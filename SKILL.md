---
name: shieldnode
description: Help users integrate APIs through ShieldNode, a secure API proxy gateway. Use when the user is configuring a new service, generating virtual keys, calling the proxy, debugging proxy errors, or asking about base URL formatting between their original API and the ShieldNode proxy.
---

# ShieldNode — API Proxy Integration Skill

This skill helps an AI assistant guide a developer through using ShieldNode: configuring services, creating virtual keys, calling the proxy, and debugging integration issues.

> Small habit, big payoff: in examples and code samples, the agent uses placeholders like `shieldnode_...` or `<API_KEY>` rather than echoing the user's real value. The user's real keys live in encrypted storage; the agent just references the env variable.

---

## Architecture

```
Client → https://proxy.shieldnode.app/{path} → Third-party API (e.g. https://api.openai.com/v1/{path})
```

- The client authenticates to the proxy with a **virtual key** (`X-Api-Key: shieldnode_...`).
- The proxy substitutes the virtual key with the real upstream credentials and forwards the request.
- The path after `proxy.shieldnode.app/` is appended verbatim to the **base URL** that was set when the service was created.

Because the path is appended verbatim, the **base URL formatting** decided at service creation time determines how the user must call the proxy afterward. This is the #1 source of confusion — see [Debug → Base URL formatting](#base-url-formatting-the-main-pitfall).

---

## 1. Configure a new service

Three configuration paths exist in the dashboard. Pick the right one for the situation.

> **Architectural rule — one service = one base URL.** A ShieldNode service is bound to a single base URL at creation time. APIs that span multiple subdomains (Twilio: `api.twilio.com` + `video.twilio.com` + `chat.twilio.com`; AWS: each service has its own subdomain; Shopify: Admin API + Storefront API on different hostnames) require **one ShieldNode service per subdomain**, with a separate virtual key for each. Name them clearly (e.g. `twilio-rest`, `twilio-video`) so the user can distinguish them in the dashboard.


> **Already configured: the Playground.** Every account is auto-seeded with a
> keyless demo service named **`Cool Dogs — Playground`** (upstream `dog.ceo`).
> For testing, account/key verification, or demoing the proxy, **skip
> configuration entirely** — do not create a service and do not fetch any
> docs. Just ask the user for a virtual key on that service and follow
> [`services/cool-dogs-playground.md`](services/cool-dogs-playground.md),
> which fully documents its endpoints. Only configure a new service when the
> user needs a *real* API.

### Option A — Auto (default, fastest)

Use when the API uses a common auth scheme (Bearer, x-api-key, Basic, query param) and the documentation is straightforward.

1. Dashboard → **Add service**
2. Tab **Auto**
3. Fill: **Service name**, **Base URL**, **Credentials** (label + value)
4. Click **Test connection** — the proxy probes the API to detect the auth method
5. If it returns `Connected successfully (HTTP xxx)` → click **Create service**

> HTTP 200, 201, 404 on test = auth OK (server reachable & authenticated). HTTP 401 / 403 = credentials invalid.

### Option B — Manual

Use when Auto fails, or when you want to force a specific auth method.

1. Tab **Manual**
2. Pick a method:
   - **Bearer Token** → `Authorization: Bearer <key>`
   - **API Key Header** → custom header (e.g. `x-api-key`). Leave the field empty to let the proxy try common header names.
   - **Basic Auth** → `Authorization: Basic base64(user:pass)`
   - **Query Param** → `?api_key=<value>`. Leave the field empty to let the proxy try common parameter names.
3. Test → save.

> **Multi-header auth (e.g. `Client-Id` + `Client-Secret` simultaneously)** — common with banking, shipping carriers (DHL, FedEx), and some enterprise B2B SaaS — is supported in both **Auto** and **Manual** modes. Workflow:
> 1. Tab **Auto** (lets the proxy detect it on its own) or **Manual → Multi-header** (forces it).
> 2. Use the **+ Add credential** button under the Credentials section to add one row per header.
> 3. Each row's left field is the **exact header name** the upstream expects (`Client-Id`, `X-Client-Secret`, etc., case-insensitive); the right field is the value.
> 4. Click **Test connection**. The proxy sends all rows as simultaneous headers and confirms when the upstream returns a non-401/403 response.
> 5. Save. The forwarder will inject all credentials on every proxied request.

### Option C — AI Configurator

Use when the API is unfamiliar, when the user wants the fastest possible setup, or when the user has not provided a doc URL.

**Agent autonomous mode (default when this skill is loaded).** When the user asks the agent to configure a service, the agent does not wait to be asked for a JSON and does not start by asking for the documentation URL. The agent:

