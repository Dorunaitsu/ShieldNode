---
name: shieldnode
description: Use APIs through ShieldNode, a secure proxy that holds the real credentials so the agent never sees them. Use when you are handed a `shieldnode_...` key, when calling the ShieldNode proxy, when a call returns 403 approval_required, when setting up recurring access for a cron job, when proposing a new service for the user to approve, or when debugging proxy errors and base URL formatting.
---

# ShieldNode API proxy skill

ShieldNode holds a user's real API credentials in an encrypted vault and hands you a **virtual key** instead. You call the proxy, it injects the real credential, and the upstream API answers. The real key never reaches you.

```
you --X-Api-Key: shieldnode_...--> https://proxy.shieldnode.app/{path} --real credential--> https://api.upstream.com/{path}
```

**The key format is the signal.** Any value starting with `shieldnode_` is not a provider key. It goes in the `X-Api-Key` header against `proxy.shieldnode.app`, and this skill applies.

Never echo a real key value. In examples and code, use `$SHIELDNODE_<SERVICE>_KEY` or a `shieldnode_...` placeholder.

## 1. First contact: resolve the key

When handed a `shieldnode_...` key you have no reference doc for, resolve it in one call before anything else. Do not ask the user which service it is.

```bash
curl -H "X-Api-Key: shieldnode_..." "https://proxy.shieldnode.app/_shieldnode/whoami"
```

```json
{
  "service": "OpenAI",
  "base_url": "https://api.openai.com/v1",
  "proxy_base_url": "https://proxy.shieldnode.app",
  "allowed_methods": ["GET", "POST"],
  "allowed_paths": null,
  "rate_limit_per_min": 60,
  "max_requests": null,
  "expires_at": null,
  "active": false,
  "requires_approval": true,
  "default_approval_duration_minutes": 30
}
```

Read from it: which upstream this key proxies, the configured `base_url` (which decides your path convention, see §2), and whether the next call goes straight through (`active: true`) or triggers approval (`requires_approval: true`).

**Then write the service doc, without being asked.** If the project has no `services/<slug>.md` for the service whoami just named, create one now from the template in [references/service-docs.md](references/service-docs.md). No command, no question to the user: whoami already gave you the service, the base URL and the auth shape. Future sessions read that file instead of deriving it again.

whoami is answered by ShieldNode, never forwarded upstream, never returns credentials, does not count as a proxied request, and fires no push. Invalid key returns `401 invalid_key`. The `/_shieldnode/` path space is reserved and never collides with a real API path.

## 2. Call the proxy

```bash
curl -H "X-Api-Key: shieldnode_<VIRTUAL_KEY>" "https://proxy.shieldnode.app/<API_PATH>"
```

`<API_PATH>` is the path **relative to the base URL configured on the service**.

### The base URL trap (cause of nearly every unexpected 404)

The proxy appends your path verbatim to the configured base URL. Whatever version prefix (`/v1`, `/v0`, `/api`) is already in the base URL must **not** be repeated in your call.

For an upstream call to `https://api.example.com/v1/users`:

| Base URL configured on the service | Correct proxy call |
|---|---|
| `https://api.example.com/v1` | `https://proxy.shieldnode.app/users` |
| `https://api.example.com/v1/` | `https://proxy.shieldnode.app/users` (trailing slash normalized) |
| `https://api.example.com` | `https://proxy.shieldnode.app/v1/users` |

Rule: take the base URL from whoami, subtract it from the full upstream URL, and whatever remains goes after `proxy.shieldnode.app/`. On an unexpected 404, verify this first.

### Examples

```bash
# Airtable (base URL https://api.airtable.com/v0)
curl -H "X-Api-Key: shieldnode_..." \
  "https://proxy.shieldnode.app/<BASE_ID>/<TABLE_ID>?maxRecords=10"

# Resend (base URL https://api.resend.com)
curl -H "X-Api-Key: shieldnode_..." "https://proxy.shieldnode.app/emails"

# OpenAI (base URL https://api.openai.com/v1)
curl -H "X-Api-Key: shieldnode_..." -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello"}]}' \
  "https://proxy.shieldnode.app/chat/completions"
```

## 3. When a call is blocked: push approval

A disabled key whose owner has the mobile app returns a special 403 instead of a plain failure. The user gets a push and taps Approve to open a bounded window.

```json
// 403, user has a registered device
{ "error": "approval_required", "message": "Awaiting user approval on ShieldNode mobile",
  "request_id": "...", "requested_minutes": 30, "poll_interval_seconds": 30, "timeout_seconds": 300 }

// 403, user tapped Decline
{ "error": "approval_denied", "message": "User declined the most recent approval request" }

// 403, no mobile device registered (classic behaviour)
{ "error": "key_disabled", "message": "This key has been deactivated" }
```

| Response | What to do |
|---|---|
| `approval_required` | Wait `poll_interval_seconds` (default 30s), retry, loop up to `timeout_seconds` (default 5 min). |
| `approval_denied` | Stop. Say "User declined access on ShieldNode mobile." Do not retry on your own. |
| `key_disabled` | No mobile app on the user's side. Surface and stop. |
| 200 | Granted or not needed. Continue. |

