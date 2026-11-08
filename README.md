# PhishClean agent skill

An [agent skill](https://skills.sh) that lets Claude Code, Codex, Cursor, Gemini CLI and other coding agents check links, emails, email headers, JWTs, web pages and source code for phishing and leaked secrets, using the free [PhishClean API](https://www.phishclean.com/developers).

```bash
npx skills add chidhu07/phishclean-skills
```

No account or API key is needed. Without a key you get 30 checks per hour per IP address.

## Install in Hermes Agent

This repo is also an [Agent Plugins v1](https://agent-plugins.org) package (`plugin.json` + `mcp.json` + `skills/`), so [Hermes Agent](https://github.com/NousResearch/hermes-agent) can install it as a plugin. That gives Hermes both the skill and the six PhishClean MCP tools:

```bash
hermes plugins install chidhu07/phishclean-skills --no-enable
hermes plugins enable phishclean
```

What the plugin does:

- **Configuration only.** No local code runs. `mcp.json` points at PhishClean's hosted MCP server, `https://www.phishclean.com/mcp` (Streamable HTTP, no sign-in, no keys in config).
- **Every tool call goes to phishclean.com.** The server receives only what the agent passes to a tool (a URL, email text, email headers, a token, source code or page HTML), analyses it in memory and doesn't store or log it. For rate limiting it keeps an hourly count per IP address (1,000 calls per hour) and deletes those counts after a few hours. See the [privacy policy](https://www.phishclean.com/privacy).
- **Read-only.** All six tools (`check_link`, `check_email`, `analyze_headers`, `decode_jwt`, `scan_secrets`, `scan_page`) are annotated `readOnlyHint: true` and change nothing. The package still marks the server `untrusted`, so Hermes would ask before any tool that isn't read-only.
- **No password tool.** The remote server leaves the password checker out on purpose.
- The tool list is served by the remote server at runtime, so it isn't pinned by a commit.

## What your agent can do with it

| Ask | What happens |
|---|---|
| "Is this link a scam? `https://paypal-secure-login.xyz/verify`" | `check-link`: brand spoofing, risky TLDs, lookalike domains, tokens in the URL |
| "Is this email phishing?" + pasted text | `check-email`: urgency, credential requests, generic greetings, bad links |
| "Was this email spoofed?" + raw headers | `analyze-headers`: SPF, DKIM, DMARC, From / Return-Path mismatch, relay hops |
| "Decode this JWT" | `decode-jwt`: `alg: none`, expiry, weak algorithms, sensitive claims |
| "Check my staged changes for API keys" | `scan-secrets`: AWS, Stripe, GitHub, Google, Slack, Twilio, SendGrid, private keys and more |
| "Is this login page stealing passwords?" + HTML | `scan-page`: forms posting to other domains, hidden iframes, HTTP downgrades |

## How the skill handles sensitive input

- It never opens the link it is checking.
- It strips the signature from a JWT before sending it, so the token can't be replayed. All checks still work.
- It sends only the relevant files to the secret scanner, and asks before sending private code when you didn't ask for PhishClean by name.
- It never sends passwords.
- It won't tell you a link is "safe", only that no warning signs were found.

The API analyses input in memory and doesn't store or log it. See the [privacy policy](https://www.phishclean.com/privacy).

## Higher limits

Set `PHISHCLEAN_API_KEY` to a key from [phishclean.com/developers](https://www.phishclean.com/developers) and the skill sends it as a bearer token.

## Other ways to use PhishClean in an agent

- **Claude connector / remote MCP:** `https://www.phishclean.com/mcp`
- **Hermes Agent plugin:** see [Install in Hermes Agent](#install-in-hermes-agent) above
- **Local MCP server:** [`phishclean-mcp`](https://github.com/chidhu07/phishclean-mcp) on npm
- **In your browser:** the [PhishClean extension](https://www.phishclean.com) checks every page you open in Chrome, Firefox and Edge.

## License

MIT