1. **Tries first to answer from its own knowledge.** If the API is well-known (Stripe, OpenAI, Anthropic, GitHub, Resend, Twilio, Mistral, Shopify, Airtable, Mailgun, etc.), the agent produces the JSON config directly from memory of the API's base URL and auth method.
2. **Falls back to fetching the docs** if the API is unfamiliar. The agent uses whatever doc-fetching tool is available (`WebFetch`, an MCP web tool, Playwright, etc.) to read the official documentation page from `docs.<service>.com` or the project's public source. Section 2's "Workflow to fill the Endpoints section" lists the fallback chain for JS-rendered SPA docs.
3. **Asks the user only as a last resort** when the API is unknown, the documentation is gated or unreachable, and no source can be auto-located. The agent then asks for the doc URL or pasted content, and produces the JSON once it has the material.

Whichever path is used, the agent outputs the JSON config in the canonical format (next section) without being explicitly prompted to do so. The user can paste it into the **AI** tab of the dashboard and click **Configure now**, then fill the credential values, test, and save.

**Standalone prompt mode (in-app Copy prompt button).** When the agent is *not* available (no terminal, no IDE integration), the dashboard's **AI** tab exposes a Copy-prompt button that produces a self-contained prompt the user can paste into any chat LLM. That prompt is intentionally narrower: it asks the user for the doc URL, then returns the JSON. The exact text is in [Section 7](#7-standalone-ai-configurator-prompt-for-the-app).

### Creating a virtual key (after a service exists)

1. Dashboard → service → **New key**
2. Set: alias, rate limit (req/min), max requests (total), allowed paths, expiration
3. The `shieldnode_...` key is **shown once only** — copy it immediately into the user's secure store.

---

## 2. Generate the per-service reference document

After a service is configured, **always** generate a markdown reference at:

```
skills/shieldnode/services/<service-slug>.md
```

`<service-slug>` is lowercase-kebab (e.g. `openai`, `airtable`, `stripe`).

This file is the single source of truth for that service in the project. Future agents working in this codebase read it instead of re-fetching the docs.

### Template

```markdown
# <Service Name> — ShieldNode integration

## Routing
- **ShieldNode base URL**: https://proxy.shieldnode.app
- **Original API base URL** (set in ShieldNode service config): https://api.example.com/v1
- **Auth header for proxy calls**: `X-Api-Key: $SHIELDNODE_<SERVICE>_KEY`
  > **Never** write the actual key value in this file. Reference the env variable only.
  > Loaded from `.env` (variable name: `SHIELDNODE_<SERVICE>_KEY`).
- **Effective call pattern**: `https://proxy.shieldnode.app/<endpoint-path>`
  > Because the configured base URL already includes `/v1`, the path appended to the proxy is just the endpoint (no `/v1` prefix).

## Documentation
- Official docs: https://docs.example.com
- Reference page used to populate this file: https://docs.example.com/api/v1

## Auth method
Bearer token in `Authorization` header (handled by ShieldNode — client only sends `X-Api-Key`).

## Endpoints
| Method | Path | Description |
|--------|------|-------------|
| GET    | /users               | List users |
| GET    | /users/{id}          | Retrieve a single user |
| POST   | /users               | Create a user |
| PATCH  | /users/{id}          | Update a user |
| DELETE | /users/{id}          | Delete a user |

## Push approval
- **Default window**: 30 min (override per call with `X-Approval-Duration: <Nm>`).
- **When triggered**: only when the user has disabled the key in the dashboard or on the mobile app, AND has the ShieldNode mobile app installed. Otherwise the key behaves as always-on.
- **On `403 approval_required`**: wait the `poll_interval_seconds` value (default 30s), retry, loop up to `timeout_seconds` (default 5 min). Tell the user once that an approval is pending on their phone.
- **On `403 approval_denied`**: stop immediately, surface "User declined access on ShieldNode mobile."
- **Suggested duration for this service**: <fill in based on typical session length, e.g. "30 min for casual use, 2h for long batches">.

