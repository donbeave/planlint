# Storage comparison

[← Integrations index](../README.md) · [Linear index](README.md)

> Preserved from the original Linear integration study.

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

