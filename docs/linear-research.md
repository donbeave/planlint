# Linear as the planlint work backend

Research cut: 2026-08-23. This document evaluates Linear as the operational
backend and observability surface for the plan-contract workflow described in
the other research documents.

## Verdict

Linear is technically suitable for the **work graph and operator UI**:

```text
initiative / program
        ↓
project / delivery unit
        ↓
milestone / phase
        ↓
parent issue / plan item
        ↓
sub-issue / executable slice
        ↘ blocked-by / blocking / related edges
        ↘ GitHub branch / commit / pull request
        ↘ agent session and activity
```

It is not a drop-in replacement for Markdown files in Git. The best design is
hybrid:

```text
Git Markdown = canonical contract, version history, review, verifier input
Linear       = operational projection, ownership, status, dependencies, UI
GitHub       = code, CI, branches, commits, pull requests
```

Use Linear as the **control-plane projection**, not as the authority that can
declare a plan item technically complete. `Done` in Linear means the work is
reported done; `planlint verify` plus a stored evidence receipt decides whether
the contract actually passed.

This meets the visibility goal while preserving deterministic acceptance and
avoiding a two-master Markdown conflict.

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

## Objective
...

## Acceptance
- ...

## Verification
- `cargo test ...`
- `planlint verify ...`
<!-- planlint-managed:end -->
```

Do not treat the issue description as an arbitrary file store. The checked
Linear sources document Markdown fields, API export, and “copy as Markdown”,
but do not document a built-in synchronization of an arbitrary repository
directory with Linear issues. The GitHub integration syncs GitHub Issues, not
arbitrary `.md` files.

Linear descriptions are therefore a useful **rendered copy**, not a lossless
Git filesystem abstraction. Keep the source path, stable contract ID, and
source content hash in the projection.

### Hierarchy

Linear’s current concepts map well to the requested hierarchy:

| planlint concept | Linear representation | Fit |
| --- | --- | --- |
| Strategic objective / program | Initiative | Strong |
| Delivery unit / large plan | Project | Strong |
| Phase / checkpoint | Project milestone | Strong |
| Contract or plan item | Issue | Strong |
| Smaller executable task | Parent issue + sub-issue | Strong |
| Cross-cutting grouping | Labels, project views, teams | Strong |
| Epic | Usually a project; sometimes a parent issue | Partial: Linear does not require an “Epic” type |
| Nested strategic programs | Sub-initiatives | Strong, but current docs state up to five levels and Enterprise availability |

Projects contain issues and optional documents, support multiple teams, and have
project views and progress graphs. Initiatives sit above projects and roll up
their projects. Milestones organize issues within a project and expose progress
in initiative and project timelines. Parent issues and sub-issues provide the
task tree; issue relations provide non-tree edges.

This is close to a plan DAG, with an important distinction: the hierarchy is a
tree and dependencies are separate edges. A plan compiler must keep both
structures explicit. A sub-issue is not automatically a dependency.

### Dependencies and relations

Linear supports issue relations for `blocked by`, `blocking`, `related`, and
`duplicate`. Blocking issues show in the issue sidebar, and a resolved blocker
causes the relation to move under Related. This gives the requested blocker
visibility for task-level work.

Linear also supports project dependencies in timeline views. The documented
project dependency model currently supports only an end-to-start dependency.
That is useful for delivery planning, but it is weaker than a general typed
plan edge such as `requires`, `informs`, `invalidates`, or `must-follow`.

Recommended rule:

- Use native Linear blocking relations for operator-visible blockers.
- Keep the authoritative dependency edge type and cycle policy in the Git
  contract.
- Reject cycles and dangling references in `planlint`, even if Linear accepts
  the corresponding issue graph.
- Project dependencies are a convenience for schedule visualization, not the
  verifier’s dependency model.

### Status and progress

Linear provides workflow statuses, board/list/timeline views, project status,
milestone progress, initiative health, project updates, and project graphs.
Project graphs use issue activity and estimates to show scope, started/completed
work, velocity, and predicted completion. That is excellent operator
observability.

It is not proof of a contract. A project can look complete while a required
test was skipped, a changed path escaped the declared scope, or an acceptance
claim was never verified. Keep these separate:

| Meaning | Owner |
| --- | --- |
| Human/agent execution state | Linear issue status |
| Project health and communication | Linear project/initiative update |
| Contract lifecycle | Git Markdown + `planlint` |
| Test/compiler/linter result | Deterministic verifier and CI |
| Final evidence | Immutable receipt linked from Linear |

Useful Linear statuses for the projection are `Backlog`, `Ready`, `In
Progress`, `Blocked`, `In Review`, `Verified`, `Done`, and `Canceled`. The
`Verified` status should only be set by the sync adapter after a verifier
receipt passes. If that policy is too restrictive for normal team use, use a
`planlint:verified` label or comment instead; never infer verification from
`Done`.

### GitHub branches, commits, and pull requests

Linear’s GitHub integration is a major advantage over local Markdown. It can:

- link PRs using issue IDs, branch names, commit messages, or magic words;
- move issues through configured statuses as branches, commits, and PRs change;
- link multiple issues to one PR and multiple PRs to one issue;
- show review state, checks, comments, and merge progress;
- display PR diffs and file changes inside Linear when code access is enabled.

This gives the desired “task → dependency → PR → review → merge” view without
building a second GitHub integration. The adapter should put the stable Linear
issue identifier in generated branch names and PR descriptions, then let the
native integration perform the link.

Linear’s code-review view requires repository code access and personal GitHub
account connections. Treat that as a workspace permission/configuration
dependency, not as a property guaranteed by the API alone.

### Agents and execution visibility

Linear supports assigning issues to agents. Its agent API supports sessions and
semantic activities such as thoughts, actions, clarification requests,
responses, and errors. This is a better fit for “which agent is working on
which task?” than a Markdown checkbox.

The agent activity stream is an observability surface, not a verifier. Emit
short progress events and links to evidence; keep verbose tool logs and exact
receipts in CI or repository artifacts. Never put secrets or sensitive source
content into issue descriptions or activity bodies just because Linear makes
them easy to display.

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

## Storage comparison

| Dimension | Markdown in Git | Linear as canonical storage | Hybrid recommendation |
| --- | --- | --- | --- |
| Exact contract text | Strong | Markdown field, but not a Git file | Git owns contract |
| Version history | Commits, branches, review | Description history and activity | Git owns normative changes |
| Offline/local execution | Strong | Requires network/auth | Verifier reads Git |
| Human visibility | Good for developers | Excellent list/board/timeline UI | Linear projection |
| Hierarchy | Manual links/headings | Native initiatives/projects/milestones/sub-issues | Project graph mirrors Git DAG |
| Dependencies | Compiler-defined, exact | Native blocker relations and project edges | Linear renders; compiler validates |
| PR linkage | Manual links or conventions | Native GitHub linking/status/reviews/diffs | Use Linear integration |
| Agent ownership/activity | Files and logs need conventions | Native assignment/session/activity | Linear UI + receipt artifacts |
| Deterministic acceptance | Natural with a local verifier | Not a native contract compiler | `planlint verify` remains authority |
| Backup/export | Git clone | API/CSV/Markdown copy | Back up both; Git is recovery source |
| Secrets/privacy boundary | Repository policy | Cloud workspace permissions | Keep sensitive data out of projection |
| Conflict behavior | Git merge/review | Last writer/activity history | One owner per field; drift is explicit |
| Portability | High | Workspace/API dependency | Adapter boundary isolates Linear |

Linear wins decisively on observability. Git wins decisively on canonical
specification and reproducibility. Making Linear the only store would trade
away the exact properties that make `planlint` useful.

## Recommended data model

Use one Linear team for the execution domain, unless cross-team ownership is a
real requirement. Create this mapping:

| Canonical field | Linear field / artifact |
| --- | --- |
| `plan_id` | Stable marker in managed description block and sync manifest |
| title | Issue title |
| contract body | Managed Markdown block in issue description |
| source path | Link/text in issue description |
| source revision | Link to commit or CI artifact |
| objective/program | Initiative |
| delivery plan | Project |
| phase | Project milestone |
| child task | Sub-issue |
| prerequisite | Blocking relation |
| execution owner | Assignee or delegated agent |
| execution state | Issue status |
| human context | Comments and project/initiative updates |
| PR and checks | Native GitHub attachment |
| verifier result | Comment/link to receipt; optionally controlled status |
| content identity | `.linear/manifest.json` in Git |

Example manifest entry:

```json
{
  "planlint/agent-cli#adapter-verify": {
    "source": "docs/plans/agent-cli.md",
    "sha256": "...",
    "linearIssueId": "uuid",
    "linearIdentifier": "ENG-123",
    "linearProjectId": "uuid",
    "lastSyncedRevision": "8f2c..."
  }
}
```

The manifest is an index, not a second contract. A missing or stale mapping
must cause a visible sync diagnostic, not a guessed new issue.

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

## Proposed workflow

```text
1. Author or approve plan contract in Git
2. planlint check/compile validates IDs, references, cycles, scope, gates
3. planlint linear sync projects contract into Linear
4. Linear shows hierarchy, ownership, blockers, progress, agent activity
5. Agent works from Git contract and updates Linear operational state
6. GitHub branch/commit/PR links automatically to the Linear issue
7. CI runs deterministic verifier and writes an evidence receipt
8. Adapter posts receipt link and changes Linear state to Verified/Done
9. Webhook records later cloud edits as events or drift
```

The agent’s context packet should contain the Linear issue URL and relevant
project/initiative context, but the contract itself should be fetched from the
checked-out Git revision. This prevents stale cloud descriptions from silently
changing the execution target.

## Minimal proof of concept

Build the smallest vertical slice before committing to Linear as a product
dependency:

1. Add three Markdown plan items with stable IDs, one parent, one child, and
   one blocking edge.
2. Implement `planlint linear sync --project ...` using the GraphQL SDK/API.
3. Run it twice and prove it creates no duplicates.
4. Verify the Linear issue shows the Markdown projection, parent/sub-issue,
   blocker edge, milestone, and source commit link.
5. Create a test branch and PR containing the Linear ID; prove the PR appears
   on the issue and status automation behaves as configured.
6. Run the verifier; post a receipt comment and set the controlled verified
   signal only on pass.
7. Edit an operational field in Linear and confirm the local manifest/cache
   updates without changing the contract file.
8. Edit the managed block in Linear and confirm the adapter reports drift and
   does not overwrite Git.
9. Test a failed webhook, duplicate delivery, GraphQL partial error, expired
   token, rate limit, missing team/project, dangling dependency, and cycle.

Acceptance for the POC:

- repeatable sync is idempotent;
- the same stable plan ID always resolves to the same Linear issue;
- no Linear UI status can bypass `planlint verify`;
- PR, review, check, and merge state are visible from the issue;
- dependency edges are visible in Linear and validated in Git;
- cloud edits cannot silently rewrite the canonical contract;
- loss of Linear access does not prevent local verification;
- the adapter can export enough metadata to rebuild the projection.

The POC must also record the live schema revision and the exact scopes needed
for each mutation. This is the remaining API-level unknown; it is not a reason
to make Linear the canonical contract store before testing.

If the POC passes, Linear is a good backend for the **operational graph**. If
the requirement changes to “edit contracts in a web UI and reproduce them
exactly as Git files,” Linear should not be the canonical store; build a
dedicated web editor backed by Git or introduce an explicit reviewed import
step.

## Risks and controls

- **Two masters:** field-level ownership, managed blocks, hashes, and drift
  detection.
- **Cloud dependency:** local verifier and Git source remain fully usable
  without Linear.
- **API drift/limits:** typed SDK, narrow filtered queries, cursor pagination,
  webhook-driven updates, retries with backoff, and GraphQL error inspection.
- **Webhook spoof/replay:** verify raw-body HMAC and timestamp; deduplicate by
  delivery ID.
- **Over-sharing:** least-privilege OAuth/API scopes; read-only MCP where
  possible; never store secrets in Linear descriptions or agent activity.
- **False completion:** only receipts from deterministic commands may produce
  the verified signal.
- **Graph mismatch:** keep typed dependency semantics and cycle policy in the
  Markdown contract/compiler; use Linear’s native edges for visualization.
- **Vendor lock-in:** isolate Linear behind an adapter and keep stable IDs,
  source files, and receipts in Git.

## Decision

Adopt Linear experimentally as a **projection and observability backend** for
planlint. Do not replace local Markdown contracts. The first implementation
should be one-way contract projection plus controlled operational-state
feedback, with GitHub’s native integration handling PR visibility and Linear’s
webhooks/API handling synchronization.

This gives the requested UI/UX—hierarchy, dependencies, owners, agent activity,
progress, and PRs—without weakening the core design: acceptance remains
versioned, local, deterministic, and independently verifiable.

## Sources

- [Linear conceptual model](https://linear.app/docs/conceptual-model)
- [Linear projects](https://linear.app/docs/projects)
- [Linear initiatives](https://linear.app/docs/initiatives)
- [Linear sub-initiatives](https://linear.app/docs/sub-initiatives)
- [Linear project milestones](https://linear.app/docs/project-milestones)
- [Linear parent and sub-issues](https://linear.app/docs/parent-and-sub-issues)
- [Linear issue relations](https://linear.app/docs/issue-relations)
- [Linear project dependencies](https://linear.app/docs/project-dependencies)
- [Linear project graph](https://linear.app/docs/project-graph)
- [Linear GitHub integration](https://linear.app/docs/github)
- [Linear Reviews / Diffs](https://linear.app/docs/diffs)
- [Linear GraphQL API](https://linear.app/developers/graphql)
- [Linear TypeScript SDK](https://linear.app/developers/sdk)
- [Linear SDK fetching and mutations](https://linear.app/developers/sdk-fetching-and-modifying-data)
- [Linear rate limiting](https://linear.app/developers/rate-limiting)
- [Linear webhooks](https://linear.app/developers/webhooks)
- [Linear OAuth2 authentication](https://linear.app/developers/oauth-2-0-authentication)
- [Linear MCP server](https://linear.app/docs/mcp)
- [Linear agent interaction](https://linear.app/developers/agent-interaction)
- [Linear agent interaction best practices](https://linear.app/developers/agent-best-practices)
- [Linear exporting data](https://linear.app/docs/exporting-data)
- [Linear documents](https://linear.app/docs/documents)
- [Linear issue templates](https://linear.app/docs/issue-templates)
