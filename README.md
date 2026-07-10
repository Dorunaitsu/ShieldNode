# ShieldNode Agent Skill 🛡️

A skill that teaches AI agents to use [ShieldNode](https://shieldnode.app), an API key proxy with push-approval access control for AI agents.

> If you want the shortest possible intro: drop this skill into your agent, then ask it *"explain ShieldNode and why I'd use it"*. The skill is structured so the agent can summarise the value, the security model, and the push-approval flow on its own. The rest of this README is for humans.

## Where ShieldNode lives

ShieldNode runs as three coordinated surfaces:

- **Web dashboard** at [shieldnode.app](https://shieldnode.app). Manage services and virtual keys, watch live request logs, see audit history, configure rate limits and spending caps. The full configuration surface.
- **Mobile app** on iOS and Android. The approval control center. Receive push notifications when an agent needs access, approve or decline in one tap, run the emergency stop that locks every key at once, browse per-key request logs with approval decisions inline.
- **Agent skill** in this repository. Drop into any agent that loads instructions (Claude Code, Cursor, OpenClaw, Codex, custom bots) and the agent immediately knows how to use virtual keys, request the right approval duration, identify itself in your notifications, and back off cleanly on a decline.

## What ShieldNode actually is

You write code with an AI agent. At some point the agent needs an API key (OpenAI, Stripe, whatever). Two unpleasant options today:

1. You paste the real key into the agent's environment. Now the key sits in shell history, in screenshots, in the agent's context window, in your editor's autocomplete, in any uploaded conversation. Rotating it later means rotating it everywhere.
2. You don't paste anything, and your agent can't actually do the thing.

ShieldNode is the third option. You give the agent a **virtual key**. That virtual key gets proxied through ShieldNode to the real API. You decide the rate limit, the request cap, the allowed endpoints, the expiry date. If something feels off, you revoke the virtual key in one tap. Your real upstream key never leaves the server.

That's the boring part. The interesting part is the next section.

## Push approval (the marquee feature)

The default state of a ShieldNode virtual key, if you want it that way, is **disabled**. Your agent has a key, but it does nothing.

When the agent tries to use it, ShieldNode does not return a generic error. It sends a time-sensitive push notification through the ShieldNode mobile app to your phone.

> *Claude is requesting 15 min access to your OpenAI key "production".*
> [Approve] [Decline]

You tap Approve. The key is active for exactly 15 minutes. The agent's next request returns 200. After 15 minutes, the key auto-disables. Next time the agent needs access, you get another push.

The flow takes about three seconds of your attention per session. You stay in control without becoming a bottleneck. Your agent stays productive without ever holding an always-on key.

Why this matters:

- Time-sensitive notifications break through Focus modes and silent profiles on iOS, so the approval request actually reaches you when an agent is blocked. Standard push categories would queue silently until your next unlock; this category does not.
- Your phone is harder to compromise than your laptop, so the approval channel is more trustworthy than the key channel.
- You see exactly which agent (Claude, Codex, Cursor, OpenClaw, your own bot) is requesting what, when, and for how long. That visibility alone changes how comfortable you are giving an agent access to sensitive things.
- If you ever wake up to an approval notification you didn't expect, you decline. Nothing was at risk, because the key was off.

This is the part you probably want to demo on launch day. The skill teaches your agent how to participate in this flow correctly (including how to request a duration, how to identify itself, how to poll without spamming).

## Inside the mobile app

The approval push is the surface, the mobile app is the control center. Available on iOS and Android.

- **Approval inbox.** Pending requests are visible on the alerts screen until they expire or you decide. You can decline late from the screen if you missed the push.
- **Per-key request logs.** Every proxied call lands here with method, path, HTTP status, latency, and timestamp. Approval decisions (approve, decline, expired) appear inline with the calls they gated, so you have one unified audit trail per key.
- **Key controls.** Pause, disable, delete a virtual key from one screen. Effective in under one second via in-memory cache invalidation.
- **Emergency stop.** A single button that disables every virtual key on your account at once, no confirmation, no delay. The big red switch you wish every credential system had.

## What this skill teaches your agent

Once the skill is loaded, the agent knows how to:

- Configure a new service in ShieldNode (base URL, auth method, credentials) with the three trade-offs explained (Auto detect, Manual, AI-assisted).
- Write a per-service reference document the first time it encounters a new API, so future sessions skip the doc-reading cost.
- Call the proxy correctly. The base-URL formatting trap (where `/v1` ends up either doubled or missing) is the number-one source of 404s and the skill bakes the rule in.
- Handle the push-approval flow end to end: detect `403 approval_required`, send `X-Agent-Name` and `X-Approval-Duration`, poll without hammering, surface a clean message when the user declines or times out.
- Diagnose errors by HTTP status instead of guessing.

The skill is opinionated about behaviour the agent should avoid: spamming the user with retry messages, requesting absurd approval durations, retrying after an explicit decline. Those rules exist because the worst case for this product is an agent that makes you regret approving anything.

## Install

### Drop into your project

```bash
mkdir -p skills && curl -sL https://github.com/RP0-undefined/shieldnode-skill/archive/refs/heads/main.tar.gz \
  | tar -xz -C skills --strip-components=1
```

Most agent runtimes (Claude Code, Cursor, Continue) auto-discover skills under a `skills/` directory.

### Single file

```bash
curl -O https://raw.githubusercontent.com/RP0-undefined/shieldnode-skill/main/SKILL.md
```

Then point your agent at it manually (system prompt, custom instructions).

### Git submodule

```bash
git submodule add https://github.com/RP0-undefined/shieldnode-skill skills/shieldnode
```

## Try the meta-test

The fastest way to know whether this skill (and by extension this product) makes sense to you:

1. Install the skill in your agent's workspace.
2. Ask: *"Read SKILL.md and tell me what ShieldNode is, what problem it solves, and whether you'd want me to use it for our work."*

Whatever your agent says is the most honest review you'll get. The skill is the manual the agent will rely on every day. If the agent can't pitch it back coherently, that's the most useful piece of feedback we can get.

## What's inside

| File             | Purpose                                                           |
|------------------|-------------------------------------------------------------------|
| `SKILL.md`       | The operational skill, read by the agent                          |
| `services/`      | Per-service reference docs generated by the agent (one `.md` each) |
| `LICENSE`        | MIT                                                               |

## Security model (short version)

Real upstream credentials are encrypted at rest with AES-256-GCM. The encryption key lives in the backend process, separately from the database. Decryption happens in memory at request time and never touches disk or logs.

Virtual keys are hashed with SHA-256 before storage. The plaintext is shown once at creation and is not recoverable afterwards. Lose it, generate a new one.

Request and response bodies are forwarded transparently. They are not logged, stored, or inspected. Only request metadata (method, path, status, latency, timestamp) is recorded for your own logs.

Disabling a virtual key propagates in under one second via in-memory cache invalidation. No client-side rotation is required.

Operational details (KMS choice, key rotation cadence, backup procedures, infrastructure boundaries) are intentionally not disclosed.

## Working with virtually any HTTP API

ShieldNode is API-agnostic. The skill includes worked patterns for AI providers (OpenAI, Anthropic, Mistral, OpenRouter, Replicate), payments (Stripe, Paddle, Lemon Squeezy), comms (Resend, Mailgun, Twilio), data (Airtable, Supabase, GitHub, Shopify), and any custom REST API you control. If your API is HTTP and uses a standard auth method, it works.

## Feedback, issues, contributions

This is the public hub for ShieldNode as a whole. If you have used the product or this skill and have something to say:

- **Bug, weird behaviour, unclear error.** [Open an issue](https://github.com/RP0-undefined/shieldnode-skill/issues/new). Even if it is about the dashboard, the proxy, or the docs and not strictly about this repo. Tag it appropriately and we will route it.
- **Feature idea or feedback.** Also issues, with the `enhancement` label or freeform.
- **Improvement to the skill.** [PR](https://github.com/RP0-undefined/shieldnode-skill/pulls) welcome. Keep it narrow (ShieldNode-related). For unrelated agent capabilities, fork.
- **Question.** [Discussions](https://github.com/RP0-undefined/shieldnode-skill/discussions) if enabled, otherwise an issue is fine.

Honest, blunt feedback is the most valuable kind. Specific complaints beat vague praise. If something feels off, say so.

## Links

- Product: [shieldnode.app](https://shieldnode.app)
- Docs: [shieldnode.app/docs](https://shieldnode.app/docs)
- Get the app: [shieldnode.app/get-app](https://shieldnode.app/get-app) (auto-detects your platform and links to App Store or Play Store)
- Mobile waitlist + Nexus waitlist: [shieldnode.app/waitlist](https://shieldnode.app/waitlist)

## License

MIT
