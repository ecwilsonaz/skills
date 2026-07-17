# eric-skills — a Claude Code plugin marketplace

A collection of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills,
packaged as plugins and served from this repo's own marketplace.

## Install

Add the marketplace once:

```
/plugin marketplace add ecwilsonaz/skills
```

Then install whichever plugins you want:

```
/plugin install panel@eric-skills
```

## Plugins

| Plugin | What it does |
|---|---|
| [`panel`](./panel/) | Advisor panel — pressure-test a strategic, judgment, or factual question from several independent AI perspectives, with a convergence/divergence map. |

Each plugin's own README covers usage, invocation, and any cost caveats.

## Layout

```
.claude-plugin/
  marketplace.json    # the marketplace catalog — lists every plugin below
<plugin-name>/
  .claude-plugin/
    plugin.json       # that plugin's manifest
  skills/
    <skill-name>/
      SKILL.md
```

To add a new plugin: create a sibling directory in this layout and append an entry to
`.claude-plugin/marketplace.json` with `"source": "./<plugin-name>"`.
