---
name: code-review-panel
description: Run parallel code reviews from a panel of AI reviewers (3 Claude models + Antigravity + Codex, expandable with OpenRouter models on request) via a Workflow, then consolidate, adversarially verify, and triage findings interactively
---

# Multi-Reviewer Code Review Panel

You are orchestrating a **multi-reviewer code review panel** — a core of five reviewers,
expandable with OpenRouter-hosted models on request. The design is hybrid: the
**main conversation owns the two user decisions** (scope selection before, finding triage
after), and a **Workflow owns everything between them** (fan-out, collection, dedup,
adversarial verification, ranking). Do not run reviewers as ad-hoc Agent/Bash calls when
the Workflow tool is available — the workflow gives structured findings, bounded external
CLIs, live progress via `/workflows`, and resume via `resumeFromRunId` if one reviewer
dies. (If the Workflow tool is unavailable, see Fallback at the end.)

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
| grok | `x-ai/grok-4.3` |
| deepseek | `deepseek/deepseek-v4-pro` |
| kimi | `moonshotai/kimi-k2.7-code` |
| glm | `z-ai/glm-5.2` |
| qwen | `qwen/qwen3-coder-plus` |

(IDs verified 2026-07-03 against `https://openrouter.ai/api/v1/models` — they rot; when
a wrapper reports an unknown-model error, refresh from that endpoint and update this
table.) Requirements: the `opencode` CLI (`brew install opencode`) and
`OPENROUTER_API_KEY` in the environment — never print the key. If models were requested
but the CLI or key is missing, say so and run core-only; do not improvise another route.

Known failure mode: "No allowed providers are available for the selected model" is the
ACCOUNT's privacy/provider policy (openrouter.ai/settings/privacy) excluding every host
of that model — not ID rot, and retrying cannot fix it. The models are hosted mostly by
third-party GPU clouds (Novita, Together, DeepInfra, Google, Fireworks, Cloudflare, …),
so a provider allowlist naming only the model-owner orgs blocks nearly everything; the
allowlist must also intersect the training/ZDR toggles or the result is empty. Diagnose
with `GET /api/v1/models/<id>/endpoints` (provider_name list) and report which models
the policy excludes; changing the policy is the user's call. Kimi note: k2.7-code is
the panel default — kimi-k2-thinking produced 10× the output cost for no unique
confirmed findings on its first run (2026-07-03).

**OpenAI models (Sol, GPT-5.x): do NOT route through OpenRouter.** The codex CLI in the
core panel is already authenticated against OpenAI directly — an extra OpenAI-model
reviewer runs as a second codex wrapper with a model override:
`codex review <scope-flag> -c model="gpt-5.6-sol"` (same background + Monitor + tail
rules as the core codex reviewer). OpenRouter adds a shared-quota/provider-policy
failure surface for no benefit (observed 2026-07-12: `openai/gpt-5.6-sol` via OpenRouter
died on an upstream quota 502 while the codex CLI path was available the whole time).
Version gate: gpt-5.6-sol rejects older CLIs — v0.133 returns 400 "requires a newer
version of Codex"; upgrade via `npm install -g @openai/codex@latest` (it is an npm
global on this machine, not brew) before launching the panel, not mid-run. (As of
2026-07-18 this machine runs codex-cli v0.144.1; a direct `codex review` used model
gpt-5.5 and prints its findings under a "Review comment:" header — see the marker note
in CODEX_WRAPPER.)

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

## Phase 1: Run the Panel Workflow

Invoke the Workflow tool with the script below and args (an actual JSON object, not a
string):

```json
{
  "scope": "<one-line scope statement, e.g. 'commit a1b2c3d' or 'all uncommitted changes'>",
  "diffCommand": "<diff command from Phase 0, exactly as a reviewer should run it>",
  "codexFlag": "<codex scope flag from Phase 0, e.g. '--commit a1b2c3d'>",
  "context": "<project context paragraph, including contract/spec doc paths>",
  "intent": "<the three-line intent block from Phase 0: Purpose / In scope / Out of scope>",
  "priorDispositions": "<content of .claude/reviews/panel-<scope-label>.md, or ''>",
  "openrouterModels": [ { "id": "grok", "model": "x-ai/grok-4.3" } ]
}
```