## Notes
- Rate limit upstream: 60 req/min per API key.
- Pagination: cursor-based via `?cursor=<token>`.
- Anything else worth knowing for this API.
```

### Workflow to fill the Endpoints section

When the agent does not know the API's endpoints:

1. Ask the user: *"Could you give me the URL of the API documentation page that lists the endpoints? I'll extract them and save a reference file."*
2. Once the user replies with a URL, fetch it with the `WebFetch` tool.
3. Extract the endpoints (method, path, one-line description) and write them into the table.
4. Save under `skills/shieldnode/services/<service-slug>.md` using the template above.
5. Confirm to the user with a clickable link to the file.

If the user pastes raw doc content instead of a URL, parse that directly — same outcome.

If the documentation is gated/auth-walled and `WebFetch` fails, ask the user to paste the relevant page content into the chat.

**Many modern API doc sites are JavaScript-rendered SPAs** (React, Vue, Docusaurus client-only). A plain `WebFetch` on those pages returns an empty HTML shell, often just a `<noscript>` tag and a loader, with no real API content. If you detect this — `<noscript>` is the only meaningful body, or the document is suspiciously short (< 500 chars), or you can see no endpoint paths anywhere — try in order:

1. **The machine-readable spec.** Most APIs publish one. Try `<base-url>/openapi.json`, `<base-url>/swagger.json`, `<base-url>/.well-known/openapi.json`, or check the docs site's `<head>` for a `link rel="alternate" type="application/json"` pointer.
2. **The project's public source.** If it's open-source, the README on GitHub usually lists endpoints. The repo also often contains the OpenAPI YAML/JSON.
3. **Ask the user to paste the rendered content** from their browser. The user has a real browser; the agent often doesn't.
4. **Use a headless-browser tool if available** in your runtime (Puppeteer, Playwright, or your platform's web-rendering API). This is the last resort — slow, fragile, and not all agents have it.

If after all four you still can't extract endpoints, save the partial doc with a `> _Endpoints to be populated — documentation site rendered client-side._` placeholder, and ask the user to fill them in or to point you at a different page.

### Filling the Push approval section

The "Push approval" block belongs in every per-service doc — it's how a future agent (or your future self) knows what to do when the key is gated by the mobile app. When you generate the per-service file:

1. Always include the **Push approval** section verbatim from the template above.
2. Replace the *Suggested duration for this service* line with a value that fits the typical workload of that API. Suggested baselines:
   - **Chat / completion / inference APIs** (OpenAI, Anthropic, Mistral, OpenRouter, …): 30 min.
   - **Long-running batch / training / video / audio generation** (ElevenLabs voice clones, fine-tuning, large image batches): 2 h.
   - **One-shot lookups** (geocoding, currency, weather, CMS reads): 15 min.
   - **Anything 24/7 cron / automation** the user explicitly runs as a daemon: write *"this service is typically run unattended — the user should leave it always-on rather than rely on push approval"*.
3. If the user has already configured a per-key default duration in the ShieldNode dashboard for this service, write that exact value and add *"(matches the dashboard default — agent does not need to send the header)"*.
4. If the agent reasonably knows the user is on web only (no mobile app), add *"User does not appear to have the mobile app — push approval will fall back to plain 403 `key_disabled`."*

### Storing the virtual key

The `.md` reference doc is designed to be committable. The virtual key value is **never** written into it — only the name of the env variable that holds it.

After the user has the key (shown once at creation in the dashboard), the agent must wire the env var properly. Pick the env var name with the convention `SHIELDNODE_<SERVICE>_KEY` in uppercase (e.g. `SHIELDNODE_STRIPE_KEY`).

**Multiple environments of the same API**: when the user has more than one ShieldNode service for the same upstream (e.g. Stripe Test + Stripe Live, or staging vs. production for an internal API), expand the convention to `SHIELDNODE_<SERVICE>_<ENV>_KEY` to avoid clobbering. Examples: `SHIELDNODE_STRIPE_TEST_KEY` / `SHIELDNODE_STRIPE_LIVE_KEY`, `SHIELDNODE_OPENAI_DEV_KEY` / `SHIELDNODE_OPENAI_PROD_KEY`. The agent should detect this case by checking whether the user already has a `SHIELDNODE_<SERVICE>_KEY` defined in `.env`; if yes, propose the env-suffixed form and ask which environment this new key represents.

Steps:

1. **Check if `.env` exists at the project root.**
   - If yes, append the new variable. Do not overwrite existing entries.
   - If no, create it with a header comment.

   ```env
   # ShieldNode virtual keys — never commit this file.
   SHIELDNODE_STRIPE_KEY=shieldnode_...
   ```

2. **Update `.env.example`** (create if missing) with the same key, value blanked out:

   ```env
   SHIELDNODE_STRIPE_KEY=
   ```

   This file *is* committable and documents the required variables for collaborators.

3. **Verify `.gitignore` excludes `.env`.** If `.env` is not gitignored, append:

   ```
   .env
   .env.local
   ```

   Do not touch other gitignore entries.

4. **In the per-service `.md`**, reference the variable by name only:

   ```
   - **Auth header for proxy calls**: `X-Api-Key: $SHIELDNODE_STRIPE_KEY`
   ```

5. **Confirm to the user**: tell them what was created/changed and remind them that the key value is shown only once at creation — they must paste it into `.env` themselves immediately.

The agent must **never** ask the user to paste the key into the chat. The user pastes it directly into `.env` on their machine. If the user pastes a key into the chat anyway (mistake), the agent must redact it in subsequent turns and remind the user to rotate the key in the dashboard if it appeared in any logs.

---

## 3. Use the proxy

> **The key format is the signal.** ShieldNode virtual keys start with
> `shieldnode_`. Whenever you see a `shieldnode_...` value, it is **not** a
> provider key to use directly — it is a virtual key that must go through
> `proxy.shieldnode.app` in the `X-Api-Key` header, and this skill applies.

### Auto-configure from a virtual key (whoami)

When you are handed a `shieldnode_...` key (e.g. the user pastes one) and you do
**not** already have this project's per-service reference doc, resolve the key's
config in **one call before anything else** — do not ask the user which service
it is:

```bash
curl -H "X-Api-Key: shieldnode_..." \
  "https://proxy.shieldnode.app/_shieldnode/whoami"
