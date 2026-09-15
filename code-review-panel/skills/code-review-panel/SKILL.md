---
name: code-review-panel
description: Run parallel code reviews from a panel of AI reviewers (3 Claude models + Antigravity + Codex, expandable with OpenRouter models on request) via a Workflow, then consolidate, adversarially verify, and triage findings interactively
---

# Multi-Reviewer Code Review Panel

You are orchestrating a **multi-reviewer code review panel** — a core of five reviewers,
expandable with OpenRouter-hosted models on request. The design is hybrid: the
**main conversation owns the two user decisions** (scope selection before, finding triage
after) **plus the codex run** (see "The standalone codex run" — codex cannot live inside
the workflow), and a **Workflow owns everything between them** (fan-out of the other four
reviewers, collection, dedup, adversarial verification, ranking). Do not run the other
reviewers as ad-hoc Agent/Bash calls when the Workflow tool is available — the workflow
gives structured findings, bounded external CLIs, live progress via `/workflows`, and
resume via `resumeFromRunId` if one reviewer dies. (If the Workflow tool is unavailable,
see Fallback at the end.)

## User Input

```text
$ARGUMENTS
```

If `$ARGUMENTS` is provided, treat it as scope instructions (e.g., specific files,
directories, "entire repo", or a description of what to focus on) and/or a panel-size
directive (see Panel size below).

## Panel size — OpenRouter expansion (chosen per invocation)

The core panel is the five reviewers in the script. `$ARGUMENTS` can expand it:

- **"whole hog"** (or "full panel", "all models") → add every model in the registry.
- **Named shortnames** ("add grok", "with kimi and glm") → add just those.
- **No directive** → core five only. Don't ask — absence of a directive means core.

| Shortname | OpenRouter model |
|-----------|------------------|
| grok | `x-ai/grok-4.20` |
| deepseek | `deepseek/deepseek-v4-pro` |
| kimi | `moonshotai/kimi-k3` |
| glm | `z-ai/glm-5.3` |
| qwen | `qwen/qwen3-coder-plus` |

**REFRESH THIS TABLE AT THE START OF EVERY PANEL THAT USES IT — one curl, before the
first wrapper runs, not when something breaks:**

```bash
curl -s https://openrouter.ai/api/v1/models | python3 -c "
import json,sys
d=json.load(sys.stdin)['data']
for pre in ['x-ai/grok','deepseek/deepseek-v4','moonshotai/kimi','z-ai/glm-5','qwen/qwen3-coder']:
    for m in sorted((x for x in d if x['id'].startswith(pre)), key=lambda x: x['id']):
        p=m.get('pricing',{})
        print(f\"{m['id']:<42} ctx={m.get('context_length'):<9} \${float(p.get('prompt',0))*1e6:.2f}/\${float(p.get('completion',0))*1e6:.2f}\")"
```

**The old rule was "refresh when a wrapper reports an unknown-model error", and that rule
CANNOT catch the case that actually happens.** An error fires only when a listed model
DISAPPEARS. A newer, better model appearing fires nothing, so the table keeps working
while quietly naming last quarter's models — which is worse than breaking, because
nothing prompts anyone to look. Found 2026-08-29: the table still said
`kimi-k2.7-code` (262k context) while `kimi-k3` had shipped at 1M, and FOUR of the five
entries were superseded. Only `qwen3-coder-plus` was still current.

Two things about reading the refreshed list. **Version numbers here are not decimals** —
`grok-4.20` is newer than `grok-4.6`, and a `-multi-agent` or `:batch` sibling usually
marks the current flagship line. And **the newest is not automatically the pick**: check
the context window against your diff size and the price against how many reviewers you
are running. Prefer a `-code`/`-coder` variant when one exists at a comparable tier.

(IDs verified 2026-08-29 against `https://openrouter.ai/api/v1/models`.) Requirements: the `opencode` CLI (`brew install opencode`) and
`OPENROUTER_API_KEY` in the environment — never print the key. If models were requested
but the CLI or key is missing, say so and run core-only; do not improvise another route.

Known failure mode: "No allowed providers are available for the selected model" is the
ACCOUNT's privacy/provider policy (openrouter.ai/settings/privacy) excluding every host
of that model — not ID rot, and retrying cannot fix it. The models are hosted mostly by
third-party GPU clouds (Novita, Together, DeepInfra, Google, Fireworks, Cloudflare, …),
so a provider allowlist naming only the model-owner orgs blocks nearly everything; the
allowlist must also intersect the training/ZDR toggles or the result is empty. Diagnose
with `GET /api/v1/models/<id>/endpoints` (provider_name list) and report which models
the policy excludes; changing the policy is the user's call. Kimi note: kimi-k3 is
the panel default since 2026-08-29. The older measurement it replaced — kimi-k2-thinking
producing 10× the output cost for no unique confirmed findings (2026-07-03) — was a
comparison against its OWN sibling, and it was read for two months as a reason not to
look at Kimi again. A cost finding about one variant says nothing about the next
generation; re-measure rather than inherit it.

**OpenAI models (Sol, GPT-5.x): do NOT route through OpenRouter.** The codex CLI in the
core panel is already authenticated against OpenAI directly — an extra OpenAI-model
reviewer runs as a second standalone codex run with a model override:
`codex review <scope-flag> -c model="gpt-5.6-sol"` (same background + tail rules as the
core standalone codex run). OpenRouter adds a shared-quota/provider-policy
failure surface for no benefit (observed 2026-07-12: `openai/gpt-5.6-sol` via OpenRouter
died on an upstream quota 502 while the codex CLI path was available the whole time).
Version gate: gpt-5.6-sol rejects older CLIs — v0.133 returns 400 "requires a newer
version of Codex"; upgrade via `npm install -g @openai/codex@latest` (it is an npm
global on this machine, not brew) before launching the panel, not mid-run. (As of
2026-07-18 this machine runs codex-cli v0.144.1; a direct `codex review` used model
gpt-5.5 and prints its findings under a "Review comment:" header — see the marker note
in "The standalone codex run".)

## Phase 0: Determine Review Scope (main conversation, do this ONCE)

Run `git status` and `git log --oneline main..HEAD 2>/dev/null | head -5` to detect scope.
Then choose ONE mode that all reviewers will use:

| Priority | Condition | Mode | Diff command | Codex flag |
|----------|-----------|------|-------------|------------|
| 1 | User said "entire repo" or similar | full-codebase | `git diff $(git rev-list --max-parents=0 HEAD)...HEAD` | `--base $(git rev-list --max-parents=0 HEAD)` |
| 2 | Uncommitted changes exist | uncommitted | `git diff` (+ `git diff --cached` + untracked) | `--uncommitted` |
| 3 | Feature branch ahead of main/master | branch | `git diff main...HEAD` | `--base main` |
| 4 | Clean tree on main, no args | ask | Use AskUserQuestion (see below) | — |