`openrouterModels` is `[]` unless the user's panel-size directive selected models
(shortname → `id`, registry value → `model`).

Before/while it runs:

- The user invoking this skill IS the multi-agent opt-in for the Workflow tool.
- VERIFY ARGS DELIVERY within the first minute — the script now normalizes a
  string-delivered `args` itself and returns `{ error, panelSize: 0 }` rather than
  running a degraded panel, but verify anyway, because the string form is not the only
  way args can arrive wrong. Count `"type":"started"` lines in the run's journal.jsonl
  (transcript dir is in the tool result) — it must equal the reviewer count *including*
  OpenRouter models. Then confirm the values actually landed:
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
  7–8 reviewer panel should start every agent at once.
- Tell the user they can watch live progress with `/workflows`.
- If the result's `reviewerStatus` shows a failed reviewer, report it honestly above the
  summary table; the wrappers already retried once internally — do not relaunch the panel
  for one failure. Consensus denominators use `panelSize` (succeeded reviewers).
  - **Codex-specific recovery:** codex is the reviewer most likely to fail this way —
    its runtime can exceed the workflow's structured-output enforcement window even when
    the wrapper waits correctly (observed 2026-07-18). To get codex's read without
    re-running the whole panel, run it DIRECTLY from the main conversation, outside the
    workflow: `codex review <codexFlag>` in the background (redirect to a log), monitor
    the log for its trailing findings block, then read the tail. Outside the workflow
    there is no structured-output enforcement window, so it runs to completion (this
    reliably recovered a force-terminated codex on 2026-07-18).
- To re-run after a partial failure, relaunch with `resumeFromRunId` — completed
  reviewers return from cache instantly.

```js
export const meta = {
  name: 'review-panel',
  description: 'Independent code reviewers in parallel (core five plus any OpenRouter models); consolidate, dedupe, adversarially verify, rank',
  phases: [
    { title: 'Review', detail: 'parallel reviewers: 3 Claude + Antigravity + Codex + optional OpenRouter via opencode' },
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
const MISSING = ['scope', 'diffCommand', 'codexFlag', 'context', 'intent'].filter((k) => !CFG[k])
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
// - agy --sandbox cannot see .git (blocked path), so it cannot run git diff itself.
// - Passing a file PATH (not diff text) as the prompt also avoids the OS E2BIG
//   argument-size limit — no large-diff fallback needed.
// - agy's --print-timeout is UNRELIABLE (observed running 3x past it and hanging);
//   the wrapper MUST bound it with a foreground `timeout` and retry once itself.
// - codex review needs exactly one scope flag, rejects a prompt argument alongside
//   it, and has no --approval-mode. It can legitimately run 30+ minutes on big
//   diffs — run it in background and wait for the completion notification; never
//   read its full transcript (it is huge), only the tail findings block it prints
//   at the very end. The header STRING has changed across CLI versions — older
//   CLIs printed "Full review comments:", v0.144.1 (observed 2026-07-18) prints
//   "Review comment:" — so match the trailing findings block generically; do not
//   grep for one fixed marker (a stale grep silently never fires and a watch
//   falls through to process-exit).
// - The wrapper must NEVER wait for codex by ending its turn or via a foreground
//   sleep/kill-0 loop: the harness converts such loops into ANOTHER background
//   task, the turn ends with nothing to report, the workflow fails the reviewer
//   after one structured-output nudge, and the orphaned codex process is killed
//   mid-review (observed 2026-07-03, chunk-3 run). Waiting happens ONLY via the
//   Monitor tool, which keeps the turn alive.
// TWO SEPARATE agy failure modes, both observed 2026-07-28 in one run. They look alike
// from outside (no findings) and neither is a clean review:
//  1. HEADLESS AUTO-DENY. agy asks for the `command` permission, headless mode cannot
//     prompt, so it auto-denies and returns "jetski: no output produced". `--sandbox`
//     alone does not fix this — sandbox restricts the terminal, it does not pre-approve.
//     `--dangerously-skip-permissions` auto-approves prompts and COMPOSES with --sandbox
//     (orthogonal flags: one restricts what tools may do, the other stops the prompt).
//     The alternative is a `permissions.allow` entry in
//     ~/.gemini/antigravity-cli/settings.json of the form command(<target>), per agy's
//     own error text; the syntax is undocumented in `agy --help`, so the flag is the
//     reliable route.
//     NOT YET OBSERVED WORKING END-TO-END. The flag pairing is reasoned from `agy --help`
//     (the two flags are documented as independent) rather than confirmed by a run. If a
//     future panel still reports the auto-deny, that is the thing to check first, and the
//     permissions.allow entry is the fallback.
//  2. SAFETY REFUSAL ON THE PROMPT WORDING. Asked to look for "security issues" and
//     "vulnerabilities", Gemini refused outright: "My safety guidelines strictly prohibit
//     me from performing vulnerability analysis or security scanning on user-provided
//     code." Phrase agy's prompt as ordinary code review — correctness, data integrity,
//     error handling, robustness. It still reports security-relevant defects; it will not
//     accept a security-scanning framing. Do NOT reintroduce the words "vulnerability",
//     "security scanning" or "exploit" into this prompt.
const AGY_WRAPPER = `You are a wrapper around the Antigravity CLI (agy), producing an external (Gemini) code-review perspective.
Steps:
1. DIFF_FILE="$(mktemp -t agy_panel_diff.XXXXXX)" then write the diff: ${CFG.diffCommand} -- . ':(exclude)package-lock.json' ':(exclude)*.lock' > "$DIFF_FILE" (if the exclude pathspec form fails for this diff command, use it without the excludes).
2. Run with a hard 9-minute bound: timeout 540 agy --sandbox --dangerously-skip-permissions --add-dir "$(dirname "$DIFF_FILE")" --model "Gemini 3.1 Pro (High)" --print-timeout 8m --prompt "Review the code changes in the diff file at $DIFF_FILE. Read that file with your file tools, and open full source files in this workspace for context when needed. Look for bugs, logic errors, incorrect error handling, data integrity problems, race conditions, and code quality issues. For each finding provide: severity (CRITICAL/HIGH/MEDIUM/LOW), file path, line numbers, description, recommended fix. Be concise - findings list only."
   PROMPT WORDING IS LOAD-BEARING: do not add "security issues", "vulnerabilities" or "security scanning" — that phrasing triggers a flat safety refusal from this model (observed 2026-07-28), and a refusal is indistinguishable from a clean review in the output.
