# Integration surfaces

[← Integrations index](../README.md) · [Linear index](README.md)

> Preserved from the original Linear integration study.

## Integration surfaces

### GraphQL API and TypeScript SDK

Linear’s public API is GraphQL at `https://api.linear.app/graphql`, with
personal API keys and OAuth2 authentication. The official TypeScript SDK wraps
the schema and exposes typed queries and mutations.

This is the right surface for a deterministic `planlint linear sync` command:

1. resolve the configured workspace/team/project IDs;
2. read the canonical Markdown contract;
3. compute its stable ID and content hash;
4. create or update the corresponding Linear object;
5. create parent/sub-issue and blocking relations;
6. attach the source path, Git revision, and verifier links;
7. persist the Linear ID mapping in a repository manifest;
8. fail loudly on GraphQL `errors`, partial success, permission failures, or
   rate limits.

The public docs prove the object model and user-facing relationships. The
exact current mutation/input names for every relation, milestone, delegation,
and project-update operation should be obtained from live GraphQL introspection
and pinned in the POC; do not hard-code names from an old SDK example.

Linear documents cursor pagination, query filtering, and webhooks rather than
per-issue polling. The adapter should use those primitives from the beginning.

### Remote MCP server

Linear provides a remote MCP endpoint at `https://mcp.linear.app/mcp`, plus a
read-only endpoint at `https://mcp.linear.app/mcp/readonly`. Its documented
tool surface includes finding, creating, and updating Linear objects such as
issues, projects, and comments. Linear’s own roadmap-planning example starts
from a planning document and creates a project with issues, milestones, and
relationships.

MCP is useful for interactive agent workflows:

- ask an agent to inspect the current Linear graph;
- create or update an issue after human-visible confirmation;
- post a concise progress comment;
- inspect blockers and linked work.

Use the GraphQL SDK/API for the repository sync compiler. MCP tool availability
can evolve, and a model-mediated tool call is not a suitable idempotency,
conflict, or acceptance boundary. Prefer the read-only MCP endpoint for
exploration and least-privilege API scopes where writes are not needed.

### Webhooks

Linear webhooks can notify an HTTPS consumer when issues, comments, projects,
project updates, documents, initiatives, initiative updates, cycles, and other
supported objects are created, updated, or removed. Payloads include the
entity, action, URL, actor, updated fields, delivery ID, and timestamp.

The receiver must acknowledge quickly. Linear documents retries after failed
delivery and HMAC-SHA256 signature verification over the raw request body,
with timestamp checking to reduce replay risk.

Use a webhook to invalidate or update a local projection cache, not to mutate
the canonical contract silently. For a Linear-authored description or status
change, the adapter should apply the defined ownership rule:

```text
managed contract body changed in Linear
    → record drift
    → do not overwrite Git automatically
    → generate a reviewable patch or pull request

operational status/comment changed in Linear
    → accept as operational state
    → optionally mirror an event to Git/receipt storage
```

Linear’s API guidance explicitly recommends webhooks over polling and advises
checking GraphQL errors even when HTTP returns 200.

