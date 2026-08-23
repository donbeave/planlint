# Verification

[← Integrations index](../README.md) · [Linear index](README.md)

> Preserved from the original Linear integration study.

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

