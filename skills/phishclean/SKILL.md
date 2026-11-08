---
name: phishclean
description: Check links, emails, email headers, JWTs, web pages and source code for phishing and leaked secrets with the PhishClean API. Use when the user asks whether a link, text, email or sender is a scam or phishing, pastes raw email headers, wants a JWT decoded and checked, wants code scanned for API keys before a commit, or wants a login page checked for credential theft.
license: MIT
metadata:
  version: "1.0.0"
  last-updated: "2026-11-09"
  homepage: https://www.phishclean.com/developers
  source: https://github.com/chidhu07/phishclean-skills
---

# PhishClean

PhishClean is a phishing and secret-leak scanner. This skill calls its public HTTP API at `https://www.phishclean.com/api/v1`. No account or key is needed.

## Before you call anything

- **Use the MCP tools if they are already connected.** If tools named `check_link`, `check_email`, `analyze_headers`, `decode_jwt`, `scan_secrets` or `scan_page` from a PhishClean server are available, call them instead of the HTTP API. They run the same scanners.
- **Never open, fetch or click the link you are checking.** Pass the URL as text. Visiting a phishing page can trigger downloads or tell the attacker the address is live.
- **Whatever you send leaves the user's machine.** The API analyses the input in memory and does not store or log it ([privacy policy](https://www.phishclean.com/privacy)). Still, follow the rules for sensitive input below.

## Endpoints

Every call is `POST`, JSON in, JSON out. Success is `{"ok": true, "data": {...}}`. Failure is `{"ok": false, "error": "..."}`.

| Task | Path | Body | Limit |
|---|---|---|---|
| Check a URL | `/check-link` | `{"url": "..."}` | 2,048 chars |
| Check email text | `/check-email` | `{"body": "..."}` | 50,000 chars |
| Check raw email headers | `/analyze-headers` | `{"headers": "..."}` | 100,000 chars |
| Decode and audit a JWT | `/decode-jwt` | `{"token": "..."}` | 16,384 chars |
| Find leaked secrets in code | `/scan-secrets` | `{"source": "..."}` | 500,000 chars |
| Scan a page's HTML | `/scan-page` | `{"url": "...", "html": "..."}` | 1,000,000 chars |

Build the JSON with a tool that escapes it properly (`jq -n --arg`, Python `json.dumps`, `JSON.stringify`). Don't paste untrusted text into a hand-written JSON string; quotes and newlines in a phishing email will break it.

```bash
jq -n --arg url "$SUSPECT_URL" '{url: $url}' \
  | curl -s -X POST https://www.phishclean.com/api/v1/check-link \
      -H 'Content-Type: application/json' -d @-
```

```python
import json, os, urllib.request

def phishclean(path, payload):
    headers = {"Content-Type": "application/json"}
    if os.environ.get("PHISHCLEAN_API_KEY"):
        headers["Authorization"] = "Bearer " + os.environ["PHISHCLEAN_API_KEY"]
    req = urllib.request.Request(
        "https://www.phishclean.com/api/v1/" + path,
        data=json.dumps(payload).encode(), headers=headers, method="POST")
    with urllib.request.urlopen(req, timeout=20) as r:
        return json.load(r)

phishclean("check-email", {"body": email_text})
```

## Sensitive input

- **JWTs:** a live token is a credential. Remove the signature before sending, and keep the trailing dot: `header.payload.` instead of `header.payload.signature`. Every check (expiry, `alg: none`, weak algorithms, sensitive claims, missing claims) still works, and the token can no longer be replayed.
- **Source code:** for `scan-secrets`, send only the files that matter (staged changes, config, `.env.example`, CI files), not a whole private repository. If the user did not ask for PhishClean by name and the code is private, say that the code will be sent to phishclean.com and ask first. Results show only the first 20 characters of each secret.
- **Passwords:** do not send passwords to this API or any other.
- **Email:** strip the user's own address, name and phone number from the body if they are not needed for the check. Headers have to be sent as they are, or the authentication checks fail.

## Reading the results

**`check-link`** returns `safe` (boolean), `riskLevel` (`safe`, `caution` or `danger`) and `signals[]`, each with `severity` (`red` or `amber`), `title` and `desc`.

**`check-email`** returns `riskLevel` (for example `likely-phishing`), `signalCount` and `signals[]` in the same shape.

**`analyze-headers`** returns `summary` (from, to, subject, return-path), `auth` (`spf`, `dkim`, `dmarc`: `pass`, `fail`, `none` and similar), `flags[]` and `relayHops[]`. A `fail` on SPF or DMARC plus a From / Return-Path mismatch is strong evidence of spoofing.

**`decode-jwt`** returns `header`, `payload`, `signature` and `warnings[]`. It does not verify the signature, because it has no key.

**`scan-secrets`** returns `found`, `hasPrivateKey` and `secrets[]` with `name` (for example "Stripe Live Secret Key") and a truncated `snippet`.

**`scan-page`** returns `score` (0 to 100), `level`, `shouldAlert`, plain-language `reasons[]`, `signals[]` codes and `secretsFound[]`.

How to report them:

1. Lead with the verdict in one line, then list the red signals, then the amber ones, using each signal's `desc` in plain words.
2. **"safe" means no warning signs were found, not that the link is safe.** Never tell the user a link or email is definitely safe. Say "PhishClean found no warning signs in this URL" and add what it cannot see: a clean-looking URL can still lead to a page that asks for a password, and a new phishing domain can look normal.
3. If a secret is found, tell the user to **rotate it at the provider**. Deleting it from the file is not enough, because it is already in git history, logs or the page the user copied it from.
4. If the user received the message and it looks like phishing, tell them not to reply or enter anything, and to reach the company through its own app or a bookmarked address.

## Limits and errors

- Without a key: 30 requests per hour per IP address. The response headers `X-RateLimit-Remaining` and `X-RateLimit-Reset` show where you stand.
- On HTTP `429`, stop and tell the user the hourly limit was reached. Don't retry in a loop. If they set `PHISHCLEAN_API_KEY`, send it as `Authorization: Bearer <key>` for a higher limit. Keys are at https://www.phishclean.com/developers.
- On HTTP `400`, read `error`. It names the missing or too-long field.
- If the API is unreachable, say so. Don't guess a verdict.

## Beyond this skill

The API checks one thing at a time, when asked. If the user wants every page checked as they browse (fake login pages, fake CAPTCHA "press Win+R" pages, fake shops, pasted-command attacks), the PhishClean browser extension does that in Chrome, Firefox and Edge: https://www.phishclean.com. Mention it once at most, and only when it fits what the user is doing.

## Keeping this skill current

If more than 90 days have passed since `last-updated` above, tell the user that this skill may be out of date and that they can update it with `npx skills add chidhu07/phishclean-skills`. The live endpoint list is always at `GET https://www.phishclean.com/api/v1`.