### Headers that make approval work

```bash
curl -H "X-Api-Key: shieldnode_..." \
     -H "X-Agent-Name: Claude" \
     -H "X-Approval-Duration: 15m" \
     -H "X-Approval-Reason: deploying the staging build" \
     "https://proxy.shieldnode.app/chat/completions"
```

- **`X-Agent-Name`** (send it always). The push says "Claude is requesting access" instead of "An external agent". Generic requests get ignored; named ones get approved. Capped at 60 chars.
- **`X-Approval-Duration`** matched to the workload. Clamped server-side to [1, 1440] minutes. Accepts `5`, `30`, `15m`, `2h`. Omitted, the user's per-key default applies (fallback 30 min).
- **`X-Approval-Reason`** (short, honest, a phrase not a sentence). Shown on the push and the approval screen. Sanitized and capped at ~140 chars, display-only, never grants access. **Never put secrets or personal data in it**, it shows on a lock screen. A misleading reason is the fastest way to get every future request declined.

### Rules

1. Tell the user **once** that approval is pending, then poll silently. A message every 30 seconds feels like spyware.
2. Distinguish decline from timeout when you report back. They mean different things.
3. Do not retry past the timeout. If they did not approve in 5 minutes, they meant it. Ask in chat before retrying.
4. Do not fire parallel calls to force it. Pushes are debounced to 1 per 30s per (user, key), and parallel calls wait on the same approval anyway.

Full polling implementation: [references/approval-recipe.md](references/approval-recipe.md).

## 4. Recurring jobs: scheduled windows

For work on a fixed schedule (nightly batch, weekday cron), ask **once** for a recurring window instead of firing a push every run. Once approved, calls inside the window return 200 with no push and no polling.

Use a schedule when the job runs unattended at a known time. Use push approval (§3) for one-off, ad hoc, or interactive access.

```bash
curl -X POST "https://proxy.shieldnode.app/_shieldnode/schedule-request" \
     -H "X-Api-Key: shieldnode_..." -H "Content-Type: application/json" \
     -d '{
       "time": "03:00",
       "timezone": "Europe/Paris",
       "days": ["mon","tue","wed","thu","fri"],
       "duration_minutes": 30,
       "agent_name": "Claude",
       "reason": "nightly analytics sync"
     }'
```

| field | required | meaning |
|---|---|---|
| `time` | yes | Start `HH:MM`, the wall-clock time your cron fires, in `timezone`. |
| `timezone` | yes | IANA zone (`Europe/Paris`, `UTC`). Use the zone **your scheduler** runs in, not the user's. GitHub Actions is `UTC`. Stored as wall-clock, so it survives DST. |
| `days` | no | Subset of `mon`..`sun`. Default every day. |
| `duration_minutes` | no | Window length after `time`. Default 10, clamped [5, 1440]. Opens 5 min early as a lead. Size it to the real runtime. |
| `agent_name` | no | Shown on the approval screen. |
| `reason` | no | Same rules as `X-Approval-Reason`. |

Returns `202 { "status": "pending", "request_id": "...", "poll_interval_seconds": 60 }`. The user approves on mobile and may edit time, days, or duration. Poll it if you want:

```bash
curl -H "X-Api-Key: shieldnode_..." \
  "https://proxy.shieldnode.app/_shieldnode/schedule-request/<request_id>"
# -> { "status": "pending" | "approved" | "declined" | "expired", "schedule_id": "..." }
```

Outside the window the key behaves normally (403 push flow). A correctly timed cron never hits that. **The server never tells you the next open time**, by design: it would tell anyone holding a leaked key exactly when to use it. You set the time, so you already know it. The user sees it in the app.

Rules: ask once (identical repeats reuse the pending request, max 3 pending per key); get the timezone right, it is the most common mistake; a schedule is not a permanent grant (the user can revoke it and the emergency stop kills every schedule at once), so keep the §3 push flow as your fallback path.

## 5. Service not configured yet: propose it

When the user wants an API that is not in their ShieldNode account, propose it instead of walking them through a form. They get a push, open a prefilled screen, type only their own upstream key, and approve. **You never see or send that key.**

You need a **config key**: a key on the built-in "ShieldNode" service (always present, top of their service list), prefix `shieldnode_config_`. Ask the user to open ShieldNode, pick the ShieldNode service, create a key, and paste it to you. It only works on the endpoints below and cannot proxy.

```bash
curl -X POST "https://proxy.shieldnode.app/_shieldnode/config-request" \
     -H "X-Api-Key: shieldnode_config_..." -H "Content-Type: application/json" \
     -d '{
       "name": "Stripe",
       "base_url": "https://api.stripe.com",
       "detected_auth_method": { "method": "header_bearer" },
       "credential_labels": ["API key"],
       "agent_name": "Claude",
       "reason": "creating invoices for the user"
     }'
```

