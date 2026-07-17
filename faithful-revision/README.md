# Faithful Revision — structural + line editing for Claude Code

Revise prose under one of two explicit contracts, chosen up front:

## POLISH — lossless revision

One promise: the strength and arrangement of the writing change; the meaning,
facts, quotations, attributions, and voice never do.

- **Phase 1 — ARCHITECT:** a one-shot structural proposal for the whole section
  (reorder, merge/split, surface the buried point, fix the lead and kicker), with a
  hard checkpoint: at a genuine structural fork, *you* choose, not the model.
- **Phase 2 — POLISH:** an iterative declutter loop run to convergence on each
  paragraph, every round passing an adversarial integrity gate against the true
  original.

Built especially for reported/journalistic prose, where attributions are sacred:
the gates explicitly guard against dropped attributors, manufactured implications
from reordering, and invented actors when un-passivizing a sentence.

## BRIEF — lossy executive compression

For exec summaries, emails to leadership, and status updates, where concision
outranks thoroughness. Deliberately lossy — with the loss kept visible:

- **BLUF spine:** the recommendation or key finding leads (≤6-word tease,
  one-sentence lede, bolded "why it matters"), support capped at three points,
  explicit ask with owner and date.
- **Hard caps sized to reader-minutes:** an email fits one screen (~250 words);
  a summary item ~200.
- **The CUT LEDGER:** every substantive omission is listed, one line each, for
  you to veto — cuts are proposals, never silent. Detail relocates to an appendix
  before it gets deleted.
- **Same fidelity core:** retained facts stay true and attributed; a source's
  hedged claim never flattens into asserted fact; no implication manufactured by
  seating two surviving facts together. Tone may outrank grammatical completeness
  — never clarity.

BRIEF never auto-runs: it triggers only on an explicit ask ("brief this," "exec
summary," "BLUF this") or when you accept the skill's one-line offer on a
workplace-directed document.

## Install

This plugin is distributed via the `eric-skills` marketplace at the root of this repo:

```
/plugin marketplace add ecwilsonaz/skills
/plugin install faithful-revision@eric-skills
```

Invoke it by pasting a paragraph or section and asking to "tighten this,"
"restructure this," "edit this like Zinsser," or "brief this for [reader]" — or
explicitly as `/faithful-revision:faithful-revision`.

### Manual install (no plugin system)

```bash
mkdir -p ~/.claude/skills/faithful-revision
cp -R skills/faithful-revision/* ~/.claude/skills/faithful-revision/
```

With a manual install the skill is invoked as plain `/faithful-revision`.

## Intellectual lineage

The ruleset is distilled from more than ten books on writing craft — buy them;
this skill is a working companion to them, not a substitute.

**Behind POLISH:**

- William Zinsser, *On Writing Well* — the polish loop's philosophy: simplicity,
  clutter, voice, unity
- Joseph M. Williams, *Style: Toward Clarity and Grace* — cohesion, stress
  position, motivation
- Barbara Minto, *The Pyramid Principle* — governing thought, grouping, SCQA
- Steven Pinker, *The Sense of Style* — coherence relations, given-before-new
- Roy Peter Clark, *Writing Tools* — emphatic order, gold coins, reverse outline
- John McPhee, *Draft No. 4* — structure from material, selection, the kicker

**Behind BRIEF:**

- Jim VandeHei, Mike Allen & Roy Schwartz, *Smart Brevity* — the lede/why-it-matters
  spine, length ceilings, write-like-you-talk
- Joseph McCormack, *Brief* — attention economics, detail tiers, halve-it-twice,
  the deep-vs-light brevity warning
- Josh Bernoff, *Writing Without Bullshit* — the reader's-time imperative, word
  budgets, always-recommend, bad-news-first
- William Strunk Jr., *The Elements of Style* (1918) — omit needless words,
  positive form, emphatic ends
- Colin Bryar & Bill Carr, *Working Backwards* — the counterpoint: when connected
  prose under a hard cap beats bullets (BRIEF's "when not to use")
- U.S. Army BLUF doctrine (AR 25-50) — bottom line up front

The principles and categories follow these authors' published ideas (ideas and
methods, with attribution); the worked examples, word catalogs, and prose in this
skill's reference files are the skill's own.

## Layout

```
.claude-plugin/
  plugin.json          # plugin manifest
skills/
  faithful-revision/
    SKILL.md           # the phases, modes, and gates
    references/
      architecture.md  # Phase 1 structural engine (Williams/Minto/Pinker/Clark/McPhee/Zinsser)
      principles.md    # Phase 2 philosophy
      operations.md    # line-edit moves by magnitude
      clutter-list.md  # clutter catalog
      examples.md      # worked BEFORE→AFTER revisions
      voice-guard.md   # what the integrity gate protects as voice
      consistency.md   # tense/person/mood unity check
      brief.md         # BRIEF mode: BLUF spine, compression ops, gate, cut ledger
```
