---
name: shieldnode
description: Call APIs through a proxy that holds the real keys, with push approval on your phone.
version: 1.0.0
author: Dorunaitsu
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [security, secrets, api-keys, proxy, approval]
    category: security
setup:
  help: "Create a free account at https://shieldnode.app, add a service, then create a virtual key on it. The key is shown once."
  collect_secrets:
    - env_var: SHIELDNODE_KEY
      prompt: "ShieldNode virtual key (starts with shieldnode_)"
      provider_url: "https://shieldnode.app"
      secret: true
---

# ShieldNode

Use this skill when the user wants an API called without the real credential ever reaching the agent. ShieldNode keeps their key in an encrypted vault and gives Hermes a **virtual key** instead. Hermes calls the proxy, the proxy injects the real credential, the upstream API answers.

```
Hermes --X-Api-Key: shieldnode_...--> proxy.shieldnode.app/{path} --real key--> api.upstream.com/{path}
```

The user can leave the key **disabled** by default. Hermes then gets a push approval on their phone, they tap Approve for a bounded window, and the call goes through. A leaked virtual key is inert until its owner says otherwise.

## Requirements

- A ShieldNode account (free tier: 2 services, 3 keys per service, 500 requests a month)
- A virtual key in `SHIELDNODE_KEY`
- The ShieldNode mobile app (iOS/Android) for push approval. Without it, a disabled key simply fails.

## When to Use

- The user hands you a value starting with `shieldnode_`
- Calling any API the user has put behind ShieldNode
- A call returns `403 approval_required`
- Setting up recurring access for a cron job
- The user wants to use an API they have not configured yet
- Debugging proxy errors, especially unexpected 404s

## Setup

Hermes collects `SHIELDNODE_KEY` at install time and injects it as an env var. To add it manually, or to add more services, put it in `~/.hermes/.env`:

```env
SHIELDNODE_KEY=shieldnode_...
SHIELDNODE_STRIPE_KEY=shieldnode_...
SHIELDNODE_OPENAI_KEY=shieldnode_...
```

One virtual key maps to one service, so use `SHIELDNODE_<SERVICE>_KEY` once the user has more than one. The values can also be mapped from Bitwarden or 1Password if the user manages secrets there. No ShieldNode-specific tooling is needed: it is a plain HTTPS call with a header.

Never write a key value into a file the user commits, and never print one back into the conversation.

## Hermes Execution Pattern

Resolve an unknown key **before** doing anything else, in one call. Do not ask the user which service it belongs to.

```bash
curl -sS -H "X-Api-Key: $SHIELDNODE_KEY" \
  "https://proxy.shieldnode.app/_shieldnode/whoami"
```

```json
{
  "service": "OpenAI",
  "base_url": "https://api.openai.com/v1",
  "allowed_methods": ["GET", "POST"],
  "rate_limit_per_min": 60,
  "active": false,
  "requires_approval": true,
  "default_approval_duration_minutes": 30
}
```

That tells you the upstream, the configured `base_url` (which fixes your path convention), and whether the next call goes straight through (`active: true`) or will trigger an approval push. whoami never returns credentials, is never forwarded upstream, and fires no push.

Then call the API:

```bash
curl -sS -H "X-Api-Key: $SHIELDNODE_KEY" \
     -H "X-Agent-Name: Hermes" \
     "https://proxy.shieldnode.app/<API_PATH>"
```

### The base URL trap

The proxy appends your path **verbatim** to the configured base URL. A version prefix already in the base URL must not be repeated. For an upstream `https://api.example.com/v1/users`:

| Base URL on the service | Correct proxy call |
|---|---|
| `https://api.example.com/v1` | `https://proxy.shieldnode.app/users` |
| `https://api.example.com` | `https://proxy.shieldnode.app/v1/users` |

Take `base_url` from whoami, subtract it from the full upstream URL, and what remains goes after `proxy.shieldnode.app/`. This causes nearly every unexpected 404, so check it first.

## Push Approval

A disabled key whose owner has the mobile app returns a structured 403 instead of a plain failure.

| Response body `error` | What to do |
|---|---|
| `approval_required` | The user got a push. Wait `poll_interval_seconds` (default 30s), retry, up to `timeout_seconds` (default 5 min). |
| `approval_denied` | Stop. Report "User declined access on ShieldNode mobile." Do not retry on your own. |
| `key_disabled` | No mobile app registered. Surface and stop. |

Headers that matter:

```bash
curl -sS -H "X-Api-Key: $SHIELDNODE_KEY" \
     -H "X-Agent-Name: Hermes" \
     -H "X-Approval-Duration: 15m" \
     -H "X-Approval-Reason: sending the weekly report" \
     "https://proxy.shieldnode.app/emails"
```

