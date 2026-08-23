# Evaluation design

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Evaluation design

No public benchmark was found that directly evaluates this exact combination:
pure-Markdown plan contracts, deterministic Rust verification, and `/goal`
execution across Claude, Codex, and Grok. Build a purpose-made evaluation.

Test four conditions:

```text
A. Free-form /goal prompt
B. Markdown plan template only
C. Markdown plan + static compiler
D. Markdown plan + compiler + proof engine + host continuation adapter
```

Run identical accepted tasks from identical clean commits across all four.

### Primary metric

The primary metric is not completion rate. It is:

```text
false PASS rate
```

A false PASS occurs when completion is declared even though:

- a requirement was not implemented
- a forbidden change occurred
- zero tests executed
- the wrong target was tested
- a necessary entry point was not exercised
- a plan was stale
- scope expanded without documentation

### Additional metrics

Measure:

- task success
- unauthorized-path rate
- requirement coverage recall
- change-to-requirement precision
- correct `PLAN_GAP` detection
- verifier mutation score
- median turns to verified completion
- median tokens to verified completion
- peak working-set tokens
- repeated-run variance
- human interventions
- plans requiring re-planning

### Mutation corpus

Every verifier release is tested against deliberately broken variants:

```text
replace a test command with `true`
make a test selector match zero tests
point a gate at the wrong package
delete one requirement-to-proof edge
introduce a dependency cycle
modify one forbidden file
change a frozen input after acceptance
add an undeclared runtime dependency
make a plan require an unlisted source file
exceed the context profile
weaken `tests >= 3` to `process.exit == 0`
```

Expected result is not always `FAIL`. For an unlisted source file, the correct
result is:

```text
PLAN_GAP
```

That distinction prevents the agent from turning every plan defect into an
improvised implementation decision.

### Planning-quality corpus

The SWE-RPG benchmark adds cases before the implementation loop, not only
after it. Include tasks where the visible request hides a compatibility
boundary, the location is known but the mechanism is underspecified, or the
new behavior can pass while a regression is introduced. These cases test the
three separate failure boundaries: requirement recovery, plan quality, and
code execution.

For each case, record whether `planlint check` identifies the missing detail,
whether `PLAN_GAP` stops unlisted necessary work, and whether `planlint verify`
rejects a false pass. The external benchmark reports 31.5% average resolution
across 163 tasks in 31 repositories; that number is context for the problem,
not a directly comparable `planlint` baseline.

The detailed evidence and limits are recorded in [SWE-RPG planning
implications](../research/review/06-swe-rpg-planning-implications.md).
