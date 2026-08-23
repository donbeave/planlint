# Context-budget checker

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Context-budget checker

The linter computes both **authored size** and **working-set estimate**.

### Authored size

```text
plan Markdown tokens
+ inlined requirement excerpts
+ inlined command output examples
```

### Working-set estimate

```text
base host instructions
+ applicable AGENTS.md / CLAUDE.md
+ selected skills
+ plan
+ required spec excerpts
+ required source inputs
+ dependency-plan summaries
+ expected command output
+ reserved execution history
```

Every input is explicit. Broad discovery instructions such as these should be
warnings or errors:

```text
Read the codebase.
Inspect all relevant files.
Update anything necessary.
```

Prefer:

```markdown
