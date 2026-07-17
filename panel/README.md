# Panel — an advisor-panel skill for Claude Code

Convene a panel of independent AI advisor personas to pressure-test a strategic,
judgment, or factual question. The skill designs a persona roster tailored to your
question, fans the personas out in parallel (each blind to the others), and returns a
**convergence/divergence map**: where independent lenses agree (trust it), where they
split (the real cruxes worth your attention), and what no lens covered. You stay in the
loop at every decision gate — roster approval, which cruxes to dig into, whether to run
another round.

For code review, see its sibling skill `code-review-panel` (not included here).

## Install

Requires a recent version of [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

This plugin is distributed via the `eric-skills` marketplace at the root of this repo:

```
/plugin marketplace add ecwilsonaz/skills
/plugin install panel@eric-skills
```

Then invoke it in any session:

```
/panel:panel Should we open-source our SDK this quarter?
```

Claude will also invoke it automatically when a question calls for multi-perspective
pressure-testing ("run this by a few lenses", "what am I missing", go/no-go decisions).

### Manual install (no plugin system)

Copy the skill folder into your personal skills directory:

```bash
mkdir -p ~/.claude/skills/panel
cp skills/panel/SKILL.md ~/.claude/skills/panel/
```

Or drop it into a repo's `.claude/skills/panel/` to share it with everyone who clones
that repo. With a manual install the skill is invoked as plain `/panel`.

## A note on cost

Each panel round spawns several parallel agents (one per persona, plus optional
cross-examination), so a session using this skill consumes noticeably more tokens
than a normal conversation. The roster size is decided interactively, so you control
the spend.

## Layout

```
.claude-plugin/
  plugin.json        # plugin manifest
skills/
  panel/
    SKILL.md         # the skill itself
```