- **`X-Agent-Name: Hermes`** on every call. The push then reads "Hermes is requesting access" instead of "An external agent". Named requests get approved, generic ones get ignored.
- **`X-Approval-Duration`** sized to the job. Clamped to [1, 1440] minutes. Accepts `30`, `15m`, `2h`.
- **`X-Approval-Reason`**, a short honest phrase. It shows on the user's lock screen, so never put secrets or personal data in it.

Full polling implementation: `references/approval-recipe.md`.

## Scheduled Windows

For a cron or a nightly batch, ask **once** for a recurring window instead of firing a push every run. Inside the window, calls return 200 with no push and no polling.

```bash
curl -sS -X POST "https://proxy.shieldnode.app/_shieldnode/schedule-request" \
     -H "X-Api-Key: $SHIELDNODE_KEY" -H "Content-Type: application/json" \
     -d '{"time":"03:00","timezone":"Europe/Paris","days":["mon","tue","wed","thu","fri"],
          "duration_minutes":30,"agent_name":"Hermes","reason":"nightly analytics sync"}'
```

`time` and `timezone` are required. Use the zone **your scheduler** runs in, not the user's: that is the most common mistake. `duration_minutes` defaults to 10, clamped to [5, 1440], and the window opens 5 minutes early as a lead. Returns `202` with a `request_id` you can poll at `/_shieldnode/schedule-request/<request_id>`.

The server never tells you the next open time, by design. You set the schedule, so you already know it. Outside the window the key falls back to the normal push flow, so keep that path in your code.

## Proposing a New Service

When the user wants an API that is not in their ShieldNode account yet, propose it rather than walking them through a form. They get a push, open a prefilled screen, type only their own credential, and approve. **You never see that credential.**

This needs a **config key** (prefix `shieldnode_config_`), created on the built-in "ShieldNode" service at the top of their service list. Ask them for one and store it as `SHIELDNODE_CONFIG_KEY`.

```bash
curl -sS -X POST "https://proxy.shieldnode.app/_shieldnode/config-request" \
     -H "X-Api-Key: $SHIELDNODE_CONFIG_KEY" -H "Content-Type: application/json" \
     -d '{"name":"Stripe","base_url":"https://api.stripe.com",
          "detected_auth_method":{"method":"header_bearer"},
          "credential_labels":["API key"],
          "agent_name":"Hermes","reason":"creating invoices for the user"}'
```

Fill `base_url` and `detected_auth_method` from what you know about the API, or from its docs. Poll `/_shieldnode/config-request/<request_id>` until `approved`, then ask the user for a normal virtual key on the new service.

**Never put the user's upstream API key in this request.** You propose the non-secret shape only.

One service maps to one base URL, so APIs spanning subdomains (Twilio, Shopify Admin plus Storefront) need one service each.

## Status Codes

| Code | Meaning | Action |
|---|---|---|
| 401 | Virtual key invalid or expired | Check the key in the dashboard |
| 403 | Disabled, over quota, or path not allowlisted | See Push Approval above |
| 404 | Path missing upstream | Check the base URL trap first |
| 429 | Rate limited | Wait, or raise the key's limit |
| 413 | Body over the 90 MB cap | Use a signed-URL upload |
| 502 / 504 | Cold start or slow upstream | Wait 30s, then test the upstream directly |

## Guardrails

- Never print a virtual key back to the user, and never ask them to paste one into the chat. They put it in `~/.hermes/.env` themselves.
- Tell the user **once** that an approval is pending, then poll silently. A message every 30 seconds feels like spyware.
- Never retry after an explicit decline, and never retry past the timeout. Ask the user instead.
- Do not fire parallel calls to force an approval. Pushes are debounced to 1 per 30s per key, and parallel calls wait on the same approval anyway.
- Distinguish a decline from a timeout when reporting back. They mean different things to the user.
- Never put secrets or personal data in `X-Approval-Reason`, and keep it truthful. It is how the user decides.

## Testing Without a Real API

Every ShieldNode account is seeded with a keyless demo service, `Cool Dogs — Playground` (upstream `dog.ceo`). Use it to verify the whole flow, including approval, before touching a real credential.

## References

- `references/approval-recipe.md` for the full polling loop in code
- `references/troubleshooting.md` for compression, pagination, Cloudflare 1010 and upload limits
- `references/service-docs.md` for writing a per-service reference document
- `references/dashboard-setup.md` for configuring a service by hand
- Canonical skill repository, kept up to date: https://github.com/Dorunaitsu/ShieldNode
- https://shieldnode.app
