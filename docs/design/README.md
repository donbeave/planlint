# Design index

[← Documentation home](../README.md)

This is the current design synthesis. Pages are short and ordered from product
intent to implementation shape. They were split from the original research
archive without dropping its substantive sections.

## Product and constraints

- [Product thesis](01-product-thesis.md) — compiler pipeline, evidence basis,
  and the `PLAN_GAP` principle.
- [Context and working set](02-context-and-working-set.md) — why a fixed
  “smart context” threshold is not a quality guarantee.
- [Goal authority](03-goal-authority.md) — persistence is not acceptance
  authority across major hosts.
- [Source-level traces](04-source-level-traces.md) — implementation evidence
  behind the design.
- [Closest existing work](05-existing-work.md) — Markdown workflows,
  `agent-spec`, the Agent Execution Harness, and Tailrocks.

## Contract and proof

- [Contract model](06-contract-model.md) — the three contract layers.
- [Pure Markdown contract](07-markdown-contract.md) — canonical authoring
  dialect and example plan.
- [What the compiler should prove](08-compiler-proof.md) — static and dynamic
  acceptance conditions.
- [Non-vacuous verification](09-non-vacuous-verification.md) — avoiding
  zero-work and wrong-target proof.
- [Context-budget checker](10-context-budget-checker.md) — authored and
  working-set limits.
- [Contract inputs](11-contract-inputs.md) — explicit inputs and drift checks.
- [Input slicing discipline](12-input-slicing.md) — keeping one plan bounded.

## Runtime and delivery

- [Runtime state](13-runtime-state.md) — controller-owned lifecycle and
  receipts.
- [Goal handoff](14-goal-handoff.md) — compact host prompt and execution loop.
- [Recommended Rust architecture](15-rust-architecture.md) — crates and CLI
  boundaries.
- [Combining with agent-spec](16-agent-spec.md) — specification/runtime split.
- [Evaluation design](17-evaluation.md) — metrics and mutation corpus.
- [Delivery sequence](18-delivery-sequence.md) — versions 0.1–0.3.
- [Final recommendation](19-final-recommendation.md) — current rules and host
  recommendations.
- [Design source links](20-sources.md) — source list for this synthesis.

When a design claim needs source authority, follow the link to [Research](../research/README.md).