**If mode is "ask"** — use AskUserQuestion to let the user choose:

- **"Entire repo"** — Review all source files. Uses full-codebase mode.
- **"Recent commits"** — Ask how many commits back (default 5). Uses `--base HEAD~N`, diff `HEAD~N...HEAD`.
- **"Specific files"** — Ask which files/directories; reviewers get a targeted scope description.
- **"Specific commit"** — Ask for a SHA. Uses `--commit <sha>`, diff `<sha>~1...<sha>`.

**Start codex immediately after choosing the mode** — it is the panel's long pole and it
runs OUTSIDE the workflow (see "The standalone codex run" below). Launch it as a
background Bash task before composing the context paragraph, so it reviews while you
finish Phase 0 and the workflow runs.

Also compose a short **project context** paragraph: what the repo is, what the change set
does, and the paths of any governing contract/spec documents the code must honor.
Context-aware reviewers find contract violations that generic ones miss — spend two or
three sentences here.

Then derive two more inputs:

- **Intent block** (three lines, from the diff and conversation context): *Purpose* —
  what this change is trying to accomplish; *In scope* — the files/behaviors it
  deliberately touches; *Out of scope* — adjacent things it deliberately does not.
  Reviewers use this to classify each finding's scope (see the anchor rule in the
  script); don't skip it — it is what separates signal from review churn.
- **Scope label** (keys the disposition history): `pr-<n>` when reviewing a PR,
  `branch-<name>`, `uncommitted`, `commit-<sha>`, or `full-codebase`. Read
  `.claude/reviews/panel-<scope-label>.md` if it exists and pass its content as
  `priorDispositions` (empty string otherwise) — the merge agent uses it to label
  re-found issues with their prior fate instead of re-triaging them cold.

**Round start, when this is a CONVERGENCE campaign (a branch the owner merges only after a
clean round).** Two checks before launching, added 2026-09-08 after `feat/review-extraction`
took four rounds and seven gate passes, most of it on code no earlier round had seen:

- **Freeze the scope at round 1 and say so in the intent block.** From then on a change lands
  on the branch only as the fix for a finding. Anything else the owner wants — a new rule, a
  re-measurement, a strip, an optimisation — goes on a follow-up list in the disposition file,
  or the owner accepts, stated up front, that it costs one more full round. On that branch
  round 2 found nine of twelve issues in code round 1 never saw (a leak commit and a
  calibration read added between rounds), and gate pass 4 found eight findings all on the
  change gate pass 3 introduced. Every mid-campaign addition resets the count.
- **Grill the design before the first round, on failure semantics.** Run the `grilling` skill
  (or one pinned-opus skeptic with the same brief) over the design's decisions section with
  the questions a panel round otherwise spends a HIGH to ask: what does each floor, gate or
  budget do when its provider is down, when its input is empty, when it is called from the
  built artifact rather than the source tree. Round 1's CRITICAL (a fixture absent from
  `dist/`) and round 3's HIGH (an outage ledgered as a thin film for ninety days) were both
  design gaps a cold read of the design would have found before any code existed.

## The standalone codex run (main conversation, never inside the workflow)

Codex used to run as a fifth in-workflow wrapper agent. That design failed twice the
same way (2026-07-18, 2026-08-13): codex can legitimately run 30+ minutes, the
workflow's structured-output enforcement window is shorter, and when the wrapper is
terminated its death **orphan-kills the codex process mid-review** — the panel loses
the reviewer AND the partial run. A wrapper prompt cannot out-wait the harness, so
codex now runs from the main conversation, where no enforcement window exists, and its
findings enter the workflow as data (`codexFindings`).

Launch (background Bash, `run_in_background: true`, output redirected to a log):

```bash
codex review <codex scope flag> -c model="gpt-5.6-sol" -c model_reasoning_effort="high" \
  > <scratchpad>/codex-review-<scope-label>.log 2>&1
```

- **Model and effort are pinned explicitly** so a drifted `~/.codex/config.toml` cannot
  silently downgrade the reviewer. gpt-5.6-sol at high effort is the best verified
  configuration (confirmed running 2026-08-13 on codex-cli v0.147.0). When a newer
  model ships, update the flag here after one verified run. Version gate: gpt-5.6-sol
  rejects CLIs older than ~v0.134 with a 400; upgrade via
  `npm install -g @openai/codex@latest` before launching, not mid-run.
- Exactly one scope flag (`--base <sha>`, `--uncommitted`, `--commit <sha>`). Older
  CLIs rejected a prompt argument beside a scope flag; v0.147.0's help documents an
  optional `[PROMPT]` for custom review instructions, but that combination is
  unverified here — test it once before relying on it for focus instructions.
- The harness notifies you when the background task exits — do not poll, and NEVER
  TaskStop it to hurry the panel; a killed codex is a lost reviewer.
- On exit, read only the END of the log: the trailing findings block after the final
  "Review comment:" (v0.144.1+) or "Full review comments:" (older) header. The
  transcript above it is its working log — tail the file, never read it whole.
- Translate the findings faithfully into the workflow's findings shape — one object
  per finding: `severity` (map P1→HIGH, P2→MEDIUM, P3→LOW unless codex states
  severities), `file`, `line`, `summary`, `detail`, `fix`, `scope`
  (this-change / pre-existing / preference, classified against the diff and intent),
  `why_it_matters`. Do not add findings of your own.
- **Sequencing.** If codex finishes before you launch the workflow, pass the array as
  `codexFindings` and everything flows through dedup/consensus/verify normally. If the
  workflow finishes first, relaunch it with `resumeFromRunId` and the same args plus
  `codexFindings` filled in — the four reviewers return from cache instantly and only
  Consolidate/Verify re-run, now with codex included. Either way codex findings get
  full consensus treatment; the old "fold in by hand, consensus of 1" caveat is gone.
- **Section campaigns** (many panels over one code state): codex cannot be
  path-scoped, so run it ONCE against the whole tree, then slice the translated
  findings by each panel's paths and pass each panel its slice. Findings in files no
  panel claims get one triage pass at the end of the campaign. Re-run the standalone
  review when the tree has changed enough that the report is stale (e.g. between
  tiers, after a batch of fixes lands).

## Phase 1: Run the Panel Workflow

Invoke the Workflow tool with the script below and args (an actual JSON object, not a
string):

```json
{
  "scope": "<one-line scope statement, e.g. 'commit a1b2c3d' or 'all uncommitted changes'>",
  "diffCommand": "<diff command from Phase 0, exactly as a reviewer should run it>",
  "context": "<project context paragraph, including contract/spec doc paths>",
  "intent": "<the three-line intent block from Phase 0: Purpose / In scope / Out of scope>",
  "priorDispositions": "<content of .claude/reviews/panel-<scope-label>.md, or ''>",
  "openrouterModels": [ { "id": "grok", "model": "x-ai/grok-4.20" } ],
  "codexFindings": null
}
```

