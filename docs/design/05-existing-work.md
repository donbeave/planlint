# Closest existing work

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Closest existing work

### Markdown spec workflows establish the authoring precedent

GitHub Spec Kit and OpenSpec already make Markdown the repository-visible
authoring surface. Spec Kit separates specification, plan, and actionable task
artifacts and adds a convergence loop; OpenSpec uses plain Markdown
requirements and concrete scenarios, then generates `design.md` and `tasks.md`
for a change.

They validate the usability and portability of Markdown across coding agents.
They do not, by themselves, define the proposed authoritative runtime
contract: immutable acceptance, closed-world changed-path checking,
non-vacuous evidence, input fingerprints, or a verifier-owned PASS state.

Therefore `planlint` should interoperate with these workflows rather than
compete with their planning UX:

```text
Spec Kit / OpenSpec / human Markdown
    → reviewed bounded plan
    → planlint check + accept
    → host /goal execution
    → planlint verify
```

### `agent-spec` is the closest technical foundation

The strongest related project found is the Rust project `agent-spec`. It
describes itself as an **intent compiler** and implements this pipeline:

```text
human intent
→ requirements
→ Task Contracts
→ implementation
→ deterministic lifecycle verification
→ trace and liveness
```

Its Task Contracts are Markdown documents containing Intent, Decisions,
Boundaries, and Completion Criteria. Completion scenarios bind explicitly to
tests, and intermediate gates are intended to be deterministic and model-free.

A real `agent-spec` task contract is close to the needed specification layer:

- requirement links through `satisfies`
- allowed paths
- forbidden changes
- out-of-scope work
- explicit completion scenarios
- named proving tests

Its architecture separates code intelligence, typed code bindings, quality
providers, execution bundles, lifecycle verdicts, and trace evidence.

Its mismatch is that it primarily represents a **behavioral contract**, not a
`/goal` execution item. It does not fully represent:

- exact ordered implementation steps
- per-step verification
- preconditions and starting-state drift
- one-fresh-context execution
- whole-working-set context budgets
- explicit `PLAN_GAP` and STOP behavior
- immutable plan fingerprints
- host-specific `/goal` continuation adapters

It correctly acknowledges that passing a contract does not prove the contract
was comprehensive. That limitation must remain visible in this design.

### Agent Execution Harness validates much of the runtime idea

Agent Execution Harness has narrow tasks, typed evidence, strict command
allowlists, compact handoffs, explicit verification, completion checking, and
worker-patch validation. In strict mode, shell-style commands are blocked
unless declared.

Its architectural mismatch is important: it imports an atomic Markdown backlog
into `plan.json`, then uses JSON as the operational plan. It is also implemented
in TypeScript.

It validates demand for an execution harness while leaving a clear place for
this narrower design:

> Rust, canonical Markdown, compiler diagnostics, context budgeting, and
> native `/goal` adapters.

### Tailrocks already contains most contract semantics

The existing Tailrocks plan workflow, as supplied for this research, already
requires:

- one zero-context plan per work item
- vertical, independently verifiable slices
- one fresh executor session
- explicit paths and code shapes
- preconditions
- in-scope and out-of-scope paths
- Must NOT constraints
- done criteria
- STOP conditions
- verification commands run during planning

The goal handoff freezes the plan package with a fingerprint and requires each
final gate to have the form:

```text
command ||| proof
```

The proof must demonstrate that non-zero work executed, specifically to prevent
commands that exit successfully after running zero tests.

The new Rust project should not redesign that workflow. It should compile and
enforce the workflow already present.

