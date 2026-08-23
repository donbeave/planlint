# What Linear can represent

[← Integrations index](../README.md) · [Linear index](README.md)

> Preserved from the original Linear integration study.

## What Linear can represent

### Markdown content and issue storage

Linear issues have a required title and status, with an optional description.
The GraphQL API supports `issueCreate` and `issueUpdate`; descriptions accept
Markdown. Linear also supports Markdown in documents, comments, and agent
activity bodies.

This is enough to project a plan item into an issue containing:

```markdown
<!-- planlint-managed:start -->
**Contract:** `planlint/agent-cli#adapter-verify`
**Source:** `docs/plans/agent-cli.md`
**Source revision:** `8f2c...`