`openrouterModels` is `[]` unless the user's panel-size directive selected models
(shortname → `id`, registry value → `model`). `codexFindings` is the standalone codex
run's translated findings array when that run has already completed, else `null` —
null means "not folded in yet", and the result's `reviewerStatus.codex` will say so;
resume with the array filled in once codex exits (see "The standalone codex run").
An empty array `[]` means codex finished with zero findings and counts as a succeeded
reviewer.

Before/while it runs:

- The user invoking this skill IS the multi-agent opt-in for the Workflow tool.
- VERIFY ARGS DELIVERY within the first minute — the script now normalizes a
  string-delivered `args` itself and returns `{ error, panelSize: 0 }` rather than
  running a degraded panel, but verify anyway, because the string form is not the only
  way args can arrive wrong. Count `"type":"started"` lines in the run's journal.jsonl
  (transcript dir is in the tool result) — it must equal the reviewer count: 4 in-workflow
  core reviewers plus OpenRouter models (codex is standalone, never an in-workflow
  agent). Then confirm the values actually landed:
  `grep -o 'Scope: [^.]*' agent-*.jsonl` in that directory must show your real scope
  string, never "undefined".
  Observed twice (2026-07-03, 2026-07-27): args arrived as a JSON string, so `args.scope`
  was undefined; because undefined interpolates into a template literal as the *text*
  "undefined", every reviewer got "Scope: undefined. Obtain the change set with:
  undefined" and the OpenRouter spread was empty — 5 agents instead of 7–8, with no error
  raised. On a started-count mismatch or an `error` return: TaskStop the run, edit the
  persisted script to bake the values in as a `const CFG = {…}` literal (replace every
  `args.` reference with `CFG.`), and relaunch via scriptPath. A started-count that is
  short by exactly the number of OpenRouter models is this bug until proven otherwise —
  the concurrency cap is `min(16, cores - 2)`, so on any machine with ≥10 cores a
  6–7 agent panel should start every agent at once.
- Tell the user they can watch live progress with `/workflows`.
- If the result's `reviewerStatus` shows a failed reviewer, report it honestly above the
  summary table; the wrappers already retried once internally — do not relaunch the panel
  for one failure. Consensus denominators use `panelSize` (succeeded reviewers).
  - **Codex is never a workflow failure any more** — it runs standalone (see "The
    standalone codex run"). `reviewerStatus.codex` saying "not included yet" is not a
    failure; it is the cue to resume with `codexFindings` once the background run exits.
  - **Antigravity-specific recovery:** agy can fail in a way that emits NO start event at
    all — a classifier can block the sub-agent before it launches, so the reviewer is
    simply absent rather than failed (observed 2026-08-11; see the mode-3 note above the
    wrapper). Recover the same way as codex: run it yourself from the main conversation,
    in parallel with the workflow rather than after it, since agy takes minutes and the
    other reviewers are already running. Write the diff to a file outside the repo, then
    from the repo root run
    `timeout 540 agy --sandbox --add-dir "$REPO" --add-dir "$(dirname "$DIFF_FILE")" --model "Gemini 3.1 Pro (High)" --print-timeout 8m --output-format json --prompt "…"`
    with the same neutral wording the wrapper uses, and apply the same `denied_actions` /
    empty-response check (a denied run exits 0). Never add `--dangerously-skip-permissions`
    here either: see mode 3 — it is unsafe even run by hand.
    A wrapper that reports the "command" permission auto-deny usually means the repo was not
    an `--add-dir`; check that before concluding agy is broken. Fold its findings into triage by hand
    and say so in the report — a reviewer merged manually did not go through the dedup and
    consensus phases, so its findings carry a consensus of 1 by construction.
- **A start-event count short of the reviewer count is a real signal, not noise.** Check it
  within the first minute (see the args verification above) — the args-string bug and a
  classifier block both present this way, and they are distinguishable: with the args bug
  the transcripts show "Scope: undefined", with a block the scope is correct and one agent
  never appears at all. Neither raises an error on its own.
- To re-run after a partial failure, relaunch with `resumeFromRunId` — completed
  reviewers return from cache instantly. This is also the fix for a crash in the
  collection/merge phases: the reviewers are cached, so a resume costs only the phases
  that never ran. And it is the NORMAL second step whenever the workflow finished
  before the standalone codex run did: resume with the same args plus `codexFindings`
  filled in, and only Consolidate/Verify re-run.

```js
export const meta = {
  name: 'review-panel',
  description: 'Independent code reviewers in parallel (3 Claude + Antigravity + optional OpenRouter; codex findings supplied from a standalone run); consolidate, dedupe, adversarially verify, rank',
  phases: [
    { title: 'Review', detail: 'parallel reviewers: 3 Claude + Antigravity + optional OpenRouter via opencode (codex runs standalone outside the workflow)' },
    { title: 'Consolidate', detail: 'merge and dedupe findings across reviewers' },
    { title: 'Verify', detail: 'skeptic pass on single-reviewer findings' },
  ],
}

// Defensive arg normalization. The harness sometimes delivers `args` as a JSON
// STRING rather than an object (observed 2026-07-03 and again 2026-07-27). When
// that happens `args.scope` is undefined, and because undefined interpolates
// into a template literal as the TEXT "undefined", every reviewer silently
// receives "Scope: undefined. Obtain the change set with: undefined" and the
// OpenRouter spread comes out empty — a degraded panel that still reports
// success. Parse the string form, then FAIL LOUDLY if required fields are
// missing, so a future mangling can never run a silent no-op panel again.
let CFG
try {
  CFG = typeof args === 'string' ? JSON.parse(args) : (args ?? {})
} catch {
  return { error: 'args arrived as an unparseable string — pass args as a real JSON object, or bake a `const CFG = {…}` literal into the script and relaunch via scriptPath.', issues: [], reviewerStatus: {}, panelSize: 0 }
}
const MISSING = ['scope', 'diffCommand', 'context', 'intent'].filter((k) => !CFG[k])
if (MISSING.length) {
  return { error: `Required args missing: ${MISSING.join(', ')} (args arrived as ${typeof args}). Bake a \`const CFG = {…}\` literal into the persisted script and relaunch via scriptPath.`, issues: [], reviewerStatus: {}, panelSize: 0 }
}

