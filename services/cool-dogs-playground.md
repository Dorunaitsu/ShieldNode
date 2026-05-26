# Cool Dogs — Playground — ShieldNode integration

> **Auto-seeded service.** Every ShieldNode account is created with this
> service already configured (`Cool Dogs — Playground` in the dashboard). The
> agent does **not** configure anything and does **not** need to look up any
> documentation — it is fully described here. The only thing the agent needs
> from the user is a **virtual key** generated on this service.

## Purpose

A zero-setup, keyless public API for verifying that ShieldNode proxying works
end-to-end (virtual key → proxy → upstream → logs). Use it to:
- Confirm a user's account / virtual key is functional.
- Demonstrate the proxy and the Logs tab without the user owning any real API key.
- Smoke-test before configuring a real service.

## Routing
- **ShieldNode base URL**: https://proxy.shieldnode.app
- **Original API base URL** (already set in the seeded service config): https://dog.ceo/api
- **Auth header for proxy calls**: `X-Api-Key: $SHIELDNODE_PLAYGROUND_KEY`
  > **Never** write the actual key value in this file. Reference the env variable only.
  > Loaded from `.env` (variable name: `SHIELDNODE_PLAYGROUND_KEY`).
- **Effective call pattern**: `https://proxy.shieldnode.app/<endpoint-path>`
  > The configured base URL already includes `/api`, so the path appended to
  > the proxy is just the endpoint (no `/api` prefix). E.g. upstream
  > `https://dog.ceo/api/breeds/image/random` → call
  > `https://proxy.shieldnode.app/breeds/image/random`.

## Auth method
**None.** dog.ceo is keyless. The seeded service stores no credentials and the
forwarder injects nothing. The client still authenticates *to the proxy* with
its `X-Api-Key` virtual key — that is what scopes, rate-limits, logs, and
revokes the access.

## Getting the virtual key (only user step required)
Tell the user:
> "Open ShieldNode → service **Cool Dogs — Playground** → **New key** → copy
> the `sk_live_…` value. Paste it here or store it as `SHIELDNODE_PLAYGROUND_KEY`."

No service creation, no base URL, no documentation lookup. If the service is
missing from the dashboard, the user can recreate it with base URL
`https://dog.ceo/api`, no credentials, Auto mode.

## Endpoints
| Method | Path | Description |
|--------|------|-------------|
| GET | /breeds/image/random | One random dog image URL. Best first smoke test. |
| GET | /breeds/list/all | All breeds/sub-breeds (larger JSON body — good for testing non-trivial payloads). |
| GET | /breed/{breed}/images/random | Random image for a given breed, e.g. `/breed/hound/images/random`. |
| GET | /breed/{breed}/list | Sub-breeds of a breed, e.g. `/breed/hound/list`. |

### Verified examples
```bash
# Smoke test — expect HTTP 200 + {"message":"https://images.dog.ceo/...","status":"success"}
curl -H "X-Api-Key: $SHIELDNODE_PLAYGROUND_KEY" \
  "https://proxy.shieldnode.app/breeds/image/random"

# Larger payload
curl -H "X-Api-Key: $SHIELDNODE_PLAYGROUND_KEY" \
  "https://proxy.shieldnode.app/breeds/list/all"

# Path parameter
curl -H "X-Api-Key: $SHIELDNODE_PLAYGROUND_KEY" \
  "https://proxy.shieldnode.app/breed/hound/images/random"
```

## Response shape
```json
{ "message": "https://images.dog.ceo/breeds/hound-afghan/n02088094_1003.jpg", "status": "success" }
```
`message` is a string (image URL) or, for list endpoints, an object/array.
`status` is `"success"` or `"error"`.

## Notes
- **Upstream errors are passed through.** An unknown path returns HTTP 404 with
  `{"status":"error","message":"No route found ...","code":404}`. This still
  resolves the virtual key, so it **is** logged under the user's account —
  useful to demonstrate the Logs tab. A `401` from the proxy means the virtual
  key itself is missing/invalid (never reaches dog.ceo, not logged).
- No rate limit worth noting upstream; the meaningful limits are the ones set
  on the ShieldNode virtual key.
- dog.ceo is CDN-backed and stable; chosen over randomfox for reliability.