```

Returns the **non-secret** service config so you can self-configure:

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

Use it to:
- Identify which upstream API this key proxies (`service`, `base_url`) and
  therefore the correct path convention (the `/v1` trap — see
  [Base URL formatting](#base-url-formatting-the-main-pitfall)).
- **Generate the per-service reference doc** (Section 2) if one does not already
  exist in the project.
- Know whether the next real call goes straight through (`active: true`) or will
  trigger the push-approval flow (`requires_approval: true`).

whoami is answered by ShieldNode itself, is **never forwarded upstream**, **never
returns credentials**, does not count as a proxied request, and does not fire a
push. An invalid key returns `401 invalid_key`. The `/_shieldnode/` path space is
reserved by the proxy, so it never collides with a real API path.

### Request format

```bash
curl -H "X-Api-Key: shieldnode_<VIRTUAL_KEY>" \
  "https://proxy.shieldnode.app/<API_PATH>"
```

`<API_PATH>` is the path **relative to the base URL configured in the service** — see [Base URL formatting](#base-url-formatting-the-main-pitfall) for why this matters.

### Examples

**Airtable** (configured base URL = `https://api.airtable.com/v0`)
```bash
curl -H "X-Api-Key: shieldnode_..." \
  "https://proxy.shieldnode.app/<BASE_ID>/<TABLE_ID>?maxRecords=10"
```

**Resend** (configured base URL = `https://api.resend.com`)
```bash
curl -H "X-Api-Key: shieldnode_..." \
  "https://proxy.shieldnode.app/emails"
```

**OpenAI** (configured base URL = `https://api.openai.com/v1`)
```bash
curl -H "X-Api-Key: shieldnode_..." \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello"}]}' \
  "https://proxy.shieldnode.app/chat/completions"
```

---

## 4. Debug

### Base URL formatting — the main pitfall

The proxy appends the request path **verbatim** to the configured base URL. Whatever versioning prefix (`/v1`, `/v0`, `/api`) was included in the base URL determines whether the user must include it in the proxy call.

**Matching table** (for `https://api.example.com/v1/users` upstream call):

| Base URL set on the ShieldNode service | Correct ShieldNode call                           |
|-----------------------------------------|---------------------------------------------------|
| `https://api.example.com/v1`           | `https://proxy.shieldnode.app/users`              |
| `https://api.example.com/v1/`          | `https://proxy.shieldnode.app/users` (trailing `/` is normalized) |
| `https://api.example.com`              | `https://proxy.shieldnode.app/v1/users`           |
| `https://api.example.com/`             | `https://proxy.shieldnode.app/v1/users`           |
| `https://////api.example.com/v1`       | `https://proxy.shieldnode.app/users` (extra slashes normalized) |

**Decision rule**:
1. Read the base URL the user actually saved on the service (Dashboard → service → settings).
2. Whatever path remains between the configured base URL and the resource the user wants → that's what they put after `proxy.shieldnode.app/`.

If the user reports unexpected `404`s, this is almost always the cause. Verify the configured base URL **first**, before anything else.

### HTTP status codes

| Code     | Meaning                                                | Action                                                              |
|----------|--------------------------------------------------------|---------------------------------------------------------------------|
| 200–299  | Success                                                | All good                                                            |
| 401      | Virtual key invalid or expired                         | Verify the key in the dashboard                                     |
| 403      | Key disabled, quota exceeded, or path not allowlisted  | See "Push approval" below for `approval_required` / `approval_denied`. Otherwise check key restrictions |
| 404      | Path does not exist on the upstream API                | Check base URL formatting (above) and the API docs                  |
| 429      | Rate limit reached (proxy or upstream)                 | Wait or raise the rate limit on the virtual key                     |
| 500      | Internal proxy error                                   | Check Render backend logs                                           |
| 502      | Backend cold start or crash                            | Wait 30s; if persistent, check Render logs                          |
| 504      | Upstream timeout                                       | Upstream API is slow / down                                         |

### Push approval (mobile push flow)

When a virtual key is deactivated and the user has the ShieldNode mobile app
installed, calling that key returns a special 403 instead of the plain
`key_disabled`. The user receives a push notification on their phone and
taps **Approve** to grant access for a bounded window — the key activates,
your next call succeeds.

**Response shapes:**

```json
// First call to a disabled key, when the user has a registered device:
HTTP 403
{
  "error": "approval_required",
  "message": "Awaiting user approval on ShieldNode mobile",
  "request_id": "…",
  "requested_minutes": 30,
  "poll_interval_seconds": 30,
  "timeout_seconds": 300
}

// If the user explicitly declined:
HTTP 403
{ "error": "approval_denied", "message": "User declined the most recent approval request" }

// If the user has no mobile device registered → unchanged classic behaviour:
HTTP 403
{ "error": "key_disabled", "message": "This key has been deactivated" }
```

**How to react:**