3. If it times out or errors, retry ONCE with the prompt narrowed to the most important files in the diff, keeping the same neutral wording. If agy reports a TOOL DENIAL ("a tool required the \\"command\\" permission") or REFUSES ON POLICY GROUNDS, say so explicitly in failed: — those are reviewer failures, NOT zero findings, and reporting them as a clean bill of health is the worst available outcome. If the retry also fails, return findings: [] and failed: "<what happened>".
4. Only rm the diff file after agy has returned. (For a huge full-codebase scope where the diff is unwieldy, you may instead omit the diff file and tell agy to review the workspace source directly — it runs in the repo as its workspace.)
Translate agy's prose findings into the structured schema faithfully — do not add findings of your own. For the scope and why_it_matters fields agy does not provide, classify yourself by checking each finding against the diff (does it anchor to changed lines?) and this intent: ${CFG.intent}. ${READ_ONLY}`

const CODEX_WRAPPER = `You are a wrapper around the Codex CLI, producing an external code-review perspective.
Steps:
1. Run in BACKGROUND, redirecting output to a log file (it may take 30+ minutes): codex review ${CFG.codexFlag}
2. WAIT CORRECTLY — this is the part that has failed before. Load the Monitor tool (ToolSearch "select:Monitor") and monitor the background task / log file until codex exits or the log emits its trailing findings block (a "Review comment:"/"Full review comments:" section — the exact header varies by CLI version, so watch for the findings block itself, not a fixed string); renew the monitor for up to ~25 minutes total. NEVER end your turn to "wait for the completion notification" and NEVER wait with a foreground sleep/kill-0 Bash loop — the harness converts those to background tasks, your turn ends without structured output, the workflow fails you after one nudge, and your death orphan-kills codex mid-review. If Monitor is unavailable after loading, fall back to repeated SHORT foreground Bash checks (tail the log, test liveness) — many quick calls, never one long wait. NOTE: codex is the most timeout-prone reviewer — its runtime can exceed the workflow's structured-output enforcement window (observed 2026-07-18: force-terminated mid-review while waiting correctly). If that happens, returning findings: [] with a failed: reason is correct; the main conversation recovers it (see Phase 1).
3. When it completes, read only the END of its output — the trailing findings block (after the final "Review comment:" or "Full review comments:" header, whichever this CLI prints). The transcript above it is its working log; tail the file, never read it whole.
4. Translate its findings into the structured schema faithfully (map P1→HIGH, P2→MEDIUM, P3→LOW unless it states severities directly); do not add findings of your own. For the scope and why_it_matters fields codex does not provide, classify yourself by checking each finding against the diff (does it anchor to changed lines?) and this intent: ${CFG.intent}.
5. If codex errors out, retry ONCE; if that fails, return findings: [] and failed: "<what happened>". ${READ_ONLY}`

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
const REVIEWERS = [
  {
    id: 'code-reviewer',
    opts: { agentType: 'superpowers:code-reviewer', label: 'code-reviewer' },
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
  { id: 'antigravity', opts: { label: 'antigravity-gemini' }, prompt: AGY_WRAPPER },
  { id: 'codex', opts: { label: 'codex' }, prompt: CODEX_WRAPPER },
  // Each OpenRouter reviewer is handed its slice index. With N of them the diff is cut
  // into N disjoint slices when it is large (see orWrapper step 1), so the panel covers
  // the whole change set across the group rather than having every model time out trying
  // to read all of it alone. With one OpenRouter model the slice IS the whole diff, which
  // is the previous behaviour.
  ...(CFG.openrouterModels ?? []).map((m, i, all) => ({
    id: `or-${m.id}`,
    opts: { label: `or-${m.id}` },
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
  if (!res || res.out.failed) {
    reviewerStatus[REVIEWERS[i].id] = `FAILED: ${res ? res.out.failed : 'agent died or was skipped'}`
    continue
  }
  reviewerStatus[res.id] = `ok (${res.out.findings.length} findings)`
  for (const f of res.out.findings) all.push({ ...f, reviewer: res.id })
}
const panelSize = Object.values(reviewerStatus).filter((s) => s.startsWith('ok')).length
log(`${panelSize}/${REVIEWERS.length} reviewers succeeded, ${all.length} raw findings`)

if (all.length === 0) {
  return { issues: [], reviewerStatus, panelSize }
}

phase('Consolidate')
const merged = await agent(
  `Below are ${all.length} code-review findings from ${panelSize} independent reviewers, as JSON. Group findings that describe the SAME underlying issue (even if worded differently or anchored a few lines apart); keep genuinely distinct issues separate. For each group report: the highest severity assigned, the clearest summary/fix, merged detail noting differing observations, the distinct reviewer ids, a resolved scope, and a short-kebab-case slug.
Scope resolution (anchor rule): findings classified "preference" whose why_it_matters states no concrete failure scenario are DROPPED — list their summaries in a final log line of your reasoning, but not in issues[]. Never drop this-change or pre-existing findings. When reviewers disagree on scope, a concrete failure scenario wins.
Prior dispositions from earlier panel runs on this scope (may be empty): when an issue matches an entry, REUSE its slug and set priorDisposition to its recorded fate.\n${CFG.priorDispositions}\n\n${JSON.stringify(all)}`,
  { label: 'merge-dedupe', phase: 'Consolidate', schema: MERGE_SCHEMA },
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
    { label: `verify:${issue.file}`, phase: 'Verify', schema: VERDICT_SCHEMA },
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

Launch the three Claude reviewers as parallel Agent calls and the CLI reviewers (agy,
codex, and any selected OpenRouter models via opencode) as background Bash calls, using
the same prompts and constraints embedded in the script above (diff-file + `--add-dir`
for agy with a hard `timeout` and one retry; background + scope-flag-only for codex,
reading only its trailing findings block (header varies by CLI version — "Review comment:"
on v0.144.1, "Full review comments:" on older); in-repo diff file + `--agent plan`
for opencode). Then
consolidate, count consensus, and rank manually before triage — applying the same anchor
rule, scope classification, prior-disposition labeling, and pre-existing shelf described
above. Do not proceed to triage until every successful reviewer's findings list is
verifiably complete (watch for truncated outputs; recover from the task output file with
offset reads).
