# Input slicing discipline

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Slicing discipline

One small task does not mean one arbitrary code fragment.

A good plan item is:

- small enough for one fresh context
- a complete vertical behavior slice
- independently verifiable
- dependency-aware
- able to leave the repository green
- closed enough that the executor does not need a new architectural decision

Poor split:

```text
PLAN-001: Add database types
PLAN-002: Add service
PLAN-003: Add API
PLAN-004: Add tests
```

The first three plans have no independent behavioral proof.

Better split:

```text
PLAN-001: Establish a passing skeleton and verification baseline
PLAN-002: Support one end-to-end successful flow
PLAN-003: Add one failure flow with its proof
PLAN-004: Migrate remaining callers
PLAN-005: Remove the old path
```

The compiler checks write-set overlap:

```text
PLAN-014 writes retry.rs
PLAN-015 writes retry.rs
no dependency edge exists
```

That produces a dependency error or forces sequential execution. Plans are
safely parallel only when dependencies are verified and resolved write sets are
disjoint.