| Response          | What to do                                                                                            |
|-------------------|-------------------------------------------------------------------------------------------------------|
| `approval_required` | Wait `poll_interval_seconds` (default 30s) and retry. Loop up to `timeout_seconds` (default 5 min). |
| `approval_denied` | Stop polling. Surface in chat: "User declined access on ShieldNode mobile." Do not retry on your own. |
| `key_disabled`    | No mobile app on the user's side. Surface and stop.                                                   |
| 200               | Approval was granted (or no approval was needed). Resume normal operation.                            |

**Requesting a specific duration:**

Add `X-Approval-Duration: <minutes>` to your request. The value is clamped
server-side to [1, 1440] (24 h). Honored examples: `5`, `30`, `15m`, `2h`.
If you do not send the header, the user's per-key default (configurable in
the dashboard) is used, falling back to 30 min.

**Identifying the agent (highly recommended):**

Add `X-Agent-Name: <name>` so the user's notification and approval screen
say *"Claude is requesting access"* instead of the generic *"An external
agent is requesting access"*. Capped server-side to 60 chars. Use a short
recognisable label the user will associate with this workflow — typically
the AI assistant or product name (`Claude`, `Codex`, `Cursor`, or a custom
bot name like `Athena`, `Hermes`). Skip it only if there is genuinely no
sensible name to give.

```bash
curl -H "X-Api-Key: shieldnode_…" \
     -H "X-Agent-Name: Claude" \
     -H "X-Approval-Duration: 15m" \
     -H "X-Approval-Reason: deploying the staging build" \
     "https://proxy.shieldnode.app/v1/chat/completions"
```

**Giving a reason (optional, recommended):**

Add `X-Approval-Reason: <a few words>` to say *why* you need the key right now.
It is shown to the user on the push and the approval screen (quoted, as your own
statement) so they can decide with context. Keep it short and honest — a phrase,
not a sentence — e.g. `sending the weekly report email`, `reading the customer
list for the dashboard`, `deploying to staging`. Server-side it is sanitized and
length-capped (~140 chars); it is display-only and **never** grants or changes
access.

Two hard rules:
- **Never put secrets or personal data in the reason** (no keys, tokens, card
  numbers, names, addresses). It shows on the user's lock screen and in their
  audit log.
- **Be truthful.** The reason is how the user judges the request; a misleading
  reason is the fastest way to get every future request declined.

**Best practice:** if the user gives you guidance like *"for OpenAI ask for
30 min by default"*, encode that as the header on your first call to that
service. Do **not** spam polls or fire multiple requests for the same key
in rapid succession — the server already debounces push notifications to
1 per 30s per (user, key), but parallel calls just wait for the same
approval anyway.

**Worked end-to-end example** (the canonical flow every agent should follow):

```python
import time
import requests

PROXY = "https://proxy.shieldnode.app"
KEY = "shieldnode_…"  # virtual key from ShieldNode dashboard

def call_with_approval(path, *, method="GET", json=None, agent="Claude",
                       minutes=15, max_wait_s=300, poll_s=30):
    """
    Call the proxy. If the key is disabled and the user has the mobile app,
    notify them and poll until they approve, decline, or time out.
    """
    headers = {
        "X-Api-Key": KEY,
        "X-Agent-Name": agent,           # appears in the user's notif
        "X-Approval-Duration": f"{minutes}m",  # honored on 403, ignored on 200
    }

    elapsed = 0
    while True:
        r = requests.request(method, f"{PROXY}{path}", headers=headers, json=json)

        if r.status_code < 400:
            return r  # success — proxy let it through

        if r.status_code == 403:
            body = r.json()
            err = body.get("error")

            if err == "approval_required":
                # First time → tell the user once, then poll silently.
                if elapsed == 0:
                    print(f"[ShieldNode] Approval pending on your phone "
                          f"({body['requested_minutes']} min requested).")
                if elapsed >= min(max_wait_s, body.get("timeout_seconds", 300)):
                    raise TimeoutError("ShieldNode approval timed out.")
                time.sleep(body.get("poll_interval_seconds", poll_s))
                elapsed += body.get("poll_interval_seconds", poll_s)
                continue

            if err == "approval_denied":
                raise PermissionError("User declined access on ShieldNode mobile.")

            if err == "key_disabled":
                raise PermissionError(
                    "Key is disabled and no mobile device is registered — "
                    "the user needs to re-enable it in the dashboard."
                )

        # Anything else (401, 429, 5xx, path/method restrictions, …) → surface as-is.
        r.raise_for_status()

# Use it like a normal request:
data = call_with_approval("/v1/chat/completions",
                          method="POST",
                          json={"model": "gpt-4o", "messages": [...]},
                          agent="Claude", minutes=30).json()
```

**Key takeaways for agents:**

