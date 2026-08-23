# Product thesis

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Document context

Status: design basis, not implementation.

Reviewed: 2026-08-23.


## Verdict

This is a strong concept. The right product is not merely a Markdown linter;
it is a Rust **plan-contract compiler** with a deterministic verification
runtime.

The proposed execution model is:

```text
accepted specification
        ↓
pure-Markdown plan item
        ↓
Rust static checker
        ↓
one fresh /goal execution
        ↓
deterministic verifier
        ↓
evidence receipt
        ↓
DONE or explicit failure state
```

The agent remains probabilistic while producing code, but acceptance becomes
deterministic. The useful Rust analogy is:

```text
Rust source → compiler diagnostics → developer edits → compiler passes

Plan contract → verifier diagnostics → agent edits → verifier passes
```

Research supports the direction. CodePlan framed repository-level changes as
dependency-aware planning and, in its FSE evaluation, got 5 of 7 repositories
through validity checks while equivalent non-planning baselines got none
through. The August 2026 SWE-RPG preprint reports that current coding agents
resolved 31.5% of its tasks on average and identifies recovery of implicit
requirements as a major failure source. Small plans and mechanical gates help,
but the tool also needs an explicit **`PLAN_GAP`** state instead of allowing
the executor to guess missing requirements.

These findings do not prove that this exact product will work. They justify a
purpose-built evaluation.

