# Distribution builds

Catalog-specific builds of the skill. The canonical source stays `../SKILL.md` and `../references/`; these folders only add what a given catalog requires. Update the canonical first, then port the delta here.

## hermes/

Goes into [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) at `optional-skills/security/shieldnode/`. Copy the **contents** of `hermes/` there, not the folder itself:

```
optional-skills/security/shieldnode/
  SKILL.md
  references/
```

Deltas from canonical, kept deliberately small:

| Delta | Why |
|---|---|
| Frontmatter gains `version`, `author`, `license`, `platforms`, `metadata.hermes`, `setup.collect_secrets` | Hermes catalog schema. `collect_secrets` makes Hermes prompt for `SHIELDNODE_KEY` at install. |
| `description` shortened to one action-oriented line | Catalog style is 5 to 15 words. |
| `X-Agent-Name: Hermes` in every example | Pushes then read "Hermes is requesting access". |
| Setup section uses `~/.hermes/.env` | Their secrets model. |
| Sections renamed to their house style (Requirements, When to Use, Setup, Hermes Execution Pattern, Guardrails, References) | Matches neighbouring skills such as `security/1password`. |
| Link back to the canonical repo in References | The catalog entry and its generated docs page funnel traffic home. |

On each canonical release: port the change, bump `version` in `hermes/SKILL.md`, open a PR.
