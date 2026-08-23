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