const FINDINGS_SCHEMA = {
  type: 'object',
  required: ['findings'],
  properties: {
    findings: {
      type: 'array',
      items: {
        type: 'object',
        required: ['severity', 'file', 'summary', 'fix', 'scope', 'why_it_matters'],
        properties: {
          severity: { enum: ['CRITICAL', 'HIGH', 'MEDIUM', 'LOW', 'NIT'] },
          file: { type: 'string', description: 'repo-relative path' },
          line: { type: 'number' },
          summary: { type: 'string', description: 'one sentence, the defect itself' },
          detail: { type: 'string', description: 'evidence: why this is real, concrete failure scenario' },
          fix: { type: 'string' },
          scope: { enum: ['this-change', 'pre-existing', 'preference'], description: 'this-change: caused/worsened by the diff; pre-existing: real defect in code the diff did not cause; preference: alternative approach with no concrete failure scenario' },
          why_it_matters: { type: 'string', description: 'one sentence anchoring the finding to the stated intent — or, for pre-existing, to a concrete failure scenario' },
        },
      },
    },
    failed: { type: 'string', description: 'set ONLY if the reviewer could not run (CLI timeout/error); findings must then be []' },
  },
}

const MERGE_SCHEMA = {
  type: 'object',
  required: ['issues'],
  properties: {
    issues: {
      type: 'array',
      items: {
        type: 'object',
        required: ['severity', 'file', 'summary', 'fix', 'reviewers', 'scope', 'slug'],
        properties: {
          severity: { enum: ['CRITICAL', 'HIGH', 'MEDIUM', 'LOW', 'NIT'], description: 'highest severity any reviewer assigned' },
          file: { type: 'string' },
          line: { type: 'number' },
          summary: { type: 'string' },
          detail: { type: 'string', description: 'merged evidence, noting per-reviewer observations that differ' },
          fix: { type: 'string' },
          reviewers: { type: 'array', items: { type: 'string' }, description: 'distinct reviewer ids that flagged this issue' },
          scope: { enum: ['this-change', 'pre-existing'], description: 'resolved scope; preference findings without a failure scenario are dropped, not merged' },
          slug: { type: 'string', description: 'short-kebab-case stable identifier; REUSE the matching slug from prior dispositions when this is the same issue' },
          priorDisposition: { type: 'string', description: 'ONLY when the slug matches a prior-disposition entry: its recorded fate, e.g. "deferred 2026-07-02: <reason>"' },
        },
      },
    },
  },
}

const VERDICT_SCHEMA = {
  type: 'object',
  required: ['real', 'reason'],
  properties: {
    real: { type: 'boolean' },
    reason: { type: 'string', description: 'one or two sentences: the evidence that confirms or refutes' },
  },
}

const READ_ONLY = 'This is a READ-ONLY review: never use Edit or Write. Verify claims empirically where cheap (build, grep, run a snippet) rather than speculating.'
const COMMON = `Scope: ${CFG.scope}. Obtain the change set with: ${CFG.diffCommand} — then open full source files for context.
Project context: ${CFG.context}
Change intent: ${CFG.intent}
${READ_ONLY}
Report every finding with: severity (CRITICAL/HIGH/MEDIUM/LOW/NIT), repo-relative file path, line number when known, a one-sentence summary, supporting detail, a recommended fix, a scope classification (this-change / pre-existing / preference), and why_it_matters — one sentence anchoring it to the stated intent or, for pre-existing defects, to a concrete failure scenario.
Anchor rule: a finding must matter for this change or be a real defect with a concrete failure scenario — commentary that is merely true is not a finding. Equal-merit alternatives are not findings; the current code's approach wins ties. Do report real pre-existing defects you notice en route (scope: pre-existing) — they are shelved for the user, not dropped. Report zero findings honestly if the code is clean.`

