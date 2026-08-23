# Contract model

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Three contract layers

The clean architecture has three document types.

### 1. Specification contract — what must be true

Contains product behavior, decisions, scenarios, and forbidden behavior:

```text
REQ-RETRY-01:
The client MUST stop after the configured attempt limit.
```

`agent-spec` is a plausible backend for this layer.

### 2. Plan contract — one bounded change that produces it

Declares:

- exact outcome
- requirements covered
- dependencies
- starting inputs
- files or symbols allowed to change
- steps
- focused checks
- final proof obligations
- STOP conditions
- context budget

This is the new Rust compiler’s input.

### 3. Goal contract — when the executor may continue or stop

Should remain extremely small:

```text
Execute exactly PLAN-014.
Continue until the authoritative verifier prints PLAN PASS.
Do not broaden scope.
On PLAN_GAP, BLOCKED, STALE, or FAIL, persist that state and stop.
```

The separation prevents duplication:

```text
specification = what
plan          = how, for one bounded slice
goal          = runtime continuation rule
```

The plan references specification IDs rather than copying the entire
specification. Where zero-context execution requires an excerpt, the compiler
can assemble it into a temporary execution packet without creating a second
authored source of truth.

