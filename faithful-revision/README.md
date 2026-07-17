# Faithful Revision — structural + line editing for Claude Code

Revise prose in two phases with one promise: the strength and arrangement of the
writing change; the meaning, facts, quotations, attributions, and voice never do.

- **Phase 1 — ARCHITECT:** a one-shot structural proposal for the whole section
  (reorder, merge/split, surface the buried point, fix the lead and kicker), with a
  hard checkpoint: at a genuine structural fork, *you* choose, not the model.
- **Phase 2 — POLISH:** an iterative declutter loop run to convergence on each
  paragraph, every round passing an adversarial integrity gate against the true
  original.

Built especially for reported/journalistic prose, where attributions are sacred:
the gates explicitly guard against dropped attributors, manufactured implications
from reordering, and invented actors when un-passivizing a sentence.

## Install

This plugin is distributed via the `eric-skills` marketplace at the root of this repo:

```
/plugin marketplace add ecwilsonaz/skills
/plugin install faithful-revision@eric-skills
```

Invoke it by pasting a paragraph or section and asking to "tighten this,"
"restructure this," or "edit this like Zinsser" — or explicitly as
`/faithful-revision:faithful-revision`.

### Manual install (no plugin system)

```bash
mkdir -p ~/.claude/skills/faithful-revision
cp -R skills/faithful-revision/* ~/.claude/skills/faithful-revision/
```

With a manual install the skill is invoked as plain `/faithful-revision`.

## Intellectual lineage

The method distills craft principles from books every serious editor should own —
buy them; this skill is a working companion to them, not a substitute:

- William Zinsser, *On Writing Well* — the polish loop's philosophy: simplicity,
  clutter, voice, unity
- Joseph M. Williams, *Style: Toward Clarity and Grace* — cohesion, stress
  position, motivation
- Barbara Minto, *The Pyramid Principle* — governing thought, grouping, SCQA
- Steven Pinker, *The Sense of Style* — coherence relations, given-before-new
- Roy Peter Clark, *Writing Tools* — emphatic order, gold coins, reverse outline
- John McPhee, *Draft No. 4* — structure from material, selection, the kicker

The principles and categories follow these authors' published ideas (ideas and
methods, with attribution); the worked examples, word catalogs, and prose in this
skill's reference files are the skill's own.

## Layout

```
.claude-plugin/
  plugin.json          # plugin manifest
skills/
  faithful-revision/
    SKILL.md           # the two-phase process and its gates
    references/
      architecture.md  # Phase 1 structural engine (Williams/Minto/Pinker/Clark/McPhee/Zinsser)
      principles.md    # Phase 2 philosophy
      operations.md    # line-edit moves by magnitude
      clutter-list.md  # clutter catalog
      examples.md      # worked BEFORE→AFTER revisions
      voice-guard.md   # what the integrity gate protects as voice
      consistency.md   # tense/person/mood unity check
```
