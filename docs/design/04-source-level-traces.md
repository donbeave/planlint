# Source-level implementation traces

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Source-level implementation traces

This section traces implementations that expose enough code or documentation
to follow the execution path. **Confirmed** means the behavior is visible in
the linked source or official documentation. **Supported inference** means the
architecture follows from several confirmed behaviors. **Unknown** means the
product does not expose enough implementation detail to verify the claim.

### Comparison

| System | Goal representation | Execution architecture | Completion authority | Scope, retry, isolation |
| --- | --- | --- | --- | --- |
| Claude Code | Session condition plus evaluator state | Persistent session loop; evaluator after each turn | Evaluator over surfaced transcript; deterministic Stop hook can be separate | Goal has a bounded turn/time condition; Stop blocks cap at eight; no documented changed-path guard |
| Codex | Persisted thread goal with objective, status, and optional token budget | State machine plus internal steering and idle continuation | Model can mark `Complete` or `Blocked`; user/system controls pause, resume, clear, and budgets | Replacement confirmation; persisted state; no goal-specific test/scope verifier in inspected source |
| OpenHands SDK example | Objective passed to `run_goal`; `GoalOutcome` | Normal agent conversation wrapped by external judge/re-prompt loop | Separate judge LLM auditing transcript evidence | Hard iteration cap; same conversation history; no automatic fork or closed-world path guard in the example |
| SWE-agent | Issue/problem statement plus agent/retry configuration | Tool/action loop, optionally wrapped by retry loop | Environment/task submission and configured retry reviewer | Cost limit; retry attempts hard-reset the environment; formatting/action retries are bounded |
| Agent Execution Harness | JSON operational plan and run artifact | Explicit action state machine | `finish --check` requires claims, evidence, tasks, rollback, and scope | Declared files, command policy, evidence policy, dependency waves; isolation metadata is advisory |
| agent-spec | Markdown Task Contract | Contract compiler plus lifecycle verifier | Mechanical lint, bound tests, and explicit non-`skip`/non-`uncertain` verdict | Allowed/forbidden paths and change-set checks; caller AI verifier is secondary and inspectable |
| Rig | No native goal object; serializable run state and host APIs | `rig::AgentRun` + `AgentRunner` + `AgentHook`; host verifier | Tool-free text normally ends a run; no native goal authority or evaluator | Per-run turn/retry bounds; run serialization has version/sensitivity caveats; runner resume remains open |
| Grok Build | Persisted `GoalOrchestration`, plan/baseline files, token and verifier state | Hidden planner → worker rounds → structured evaluator → adversarial skeptic panel | Harness applies verifier quorum; deterministic project verifier remains external | Built-in caps and pause states; real diff evidence; no mechanical closed-world path guard; worker remains in parent context |

### Codex: native persistent goal state machine

Codex is the strongest inspected example of a product-native goal object.
Unlike a normal prompt, `/goal` is attached to a thread, persists outside the
ordinary transcript, has lifecycle status, can resume after idle turns, and
has user/system controls distinct from model completion. The source trace is:

```text
/goal <objective>
  → TUI SlashCommand::Goal
  → feature gate and dispatch
  → GoalDraft / SetThreadGoalDraft
  → app-server thread/goal/set
  → persisted goal state
  → GoalRuntimeHandle::continue_if_idle
  → internal goal steering context
  → ordinary model/tool loop
  → model goal update or terminal user/system transition
```

