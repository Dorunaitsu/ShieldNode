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
| Frontmatter gains `version`, `author`, `license`, `platforms`, `metadata.hermes` | Their documented schema (CONTRIBUTING.md, "SKILL.md format"). |
| `description` shortened to one action-oriented line | Catalog style is 5 to 15 words. |
| Body follows their documented section order: When to Use, Prerequisites, How to Run, Quick Reference, Procedure, Pitfalls, Verification | Same source. Deviating is the easiest way to get a skill PR bounced. |
| `X-Agent-Name: Hermes` in every example | Pushes then read "Hermes is requesting access". |
| Prerequisites uses `~/.hermes/.env` and leads with the app link | Their secrets model; the app is where the account is created. |
| No `required_environment_variables` or `setup` block | Our keys are per-service (`SHIELDNODE_<SERVICE>_KEY`), so no single variable can be declared honestly. Verified on a live Hermes agent: undeclared keys in `~/.hermes/.env` resolve fine. |
| Link back to the canonical repo in References | The catalog entry and its generated docs page funnel traffic home. |

On each canonical release: port the change, bump `version` in `hermes/SKILL.md`, open a PR.