// External-CLI wrapper constraints (verified against the installed tools — do not
// simplify these back to naive pipes):
// - agy ignores piped stdin in print mode; hand it a diff FILE via --add-dir.
// - agy MUST ALSO GET THE REPO VIA --add-dir. Headless, its file tools do not treat the cwd as
//   the workspace: without it, reads of source files are refused, the model reaches for a
//   shell command instead, and that command is denied (mode 1). Measured 2026-09-14 — this is
//   almost certainly what failed on PR 93's panel, where only the diff's temp dir was added.
//   `notes/agy-sandbox-and-permissions.md` is the research behind every agy rule here.
// - agy cannot run `git diff` itself: the `command` permission gate stops it headless. (It
//   is NOT that --sandbox hides .git — since agy 1.1.10 sandboxed commands get read-only .git.)
// - A DENIED HEADLESS RUN EXITS 0 WITH status "SUCCESS" AND AN EMPTY RESPONSE, and stops at
//   the first denial. Only `--output-format json` exposes `denied_actions`, so the wrapper
//   uses it and treats any denial or empty response as a failure, never as zero findings.
// - Passing a file PATH (not diff text) as the prompt also avoids the OS E2BIG
//   argument-size limit — no large-diff fallback needed.
// - agy's --print-timeout is UNRELIABLE (observed running 3x past it and hanging);
//   the wrapper MUST bound it with a foreground `timeout` and retry once itself.
// - There is NO codex wrapper in this script, on purpose. Codex's runtime exceeds the
//   workflow's structured-output enforcement window, and a terminated wrapper
//   orphan-kills the CLI mid-review (observed 2026-07-03, 2026-07-18, 2026-08-13 —
//   three separate waiting strategies, same death). It runs standalone in the main
//   conversation and arrives here as CFG.codexFindings. Do not reintroduce a wrapper.
// THREE SEPARATE agy failure modes. They look alike from outside (no findings) and none
// is a clean review:
//  1. HEADLESS AUTO-DENY (observed 2026-07-28). agy asks for the `command` permission,
//     headless mode cannot prompt, so it auto-denies and returns "jetski: no output
//     produced". This happens when agy wants to run a SHELL COMMAND — it is not a blanket
//     property of headless mode, which is the misreading that put a dangerous flag in this
//     wrapper for two weeks. THE PROMPT BELOW ONLY READS FILES, so it does not hit this:
//     verified 2026-08-11 by running agy with `--sandbox --add-dir` and NO permissions
//     flag, which returned a normal answer at exit 0. Keep the prompt to file tools and
//     the mode stays out of reach — PROVIDED the repo is an --add-dir (see above). If a
//     future prompt does need shell, the route is a whole-line command(<exact command>)
//     grant in ~/.gemini/config/config.json under userSettings.globalPermissionGrants.allow,
//     set by a person and verified first. NOT ~/.gemini/antigravity-cli/settings.json, which
//     headless reportedly ignores for command() (issue #548, UNVERIFIED on 1.2.3/macOS), and
//     NOT the flag, for the reasons in mode 3.
//  3. BLOCKED BY A SAFETY CLASSIFIER BEFORE THE AGENT EVER STARTS (observed 2026-08-11).
//     `--dangerously-skip-permissions` used to be in this wrapper. Putting it in a
//     SUB-AGENT prompt trips the classifier: "[Create Unsafe Agents] The sub-agent prompt
//     launches the agy CLI with --dangerously-skip-permissions, disabling its per-action
//     approval gate for a code-execution-capable agent, with no explicit user
//     authorization naming that flag." The agent never starts, `agent()` returns null, and
//     NO start event is emitted — from outside it looks like a reviewer that silently
//     vanished, which is a nasty thing to debug because every other reviewer is running
//     normally. It is also strictly worse than the mode-1 auto-deny it was added to
//     prevent: mode 1 costs one reviewer, mode 3 cost an entire completed four-reviewer
//     panel through the null-guard bug (fixed below, 836k subagent tokens lost).
//     DO NOT REINTRODUCE THE FLAG, and do not phrase around the classifier. Instructing a
//     sub-agent to disable another agent's approval gate is the act being objected to;
//     wording it more persuasively does not make it a different act. AND THE FLAG IS NOT
//     SAFE EVEN WHEN A PERSON RUNS IT (researched 2026-09-14, corrected from an earlier line
//     here that called it fine): --sandbox confines shell commands only, a blocked command's
//     error invites a retry with `BypassSandbox: true`, and the flag auto-approves that retry
//     (issue #36, confirmed intended by Google), so an instruction planted in the diff under
//     review can run any command as the user with network. The file tools were never inside
//     the sandbox either. There is no recovery path that uses the flag.
//  2. SAFETY REFUSAL ON THE PROMPT WORDING (observed 2026-07-28). Asked to look for "security issues" and
//     "vulnerabilities", Gemini refused outright: "My safety guidelines strictly prohibit
//     me from performing vulnerability analysis or security scanning on user-provided
//     code." Phrase agy's prompt as ordinary code review — correctness, data integrity,
//     error handling, robustness. It still reports security-relevant defects; it will not
//     accept a security-scanning framing. Do NOT reintroduce the words "vulnerability",
//     "security scanning" or "exploit" into this prompt.
const AGY_WRAPPER = `You are a wrapper around the Antigravity CLI (agy), producing an external (Gemini) code-review perspective.
Steps:
1. DIFF_FILE="$(mktemp -t agy_panel_diff.XXXXXX)" then write the diff: ${CFG.diffCommand} -- . ':(exclude)package-lock.json' ':(exclude)*.lock' > "$DIFF_FILE" (if the exclude pathspec form fails for this diff command, use it without the excludes).
2. Set REPO to the repository the diff command reads: the path after "git -C" if the diff command has one, else "$(git rev-parse --show-toplevel)". Set AGY_OUT="$(mktemp -t agy_panel_out.XXXXXX)". Run with a hard 9-minute bound, from "$REPO": timeout 540 agy --sandbox --add-dir "$REPO" --add-dir "$(dirname "$DIFF_FILE")" --model "Gemini 3.1 Pro (High)" --print-timeout 8m --output-format json --prompt "Review the code changes in the diff file at $DIFF_FILE. Read that file with your file tools, and open full source files in this workspace for context when needed. Look for bugs, logic errors, incorrect error handling, data integrity problems, race conditions, and code quality issues. For each finding provide: severity (CRITICAL/HIGH/MEDIUM/LOW), file path, line numbers, description, recommended fix. Be concise - findings list only." > "$AGY_OUT" 2>&1
   Then read "$AGY_OUT" as JSON. The findings are in .response. If .denied_actions is non-empty, .response is empty, or .status is not "SUCCESS", that is a FAILED run — report failed: "agy denied <the denied actions>" or "agy returned an empty response", never findings: []. A denied run exits 0 and says SUCCESS, so the exit code proves nothing. If the output is not JSON (an older agy without --output-format), say so in failed:.
   THREE THINGS ON THAT COMMAND LINE ARE LOAD-BEARING, and all fail silently if changed.
   (0) --add-dir "$REPO": without it agy's file tools cannot read the source files, it tries a shell command instead, and headless mode denies it — a run with no review in it.
   (a) PROMPT WORDING: do not add "security issues", "vulnerabilities" or "security scanning" — that phrasing triggers a flat safety refusal from this model (observed 2026-07-28), and a refusal is indistinguishable from a clean review in the output.
   (b) NO PERMISSIONS FLAG: do not add --dangerously-skip-permissions. A sub-agent prompt carrying it is blocked by a classifier before the agent starts (observed 2026-08-11), which looks like a reviewer that never existed. The prompt only uses file tools, so the flag is not needed — verified by running without it.
   Keep the prompt to reading files. The moment it asks agy to run shell commands, mode 1 comes back and the flag is not the way out of it.
3. If it times out or errors, retry ONCE with the prompt narrowed to the most important files in the diff, keeping the same neutral wording. If agy reports a TOOL DENIAL ("a tool required the \\"command\\" permission") or REFUSES ON POLICY GROUNDS, say so explicitly in failed: — those are reviewer failures, NOT zero findings, and reporting them as a clean bill of health is the worst available outcome. If the retry also fails, return findings: [] and failed: "<what happened>".
4. Only rm the diff file and "$AGY_OUT" after agy has returned and you have read the output. (For a huge full-codebase scope where the diff is unwieldy, you may instead omit the diff file and tell agy to review the workspace source directly — it runs in the repo as its workspace.)
Translate agy's prose findings into the structured schema faithfully — do not add findings of your own. For the scope and why_it_matters fields agy does not provide, classify yourself by checking each finding against the diff (does it anchor to changed lines?) and this intent: ${CFG.intent}. ${READ_ONLY}`

