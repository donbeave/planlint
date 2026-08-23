# Goal authority

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## `/goal` is persistence, not authority

This distinction is critical.

### Claude Code

Claude Code’s `/goal` keeps a session running across turns and uses a small
model after each turn to decide whether the completion condition appears
satisfied. Its evaluator does not independently run commands or inspect files;
it evaluates what was surfaced in the transcript. Claude Code also distinguishes
`/goal` from a Stop hook: a Stop hook can run a deterministic script.

**Planning and decomposition:** `/goal` does not create a separate plan or DAG.
The condition itself is the directive, and the normal Claude session decides
what tools and intermediate steps to use. A user can prepare a plan in a file,
but the documented goal mechanism does not freeze or validate that plan.

**Verification, scope, and approval:** the condition must name a measurable
result and the command/output that demonstrates it, because the evaluator cannot
read files or run commands. Manual permission mode still prompts for tool
approval on each goal turn; auto mode only changes permission behavior, not the
goal’s completion semantics. The documented goal path has no changed-path
allowlist or independent scope proof. A user can clear the goal or interrupt
the session; authentication, exhausted credits, unrepairable context overflow,
and unavailable-model errors clear it automatically.

**Context and isolation:** turns remain in the same session and resume restores
the condition, while the turn count, timer, and token baseline reset. `/goal`
does not create a clean worktree or fresh execution context. Background work
defers evaluation until its result is surfaced, so a controller should avoid
using hidden background tasks as acceptance evidence.

```text
Claude /goal evaluator = keeps trying
Rust verifier          = decides whether work passed
```

The agent must surface the verifier’s final result in the transcript, but the
transcript judgment must never be the authoritative evidence.

### Codex

Current OpenAI documentation exposes native `/goal` in Codex. It describes one
durable objective, explicit files/docs to read first, commands or artifacts
that prove progress, and checkpointed work. The documented CLI setup uses the
`features.goals` flag when the command is not available.

Codex also documents a `Stop` hook with a `stop_hook_active` loop guard and a
JSON continuation decision. Returning this from the hook starts another turn
using the reason as the continuation prompt:

```text
{ "decision": "block", "reason": "<planlint diagnostic>" }
```

Therefore the Codex adapter can use either native `/goal` or a project-local
Stop hook. Native `/goal` is the least coupled first path; the hook is useful
when the repository needs deterministic enforcement at every turn:

```text
Stop hook runs planlint verify
PLAN PASS                         → exit 0 with no continuation decision
PLAN_GAP | BLOCKED | STALE | FAIL → persist receipt; exit 0; no continuation
ordinary verifier failure         → decision: block with compact diagnostics
```

`stop_hook_active` must cap consecutive automatic retries. The compiler, not
the hook, owns that retry policy and final state.

**Planning and decomposition:** Codex documents `/plan` as a separate mode and
recommends pointing a goal at the files, docs, issue, logs, or plan it must read
first, then working in checkpoints. Native goal state does not itself expose a
task DAG or require plan approval. The TUI’s normal permission and goal
replacement surfaces are the human control points: replacing an unfinished
goal prompts for confirmation, while pause/resume/clear and budget controls
remain user/system operations.

**Verification and scope:** native `/goal` records an objective and lifecycle
state, not a proof receipt. A repository must attach `planlint verify` (or an
equivalent command) through a Stop hook or explicit tool workflow. The hook must
check changed paths, acceptance evidence, and non-vacuous test/compiler/linter
output before it allows a terminal PASS. Do not use `codex exec "/goal ..."`
as a control-plane substitute for the TUI/native goal path; the inspected
maintainer discussion reports that `exec` treats it as ordinary text.

### Grok

