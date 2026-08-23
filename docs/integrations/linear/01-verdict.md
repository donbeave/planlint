# Linear verdict

[← Integrations index](../README.md) · [Linear index](README.md)

> Preserved from the original Linear integration study.

## Document context

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