// OpenRouter reviewers run via `opencode run` (verified 2026-07-03):
// - `--agent plan` is the read-only mode; its permission gate AUTO-DENIES
//   file access outside the workspace in non-interactive runs, so the diff
//   file MUST live inside the repo (removed after the run, success or not).
// - OPENROUTER_API_KEY is read from the environment — no auth setup, and
//   never print it. A missing key/CLI or unknown model id is a failed
//   reviewer, not a panel failure.
// - Project context and intent are prepended INTO the diff file rather than
//   the shell command line, so no quoting of free text inside the prompt.
//
// THE OPENROUTER PATH DOES NOT SURVIVE A LARGE DIFF, and this is now measured twice:
//   2026-07-27  grok    — read ~3 chunks of an 11,443-line diff, then the deeper retry
//                         tripped opencode's `doom_loop` guard.
//   2026-07-28  deepseek — hit the 540s bound on BOTH the full 16,332-line diff and the
//                         narrowed retry, after reading the diff in chunks and starting to
//                         open source files. No partial findings either time.
// The failure is wall-clock, not capability: these models read a big diff file in many
// small chunks and spend the whole budget on ingestion. Two changes address it, and the
// SPLIT matters more than the timeout — a bigger budget spent the same way just fails
// later. If a diff exceeds ~6,000 lines, give each OpenRouter reviewer ONE SLICE of it
// rather than the whole thing; a lens that reviewed a third of the diff thoroughly is
// worth more than one that timed out reading all of it. Say in the report which slice
// each covered, so partial coverage is never mistaken for a clean pass.
const orWrapper = (m) => `You are a wrapper around the opencode CLI, producing an external code-review perspective from the ${m.model} model via OpenRouter.
Steps:
1. mkdir -p .claude/reviews then DIFF_FILE=".claude/reviews/.panel-diff-${m.id}-$$.tmp". FIRST measure the diff: ${CFG.diffCommand} -- . ':(exclude)package-lock.json' ':(exclude)*.lock' | wc -l.
   - Under ~6,000 lines: write the whole diff.
   - OVER ~6,000 lines: write only YOUR SLICE. Slice by top-level path so the slices are disjoint and each is coherent — list the changed directories with --stat, order them, and take the slice whose index is ${m.sliceIndex ?? 0} of ${m.sliceCount ?? 1} by passing those pathspecs to the diff command. A thorough review of one slice beats a timeout on the whole diff, and this reviewer has failed the whole-diff way twice.
   Write a header followed by the diff into it: the lines "=== Project context ===", ${JSON.stringify(CFG.context)}, "=== Change intent ===", ${JSON.stringify(CFG.intent)}, "=== Scope of THIS review ===" plus a one-line statement of which slice you took (or "the entire diff"), "=== Diff ===", then append the diff. The file must stay inside the repo — opencode's plan agent cannot read outside the workspace.
2. Run with a hard 14-minute bound: timeout 840 opencode run --model openrouter/${m.model} --agent plan "Review the code changes described in the file $DIFF_FILE in this workspace. Read that file FIRST and in LARGE chunks — it carries project context, the change intent, the scope of this review, then the diff. Budget your time: finish reading before opening source files, and open source files only where a finding depends on surrounding code. Emit findings even if you have not examined everything. Look for bugs, logic errors, incorrect error handling, data integrity problems, race conditions, and code quality issues. For each finding provide: severity (CRITICAL/HIGH/MEDIUM/LOW/NIT), file path, line numbers, description, recommended fix. Findings list only; report zero findings honestly if the code is clean."
3. If it times out, errors, trips a doom_loop/repetition guard, or reports an unknown/unavailable model, retry ONCE with the slice HALVED (not merely "narrowed") — halving is what changes the outcome; a vaguer prompt over the same volume does not. If the retry also fails, return findings: [] and failed: "<what happened, and which slice was attempted>". NEVER report a truncated or unfinished read as zero findings.
4. rm -f the diff file after opencode returns, success or failure.
5. In your summary state which slice you reviewed, so the orchestrator can report coverage honestly rather than implying the whole diff was seen.
Translate the model's prose findings into the structured schema faithfully — do not add findings of your own. For the scope and why_it_matters fields, classify yourself by checking each finding against the diff (does it anchor to changed lines?) and the stated intent. ${READ_ONLY}`

phase('Review')
// EVERY agent() call in this script pins an explicit model. Without a pin a subagent
// inherits the main-loop model, and when the orchestrator runs on a premium tier
// (e.g. Fable/Mythos-class) the whole fan-out silently bills against it. The
// orchestrator's tier buys nothing here: reviewer diversity comes from the mix, and
// merge/verify are bounded judgment tasks opus handles. Keep the pins when editing.
const REVIEWERS = [
  {
    id: 'code-reviewer',
    opts: { agentType: 'superpowers:code-reviewer', model: 'opus', label: 'code-reviewer' },
    prompt: `Review this change set for bugs, security issues, contract/spec mismatches, data integrity problems, and code quality issues, checking the code against any governing contract documents named in the context. ${COMMON}`,
  },
  {
    id: 'sonnet',
    opts: { model: 'sonnet', label: 'sonnet-broad' },
    prompt: `You are a code reviewer doing a fast, broad pass. Look for bugs, logic errors, security vulnerabilities, missing validation, API mismatches, error-handling gaps, and performance problems. ${COMMON}`,
  },
  {
    id: 'opus',
    opts: { model: 'opus', label: 'opus-deep' },
    prompt: `You are a senior reviewer doing a deep architectural pass. Focus on subtle logic bugs, race conditions, fail-open vs fail-closed behavior, edge cases in parsing/config handling, drift hazards between code and its contracts, and issues other reviewers might miss. ${COMMON}`,
  },
  { id: 'antigravity', opts: { model: 'sonnet', label: 'antigravity-gemini' }, prompt: AGY_WRAPPER },
  // Each OpenRouter reviewer is handed its slice index. With N of them the diff is cut
  // into N disjoint slices when it is large (see orWrapper step 1), so the panel covers
  // the whole change set across the group rather than having every model time out trying
  // to read all of it alone. With one OpenRouter model the slice IS the whole diff, which
  // is the previous behaviour.
  ...(CFG.openrouterModels ?? []).map((m, i, all) => ({
    id: `or-${m.id}`,
    opts: { model: 'sonnet', label: `or-${m.id}` },
    prompt: orWrapper({ ...m, sliceIndex: i, sliceCount: all.length }),
  })),
]

// Barrier is correct here: dedup/consensus genuinely needs ALL reviewers' findings.
const raw = await parallel(REVIEWERS.map((r) => () =>
  agent(r.prompt, { ...r.opts, phase: 'Review', schema: FINDINGS_SCHEMA })
    .then((out) => ({ id: r.id, out }))
))

const reviewerStatus = {}
const all = []
for (let i = 0; i < REVIEWERS.length; i++) {
  const res = raw[i]
  // THE NULL ARRIVES WRAPPED, which is why `!res` alone is not the guard.
  // `agent()` resolves to null when the agent is blocked or dies on a terminal
  // error, and the `.then((out) => ({ id, out }))` above turns that null into
  // `{ id, out: null }` — a TRUTHY object. So `!res` never fires, `res.out.failed`
  // throws, and the whole panel dies in the collection loop AFTER every surviving
  // reviewer has finished and its tokens are spent. Observed 2026-08-11: one
  // blocked reviewer destroyed a completed 4-reviewer run (836k subagent tokens).
  if (!res || !res.out || res.out.failed) {
    const why = !res
      ? 'agent died or was skipped'
      : !res.out
        ? 'agent returned no result (blocked by a classifier, or a terminal error)'
        : res.out.failed
    reviewerStatus[REVIEWERS[i].id] = `FAILED: ${why}`
    continue
  }
  reviewerStatus[res.id] = `ok (${res.out.findings.length} findings)`
  for (const f of res.out.findings) all.push({ ...f, reviewer: res.id })
}

