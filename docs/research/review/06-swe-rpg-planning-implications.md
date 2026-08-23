# SWE-RPG: implications for planlint

[← Research index](../README.md) · [Review index](README.md)

> Research detail added from the supplied SWE-RPG summary. The benchmark
> supports the problem framing and design hypotheses; it does not validate
> `planlint` itself.

## What the benchmark adds

The SWE-RPG preprint evaluates 163 real tasks from 31 repositories across
coding-agent and model combinations. It reports an average resolved rate of
31.5%. Its useful contribution for `planlint` is diagnostic: an agent can fail
at three different boundaries:

1. recovering what the request means
2. turning that understanding into a complete implementation plan
3. implementing a good plan incorrectly

Final test status collapses those causes into one result. The preprint
identifies recovery of implicit requirements as a major difficulty. An agent
can therefore produce plausible code that solves the wrong interpretation of
the task.

Source: [SWE-RPG / unified issue-resolution benchmark — arXiv:2608.09072](https://arxiv.org/abs/2608.09072).

## Hidden requirements are compatibility requirements

The supplied example is a TSV importer that incorrectly removes whitespace.
The visible fix is “preserve TSV whitespace,” but the boundary condition is
equally important: CSV behavior must remain unchanged.

This plan is too weak:

```markdown
## Task

Fix whitespace handling in separator-based imports.

## Verify

Run the importer tests.
```

It permits a shared parser change and a test run that checks only the new TSV
case. A bounded plan makes the compatibility surface explicit:

```markdown
# Preserve whitespace in TSV imports

## Goal

TSV import preserves leading and trailing whitespace when trimming is disabled.

## Requirements

- Change only the TSV import path.
- CSV behavior remains unchanged.

## Files

- `SeparatorBasedImporter.java`
- TSV importer tests

## Change

Disable automatic whitespace removal in `TsvParserSettings`.

## Must not

- Change shared CSV parser settings.
- Change default CSV behavior.
- Modify unrelated import formats.

## Verify

- Run the focused TSV whitespace test.
- Run the existing CSV import tests.
```

The point is not the exact headings. The point is that the executor should not
have to infer the affected boundary, implementation mechanism, or regression
obligation.

## Five dimensions of a useful plan

The paper's planning analysis gives `planlint` five semantic dimensions:

| Dimension | Required answer | Existing contract surface |
| --- | --- | --- |
| Goal | What must become true? | `Outcome` |
| Location | Where does the change belong? | `Inputs`, `May change`, and step locations |
| Approach | How should it work? | `Steps` and their implementation prose |
| Constraints | What must remain unchanged? | `Must not`, closed-world scope, and stop conditions |
| Validation | What proves completion and compatibility? | `Verify`, `Acceptance`, evidence, and proof predicates |

The reported difficulty increases after location: agents are less reliable at
stating the implementation approach, preserving constraints, and defining
validation. The static checker must therefore inspect semantic completeness,
not only Markdown shape.

## Consequences for the compiler

`planlint check` should reject or warn on plans that:

- have no concrete outcome;
- name no affected files, symbols, or other bounded locations;
- describe a result without an implementation mechanism;
- omit behavior that must remain unchanged;
- leave requirements without a proving acceptance criterion;
- verify only the new behavior when a compatibility boundary is declared;
- contain unresolved architecture, scope, or acceptance decisions; or
- are too broad for one execution unit.

Diagnostics should identify the missing semantic field. For example:

```text
error[PL081]: step has no implementation approach
error[PL082]: requirement REQ-3 has no acceptance proof
error[PL083]: compatibility boundary has no regression verification
PLAN_GAP: required change falls outside the declared write scope
  = help: revise the accepted plan; do not expand execution scope
```

The last case is `PLAN_GAP`, not an invitation for the worker to improvise.
If the correct fix needs an unlisted public API or file, execution stops and
the contract returns to planning.

## Check and verify are different jobs

The research supports a two-phase minimum workflow:

```text
planlint check plan.md
    ↓
/goal executes the accepted plan
    ↓
planlint verify plan.md --worktree
```

`check` asks whether the plan is clear, bounded, internally covered, and safe
to hand to an agent. It must not execute commands embedded in the Markdown.

`verify` asks whether the resulting repository satisfies the frozen plan. It
should combine:

- the focused behavior test;
- relevant regression or compatibility tests;
- compilation or lint gates; and
- a changed-path scope check.

The existing `probe` command remains useful as a pre-acceptance execution
phase. It does not collapse the distinction: a successful plan check is not
implementation proof, and a successful command exit is not sufficient proof
by itself.

## Plan size and context

The benchmark's validated reference plans were generally small: about two
steps and 2.5 file locations on average, with benchmark maxima of ten steps
and ten locations. This supports a default execution unit shaped as:

```text
one behavior
one small file/symbol set
one explicit implementation approach
one or two proof commands
one fresh execution context
```

This does not establish a universal context-window threshold. In particular,
it does not validate 150K as a quality boundary. `goal-150k` should remain a
configurable upper profile, with room reserved for repository inspection,
tool output, errors, and correction rather than being treated as a target to
fill.

## What the paper does not establish

The paper is not an experiment with a Rust Markdown compiler. It does not
show that `planlint` improves completion, reduces false passes, or selects
the best plan schema. It also does not compare small and large plans, context
window sizes, or human-approved plans. Its repository set is limited to
Python and Java; Grok was not evaluated. Some failure classification uses an
LLM judge alongside manual reference review.

Use the findings to define hypotheses and failure cases for the local
evaluation, not as proof of product effectiveness.

## Evaluation additions

Add benchmark-inspired cases to the mutation corpus and acceptance study:

- a visible fix with an unstated compatibility boundary;
- a plan with a correct location but vague implementation approach;
- a new-behavior test that passes while regression behavior breaks;
- a required file or API surface omitted from the write scope;
- a requirement with no proof edge; and
- a verification command that executes zero relevant tests.

Measure whether the compiler catches the planning defect before execution,
returns `PLAN_GAP` for unauthorized necessary work, and prevents a false
`PASS` after implementation. Do not use the 31.5% benchmark result as a
baseline for this product without reproducing the task and evaluation setup.
