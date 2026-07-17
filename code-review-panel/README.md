# Code Review Panel — a multi-model code-review skill for Claude Code

Run a code review as a **panel of AI reviewers** working in parallel — several Claude
models plus external reviewers (Antigravity, Codex, and optionally any OpenRouter
model) — then consolidate their findings, adversarially verify each one, and triage
the survivors with you interactively. Independent reviewers catch different bug
classes; the verification pass kills plausible-but-wrong findings before you see them.

For non-code questions, see its sibling skill [`panel`](../panel/).

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

- **Codex** — OpenAI's CLI (`codex`), run via `npx codex@latest` or a local install;
  requires OpenAI authentication.
- **Antigravity** — Google's agentic IDE CLI; requires its own setup.
- **OpenRouter models** (on request) — set `OPENROUTER_API_KEY`. Note that requests go
  to third-party model providers; review OpenRouter's privacy settings.

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
