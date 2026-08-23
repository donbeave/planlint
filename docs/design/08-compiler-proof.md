# What the compiler should prove

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## What the compiler should prove

The authoritative condition is approximately:

```text
PASS(plan, diff, evidence) =
    contract fingerprint matches
    AND all dependencies are VERIFIED
    AND all preconditions hold
    AND changed paths are a subset of allowed paths
    AND every covered requirement has passing evidence
    AND every changed path is justified by a step
    AND every acceptance gate executed non-zero work
    AND no Must-not or Stop condition was triggered
```

### Static checks

The side-effect-free `check` command validates Markdown without running
anything.

| Rule | Blocking condition |
| --- | --- |
| `PL001` | Missing or duplicate required section |
| `PL010` | Missing or duplicate plan, step, precondition, or acceptance ID |
| `PL020` | Dangling plan dependency or dependency cycle |
| `PL021` | Two supposedly parallel plans have overlapping write scopes |
| `PL030` | Requirement is declared under `Covers` but has no proving acceptance criterion |
| `PL031` | Step or changed surface has no requirement justification |
| `PL040` | Scope is absent, contradictory, or excessively broad |
| `PL041` | Verification can pass without demonstrating executed work |
| `PL050` | Estimated working set exceeds its context profile |
| `PL060` | Plan or input fingerprint has drifted |
| `PL070` | Command uses undeclared shell interpolation, network, environment, or working directory |
| `PL080` | Placeholder, unresolved decision, `TBD`, or ambiguous implementation choice |

`PL080` may be a warning for ordinary prose ambiguity. It must be an error
when ambiguity would require the executor to choose architecture, scope, or
acceptance semantics.

### Compiler-style diagnostics

Output should look like Rust diagnostics:

```text
error[PL041]: verification can succeed without executing work
  --> roadmap/retry/plan/014-bounded-retry.md:63:15
   |
63 | - **Proves:** `process.exit == 0`
   |               ^^^^^^^^^^^^^^^^^^ no evidence unit is required
   |
   = help: require a non-zero count or exact selector, for example:
           `tests >= 1 && failures == 0`
```

This is more useful to humans and agents than “Plan is not detailed enough.”

### Dynamic checks

Execution is separated into explicit commands:

```text
planlint check   # parse and lint; no execution
planlint probe   # run preconditions and gates before human acceptance
planlint verify  # verify an implementation against the frozen plan
```

Merely linting a Markdown file must never execute commands from it.