// Codex runs OUTSIDE the workflow (standalone in the main conversation — see the
// wrapper-constraints comment above). Its findings arrive pre-translated via
// CFG.codexFindings: an array (possibly empty — a clean codex pass is a succeeded
// reviewer) folds it into consolidation with full consensus treatment; null/absent
// means the standalone run has not finished — resume this run with codexFindings
// filled in, and the four in-workflow reviewers return from cache.
if (Array.isArray(CFG.codexFindings)) {
  reviewerStatus['codex'] = `ok (${CFG.codexFindings.length} findings, from the standalone run)`
  for (const f of CFG.codexFindings) all.push({ ...f, reviewer: 'codex' })
} else {
  reviewerStatus['codex'] = 'not included yet: standalone codex still running — resume with codexFindings to fold it in'
}
const panelSize = Object.values(reviewerStatus).filter((s) => s.startsWith('ok')).length
log(`${panelSize} reviewers succeeded (${REVIEWERS.length} in-workflow; standalone codex ${Array.isArray(CFG.codexFindings) ? 'included' : 'pending'}), ${all.length} raw findings`)

if (all.length === 0) {
  return { issues: [], reviewerStatus, panelSize }
}

phase('Consolidate')
const merged = await agent(
  `Below are ${all.length} code-review findings from ${panelSize} independent reviewers, as JSON. Group findings that describe the SAME underlying issue (even if worded differently or anchored a few lines apart); keep genuinely distinct issues separate. For each group report: the highest severity assigned, the clearest summary/fix, merged detail noting differing observations, the distinct reviewer ids, a resolved scope, and a short-kebab-case slug.
Scope resolution (anchor rule): findings classified "preference" whose why_it_matters states no concrete failure scenario are DROPPED — list their summaries in a final log line of your reasoning, but not in issues[]. Never drop this-change or pre-existing findings. When reviewers disagree on scope, a concrete failure scenario wins.
Prior dispositions from earlier panel runs on this scope (may be empty): when an issue matches an entry, REUSE its slug and set priorDisposition to its recorded fate.\n${CFG.priorDispositions}\n\n${JSON.stringify(all)}`,
  { label: 'merge-dedupe', phase: 'Consolidate', model: 'opus', schema: MERGE_SCHEMA },
)

phase('Verify')
// Multi-reviewer consensus is already cross-validation; only single-reviewer
// findings get a skeptic. Refuted findings are kept and labeled, never dropped —
// the user decides their fate in triage.
const verified = await parallel(merged.issues.map((issue) => () => {
  if (issue.reviewers.length >= 2) {
    return Promise.resolve({ ...issue, verdict: 'CONSENSUS' })
  }
  return agent(
    `Adversarially verify this code-review finding — try to REFUTE it by reading the actual code (and running cheap checks). Default to real=false if you cannot confirm the failure scenario concretely. For scope "pre-existing" findings be extra demanding: a genuine defect needs a concrete failure scenario in the code as it stands — an alternative approach dressed up as a bug is real=false.\nFinding: ${JSON.stringify(issue)}\nScope: ${CFG.scope}. ${READ_ONLY}`,
    { label: `verify:${issue.file}`, phase: 'Verify', model: 'opus', schema: VERDICT_SCHEMA },
  ).then((v) => ({ ...issue, verdict: v.real ? 'CONFIRMED' : 'REFUTED', verdictReason: v.reason }))
}))

const SEV = { CRITICAL: 0, HIGH: 1, MEDIUM: 2, LOW: 3, NIT: 4 }
const ranked = verified.filter(Boolean).sort((a, b) => {
  const refA = a.verdict === 'REFUTED' ? 1 : 0
  const refB = b.verdict === 'REFUTED' ? 1 : 0
  if (refA !== refB) return refA - refB // refuted sink to the bottom
  const preA = a.scope === 'pre-existing' ? 1 : 0
  const preB = b.scope === 'pre-existing' ? 1 : 0
  if (preA !== preB) return preA - preB // shelf: pre-existing below this-change, above refuted
  if (SEV[a.severity] !== SEV[b.severity]) return SEV[a.severity] - SEV[b.severity]
  return b.reviewers.length - a.reviewers.length
})

return { issues: ranked, reviewerStatus, panelSize }
```

## Phase 2: Interactive Triage (main conversation)

**Notify the user first** — the panel takes long enough that they may have stepped away.
Send a Pushover notification with a one-line summary before presenting the table (this
no-ops silently when no credentials are configured; never print the credential values):

```bash
[ -f ~/.config/pushover/credentials ] && . ~/.config/pushover/credentials
if [ -n "$PUSHOVER_TOKEN" ] && [ -n "$PUSHOVER_USER" ]; then
  curl -s --form-string "token=$PUSHOVER_TOKEN" --form-string "user=$PUSHOVER_USER" \
    --form-string "title=Code review panel ready" \
    --form-string "message=<N> issues (<n> CRITICAL/HIGH) await triage in <repo name>" \
    https://api.pushover.net/1/messages.json >/dev/null
