# Proposed workflow

[← Integrations index](../README.md) · [Linear index](README.md)

> Preserved from the original Linear integration study.

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