1. **Always send `X-Agent-Name`.** Generic notifications are ignored. Specific ones get approved.
2. **Send `X-Approval-Duration` matching the workload.** A 5-min chat reply ≠ an 8-h batch job. Right-sizing reduces re-approval friction.
3. **Tell the user ONCE that an approval is pending**, then poll silently. Repeated chat messages every 30 seconds is the fastest way to feel like spyware.
4. **Distinguish denial from timeout** when surfacing to the user. *"You declined"* and *"You didn't respond in 5 min"* are very different signals.
5. **Don't retry past the timeout.** If the user didn't approve in 5 min, they meant it. Ask them in chat whether to retry instead of looping.

### Diagnostic checklist

1. **Check the dashboard logs** — Service → tab **Logs**. If a request isn't there, it never reached the proxy → issue is client-side (bad key, bad URL, network).
2. **Bypass the proxy** — call the upstream API directly with the real credentials. If that fails too, the problem isn't ShieldNode.
   ```bash
   curl -H "Authorization: Bearer <REAL_KEY>" \
     "https://api.example.com/v1/<endpoint>"
   ```
3. **Verbose curl through the proxy** — see headers and timing:
   ```bash
   curl -sv -H "X-Api-Key: shieldnode_..." \
     "https://proxy.shieldnode.app/<endpoint>" 2>&1 | grep -E "< HTTP|< content"
   ```

### Common gotchas