fi
```

(If the built-in PushNotification tool is available and Pushover credentials are not,
use it instead with the same summary.)

Present the workflow's `issues` as a numbered summary table (note any failed reviewer
above it, from `reviewerStatus`):

```
| #  | Severity | Scope        | Consensus | Verdict   | Issue Summary | Files |
|----|----------|--------------|-----------|-----------|---------------|-------|
| 1  | CRITICAL | this-change  | 4/5       | CONSENSUS | Brief desc    | path  |
| 2  | HIGH     | pre-existing | 1/5       | CONFIRMED | Brief desc    | path  |
| ...                                                                          |
```

Consensus denominator is `panelSize`. An issue carrying `priorDisposition` shows it in
the summary cell (e.g. "deferred 2026-07-02: …") — don't re-litigate it cold; ask only
whether the prior call still stands. **Pre-existing findings are the shelf**: real
defects found outside the change's scope, ranked after in-scope issues. Present them
under their own "Out of scope — real, your call" heading and triage them like the rest;
they were kept precisely so the user gets the choice. REFUTED findings appear at the
bottom — mention each in one line with its refutation reason; only triage one if the
user asks.

Then work through issues with AskUserQuestion:

- **One at a time** for CRITICAL/HIGH; **themed bundles** for related MEDIUMs when the
  user's time is tight; **3–5 at a time with multiSelect** for LOW/NIT.
- For each issue/bundle present: what it is (1–2 sentences), which reviewers flagged it
  (plus the verify verdict for single-reviewer findings), your recommendation marked
  "(Recommended)" as the first option, and 2–4 action choices (Fix now / Fix differently
  / Defer / Skip).

When the user chooses a fix, implement it immediately, verify it (build/tests/the checks
the finding implies), then move to the next issue. If a fix is deferred, leave a paper
trail where the project tracks such things (task list, TODO doc) rather than only in
conversation.

**Fix verification gate (BEFORE the commit that closes the round).** Added 2026-08-13
after a convergence campaign where rounds 5, 6 and 7 each found a defect IN THE PREVIOUS
ROUND'S FIX — every one caught by a reviewer who MEASURED (an A/B sweep or diff probe
against real data) while the fix had only been reasoned about and spot-checked against
already-known cases. A full round exists to catch what ten minutes at fix time could
have. So, once a round's fix batch is implemented and its ordinary checks pass, and
before committing:

1. **A/B blast-radius sweep**, when the fix changes a decision function with a real
   input population available (a resolver, matcher, ranker, parser): run OLD code vs NEW
   code over the full real population (or a large sample) and diff the outcomes. Every
   changed outcome must be classified — intended improvement with evidence, or the fix
   is wrong. Never calibrate a threshold from the finding's own examples; they are the
   sample the reviewer happened to hit, and the counter-population is what the sweep is
   for.
2. **codex on the uncommitted diff** (`codex review --uncommitted`, same background +
   tail rules as the standalone run). Cheap, already authenticated, and it covers the
   class a data sweep cannot see: lock lifetimes, interleavings, error-path ordering.
3. **One skeptic subagent** (pinned model, never the orchestrator's) prompted to REFUTE
   the fix: enumerate the truth table of every new/changed predicate and hunt the empty
   cell — the case nobody named. (The three fix regressions above all lived in an
   unconsidered cell: a sub-floor tie, an equal year distance, a lock released after
   its directory was deleted.)

Findings from the gate are fix iterations inside the SAME round — fix, re-run the gate,
then commit. Skip the gate only for mechanical fixes (doc wording, dead-code deletion,
test-only changes); anything touching decision logic, money paths, or locks gets it.
Two authoring rules that need no tooling: enumerate the truth table of any new boolean
predicate before writing it, and any test fixture claiming to mirror real data must be
QUERIED from that data at authoring time, never typed from memory.

Two more, added 2026-09-08 after a four-round, seven-gate-pass campaign
(`feat/review-extraction`) whose churn traced to exactly these:

4. **A fix that adds a threshold, ratio, classifier or diagnosis branch gets the OWNER'S
   RULE FIRST, as a written truth table, then the sweep.** Not the reviewer's proposal and
   not the fixer's guess. The refusal diagnosis on that branch moved FOUR times in one day
   — slot count, ratio of medians, per-page ratio, drop evidence — each version encoding a
   guess about what the rule meant, each falsified by the next skeptic measuring the
   population (37% of the sub-floor pages sat under the ratio with every anchor verified).
   One AskUserQuestion at round 1 ("drop evidence, ratio, or slot count?") would have cut
   two gate passes. The tell: a finding whose fix could be written several plausible ways
   is a design question wearing a bug's clothes; the round-3 outage threshold repeated the
   pattern (all-failed → any-unheard → status classes → refusal evidence → count location).
5. **A parser or text-transform fix needs at least one fixture CAPTURED from a real page,
   and a mutation that removes the feature must fail it.** The paragraph unit the whole
   roundup rule rested on was DEAD IN PRODUCTION under 4,766 green tests, because every
   fixture was typed markup that never exercised the `\s+` collapse in the real cleaner;
   codex's cold read of the fetcher found it in gate pass 3, after two passes had built on
   top of it. Rule 1's "queried, never typed" covers data; this is its markup form.

**Routed findings are dispositions, not side-channel notes.** When a reviewer surfaces
a real finding OUTSIDE the review's scope (a design doc, another feature's spec, a file
another owner holds), it goes into the disposition table as its own row with disposition
`routed`, naming who it went to — never only into a JSON side file nothing points at.
Added 2026-08-13 after codex found four P1s in a concurrent feature's specs across three
rounds and each reached that feature's owner only because a human happened to relay it;
two would have shipped otherwise. Follow-up rounds check routed rows for closure the way
they check deferred ones.

**Amended docs get read cold.** When a prior round's routed finding caused a design doc
to be amended, the next round's standalone codex (or a reviewer) reads the AMENDED file
fresh, hunting the append-contradiction: a correction added in one place while the
original claim stands elsewhere in the same document. Observed twice in one campaign — a
task list that both added and forbade the same column, and a data-model correction whose
task list still said the opposite — both caught only by a cold read of the whole amended
file, neither by the author of the amendment.

**After triage, append the disposition history** (append-only — never rewrite prior
runs) to `.claude/reviews/panel-<scope-label>.md`, creating it with a title line if
absent. One section per run:

```markdown
## Run 2026-07-03 (commit <sha>, <N> issues)
| Slug | Sev | Scope | Verdict | Disposition | Reason / commit |
|------|-----|-------|---------|-------------|-----------------|
| media-kb-kib-mismatch | LOW | this-change | CONSENSUS | fixed | 90eba98 |
| stale-lockfile-pin    | MED | pre-existing | CONFIRMED | deferred | user: US4 touches this file |
```

Disposition values: fixed / deferred / skipped / refuted / no-change-needed, with the
user's stated reason or the fix commit. This file is what future runs receive as
`priorDispositions`; whether it gets committed follows the project's convention.

## Phase 3: Wrap Up

1. Run the project's test suite to verify nothing broke.
2. Run the project's linter/type-checker to verify clean output.
3. Compute a **verdict line**: `REQUEST CHANGES` if any confirmed CRITICAL/HIGH issue
   remains unfixed (deferred or skipped); otherwise `APPROVE` (unfixed MEDIUMs get an
   `APPROVE — with noted reservations` variant listing them). State it first.
4. Present a final summary: findings per reviewer, deduped issue count, fixed vs
   deferred vs skipped vs refuted (and anything dropped by the anchor rule, from the
   merge agent's log line), disposition-history path, test/lint results, and any
   reviewer failures.

## Fallback (Workflow tool unavailable)

Launch the three Claude reviewers as parallel Agent calls and the CLI reviewers (agy
and any selected OpenRouter models via opencode) as background Bash calls, using
the same prompts and constraints embedded in the script above (diff-file + `--add-dir`
for agy with a hard `timeout` and one retry; in-repo diff file + `--agent plan`
for opencode). Codex already runs standalone in the main conversation in both designs —
"The standalone codex run" applies unchanged. Then
consolidate, count consensus, and rank manually before triage — applying the same anchor
rule, scope classification, prior-disposition labeling, and pre-existing shelf described
above. Do not proceed to triage until every successful reviewer's findings list is
verifiably complete (watch for truncated outputs; recover from the task output file with
offset reads).
