# Final recommendation

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Final recommendation

Build a small Rust project tentatively called **`planlint`**, designed
internally as a **plan-contract compiler**.

Defining rules:

```text
Markdown is the canonical source.
One plan is one coherent vertical slice.
One plan runs in one fresh context.
The plan is frozen after human acceptance.
Unlisted necessary work is PLAN_GAP.
Exit zero is not evidence.
Every requirement needs a non-vacuous proof.
Every changed surface needs a requirement.
Only the Rust controller can declare PASS.
```

Target Tailrocks pipeline:

```text
tailrocks-plan
    → planlint check
    → planlint probe
    → human acceptance
    → /goal executes one plan
    → planlint verify
    → evidence receipt
    → reconcile
    → DONE
```

This is a credible path toward making agent execution behave less like “ask an
LLM to implement the plan” and more like **compile an accepted contract,
iterate on diagnostics, and accept only a proven result**.

### Concrete host recommendations

Use one host-neutral contract for every adapter:

```text
objective        one bounded vertical slice
inputs           explicit files, symbols, docs, and accepted fingerprints
may_change      closed-world path set
must_not        forbidden paths, APIs, dependencies, and behavior
acceptance       named commands plus structured evidence predicates
retry_budget     maximum corrective passes and total tool/turn budget
stop             PASS | PLAN_GAP | BLOCKED | STALE | FAIL
```

The host’s goal feature may keep the model working, but it must not be the
authority for PASS. The controller/verifier owns receipts, scope checks, retry
limits, and terminal status.

**Claude Code**

- Use `/goal` only as the session continuation wrapper. Put one measurable
  end-state, the exact commands that prove it, forbidden changes, and a turn or
  time bound in the condition.
- In the native profile, make the worker run `planlint verify`, surface its
  canonical result, and have the outer process independently validate the
  receipt. In the strict profile, replace native `/goal` with a deterministic
  command Stop hook. Do not stack two Stop-hook completion controllers without
  explicit coordination.
- Accept the plan and permissions before enabling unattended turns. Keep one
  plan per session; do not rely on the evaluator to discover missing files or
  unauthorized changes.

**Codex**

- Use native `/goal` for durable thread state and `/plan` for decomposition or
  architecture review. Put detailed contract content in a file when it exceeds
  the goal field; use the documented TUI goal path rather than assuming
  `codex exec` parses slash commands.
- Require a deterministic verifier receipt before `update_goal Complete`; the
  outer process independently validates it. If a Stop hook owns strict
  continuation, use it as the goal equivalent instead of stacking an
  uncoordinated second loop on native goal state.
- Keep normal tool permissions and goal replacement confirmation enabled. Treat
  pause/resume/clear and token/turn budgets as controller state, not model
  suggestions.

**Rig**

- Use Rig as a lower-level execution substrate. Keep the durable objective,
  accepted contract, evidence receipt, scope policy, retry cap, and terminal
  state in a planlint controller; Rig has no native `/goal` lifecycle.
- Use `AgentRunner` plus `AgentHook` for normal runs, per-run policy, and
  telemetry. Use `rig::AgentRun` for mid-run approval/persistence only when
  the controller is willing to own model transport, tool execution, and the
  current-version serialization boundary.
- Run deterministic proof commands and changed-path enforcement in planlint.
  A custom evaluator agent can route a correction, but only planlint declares
  `PASS`.

**Grok**

- Human-approve and freeze one contract first, then invoke native `/goal` with
  the exact artifact path and hash. Treat the hidden goal plan as derived
  execution state, not the approval authority.
- Reuse native planner/worker/evaluator/skeptic orchestration, token accounting,
  pause/resume, and retry bounds. Require the accepted plan's deterministic
  verifier command and receipt as a gating step that native skeptics can audit.
- Enforce allowed paths preventively with permissions/hooks/sandbox and again
  mechanically from the final diff. Prefer a worktree per run. Treat every
  pause state as terminal review; resume only after its recorded cause changes.

