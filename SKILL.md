---
name: shieldnode
description: Help users integrate APIs through ShieldNode, a secure API proxy gateway. Use when the user is configuring a new service, generating virtual keys, calling the proxy, debugging proxy errors, or asking about base URL formatting between their original API and the ShieldNode proxy.
---

# ShieldNode — API Proxy Integration Skill

This skill helps an AI assistant guide a developer through using ShieldNode: configuring services, creating virtual keys, calling the proxy, and debugging integration issues.

> **Security rule — never violate**: never log, display, or transmit user-provided secrets (API keys, tokens, virtual keys, passwords). When you generate examples, use placeholders like `sk_live_...` or `<API_KEY>`.

---

## Architecture

```
Client → https://proxy.shieldnode.app/{path} → Third-party API (e.g. https://api.openai.com/v1/{path})
```

- The client authenticates to the proxy with a **virtual key** (`X-Api-Key: sk_live_...`).
- The proxy substitutes the virtual key with the real upstream credentials and forwards the request.
- The path after `proxy.shieldnode.app/` is appended verbatim to the **base URL** that was set when the service was created.

Because the path is appended verbatim, the **base URL formatting** decided at service creation time determines how the user must call the proxy afterward. This is the #1 source of confusion — see [Debug → Base URL formatting](#base-url-formatting-the-main-pitfall).

---

## 1. Configure a new service

Three configuration paths exist in the dashboard. Pick the right one for the situation.

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

### Option C — AI Configurator

Use when the API is unfamiliar and the user has the doc URL.

1. Tab **AI** → click **Copy prompt**
2. Paste prompt into Claude/ChatGPT
3. The AI asks for the documentation URL → user provides it
4. The AI returns a structured JSON
5. Paste JSON back into the textarea → click **Configure now** → form auto-fills
6. User fills credential values → test → save

