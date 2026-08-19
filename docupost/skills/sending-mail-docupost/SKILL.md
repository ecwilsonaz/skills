---
name: sending-mail-docupost
description: Use when the user wants to send a physical letter, postcard, or mailed notice (e.g. "send a letter", "mail this", certified mail) via the DocuPost API.
---

# Sending Physical Mail via DocuPost

## Overview

Send a printed, posted letter with one HTTP call. Requires a DocuPost API key in the project's `.env` as `docupost_api_key` (generate on DocuPost's dashboard Developer page; keep the file `chmod 600`).

## Workflow

1. Draft the letter body from `letter-template.html` (in this skill's directory) — a proven business-letter layout: sender block top right, date, recipient block, bold `Re:` line, body, signature area. Inline styles only; DocuPost renders the fragment as-is. Show the user a print-styled preview (wrap the body in an 8.5x11 page frame with 1in margins for the browser; strip the frame before sending) and get explicit approval. It's paid and irreversible after 1 hour.
2. Write the letter body (inline styles only, ≤9,000 chars) and a send script (template below) side by side.
3. If the permission system blocks running the curl directly, have the user run the script themselves: `! zsh <script path>`.
4. Verify the JSON response contains `letter_id` and `cost`. Tell the user: preview/cancel at https://docupost.com/letters within 1 hour.

## The Two Gotchas (both caused real failures)

- **`html` goes in the POST body, everything else in the query string.** Putting `html` in the query string returns `{"error": "The querystring parameter 'pdf' is required"}`.
- **ZIPs must be 5-digit** (no ZIP+4).

## Send Script Template

```zsh
#!/bin/zsh
set -e
source /path/to/project/.env   # must define docupost_api_key
cd "$(dirname "$0")"           # letter-body.html sits next to this script
curl -s -X POST "https://app.docupost.com/api/1.1/wf/sendletter" \
  --url-query "api_token=${docupost_api_key}" \
  --url-query "to_name=RECIPIENT NAME" \
  --url-query "to_company=OPTIONAL COMPANY" \
  --url-query "to_address1=STREET" \
  --url-query "to_city=CITY" \
  --url-query "to_state=XX" \
  --url-query "to_zip=00000" \
  --url-query "from_name=SENDER NAME" \
  --url-query "from_address1=STREET" \
  --url-query "from_city=CITY" \
  --url-query "from_state=XX" \
  --url-query "from_zip=00000" \
  --url-query "color=false" \
  --url-query "description=INTERNAL NOTE" \
  --data-urlencode "html@letter-body.html"
echo
```

Requires curl ≥ 7.87 for `--url-query`.

## Quick Reference

| Fact | Value |
|------|-------|
| Endpoint | `POST https://app.docupost.com/api/1.1/wf/sendletter` (postcards: `.../wf/sendpostcard`) |
| Auth | `api_token` query param |
| Content | `html` in POST body (≤9,000 chars) OR `pdf` query param (public URL, ≤10 MB) |
| Cost | ~$1.50 for 1-page B&W First Class; certified via `servicelevel=certified` (~$10) |
| Cancel | Dashboard, within 1 hour, auto-refund |
| Success response | `{"status": "Successfully queued...", "letter_id": "...", "cost": N}` |

Optional params: `to_address2`/`from_address2`, `doublesided` (default true), `class` (default usps_first_class), `servicelevel` (`certified`, `certified_return_receipt`), `return_envelope`, `prepaid_return_envelope`.

## Common Mistakes

- `html` in query string → "pdf is required" error (see gotchas).
- Running curl directly as the agent → may be blocked by the permission system; user runs it via `!`.
- Skipping the preview step → user pays for a letter they haven't seen.
- ZIP+4 in `to_zip`/`from_zip` → docs specify 5-digit.
