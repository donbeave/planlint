# Sync ownership and conflict policy

[← Integrations index](../README.md) · [Linear index](README.md)

> Preserved from the original Linear integration study.

## Sync ownership and conflict policy

Do not allow both systems to freely edit the same fields. Define ownership by
field class:

| Field class | Owner | Sync direction |
| --- | --- | --- |
| Contract title, acceptance, verification, scope | Git | Git → Linear |
| Issue status, assignee, cycle, comments | Linear | Linear → event/cache; selected status → receipt |
| PR state, checks, merge state | GitHub | GitHub → Linear native integration |
| Verifier result and evidence URL | `planlint`/CI | verifier → Linear comment/status |
| Initiative/project summaries | Human or explicit sync command | reviewed, not implicit two-way |

The issue description should contain a clearly delimited managed block. Human
comments and non-managed context must survive a sync. If a user edits the
managed block in Linear, mark the issue `drifted` or add a drift label and open
a reviewable change; never silently pull cloud text into the canonical file.

Use content hashes and stable IDs for idempotency. Titles are not identities:
they change, can collide, and can be translated.

