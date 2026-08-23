# Concrete operating model

[← Research index](../README.md) · [Goal-execution index](README.md)

> Preserved from the original goal-execution research report.

## Concrete operating model for predictable execution

Use one immutable execution contract per goal:

```text
goal id and exact objective
accepted base commit and input fingerprints
allowed and forbidden paths
requirement ids
one coherent vertical slice
exact final commands
structured proof predicates
maximum corrective retries
terminal states: PASS | PLAN_GAP | STALE | BLOCKED | FAIL
```

The external verifier must own `PASS`. A passing process exit alone is not
enough: it must prove non-zero relevant work, such as named tests, compiler
artifacts, a structured lint report, or a bounded diff. Store the plan hash,
commands, evidence hashes, and changed paths in a receipt.

### Claude Code

1. Plan and approve one bounded contract before `/goal`; do not place a backlog
   in the condition.
2. For the native profile, make the condition require one exact verifier
   command, its canonical `PASS <goal-id> <receipt-hash>` output, the allowed
   path set, and a fixed turn/time cap. Claude's evaluator sees only transcript
   evidence, so require the worker to surface that exact line. After Claude
   returns, the outer process must independently read and validate the receipt.
3. For the strict profile, use a deterministic command Stop hook *instead of*
   native `/goal`. `/goal` is itself a prompt Stop hook; stacking two completion
   controllers creates ordering ambiguity. The command hook runs the verifier,
   blocks only for bounded correctable failures, and records terminal
   `PLAN_GAP`, `STALE`, `BLOCKED`, or `FAIL` without another continuation.
4. Use Auto mode only after reviewing plan and tool permissions. On cap,
   impossible, or failed receipt, replan rather than append unrelated work to
   the old session.

Native payload shape:

```text
/goal Goal CCG-42 is met only when `planlint verify .agent/goals/CCG-42.md
--receipt .agent/receipts/CCG-42.json` prints `PASS CCG-42 <receipt-sha>`;
the receipt lists only the contract's allowed paths and every required gate is
non-vacuously proven. Surface that exact line. Otherwise continue from the
verifier diagnostic, or report impossible after 12 evaluated turns.
```

### Codex

1. Run planning separately, freeze the contract, then create one native
   `/goal` for its execution. The source shows goal accounting is not active for
   Plan-mode turns.
2. Let native goals supply persistence, progress accounting, pause/resume, and
   continuation. Require `update_goal(complete)` only after a deterministic
   verifier has emitted and persisted `PASS`; the tool call remains a
   model-mediated lifecycle transition, not acceptance authority.
3. Do not stack native goal continuation and an independent Stop-hook loop
   unless their state machines are explicitly coordinated. If a Stop hook is
   the strict controller, use it as the goal equivalent: bound retries with
   `stop_hook_active`, persist terminal non-success, and do not rely on native
   goal status.
4. The outer process revalidates frozen inputs, receipt hash, and changed paths
   before final `PASS`. Needed work outside the contract produces `PLAN_GAP`,
   never an autonomous redesign.

Native payload shape:

```text
/goal Execute exactly .agent/goals/CX-42.md at the recorded SHA-256. Keep this
goal active until `planlint verify .agent/goals/CX-42.md --receipt
.agent/receipts/CX-42.json` returns `PASS CX-42 <receipt-sha>`. Apply only the
verifier's bounded diagnostics. Call update_goal complete only after PASS; call
it blocked only under the native repeated-blocker rule.
```

### Rig

1. Treat Rig as the execution substrate, not the `/goal` controller. Keep the
   objective, accepted contract, evidence receipt, retry budget, and terminal
   status in an outer controller.
2. Use `AgentRunner` for normal configured runs and `AgentHook` for per-run
   policy, telemetry, and bounded model-turn retry/stop decisions. Do not use a
   hook's run-scoped scratchpad as durable goal state.
3. Use `rig::AgentRun` for approval-sensitive mid-run persistence only when
   the controller is prepared to own model transport, tool execution, and
   version compatibility. Otherwise run corrective rounds through
   `AgentRunner` with explicit persisted controller state.
4. Run deterministic proof commands and changed-path enforcement in the
   external verifier. A custom evaluator agent may route corrections, but only
   the verifier may emit `PASS`.

### Grok

1. Produce and human-approve one contract first, then invoke native `/goal`
   with an objective that names that exact file and hash. Grok's hidden planner
   is useful decomposition, but it is not the approval boundary.
2. Let native goal mode own persistence, worker continuation, evaluator
   routing, skeptic retries, pause/resume, and token accounting. Do not rebuild
   those layers with a slash skill or passive Stop hook.
3. Put the deterministic verifier command and expected structured receipt in
   the approved contract's gating verification plan. Require the worker to save
   its output; Grok's skeptics are designed to audit those files.
4. Add a preventive `PreToolUse`/sandbox path policy and a final mechanical
   diff check. Native verification sees the complete changed-file list but does
   not itself enforce a closed-world allowlist.
5. Use explicit token and verifier-attempt bounds. Treat no-progress,
   infrastructure, contradiction, unverifiable, and budget states as terminal
   review points; resume only after the recorded cause changes. Prefer a
   dedicated worktree for every goal.

Native payload shape:

```text
/goal Execute exactly .agent/goals/GX-42.md at the recorded SHA-256. Preserve
its acceptance criteria and allowed-path set in the derived goal plan. Run
`planlint verify .agent/goals/GX-42.md --receipt
.agent/receipts/GX-42.json`; completion requires `PASS GX-42 <receipt-sha>`.
Treat PLAN_GAP, STALE, BLOCKED, or FAIL as non-fixable terminal evidence.
--budget 150000
```

