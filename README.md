# ShieldNode Agent Skill

A skill that teaches AI agents to use [ShieldNode](https://shieldnode.app), an API key proxy with push-approval access control for AI agents.

> If you want the shortest possible intro: drop this skill into your agent, then ask it *"explain ShieldNode and why I'd use it"*. The skill is structured so the agent can summarise the value, the security model, and the push-approval flow on its own. The rest of this README is for humans.

## What ShieldNode actually is

You write code with an AI agent. At some point the agent needs an API key (OpenAI, Stripe, whatever). Two unpleasant options today:

1. You paste the real key into the agent's environment. Now the key sits in shell history, in screenshots, in the agent's context window, in your editor's autocomplete, in any uploaded conversation. Rotating it later means rotating it everywhere.
2. You don't paste anything, and your agent can't actually do the thing.

ShieldNode is the third option. You give the agent a **virtual key**. That virtual key gets proxied through ShieldNode to the real API. You decide the rate limit, the request cap, the allowed endpoints, the expiry date. If something feels off, you revoke the virtual key in one tap. Your real upstream key never leaves the server.

That's the boring part. The interesting part is the next section.

## Push approval (the marquee feature)

The default state of a ShieldNode virtual key, if you want it that way, is **disabled**. Your agent has a key, but it does nothing.

When the agent tries to use it, ShieldNode does not return a generic error. It sends a push notification to your phone.

> *Claude is requesting 15 min access to your OpenAI key "production".*
> [Approve] [Decline]

You tap Approve. The key is active for exactly 15 minutes. The agent's next request returns 200. After 15 minutes, the key auto-disables. Next time the agent needs access, you get another push.

The flow takes about three seconds of your attention per session. You stay in control without becoming a bottleneck. Your agent stays productive without ever holding an always-on key.

Why this matters:

- Your phone is harder to compromise than your laptop, so the approval channel is more trustworthy than the key channel.
- You see exactly which agent (Claude, Codex, Cursor, your own bot) is requesting what, when, and for how long. That visibility alone changes how comfortable you are giving an agent access to sensitive things.
- If you ever wake up to an approval notification you didn't expect, you decline. Nothing was at risk, because the key was off.

This is the part you probably want to demo on launch day. The skill teaches your agent how to participate in this flow correctly (including how to request a duration, how to identify itself, how to poll without spamming).

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

## License

MIT