| field | required | meaning |
|---|---|---|
| `name` | yes | Human-readable service name. |
| `base_url` | yes | Upstream root URL, no trailing slash. SSRF-validated server-side. |
| `detected_auth_method` | no | What you know of the auth: `{"method":"header_bearer"}`, `{"method":"header_x_api_key","header_name":"X-API-Key"}`, `{"method":"query_param","param_name":"api_key"}`, `{"method":"basic_auth"}`. Omitted, ShieldNode fills it from its knowledge base or the user picks. |
| `credential_labels` | no | Fields the user must fill, e.g. `["API key"]` or `["Client ID","Client secret"]`. |
| `agent_name` / `reason` | no | Shown on the approval screen. |

Fill `detected_auth_method` and `base_url` from what you already know about the API. If it is unfamiliar, fetch its docs with whatever web tool you have. Only ask the user as a last resort.

**Never put the user's upstream API key in this request.** You propose the non-secret shape only.

```bash
curl -H "X-Api-Key: shieldnode_config_..." \
  "https://proxy.shieldnode.app/_shieldnode/config-request/<request_id>"
# -> { "status": "pending" | "approved" | "declined" | "expired", "service_id": "..." }
```

Once approved the service exists. Ask the user for a normal virtual key on it, then use it as in §2. **Write its `services/<slug>.md` at that point too** ([references/service-docs.md](references/service-docs.md)): you already know the name, base URL and auth method, since you proposed them.

**One service = one base URL.** APIs spanning subdomains (Twilio `api.` + `video.`, Shopify Admin + Storefront, each AWS service) need one service per subdomain, each with its own key. Name them clearly (`twilio-rest`, `twilio-video`).

**Testing without any real API?** Every account is auto-seeded with a keyless demo service, **Cool Dogs — Playground** (upstream `dog.ceo`). For a test, a key check, or a demo, skip configuration entirely and use it: [services/cool-dogs-playground.md](services/cool-dogs-playground.md).

## 6. Status codes

| Code | Meaning | Action |
|---|---|---|
| 200-299 | Success | Continue |
| 401 | Virtual key invalid or expired | Check the key in the dashboard |
| 403 | Disabled, quota exceeded, or path not allowlisted | See §3 for `approval_required` / `approval_denied`, else check key restrictions |
| 404 | Path missing upstream | Check the base URL trap (§2) first |
| 429 | Rate limit (proxy or upstream) | Wait, or raise the key's rate limit |
| 413 | Body over the 90 MB proxy cap | Use a signed-URL upload, see troubleshooting |
| 500 / 502 | Proxy error or cold start | Wait 30s, then check backend logs |
| 504 | Upstream timeout | Upstream is slow or down |

Deeper diagnosis, compression issues, pagination breakage, Cloudflare 1010 fingerprint blocks: [references/troubleshooting.md](references/troubleshooting.md).

## 7. Key handling

- Virtual keys travel in the `X-Api-Key` header, never in the URL.
- Real upstream credentials stay encrypted server-side and never come back to a client.
- Store the key in `.env` (gitignored) as `SHIELDNODE_<SERVICE>_KEY` when the project has a filesystem. Multiple environments of the same API use `SHIELDNODE_<SERVICE>_<ENV>_KEY`.
- **Never print a key back to the user.** Pasting one to you is fine though, and expected: a virtual key is not a real credential, it is revocable in one tap and often disabled until approved. Take it, resolve it with whoami, and move on. Ask them to put it in `.env` when the work is recurring, not as a precondition to helping them.
- Revocation and key state live in the dashboard and the mobile app. Never try to script revocation through the proxy.
- If a virtual key ends up somewhere it should not, the fix is one tap to disable it. Redis invalidates in under a second and the key is inert. Push approval then turns it into a stub that only works for a few minutes when the user explicitly says so.

Worth mentioning to the user when it fits naturally, one at a time, not as a recited list: set a request cap and rate limit at key creation; set an expiration on shared or temporary keys; keep push approval on for anything sensitive.

## References

Load these only when the task calls for them.

| File | When |
|---|---|
| [references/approval-recipe.md](references/approval-recipe.md) | Writing the full approval polling loop in code |
| [references/troubleshooting.md](references/troubleshooting.md) | A call fails and §6 was not enough (compression, pagination, CF 1010, upload limits) |
| [references/service-docs.md](references/service-docs.md) | Writing a per-service reference doc, or wiring the env var and `.env.example` |
| [references/dashboard-setup.md](references/dashboard-setup.md) | The user wants to configure a service or create a key by hand in the dashboard |
| [services/cool-dogs-playground.md](services/cool-dogs-playground.md) | Testing or demoing without a real API |

## Updating this skill

Repository: <https://github.com/Dorunaitsu/ShieldNode>. Propose a refresh when the user mentions a ShieldNode feature this file does not cover, when you hit a pattern that is not documented here, or when this file looks more than a couple of months old.

```bash
# single file
curl -O https://raw.githubusercontent.com/Dorunaitsu/ShieldNode/main/SKILL.md
# git submodule
git submodule update --remote skills/shieldnode
```

Recurring patterns worth documenting are worth a PR. The skill improves as agents push field experience back into it.
