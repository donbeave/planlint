# Pure Markdown contract

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Pure Markdown dialect

Pure Markdown is a good canonical format, but it must be constrained.

Rules:

- no JSON plan source
- no embedded XML
- no mandatory YAML frontmatter
- standard headings, lists, code spans, and fenced blocks
- one plan file is one compilation unit
- an internal typed Rust IR is never treated as the authored source
- JSON or SARIF is allowed only for generated diagnostics and receipts

Example plan:

```markdown
# PLAN-014: Enforce bounded client retries

## Contract

- **Version:** `1`
- **Covers:** `REQ-RETRY-01`, `REQ-RETRY-02`
- **Depends on:** `PLAN-013`
- **Context profile:** `goal-150k`
- **Planned at:** `4f12a8c`

## Outcome

The client stops retrying after the configured attempt budget and returns
the existing terminal error without changing the public API.

## Inputs

- `crates/client/src/retry.rs#RetryPolicy`
- `crates/client/tests/retry.rs`
- `specs/client-retry.spec.md#REQ-RETRY-01`

## Scope

### May change

- `crates/client/src/retry.rs`
- `crates/client/tests/retry.rs`

### May create

- None.

### Must not

- Change any public function signature.
- Add a runtime dependency.
- Change retry timing outside the attempt-counting path.
- Modify a path not listed under **May change**.

## Preconditions

### PRE-1: Existing retry tests are green

- **Run:** `mise run test-client-retry`
- **Evidence:** `junit:target/nextest/ci/junit.xml`
- **Proves:** `tests >= 2 && failures == 0`

### PRE-2: The planned source has not drifted

- **Run:** `planlint drift PLAN-014`
- **Proves:** `changed_inputs == 0`

## Steps

### STEP-1: Count completed attempts

Add attempt accounting inside `RetryPolicy` without changing its public
construction API.

- **Verify:** `mise run test-client-retry-focused`
- **Evidence:** `junit:target/nextest/ci/focused.xml`
- **Proves:** `tests == 1 && failures == 0`
- **Covers:** `REQ-RETRY-01`

### STEP-2: Preserve the terminal error

Add the exhausted-budget case to the existing retry integration test.

- **Verify:** `mise run test-client-retry`
- **Evidence:** `junit:target/nextest/ci/junit.xml`
- **Proves:** `tests >= 3 && failures == 0`
- **Covers:** `REQ-RETRY-02`

## Acceptance

### ACC-1: Retry behavior

- **Run:** `mise run test-client-retry`
- **Evidence:** `junit:target/nextest/ci/junit.xml`
- **Proves:** `tests >= 3 && failures == 0`

### ACC-2: Rust static gate

- **Run:** `mise run lint-client`
- **Evidence:** `cargo-json:stdout`
- **Proves:** `compiler_artifacts >= 2 && error_diagnostics == 0`

### ACC-3: Scope gate

- **Run:** `planlint scope PLAN-014 --worktree`
- **Proves:** `unexpected_paths == 0`

## Stop conditions

- `PLAN_GAP` if a required change falls outside **May change**.
- `BLOCKED` if a precondition fails.
- `STALE` if an input fingerprint differs from the accepted snapshot.
- `FAIL` if the same verification fails twice after a focused correction.
```

Meaning must come from headings, identifiers, and fields—not an LLM
interpreting arbitrary prose.

## Research-derived completeness rule

The SWE-RPG evidence adds a semantic requirement to this syntax. Each
implementation step must make five answers recoverable without executor
guessing:

| Answer | Contract field |
| --- | --- |
| What becomes true? | `Outcome` and covered requirements |
| Where does it change? | `Inputs`, `May change`, and step location |
| How does it change? | step implementation approach |
| What remains unchanged? | `Must not` and compatibility requirements |
| How is it proven? | `Verify`, `Acceptance`, and evidence predicates |

The headings need not reproduce this table, but every answer must be present.
A step that says “update as needed” without naming the mechanism is not
complete. A new-behavior check without a declared regression obligation is
not sufficient when the plan crosses a compatibility boundary. Missing
details are compiler diagnostics; work outside the accepted scope is
`PLAN_GAP`.

See [SWE-RPG planning implications](../research/review/06-swe-rpg-planning-implications.md)
for the evidence and the TSV/CSV boundary example.
