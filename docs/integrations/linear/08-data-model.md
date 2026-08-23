# Recommended data model

[← Integrations index](../README.md) · [Linear index](README.md)

> Preserved from the original Linear integration study.

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