**Confirmed.** The TUI declares `SlashCommand::Goal` and permits it during a
task in [`tui/src/slash_command.rs`](https://github.com/openai/codex/blob/main/codex-rs/tui/src/slash_command.rs).
`chatwidget/slash_dispatch.rs` routes inline arguments, handles `clear`,
`edit`, `pause`, and `resume`, constructs a `GoalDraft`, and sends
`SetThreadGoalDraft`; see
[`dispatch_prepared_command_with_args`](https://github.com/openai/codex/blob/main/codex-rs/tui/src/chatwidget/slash_dispatch.rs#L3735-L3932).
The goal menu materializes the draft and calls the app-server goal setter in
[`thread_goal_actions.rs`](https://github.com/openai/codex/blob/main/codex-rs/tui/src/app/thread_goal_actions.rs).
Replacing an active, paused, blocked, usage-limited, or budget-limited goal
requires confirmation; replacing a completed goal does not.

**Confirmed.** The app-server exposes `thread/goal/set`, `get`, and `clear`
and persists goal updates. The request path is visible in
[`thread_goal_processor.rs`](https://github.com/openai/codex/blob/main/codex-rs/app-server/src/request_processors/thread_goal_processor.rs);
the extension surface is assembled in
[`ext/goal/src/lib.rs`](https://github.com/openai/codex/blob/main/codex-rs/ext/goal/src/lib.rs).
The goal tool accepts an objective and optional token budget in
[`ext/goal/src/tool.rs`](https://github.com/openai/codex/blob/main/codex-rs/ext/goal/src/tool.rs).
Create refuses a second unfinished goal. Model-driven update is deliberately
narrow: it may mark a goal `Complete` or `Blocked`; pause, resume, budgets,
and usage limits remain user/system controls.

**Confirmed.** On resume or an idle boundary, `GoalRuntimeHandle` restores the
active persisted goal and can inject a continuation in
[`ext/goal/src/runtime.rs`](https://github.com/openai/codex/blob/main/codex-rs/ext/goal/src/runtime.rs#L2034-L2174).
The objective, continuation text, token use, and remaining budget are added as
internal contextual user fragments tagged with the `goal` context source in
[`ext/goal/src/steering.rs`](https://github.com/openai/codex/blob/main/codex-rs/ext/goal/src/steering.rs).
This is more durable than repeating `/goal` in the visible chat, and it gives
the runtime a place to account for status and continuation.

**Supported inference.** The native design is a persistent state machine plus
prompt/context steering and a continuation controller. It is not an exposed
planner/executor/verifier DAG: `/plan` is a separate feature, and the inspected
goal path contains no independent compiler, test runner, linter, or changed-path
verifier. Ordinary tools can run commands and tests when the model chooses or
the objective requests them, but the goal state itself does not prove those
commands passed.

**Unknown.** The public source does not establish a deterministic rule that
`Complete` requires a successful test, compiler, or linter receipt. It also
does not show a goal-specific closed-world scope check, fresh context per retry,
or independent verifier. Codex’s official hook protocol can supply a separate
deterministic verifier, but that is an adapter/hook architecture, not native
goal semantics.

One operational caveat is confirmed by the Codex maintainer discussion in
[issue #26949](https://github.com/openai/codex/issues/26949): `codex exec
"/goal ..."` is ordinary text rather than guaranteed TUI slash-command
parsing. Automation should use the goal tool/control plane or an explicit hook,
not assume TUI command parity in `exec`.

### OpenHands: external judge loop around a normal conversation

OpenHands’ SDK includes a directly readable goal-completion example. Its trace
is intentionally different from Codex:

```text
objective + conversation + independent judge LLM
  → conversation.run()
  → judge audits surfaced transcript evidence
  → complete: GoalOutcome(complete)
    or missing evidence: feedback prompt → conversation.run()
  → hard cap: GoalOutcome(capped)
```

**Confirmed.** The example’s docstring says plain `conversation.run()` stops
when the agent thinks it is finished; `run_goal` adds a second judge that
audits file contents, command output, and test results, then re-prompts until
the judge accepts or the iteration cap is reached. The implementation and
example objective are in
[`54_goal_completion_loop.py`](https://github.com/OpenHands/software-agent-sdk/blob/main/examples/01_standalone_sdk/54_goal_completion_loop.py#L344-L434).
It creates separate `agent_llm` and `judge_llm` instances, requires
`python -m pytest -q`, and sets `max_iterations=3`.

**Confirmed.** The example returns a `GoalOutcome` with `complete` or
`capped`, iteration count, and the final verdict. It drives the supplied
conversation in place; the objective, work, and judge follow-ups remain in
the same `conversation.state.events` history. The SDK separately supports
persistent event state, iteration limits, budget limits, stuck detection, and
forking in
[`local_conversation.py`](https://github.com/OpenHands/software-agent-sdk/blob/main/openhands-sdk/openhands/sdk/conversation/impl/local_conversation.py),
but this goal example does not fork.

**Supported inference.** This is a planner/executor/verifier loop, but the
planner is only the judge’s missing-evidence feedback and the executor remains
the same agent conversation. The judge is stronger than self-reported
completion because it is a separate model instance, but the inspected example
still asks the judge to inspect transcript evidence rather than independently
run the repository’s commands. A deterministic verifier should therefore sit
under or beside this loop.

**Planning and approval:** there is no separate planner or human approval
surface in the shown `run_goal` function. The agent LLM plans implicitly inside
the conversation; the judge supplies corrective next-step feedback. If a
human-approved plan is required, it must be established before calling
`run_goal` and persisted independently of the conversation.

**Unknown.** The public example does not define an allowed-path policy,
transactional workspace, approval gate, or judge tool permissions. Those may be
provided by the surrounding SDK/runtime, but they are not part of the shown
`run_goal` contract.

### SWE-agent: bounded retries and fresh attempts

SWE-agent has no native `/goal` command in the inspected interface, but its
issue-execution loop supplies a useful equivalent pattern:

```text
problem statement + retry configuration
  → DefaultAgent action/observation loop
  → submission/reviewer result
  → retry decision
  → hard-reset environment + new attempt
  → choose best saved trajectory
```

**Confirmed.** `RetryAgent.run` repeatedly steps the current agent, saves the
trajectory, submits the completed attempt to a configured retry loop, and
starts another attempt when the retry loop asks for one. `_next_attempt` calls
`hard_reset()` before setting up the next agent. See
[`sweagent/agent/agents.py`](https://github.com/SWE-agent/SWE-agent/blob/main/sweagent/agent/agents.py#L257-L440).
The retry loop has a cost limit and per-attempt budget; each attempt receives
its own output directory and trajectory.

**Confirmed.** The default agent bounds model re-queries after formatting,
blocked-action, content-policy, and shell-syntax failures with
`max_requeries=3`; the error is converted into corrective history before the
next model call. See
[`DefaultAgent.forward`](https://github.com/SWE-agent/SWE-agent/blob/main/sweagent/agent/agents.py#L1088-L1145).

**Supported inference.** This architecture separates recovery from ordinary
tool execution and prevents a failed attempt’s filesystem mutations from
contaminating the next attempt. It is a strong pattern for bounded retry and
fresh context, but not a complete `/goal`: the inspected path does not define a
portable Markdown objective, acceptance contract, changed-path guard, or
independent deterministic verifier. SWE-bench evaluation supplies external
task validation rather than the goal command itself.

**Planning and approval:** the inspected retry path receives a problem
statement/configuration, not a user-approved plan or explicit decomposition
graph. Reviewer/retry configuration chooses whether to make another attempt;
the source path exposes no human approval gate. This makes its hard-reset and
budget controls reusable, but not its task-contract semantics.

### Rig: host-extensible run state, not native `/goal`

**Confirmed.** The current source separates protocol state from I/O. The
[`rig::AgentRun` module](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/run/mod.rs#L1-L34)
describes a serializable, sans-IO state machine whose driver handles model and
tool effects. Its public step protocol is
[`CallModel` / `CallTools` / `Done`](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/run/mod.rs#L154-L179);
there is no goal/objective/status state in that protocol.
Post-release `main` at
[`2b271b66ca21b5baa230e42589ca00184f43af59`](https://github.com/0xPlaygrounds/rig/tree/2b271b66ca21b5baa230e42589ca00184f43af59)
extracts this state machine into `rig-run`; it does not add goal semantics.

**Confirmed.** [`AgentRunner`](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/runner.rs#L142-L220)
is the configured high-level driver. It owns model/tool I/O, conversation
memory, tracing, and the hook stack. Its `max_turns` is a per-run model-call
budget. [`AgentRun` completes on an accepted tool-free model
turn](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/run/mod.rs#L718-L955),
so text means run completion rather than proof of the objective.

**Confirmed.** [`AgentHook`](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/hook.rs#L584-L704)
can retry or stop model turns and observe or steer tool events. A goal adapter
can run a deterministic verifier before accepting a tool-free turn, retry with
bounded feedback on a correctable failure, and stop on terminal non-success.
The hook scratchpad is ephemeral; tools and hooks are not serialized inside
`AgentRun`.

**Confirmed documentation drift.** The [evals concepts page](https://rig.rs/docs/concepts/evals)
describes APIs absent from v0.42.0. The release [deleted the unused evals module
and `experimental` feature](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-core/CHANGELOG.md#L481-L506),
and the [release feature list](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/Cargo.toml#L324-L385)
has no `experimental` flag. Current Rig therefore has no built-in evaluator.

**Confirmed.** The [Rig interactive-agent roadmap](https://github.com/0xPlaygrounds/rig/issues/2118)
puts workflow, permissions, storage, and orchestration in the host layer. The
[runner resume issue](https://github.com/0xPlaygrounds/rig/issues/2244) says a
serialized `AgentRun` can be hand-driven across a process boundary, but the
high-level runner does not yet accept a restored run with hooks, tool-server
dispatch, memory, and telemetry intact.

**Supported inference.** Planlint should integrate below its contract compiler:
`AgentRunner` for ordinary execution, `AgentHook` for per-run policy, and
`rig::AgentRun` only for a custom driver that accepts ownership of model
transport and tool execution. Planlint remains the owner of the durable goal,
plan, deterministic verifier, scope policy, retry cap, and `PASS` receipt.

### Deterministic contract/harness references

These are not host agents, but they expose the pieces missing from the native
goal loops.

**Agent Execution Harness — confirmed.** The TypeScript harness turns each
agent request into typed actions over a persisted run state in
[`src/core/runner.ts`](https://github.com/lordaeternus/agent-execution-harness/blob/main/src/core/runner.ts#L20-L56).
`finish --check` refuses completion when tasks, gates, claims, evidence,
rollback, or scope checks are incomplete in
[`src/core/finish-check.ts`](https://github.com/lordaeternus/agent-execution-harness/blob/main/src/core/finish-check.ts#L21-L80).
Its scope guard compares declared paths with the actual Git worktree in
[`src/core/scope-guard.ts`](https://github.com/lordaeternus/agent-execution-harness/blob/main/src/core/scope-guard.ts#L20-L36).
The repository documents dependency waves and worker handoff validation, but
worker `isolation` is advisory; it does not automatically create worktrees or
sandboxes. This is a stateful execution harness, not an LLM planner.
The documented workflow treats plan approval as an external human step before
execution; the harness then owns evidence-backed completion.

**agent-spec — confirmed.** The Rust project compiles Markdown Task Contracts
into lifecycle checks. `quality_gate` rejects lint errors or insufficient
quality; `verify` and `is_passing` reject failed, skipped, or uncertain
scenarios in
[`src/spec_gateway/lifecycle.rs`](https://github.com/ZhangHanDong/agent-spec/blob/main/src/spec_gateway/lifecycle.rs#L65-L229).
The project’s documentation binds scenarios to named tests, enforces
allowed/forbidden changed paths, and supports a caller AI verifier whose
uncertain decisions are written as explicit requests and decisions rather than
treated as automatic PASS. This is a contract compiler/verifier, not a
persistent goal controller. Its documented acceptance surface separates human
review of the contract from machine review of verification results.

### Closed-source boundary: Claude Code

**Confirmed.** Claude Code documents `/goal` as a per-session continuation
condition with a small evaluator after each turn. The evaluator receives the
condition and surfaced conversation, does not independently read files or run
commands, and has bounded conditions/turn behavior. Its deterministic Stop
hook is a separate control path. See the [goal documentation](https://code.claude.com/docs/en/goal)
and [hooks documentation](https://code.claude.com/docs/en/hooks).

**Supported inference.** Claude can host a deterministic verifier through Stop
hooks, but its native goal evaluator remains a continuation authority rather
than a repository-state authority.

**Unknown.** Claude Code does not expose its source-level goal state machine,
planner decomposition, model-facing evaluator prompt, retry persistence, or
changed-path enforcement. Claims about those internals would be speculation.

Grok Build is no longer part of this closed-source boundary. Its public
source-level trace is maintained in
[Goal-execution research](../research/goal-execution/04-implementation-traces.md#grok-build).