The exact prompt is in [Section 7](#7-standalone-ai-configurator-prompt-for-the-app).

### Creating a virtual key (after a service exists)

1. Dashboard → service → **New key**
2. Set: alias, rate limit (req/min), max budget (USD), allowed paths, expiration
3. The `sk_live_...` key is **shown once only** — copy it immediately into the user's secure store.

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

### Storing the virtual key

The `.md` reference doc is designed to be committable. The virtual key value is **never** written into it — only the name of the env variable that holds it.

After the user has the key (shown once at creation in the dashboard), the agent must wire the env var properly. Pick the env var name with the convention `SHIELDNODE_<SERVICE>_KEY` in uppercase (e.g. `SHIELDNODE_STRIPE_KEY`).

Steps:

1. **Check if `.env` exists at the project root.**
   - If yes, append the new variable. Do not overwrite existing entries.
   - If no, create it with a header comment.

   ```env
   # ShieldNode virtual keys — never commit this file.
   SHIELDNODE_STRIPE_KEY=sk_live_...
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

### Request format

```bash
curl -H "X-Api-Key: sk_live_<VIRTUAL_KEY>" \
  "https://proxy.shieldnode.app/<API_PATH>"
```

`<API_PATH>` is the path **relative to the base URL configured in the service** — see [Base URL formatting](#base-url-formatting-the-main-pitfall) for why this matters.

### Examples

**Airtable** (configured base URL = `https://api.airtable.com/v0`)
```bash
curl -H "X-Api-Key: sk_live_..." \
  "https://proxy.shieldnode.app/<BASE_ID>/<TABLE_ID>?maxRecords=10"
```

**Resend** (configured base URL = `https://api.resend.com`)
```bash
curl -H "X-Api-Key: sk_live_..." \
  "https://proxy.shieldnode.app/emails"
```

**OpenAI** (configured base URL = `https://api.openai.com/v1`)
```bash
curl -H "X-Api-Key: sk_live_..." \
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
| 403      | Key disabled, quota exceeded, or path not allowlisted  | Check key restrictions                                              |
| 404      | Path does not exist on the upstream API                | Check base URL formatting (above) and the API docs                  |
| 429      | Rate limit reached (proxy or upstream)                 | Wait or raise the rate limit on the virtual key                     |
| 500      | Internal proxy error                                   | Check Render backend logs                                           |
| 502      | Backend cold start or crash                            | Wait 30s; if persistent, check Render logs                          |
| 504      | Upstream timeout                                       | Upstream API is slow / down                                         |

### Diagnostic checklist

1. **Check the dashboard logs** — Service → tab **Logs**. If a request isn't there, it never reached the proxy → issue is client-side (bad key, bad URL, network).
2. **Bypass the proxy** — call the upstream API directly with the real credentials. If that fails too, the problem isn't ShieldNode.
   ```bash
   curl -H "Authorization: Bearer <REAL_KEY>" \
     "https://api.example.com/v1/<endpoint>"
   ```
3. **Verbose curl through the proxy** — see headers and timing:
   ```bash
   curl -sv -H "X-Api-Key: sk_live_..." \
     "https://proxy.shieldnode.app/<endpoint>" 2>&1 | grep -E "< HTTP|< content"
   ```

### Common gotchas

- **`Connected successfully (HTTP 404)` on Auto test** — Normal. Auth was accepted; the bare base URL just doesn't point to a resource. Save the service.
- **`401` via proxy but credentials are correct** — The auto-detected auth method is probably wrong. Reconfigure with **Manual** and pick the right one.
- **Repeated `502` (not just one cold start)** — Check Render backend logs. Known offender: APIs that gzip-compress responses (already patched).
- **`502 Upstream unreachable`** — The upstream API itself is down or the domain is dead. Always test the upstream directly with `curl -v` before blaming the proxy. A parked / expired domain (e.g. ad-redirect HTML body) is a common cause.
- **Truncated or empty response** — Possibly the 30s proxy timeout. SSE / WebSocket streams aren't supported by the HTTP proxy.
- **Upstream `429` despite low traffic** — ShieldNode does not aggregate rate-limit budgets across virtual keys hitting the same service. Multiple keys → multiple traffic sources to the upstream.
- **Virtual key suddenly stops working** — Check expiration, max budget reached, manual disable. Dashboard → key → status.

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

## 5. Security rules

### Rules the agent itself must follow
- Virtual keys (`sk_live_...`) travel **only** in the `X-Api-Key` header — never in the URL.
- Real upstream credentials are encrypted at rest and never returned to clients.
- A leaked virtual key → disable it in the dashboard (Redis invalidation < 1s).
- Never paste a virtual key into public code, public logs, or chat history. The agent must redact them when echoing user input.
- **The per-service `.md` doc is committable. The key value never goes in it.** Only the env var name is referenced. The actual value lives in `.env` (gitignored). See [Storing the virtual key](#storing-the-virtual-key).
- **Prefer not to ask the user to paste a key into chat history when there's a better channel.** `.env`, password managers, or platform secret stores are safer.
- **However, many agents are chat-only** (Telegram bots, web chat assistants, voice assistants, no-code automation) and have no file system to drop a `.env` into. For those, the user has no choice but to share the key in the conversation, and that is **not automatically a security incident**. In that case, do not jump to "rotate immediately". Instead, recommend the user constrain the key with ShieldNode's controls:
  - lower the rate limit (req/min) on the key,
  - cap the spending budget,
  - set a short expiration date,
  - **disable the key in the dashboard or mobile app the moment the chat session ends**.
- Rotation is the right response only when the chat history is genuinely public or shared beyond the trusted user (livestream, screen-shared call, forum, indexed log file). For a private one-on-one chat with the user's own agent, ShieldNode's revocation and rate-limit controls are the proportionate response.

### Reminders to surface to the user (when relevant)

When the agent finishes a setup, sees the user about to commit code, or detects a key that has been idle for a while, it should proactively pass along these reminders. They are short and direct — one or two sentences each, dropped into the conversation when contextual, not as an unsolicited security lecture.

- **Don't leave virtual keys in code, screenshots, public Notion pages, or pinned chat messages.** When the user has a file system: `.env` (gitignored), a password manager, or the hosting platform's secret store. When the user is on a chat-only agent (Telegram, voice, web chat with no file access): treat the key as disposable — see the bullets below.
- **Disable the virtual key on ShieldNode when you're not actively using it.** A key that's idle for hours doesn't need to stay live — toggle it off in the web dashboard (`shieldnode.app`) or the mobile app, and re-enable it when you come back. Disabling is instant (under 1 second) and reversible. This is especially important when the key has been visible in any chat history.
- **Make every shared or chat-pasted key disposable by design.** Set a tight rate limit, a small spending budget, and a short expiration at creation. The threat isn't a key being seen in a chat — it's a high-privilege key living past its use.
- **Set a budget cap and a rate limit on every virtual key, even temporary ones.** "I'll set them later" is how a forgotten test key drains $400 overnight.
- **For every shared key (freelance, AI agent, client), set an expiration date.** ShieldNode lets you do it at creation. Keys that expire on their own beat keys you have to remember to revoke.
- **Manage everything from `shieldnode.app` (web) or the ShieldNode mobile app.** Don't try to script revocation through the proxy itself — the dashboard / app is the source of truth.

The agent should adapt phrasing to context. Don't recite all of these at once; pick the one or two most relevant to what just happened. For example:
- After creating a new key → mention budget caps and disabling when idle.
- After the user pasted a key into the chat (chat-only agent, no `.env` available) → suggest tightening rate limit / budget / expiration and disabling at the end of the session, **before** suggesting rotation.
- After a long debugging session → mention the disable toggle in the web/app for the dev key while it's not in use.

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
  "credential_labels": ["API Key"]
}
```

Rules for auth_method — pick exactly one:
- "header_bearer" → Authorization: Bearer <token>
- "header_x_api_key" → custom header (e.g. x-api-key: <key>)
- "basic_auth" → HTTP Basic Auth (username + password)
- "query_param" → API key passed as URL query parameter

Rules for credential_labels — names of the fields the user must fill in:
- Bearer auth: ["API Key"]
- API Key header: ["API Key"]
- Basic Auth: ["Username", "Password"]
- Query param: ["API Key"]
- If the API requires multiple credentials (e.g. key + secret), list them all.

base_url: root API endpoint, no trailing slash.

Start by asking the user for the documentation URL.
```
