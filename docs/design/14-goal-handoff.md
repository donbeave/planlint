# Goal handoff

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Suggested `/goal` handoff

The goal prompt stays tiny:

```text
/goal Execute only `roadmap/retry/plan/014-bounded-retry.md`.

Follow its scope, steps, Must-not rules, and Stop conditions exactly.
Do not modify the plan.

Continue until:

planlint verify roadmap/retry/plan/014-bounded-retry.md --worktree

prints `PLAN PASS` for the current accepted contract hash.

On `PLAN_GAP`, `BLOCKED`, `STALE`, or `FAIL`, persist that state and stop.
Do not broaden scope or replace a failed proof with a weaker proof.
```

Runtime loop:

```text
1. Rust controller selects one ready plan.
2. Fresh agent context starts.
3. Agent reads the plan and exact declared inputs.
4. Agent implements one step.
5. Focused verifier runs.
6. Diagnostics return to the agent.
7. Agent corrects the implementation.
8. Final verifier runs.
9. Host adapter continues only on a correctable verifier failure and permits
   completion only on PLAN PASS.
10. Controller stores receipt and transitions VERIFIED → DONE.
```

Native `/goal` evaluators are useful persistence layers where available; the
Grok adapter supplies equivalent persistence outside the host. The Rust
verifier’s verdict is the fact.

