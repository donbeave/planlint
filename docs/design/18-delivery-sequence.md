# Delivery sequence

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Delivery sequence

### Version 0.1 — static plan compiler

Implement only:

- strict Markdown grammar
- source-span diagnostics
- IDs and required sections
- dependency DAG
- write-scope overlap
- requirement coverage
- closed-world scope
- proof syntax validation
- context estimation
- contract fingerprinting
- fixtures and mutation tests

No command execution and no LLM-based rules.

### Version 0.2 — proof engine

Add:

- `probe`
- `verify`
- JUnit
- Cargo JSON
- SARIF
- Git scope verification
- non-vacuity checking
- strict argv execution
- timeout and output limits
- evidence receipts
- clean-worktree verification

### Version 0.3 — `/goal` controller

Add:

- Claude Code Stop hook
- Codex native `/goal` handoff and Stop hook
- Grok skill plus headless/ACP continuation controller
- one-plan-per-fresh-session scheduler
- runtime state machine
- actual context telemetry
- reconcile
- automatic blocking diagnostics

Only after these versions are evaluated should the tool consider parallel
agents, remote sandboxes, or broader orchestration.