Current Grok Build ships a native `/goal` and publishes its Rust source. The
command creates durable `GoalOrchestration`, captures token and Git baselines,
runs a hidden planner subagent, executes ordinary worker turns, evaluates each
turn with a strict tool-free classifier, and sends candidate completion to an
adversarial panel of tool-using skeptic subagents. Achieved quorum completes;
ordinary gaps are fed back; repeated gaps, repeated blockers, verifier caps,
budgets, contradictions, unverifiable requirements, and infrastructure errors
pause rather than pass. See the source-pinned trace in
[Goal-execution research](../research/goal-execution/04-implementation-traces.md#grok-build).

**Planning and approval:** native goal planning produces a structured
acceptance/verification plan and snapshots its original form for later
anti-weakening review, but the plan is hidden and not human-approved. Grok’s
separate documented `/plan` mode provides the plan preview and explicit
approval surface. A predictable adapter should therefore approve and freeze a
contract first, then point native `/goal` at that exact artifact and hash.

**Execution and verification:** native continuation, retry, pause/resume, and
model-verifier orchestration should be reused. The project controller still
owns deterministic `PASS`: run exact proof predicates, enforce a closed-world
changed-path set, and store a receipt. Native skeptics inspect the real diff,
changed files, tests, and captured outputs, but their quorum is probabilistic
and the harness does not mechanically deny undeclared paths.

### Rig

Rig is a Rust framework/runtime, not a host product with a native `/goal`
controller. Its latest release, [v0.42.0](https://github.com/0xPlaygrounds/rig/releases/tag/v0.42.0)
at
[`d5a34986a1ad57f1e9c5984b82f8d7438ffc717e`](https://github.com/0xPlaygrounds/rig/tree/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e),
contains no goal-specific command, objective record, continuation controller,
or goal evaluator. The closest primitives are a serializable `rig::AgentRun`,
`AgentRunner`, and per-run `AgentHook`s.

**Planning and decomposition:** Rig does not plan or decompose a goal. The
official architecture is provider/model + context + tools + agent; the host
chooses workflow, storage, permissions, and orchestration. Human approval and
an immutable plan contract therefore belong outside Rig.

**Execution and context:** `AgentRun` is a sans-IO `CallModel` → `CallTools` →
`Done` state machine. It serializes the accumulated conversation and completed
provider responses, so a custom driver can pause at a tool boundary and resume
the same run. This is run persistence, not a fresh executor context or a
durable goal. The high-level `AgentRunner` owns model/tool I/O, memory,
telemetry, and hooks, but current runner-level resume is an open issue; a
restored run cannot yet re-enter that full path without a custom driver. Its
conversation-memory append is not a goal checkpoint: append failure is logged
while the final response still returns.

**Verification and retry:** `AgentRunner::max_turns` bounds model calls within
one run. A tool-free model turn normally ends that run. A model-turn hook can
instead retry with verifier feedback or stop, but retries consume the same turn
budget and have no separate cap. None of these primitives supplies an outer
corrective retry policy, changed-path guard, requirement-to-proof binding, or
authoritative repository `PASS`. The official evals page is stale against
v0.42.0: the release deleted the unused evals module and `experimental` feature.

**Recommendation:** implement `/goal` as a host/controller layer over Rig. Use
`AgentRunner` for normal configured runs, `AgentHook` for per-run policy and
telemetry, `rig::AgentRun` only when the controller must own mid-run
approval and persistence, and `planlint` for the accepted contract, deterministic
verifier, retry/terminal state, and receipt. This is a supported integration
architecture, not native Rig behavior. The [interactive-agent roadmap](https://github.com/0xPlaygrounds/rig/issues/2118)
explicitly keeps host control and durable session/run-control work separate;
[runner resume](https://github.com/0xPlaygrounds/rig/issues/2244) remains open.

For calling real coding-agent products from that host, see
[Rig agent CLI adapters](../integrations/rig-cli/README.md). The adapter boundary is a child
process, ACP JSON-RPC session, or HTTP client—not a Rig model provider. The
controller must retain authority over the accepted contract, changed paths,
deterministic proof, retry budget, and final `PASS`.