- **`Connected successfully (HTTP 404)` on Auto test** — Normal. Auth was accepted; the bare base URL just doesn't point to a resource. Save the service.
- **`401` via proxy but credentials are correct** — The auto-detected auth method is probably wrong. Reconfigure with **Manual** and pick the right one.
- **Repeated `502` (not just one cold start)** — Check Render backend logs. Known offender: APIs that gzip-compress responses (already patched).
- **`502 Upstream unreachable`** — The upstream API itself is down or the domain is dead. Always test the upstream directly with `curl -v` before blaming the proxy. A parked / expired domain (e.g. ad-redirect HTML body) is a common cause.
- **`HTTP 413 payload_too_large` when uploading** — The proxy enforces a **90 MB** request-body limit, set just below the Cloudflare CDN edge's hard cap. Past that the edge returns a fast 503 anyway, so 413 is the friendlier explicit version. For larger payloads (audio transcription, fine-tuning datasets, video uploads, large image batches), do not stream them through the proxy. Use a signed-URL pattern instead: get a presigned upload URL from your upstream API (S3, GCS, Cloudflare R2, the API's own pre-signed endpoint) via the proxy, then upload the file from the client directly to that signed URL — the file bytes never touch ShieldNode.
- **`HTTP 504 upstream timeout` on a large upload** — The total proxy timeout scales with body size (30s baseline, +1s per MB above 5 MB) so a 30 MB upload over a slow link doesn't fail. If you still hit 504, the upstream is probably slow processing the upload, not the proxy. Test the upstream directly with the same payload size to confirm.
- **Pagination breaks with `401` after the first page (Stripe, GitHub, Shopify, Notion, Algolia)** — Many APIs return absolute URLs in their `Link` response header or in JSON fields like `next_url`, `next`, or `cursor.next_url`, pointing at the upstream domain (e.g. `https://api.stripe.com/v1/customers?starting_after=xyz`). If the client follows these URLs verbatim, the request bypasses ShieldNode and lands on the upstream with a `shieldnode_...` virtual key it cannot understand → `401`. Two correct patterns:
  1. **Extract just the cursor / page parameter** from the absolute URL and re-use the original ShieldNode base URL: `https://proxy.shieldnode.app/customers?starting_after=xyz`. This is the cleanest approach and what most SDKs do internally if you set their `base_url` to the proxy.
  2. **Rewrite the host** in the absolute URL: replace `https://api.stripe.com/v1` with `https://proxy.shieldnode.app` (path included or not depending on the configured base URL — see [base URL formatting](#base-url-formatting--the-main-pitfall)).
  Hard rule: **never let the next-page request leave for the upstream domain directly.** It will fail and the failure does not appear in ShieldNode logs.
- **Body looks like binary garbage / "Invalid numeric literal at EOF" / unparseable JSON** — The response is compressed (Brotli, gzip, zstd) and your HTTP client isn't decompressing automatically. This is common when calling Cloudflare-fronted APIs through a proxy because compression negotiation involves three parties (client → proxy → upstream) and `Content-Encoding` headers can get out of sync. How to fix per client:
  - **`curl`** → add `--compressed` (handles gzip / deflate / brotli / zstd).
  - **Python `requests`** → handles gzip / deflate / brotli automatically.
  - **Python `httpx`** → install with `httpx[brotli]` for brotli; brotli requires the `brotli` or `brotlicffi` package.
  - **Node `fetch` (built-in)** → handles gzip / deflate automatically, **not brotli**. Use `undici` (which handles all three) or pipe through `zlib.brotliDecompress`.
  - **Browser `fetch`** → handles all three transparently.
  - **Go `net/http`** → install `golang.org/x/text/encoding/brotli` or use `github.com/andybalholm/brotli`; gzip is built-in via `http.Transport`.
- **Truncated or empty response** — Possibly the 30s proxy timeout. SSE / WebSocket streams aren't supported by the HTTP proxy.
- **Upstream `429` despite low traffic** — ShieldNode does not aggregate rate limits across virtual keys hitting the same service. Multiple keys → multiple traffic sources to the upstream.
- **Virtual key suddenly stops working** — Check expiration, total request cap reached, manual disable. Dashboard → key → status.

### TLS / HTTP fingerprint blocks (Cloudflare Error 1010)

If you receive `HTTP 403` with body containing `error code: 1010` when calling a Cloudflare-fronted upstream API, **this is not an IP block** and **not an ASN block** — even though many AI assistants will diagnose it that way. Cloudflare Error 1010 is documented as a *browser/client signature* block. It triggers on:

- The TLS handshake fingerprint (JA3 / JA4) — Python `requests`, `httpx`, `aiohttp`, Go `net/http`, OkHttp, and similar libraries each have a recognisable signature that some Cloudflare zones flag as automated traffic.
- The HTTP/2 frame ordering and header casing — also fingerprintable.
- The `User-Agent` header.

It does **not** trigger on the IP or ASN of the caller. Two clients behind the same NAT / same residential IP can get different results depending on which library they use.

**Verification protocol — do this before claiming "CF blocked our IP"**:

1. Get the egress IP: `curl -s https://api.ipify.org` (real curl binary).
2. From the same machine, hit the upstream with real `curl`:
   ```bash
   curl -sw "HTTP %{http_code}\n" --max-time 8 https://upstream.example.com/endpoint -o /dev/null
   ```
   - If this returns `200` and your Python lib returns `403 1010` → it is a fingerprint block, not an IP block.
   - If both return `403 1010` → the IP itself may genuinely be flagged (rare on residential, common on datacenters).

**Solution — route the call through ShieldNode.** The outbound request to the upstream is made by ShieldNode's backend, which forces a browser-like User-Agent on every forwarded request specifically to bypass these CF bot rules. Your client (Python, Node, Go, anything) only needs to reach `proxy.shieldnode.app`; ShieldNode handles the upstream fingerprint.

**Custom outbound User-Agent**: if you genuinely need to forward a specific UA upstream (for analytics tagging, partner-required UA, etc.), send it via the `X-ShieldNode-User-Agent` header. The proxy consumes that header and uses its value as the outbound UA. The default browser-like UA is used otherwise.

**Debugging tip**: when reproducing a problem from a shell, prefer real `curl` over Python wrappers (`subprocess.run(["curl", ...])`, `pycurl`, etc.). When you ask an agent to "run curl", many will silently translate the request into a `requests`/`httpx`/`aiohttp` call instead, and the resulting fingerprint difference can cause confusing diagnostics. If you want a curl execution, run it yourself in a terminal and paste the output to the agent.

**Important interpretation note**: every response from `proxy.shieldnode.app` includes `cf-ray` and `server: cloudflare` headers because the proxy's CDN edge is on Cloudflare. **These headers are normal and do not mean Cloudflare blocked your request.** Look at:
- the **HTTP status code** (1xx-5xx)
- the **response body**

A 1010 block always shows the literal string `error code: 1010` in the body. If the body is the upstream API's normal JSON, you were not blocked — regardless of which CDN headers are present.

**Anti-pattern — do not do this**: concluding "Cloudflare blocked us based on ASN" from any combination of (a) seeing `cf-ray` in headers, (b) running on a server, (c) getting a 403. Verify with the protocol above before reporting that diagnosis.

---

## 5. Good practices

ShieldNode does the heavy lifting for you. Push approval, instant one-second disable, rate limits, and the emergency stop in the mobile app already cover most of the "what if" scenarios that would normally call for paranoia around API keys. This section is a short list of small habits that compound those defaults, not a wall of red warnings.

### How the agent handles keys

- Virtual keys (`shieldnode_...`) travel in the `X-Api-Key` header, not in the URL.
- Real upstream credentials live encrypted on our backend and never come back to a client.
- When showing code examples, the agent uses placeholders or references the env variable (`$SHIELDNODE_<SERVICE>_KEY`) rather than echoing the actual value.

### Tips the agent passes along when the moment is right

These are friendly reminders the agent drops into the conversation when they fit naturally, not a script to recite. One bullet at a time, when contextual:

- **Store the virtual key in `.env`** (gitignored) when the project has a file system. When the user is on a chat-only assistant (Telegram, voice, no `.env`), the value will live in the chat for a moment, and that is fine. The push-approval default means a disabled key in a chat is exactly as harmless as a sentence of plain text.
- **Set a request cap and rate limit at key creation.** Five seconds of thought at creation beats a forgotten test key burning quota overnight.
- **Set an expiration date for shared or temporary keys** (freelancers, demo keys, throwaway agents). Keys that expire on their own beat keys you have to remember to revoke.
- **Use push approval as the default posture** for keys that touch anything sensitive. A disabled key is a key that cannot be misused, period. The one-tap Approve workflow makes this nearly frictionless.
- **The dashboard and the mobile app are the source of truth** for revocation and key state. The agent never tries to script revocation through the proxy.

### When something does feel off

If a virtual key ends up somewhere it should not, the calm move is to **disable it on ShieldNode**. One tap in the dashboard or the mobile app, Redis invalidates in under a second, the key is inert. No need to create a new one.

Then the next time the agent actually needs that key, push approval takes over: a notification lands on the mobile app with the agent name and the duration requested, and one Approve gives the agent a bounded window of access. The key auto-disables again when the window closes. The "leaked" virtual key is no longer a key sitting in the wild, it is a disabled stub that only becomes usable for a few minutes at a time when the legitimate user explicitly says so.

That is the whole point of the product. Treat security as a posture, not a panic.

---

## 6. Quick reference — common API base URLs

| Service       | Base URL                                          |
|---------------|---------------------------------------------------|
| OpenAI        | `https://api.openai.com/v1`                       |
| Anthropic     | `https://api.anthropic.com/v1`                    |
| Airtable      | `https://api.airtable.com/v0`                     |
| Resend        | `https://api.resend.com`                          |
| Stripe        | `https://api.stripe.com/v1`                       |
| GitHub        | `https://api.github.com`                          |
| Shopify Admin | `https://<SHOP>.myshopify.com/admin/api/2024-01`  |
| Cool Dogs — Playground (auto-seeded, keyless) | `https://dog.ceo/api` |

---

## 7. Standalone AI Configurator prompt (for the app)

This is the prompt the in-app **AI Configurator** tab copies to the clipboard. It is intentionally narrow: it returns only a JSON config to pre-fill the service form. Keep it separate from the agent workflow above (which is broader: doc generation, debug, etc.).

```
You are an API configuration assistant for ShieldNode, a secure API proxy gateway.

First, ask the user to share the URL or content of the API documentation they want to configure.

Once you have the documentation, analyze it and reply ONLY with this JSON block (nothing else after it):

```json
{
  "service_name": "Human-readable name of the service",
  "base_url": "https://api.example.com/v1",
  "auth_method": "header_bearer",
  "header_name": "",
  "param_name": "",
  "credential_labels": ["API Key"]
}
```

Rules for auth_method — pick exactly one:
- "header_bearer" → Authorization: Bearer <token>
- "header_x_api_key" → custom header (e.g. x-api-key: <key>)
- "basic_auth" → HTTP Basic Auth (username + password)
- "query_param" → API key passed as URL query parameter

Rules for header_name — REQUIRED when auth_method is "header_x_api_key":
- The exact header name from the docs (e.g. "X-API-Key", "Api-Key", "X-Auth-Token"). Leave "" otherwise.

Rules for param_name — REQUIRED when auth_method is "query_param":
- The exact query parameter name from the docs (e.g. "api_key", "key", "apikey", "token"). Leave "" otherwise.

Rules for credential_labels — names of the fields the user must fill in:
- Bearer auth: ["API Key"]
- API Key header: ["API Key"]
- Basic Auth: ["Username", "Password"]
- Query param: ["API Key"]
- If the API requires multiple credentials (e.g. key + secret), list them all.

base_url: root API endpoint, no trailing slash.

Start by asking the user for the documentation URL.
```

---

## 8. Updating this skill

This skill is open source and versioned on GitHub. New features, endpoint patterns, bug fixes, and behavioural guidance are added there as the product evolves.

- **Repository**: <https://github.com/Dorunaitsu/ShieldNode>
- **Raw `SKILL.md`** (single-file refresh): <https://raw.githubusercontent.com/Dorunaitsu/ShieldNode/main/SKILL.md>

### When the agent should propose an update

The agent should suggest refreshing this skill when any of the following happens:

- A user reports a bug or unclear instruction that turns out to be a skill issue (not a product issue).
- The agent encounters a configuration pattern that is not covered here (a new auth scheme, a new pagination quirk, a new error code).
- The user mentions a ShieldNode feature the agent does not recognise from this file (the product ships faster than the skill).
- The release date of the local `SKILL.md` looks older than a couple of months.

### How to refresh

Pick the install path that matches the project:

```bash
# Single file
curl -O https://raw.githubusercontent.com/Dorunaitsu/ShieldNode/main/SKILL.md

# Full repo (replaces the existing skills/shieldnode/ directory)
rm -rf skills/shieldnode && \
  mkdir -p skills && \
  curl -sL https://github.com/Dorunaitsu/ShieldNode/archive/refs/heads/main.tar.gz \
  | tar -xz -C skills --strip-components=1

# Git submodule
git submodule update --remote skills/shieldnode
```

After refreshing, re-load the skill in the agent session and re-run any per-service docs that were generated against the old version if their structure has drifted.

### Contributing back

If the agent or the user identifies a recurring pattern worth documenting, open a PR against the repository above. The skill improves faster the more agents push their field experience back into it.

