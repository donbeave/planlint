# planlint documentation

`planlint` is a proposed Rust plan-contract compiler for deterministic
verification of AI coding-agent work. This repository is a research and
design basis; it does not yet claim to implement the compiler.

## Start here

Read in this order:

1. [Product thesis](design/01-product-thesis.md) — what the project is and
   why the contract/verifier boundary matters.
2. [Contract model](design/06-contract-model.md) — specification, plan, and
   goal contracts.
3. [What the compiler should prove](design/08-compiler-proof.md) — the
   acceptance predicate and static checks.
4. [Final recommendation](design/19-final-recommendation.md) — the current
   product rules and host recommendations.

## Choose by question

| Question | Go to |
| --- | --- |
| What is the proposed product? | [Design index](design/README.md) |
| What is actually supported by sources? | [Research index](research/README.md) |
| How should contracts be authored? | [Markdown contract](design/07-markdown-contract.md) |
| How does proof avoid false passes? | [Non-vacuous verification](design/09-non-vacuous-verification.md) |
| How should context be bounded? | [Context and working set](design/02-context-and-working-set.md) |
| How should a host continue or stop an agent? | [Goal handoff](design/14-goal-handoff.md) |
| How should Rig drive real CLIs? | [Rig CLI index](integrations/rig-cli/README.md) |
| Should Linear be the backend? | [Linear decision](integrations/linear/13-decision.md) |

## Documentation map

### [Design](design/README.md)

Current synthesis: product shape, contracts, compiler/proof design, runtime,
evaluation, delivery sequence, and recommendation.

### [Research](research/README.md)

Evidence and source-level studies: goal implementations, current claims,
qualified findings, and research limits.

### [Integrations](integrations/README.md)

Operational studies for Linear and for driving coding-agent CLIs from Rig.

## How to read evidence

- **Confirmed** means official documentation or inspected source code supports
  the claim.
- **Supported inference** means the design follows documented primitives but is
  not itself a product feature.
- **Unknown** means the inspected material does not establish the claim.

The [research review](research/review/README.md) is the authority for refreshed
claims and qualifications. Design pages describe the proposed system, not
implemented behavior.

Research cut: 2026-08-23.
