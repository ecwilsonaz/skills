# Code Review Panel — a multi-model code-review skill for Claude Code

Run a code review as a **panel of AI reviewers** working in parallel — several Claude
models plus external reviewers (Antigravity, Codex, and optionally any OpenRouter
model) — then consolidate their findings, adversarially verify each one, and triage
the survivors with you interactively. Independent reviewers catch different bug
classes; the verification pass kills plausible-but-wrong findings before you see them.

For non-code questions, see its sibling skill [`panel`](../panel/).

## How it works

```mermaid
flowchart TD
    start(["You run /code-review-panel"])

    scope["Pick scope<br/>branch, uncommitted, commit, or repo"]

    subgraph P1["Review: reviewers run in parallel, then findings are checked&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"]
        cr["Claude<br/>code-reviewer"]
        son["Claude Sonnet<br/>broad pass"]
        opus["Claude Opus<br/>deep pass"]
        agy["Antigravity<br/>(Gemini)"]
        orm["OpenRouter models<br/>(optional)"]
        codex["Codex<br/>(runs outside the workflow)"]
        merge["Merge and dedupe findings"]
        multi{"Found by 2+<br/>reviewers?"}
        verify["Skeptic tries<br/>to refute it"]
        rank["Ranked findings"]
    end

    subgraph P2["Triage: you choose, Claude fixes&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"]
        triage{{"You decide each finding"}}
        fix["Fix it"]
        gate["Fix gate: Codex + skeptic<br/>re-review the fix"]
        commit["Commit"]
        history[("Disposition history<br/>(read by the next run)")]
    end

    converged{"HIGH or CRITICAL<br/>this round?"}
    verdict(["Tests + lint, then verdict:<br/>APPROVE or REQUEST CHANGES"])

    start --> scope
    scope --> cr & son & opus & agy & orm & codex
    cr & son & opus & agy & orm & codex --> merge
    merge --> multi
    multi -- "yes" --> rank
    multi -- no --> verify --> rank
    rank --> triage
    triage -- fix --> fix --> gate
    gate -- "new issue" --> fix
    gate -- clean --> commit --> history
    triage -- "defer / skip" --> history
    history --> converged
    converged -- "yes: run another round" --> scope
    converged -- "no: converged" --> verdict

    classDef you fill:#fde68a,stroke:#b45309,stroke-width:2px,color:#1c1917
    classDef main fill:#bfdbfe,stroke:#1d4ed8,stroke-width:2px,color:#0f172a
    classDef claude fill:#ddd6fe,stroke:#6d28d9,stroke-width:2px,color:#1e1b4b
    classDef ext fill:#a7f3d0,stroke:#047857,stroke-width:2px,color:#022c22
    classDef extopt fill:#a7f3d0,stroke:#047857,stroke-width:2px,stroke-dasharray:5 3,color:#022c22
    classDef store fill:#e5e7eb,stroke:#4b5563,stroke-width:2px,color:#111827

    class start,triage you
    class scope,fix,commit,verdict main
    class cr,son,opus,merge,multi,verify,rank,gate claude
    class converged main
    class agy,codex ext
    class orm extopt
    class history store

    style P1 fill:none,stroke:#8b949e,stroke-dasharray:4 4
    style P2 fill:none,stroke:#8b949e,stroke-dasharray:4 4
```

Yellow = you · Blue = main conversation · Purple = Claude agents · Green = external CLIs (dashed = optional)

Reviewers run in parallel inside a Workflow. Codex is the slowest seat, so it runs in the
background from the main conversation, and its findings join the rest at the merge. A
finding only one reviewer reported goes to a skeptic agent that tries to refute it. You
then decide each finding, and every fix passes a verification gate before it is committed. Rounds
repeat until a full panel round returns no HIGH or CRITICAL findings.

## Install

This plugin is distributed via the `eric-skills` marketplace at the root of this repo:

```
/plugin marketplace add ecwilsonaz/skills
/plugin install code-review-panel@eric-skills
```

Then invoke it in any session:

```
/code-review-panel:code-review-panel
```

### Manual install (no plugin system)

```bash
mkdir -p ~/.claude/skills/code-review-panel
cp skills/code-review-panel/SKILL.md ~/.claude/skills/code-review-panel/
```

With a manual install the skill is invoked as plain `/code-review-panel`.

## Optional external reviewers

The Claude-model reviewers work out of the box. The external panel seats need their
own tooling, and the skill degrades gracefully when one is missing:

- **Codex** — OpenAI's CLI (`codex`), pinned to `gpt-6.1-sol` at high reasoning;
  requires OpenAI authentication.
- **Antigravity** — Google's agentic IDE CLI; requires its own setup.
- **OpenRouter models** (on request) — set `OPENROUTER_API_KEY`. Note that requests go
  to third-party model providers; review OpenRouter's privacy settings.

## Fixing what the panel finds

The review is half the skill. The other half is what happens after you accept a finding,
because a fix written from a reviewer's example is the most reliable way to introduce the
next defect — three consecutive rounds of one campaign each found a defect in the previous
round's fix.

So a round does not close at "the tests pass". Before the commit, the skill runs a **fix
verification gate**:

- an **A/B blast-radius sweep** whenever the fix changes a decision function and a real
  input population exists — old code against new over the whole population, every changed
  outcome classified as an intended improvement with evidence or as a defect. A threshold
  is never calibrated from the finding's own examples; those are the cases one reviewer
  happened to hit, and the counter-population is the point of the sweep;
- **Codex over the isolated fix diff**, which covers what a data sweep structurally cannot
  see — lock lifetimes, interleavings, error-path ordering;
- **one Claude Opus 5.5 skeptic subagent** whose brief is to refute the fix by enumerating the truth table
  of every changed predicate and hunting the empty cell. The fix regressions that motivated
  this all lived in a cell nobody had named.

Gate findings are iterations inside the same round: fix, re-run the gate, then commit.
The two reviewer seats run after **every** panel fix, including doc wording, CSS, dead code,
and test-only edits. The A/B sweep applies when a decision function changed and real
input data is available. A failed reviewer leaves the gate incomplete.

Two rules exist because a whole campaign's churn traced to skipping them. **A fix that adds
a threshold, ratio, classifier or diagnosis branch is a design question wearing a bug's
clothes** — it gets the owner's rule as a written truth table before any code, because a fix
that could be written several plausible ways will be written several plausible ways, each
falsified by the next reviewer. And **a parser or text-transform fix needs a fixture captured
from a real page, with a mutation that removes the feature failing it**: one such unit was
dead in production under 4,766 green tests, because every fixture was typed by hand and none
exercised the real cleaner.

A finding that is real but outside the review's scope is not dropped and not a side note: it
becomes its own row in the disposition table, routed to whoever owns it.

## A note on cost

Each review round fans out to every panelist in parallel, and findings then get
adversarially verified — this uses substantially more tokens (and external API credits,
if configured) than a single-model review. Panel composition is decided interactively,
so you control the spend.

## Layout

```
.claude-plugin/
  plugin.json        # plugin manifest
skills/
  code-review-panel/
    SKILL.md         # the skill itself
```
