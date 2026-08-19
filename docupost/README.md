# DocuPost — send physical mail from Claude Code

Draft, preview, and mail a real printed letter through the
[DocuPost](https://docupost.com) print-and-mail API. The skill drafts the letter as
HTML from a proven business-letter template, shows you a print-styled 8.5x11 preview
in your browser, and only after your explicit approval submits it to DocuPost
(~$1.50 for a one-page black-and-white First Class letter; certified mail supported).
DocuPost gives a 1-hour cancellation window with auto-refund.

## Install

Requires a recent version of [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

This plugin is distributed via the `eric-skills` marketplace at the root of this repo:

```
/plugin marketplace add ecwilsonaz/skills
/plugin install docupost@eric-skills
```

## Setup

1. Create a DocuPost account and generate an API token on the dashboard's Developer page.
2. Put it in your project's `.env`:

```
docupost_api_key=YOUR_TOKEN
```

3. `chmod 600 .env`, and gitignore it.

DocuPost is pay-as-you-go with a $10 minimum balance deposit.

## Use

Say "send a letter to ..." in a project with the key configured. You approve the
preview before anything is sent, and you run the final send command yourself.
