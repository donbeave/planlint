# planlint research archive

Status: design basis, not implementation.

Reviewed: 2026-08-23.

## Verdict

This is a strong concept. The right product is not merely a Markdown linter;
it is a Rust **plan-contract compiler** with a deterministic verification
runtime.

The proposed execution model is:

```text
accepted specification
        ↓
pure-Markdown plan item
        ↓
Rust static checker
        ↓
one fresh /goal execution
        ↓
deterministic verifier
        ↓
evidence receipt
        ↓
DONE or explicit failure state
```

The agent remains probabilistic while producing code, but acceptance becomes
deterministic. The useful Rust analogy is:

```text
Rust source → compiler diagnostics → developer edits → compiler passes

Plan contract → verifier diagnostics → agent edits → verifier passes
```

Research supports the direction. CodePlan framed repository-level changes as
dependency-aware planning and, in its FSE evaluation, got 5 of 7 repositories
through validity checks while equivalent non-planning baselines got none
through. The August 2026 SWE-RPG preprint reports that current coding agents
resolved 31.5% of its tasks on average and identifies recovery of implicit
requirements as a major failure source. Small plans and mechanical gates help,
but the tool also needs an explicit **`PLAN_GAP`** state instead of allowing
the executor to guess missing requirements.

These findings do not prove that this exact product will work. They justify a
purpose-built evaluation.

## The “150K smart context” correction

The 150K hypothesis is directionally useful, but **150K is not a scientifically
established universal smart-context threshold**.

Long-context research repeatedly shows that advertised capacity is not the
same as effective reasoning capacity:

- *Lost in the Middle* found that models often perform much worse when
  important information is buried in the middle of a long context.
- RULER evaluated 17 long-context models and found that only half maintained
  satisfactory performance at 32K, despite all claiming at least 32K support.
- NoLiMa evaluated 13 models claiming at least 128K contexts and found that 11
  fell below half of their short-context baseline by 32K on tasks requiring
  associative retrieval rather than direct keyword matching.

These are not coding-agent benchmarks. They do not prove that coding quality
collapses at 32K or that 150K is optimal. They establish a more useful rule:

> Minimize the task’s effective working set instead of trying to fill the
> advertised context window.

The working set is not just plan Markdown:

```text
working set =
    system and repository instructions
  + skills loaded by the agent
  + the plan item
  + referenced requirements and decisions
  + mandatory source files
  + compiler and test output
  + patch history
  + reasoning and retry reserve
```

Therefore, `150K` should be a configurable **context profile**, not a claim
embedded in the linter.

Current product context windows make the distinction more important, not less:
the current OpenAI model catalog lists 1.05M-token context for GPT-5.6, while
GPT-5.2-Codex lists 400K; current xAI documentation lists 500K for Grok 4.6.
Those are capacity numbers, not evidence that an agent can keep all repository
facts active. The linter should enforce a measured working-set budget and
calibrate it per host/model, rather than treating 150K as a model limit.

Coding-agent-specific evidence makes the rule more precise:

- *The Working Set of a Coding Agent* reports that missing task facts caused
  agents to guess wrong, while additional token spend did not recover withheld
  facts. Passing harnesses differed by more than 10× in token use because some
  repeatedly reconstructed the same context.
- *When and How Context Rot Appears in Coding Agents* reports 8/10 passes with
  a 10,991-character clean context versus 3/10 with 299,140-character relevant
  or irrelevant contexts on one audit task. A second task passed in every
  condition, so this does not establish a universal threshold. A detailed
  external checklist passed 10/10 versus 5/10 for a generic self-check.
- *Coding Agents are Effective Long-Context Processors* reports that agents can
  use filesystems and executable tools to process corpora far larger than their
  prompt windows. This argues against inlining the whole repository into a
  supposedly self-contained plan.

The design target is therefore not “smallest prompt.” It is **smallest complete
working set**: every required fact available, irrelevant history excluded,
exact inputs addressable through tools, and an external checklist enforced by
the verifier.

Initial conservative profile to evaluate:

```toml
[profiles.goal-150k]
hard_peak_tokens = 150000
target_peak_tokens = 100000
max_initial_packet_tokens = 60000
max_plan_tokens = 12000
max_changed_paths = 8
max_steps = 8
max_acceptance_gates = 2
max_tool_output_bytes = 250000
```

These are starting policies, not universal values. Calibrate separately for
Claude, Codex, Grok, model class, repository, and task type.

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
[Goal-execution research](goal-execution.md#grok-build).

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
[Goal-execution research](goal-execution.md#grok-build).

## Closest existing work

### Markdown spec workflows establish the authoring precedent

GitHub Spec Kit and OpenSpec already make Markdown the repository-visible
authoring surface. Spec Kit separates specification, plan, and actionable task
artifacts and adds a convergence loop; OpenSpec uses plain Markdown
requirements and concrete scenarios, then generates `design.md` and `tasks.md`
for a change.

They validate the usability and portability of Markdown across coding agents.
They do not, by themselves, define the proposed authoritative runtime
contract: immutable acceptance, closed-world changed-path checking,
non-vacuous evidence, input fingerprints, or a verifier-owned PASS state.

Therefore `planlint` should interoperate with these workflows rather than
compete with their planning UX:

```text
Spec Kit / OpenSpec / human Markdown
    → reviewed bounded plan
    → planlint check + accept
    → host /goal execution
    → planlint verify
```

### `agent-spec` is the closest technical foundation

The strongest related project found is the Rust project `agent-spec`. It
describes itself as an **intent compiler** and implements this pipeline:

```text
human intent
→ requirements
→ Task Contracts
→ implementation
→ deterministic lifecycle verification
→ trace and liveness
```

Its Task Contracts are Markdown documents containing Intent, Decisions,
Boundaries, and Completion Criteria. Completion scenarios bind explicitly to
tests, and intermediate gates are intended to be deterministic and model-free.

A real `agent-spec` task contract is close to the needed specification layer:

- requirement links through `satisfies`
- allowed paths
- forbidden changes
- out-of-scope work
- explicit completion scenarios
- named proving tests

Its architecture separates code intelligence, typed code bindings, quality
providers, execution bundles, lifecycle verdicts, and trace evidence.

Its mismatch is that it primarily represents a **behavioral contract**, not a
`/goal` execution item. It does not fully represent:

- exact ordered implementation steps
- per-step verification
- preconditions and starting-state drift
- one-fresh-context execution
- whole-working-set context budgets
- explicit `PLAN_GAP` and STOP behavior
- immutable plan fingerprints
- host-specific `/goal` continuation adapters

It correctly acknowledges that passing a contract does not prove the contract
was comprehensive. That limitation must remain visible in this design.

### Agent Execution Harness validates much of the runtime idea

Agent Execution Harness has narrow tasks, typed evidence, strict command
allowlists, compact handoffs, explicit verification, completion checking, and
worker-patch validation. In strict mode, shell-style commands are blocked
unless declared.

Its architectural mismatch is important: it imports an atomic Markdown backlog
into `plan.json`, then uses JSON as the operational plan. It is also implemented
in TypeScript.

It validates demand for an execution harness while leaving a clear place for
this narrower design:

> Rust, canonical Markdown, compiler diagnostics, context budgeting, and
> native `/goal` adapters.

### Tailrocks already contains most contract semantics

The existing Tailrocks plan workflow, as supplied for this research, already
requires:

- one zero-context plan per work item
- vertical, independently verifiable slices
- one fresh executor session
- explicit paths and code shapes
- preconditions
- in-scope and out-of-scope paths
- Must NOT constraints
- done criteria
- STOP conditions
- verification commands run during planning

The goal handoff freezes the plan package with a fingerprint and requires each
final gate to have the form:

```text
command ||| proof
```

The proof must demonstrate that non-zero work executed, specifically to prevent
commands that exit successfully after running zero tests.

The new Rust project should not redesign that workflow. It should compile and
enforce the workflow already present.

## Three contract layers

The clean architecture has three document types.

### 1. Specification contract — what must be true

Contains product behavior, decisions, scenarios, and forbidden behavior:

```text
REQ-RETRY-01:
The client MUST stop after the configured attempt limit.
```

`agent-spec` is a plausible backend for this layer.

### 2. Plan contract — one bounded change that produces it

Declares:

- exact outcome
- requirements covered
- dependencies
- starting inputs
- files or symbols allowed to change
- steps
- focused checks
- final proof obligations
- STOP conditions
- context budget

This is the new Rust compiler’s input.

### 3. Goal contract — when the executor may continue or stop

Should remain extremely small:

```text
Execute exactly PLAN-014.
Continue until the authoritative verifier prints PLAN PASS.
Do not broaden scope.
On PLAN_GAP, BLOCKED, STALE, or FAIL, persist that state and stop.
```

The separation prevents duplication:

```text
specification = what
plan          = how, for one bounded slice
goal          = runtime continuation rule
```

The plan references specification IDs rather than copying the entire
specification. Where zero-context execution requires an excerpt, the compiler
can assemble it into a temporary execution packet without creating a second
authored source of truth.

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

## Verification must be non-vacuous

A simple verification command is necessary but not sufficient.

Weak gates include:

```text
cargo test exits 0
npm test succeeds
the project builds
```

They can pass when:

- the package name no longer resolves as expected
- a test filter matches zero tests
- all matching tests are ignored
- the command checks the wrong workspace
- expected output was never generated
- a test command succeeds without exercising the changed entry point

A strong gate has three parts:

```text
command
+ structured evidence source
+ proof predicate
```

Example:

```markdown
- **Run:** `mise run test-client-retry`
- **Evidence:** `junit:target/nextest/ci/junit.xml`
- **Proves:** `tests >= 3 && failures == 0`
```

First proof adapters:

1. `junit` — test count, failures, skipped tests, named cases.
2. `cargo-json` — packages, targets, compiler artifacts, diagnostics.
3. `sarif` — static-analysis findings.
4. `git-diff` — changed, created, deleted, and renamed paths.
5. `file` — path existence, count, hash, and exact-content predicates.
6. `command-json` — repository-specific tools that emit a documented JSON schema.

Arbitrary terminal-text regexes should be lower-assurance fallback only.
Structured reports are more stable.

The command runner declares:

```text
executable
arguments
working directory
timeout
output limit
network policy
environment allowlist
```

Strict mode rejects shell pipes, redirections, command substitution, and
undeclared variables. A human may explicitly opt into shell execution for a
trusted repository; it is not the default.

## Context-budget checker

The linter computes both **authored size** and **working-set estimate**.

### Authored size

```text
plan Markdown tokens
+ inlined requirement excerpts
+ inlined command output examples
```

### Working-set estimate

```text
base host instructions
+ applicable AGENTS.md / CLAUDE.md
+ selected skills
+ plan
+ required spec excerpts
+ required source inputs
+ dependency-plan summaries
+ expected command output
+ reserved execution history
```

Every input is explicit. Broad discovery instructions such as these should be
warnings or errors:

```text
Read the codebase.
Inspect all relevant files.
Update anything necessary.
```

Prefer:

```markdown
## Inputs

- `crates/client/src/retry.rs#RetryPolicy`
- `crates/client/src/error.rs#ClientError`
- `crates/client/tests/retry.rs`
```

At `probe` time, resolve inputs and record:

- file hashes
- symbol locations
- byte count
- estimated tokens
- accepted commit
- expected tool-output budget

A broad glob may be permitted, but it must be expanded and frozen during
acceptance. If `crates/client/**` resolved to 41 files when accepted and later
resolves to 47, the plan becomes `STALE` until reviewed.

The runtime receipt records estimated and actual context use. Claude’s current
`/goal` status reports token spend, which can be captured for calibration.

## Slicing discipline

One small task does not mean one arbitrary code fragment.

A good plan item is:

- small enough for one fresh context
- a complete vertical behavior slice
- independently verifiable
- dependency-aware
- able to leave the repository green
- closed enough that the executor does not need a new architectural decision

Poor split:

```text
PLAN-001: Add database types
PLAN-002: Add service
PLAN-003: Add API
PLAN-004: Add tests
```

The first three plans have no independent behavioral proof.

Better split:

```text
PLAN-001: Establish a passing skeleton and verification baseline
PLAN-002: Support one end-to-end successful flow
PLAN-003: Add one failure flow with its proof
PLAN-004: Migrate remaining callers
PLAN-005: Remove the old path
```

The compiler checks write-set overlap:

```text
PLAN-014 writes retry.rs
PLAN-015 writes retry.rs
no dependency edge exists
```

That produces a dependency error or forces sequential execution. Plans are
safely parallel only when dependencies are verified and resolved write sets are
disjoint.

## Runtime state is separate from the plan

The Markdown plan is immutable after acceptance. Do not put mutable checkboxes
or current status inside the plan contract.

Use a separate state machine:

```text
TODO
  ↓
CLAIMED
  ↓
CANDIDATE
  ↓
VERIFIED
  ↓
DONE
```

Failure states:

```text
BLOCKED   — environment or precondition failure
PLAN_GAP  — correct implementation requires unapproved work
STALE     — plan or starting inputs changed
FAIL      — proof failed within the permitted correction loop
REJECTED  — human explicitly chose not to execute it
```

Only the Rust controller transitions state. The agent reports evidence but
cannot mark itself `DONE`.

Generated receipts may be JSON because they are output artifacts, not the
authored contract:

```json
{
  "plan_id": "PLAN-014",
  "contract_hash": "blake3:...",
  "base_commit": "4f12a8c",
  "head_commit": "98ca301",
  "status": "verified",
  "changed_paths": [
    "crates/client/src/retry.rs",
    "crates/client/tests/retry.rs"
  ],
  "checks": [
    {
      "id": "ACC-1",
      "tests": 3,
      "failures": 0,
      "evidence_hash": "blake3:..."
    }
  ],
  "unexpected_paths": 0
}
```

Bind the receipt to:

```text
accepted plan hash
+ accepted starting commit
+ resulting commit
+ changed paths
+ exact commands
+ structured evidence hashes
```

This makes the result replayable and auditable.

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

## Recommended Rust architecture

Do not begin by writing a complete multi-agent orchestrator. Build the smallest
authoritative core first.

```text
crates/
├── plan-contract/
│   ├── Markdown parser
│   ├── typed PlanContract IR
│   ├── source spans
│   ├── lint rules
│   ├── dependency graph
│   └── context estimator
│
├── plan-proof/
│   ├── command policy
│   ├── JUnit adapter
│   ├── Cargo JSON adapter
│   ├── SARIF adapter
│   ├── Git diff adapter
│   └── evidence receipts
│
├── plan-hosts/
│   ├── Claude Code hook renderer
│   ├── Codex hook renderer
│   └── Grok skill and headless/ACP controller
│
└── planlint/
    └── CLI
```

Practical stack:

- `comrak` for CommonMark/GFM parsing and AST traversal
- `miette` for Rust-like diagnostics with source spans
- `petgraph` for dependency graphs and cycle detection
- `cargo_metadata` for Cargo metadata and JSON message processing
- `globset`, `ignore`, and `camino` for path handling
- `blake3` for fingerprints
- `serde` for generated receipts, never the canonical plan format
- `clap` for the CLI

Suggested commands:

```bash
# Static, side-effect-free compilation
planlint check roadmap/<slug>/plan/

# Resolve inputs and execute every proposed proof once
planlint probe roadmap/<slug>/plan/014-bounded-retry.md

# Record human acceptance and freeze the contract
planlint accept roadmap/<slug>/plan/014-bounded-retry.md

# Render the minimal context packet
planlint packet roadmap/<slug>/plan/014-bounded-retry.md

# Verify current changes
planlint verify roadmap/<slug>/plan/014-bounded-retry.md --worktree

# Reconcile plan state against repository evidence
planlint reconcile roadmap/<slug>/plan/

# Generate host integrations
planlint host install claude
planlint host install codex
planlint host install grok   # skill plus headless/ACP controller

# Explain a diagnostic
planlint explain PL041
```

Output formats:

```text
human compiler diagnostics
JSON
SARIF 2.1.0
compact agent diagnostics
```

Compact agent output contains only:

```text
diagnostic ID
file and span
observed fact
required fact
one suggested correction
```

This prevents the verifier from consuming excessive context.

## Combining with `agent-spec`

Do not fork all of `agent-spec` immediately.

Use this division:

```text
agent-spec
    validates the behavioral/specification contract

planlint
    validates the bounded execution contract

host continuation adapters
    provide native /goal, hook, or outer-controller persistence

Tailrocks reconcile
    validates package-level completion and state
```

A plan acceptance criterion can invoke `agent-spec` directly:

```markdown
### ACC-4: Behavioral contract

- **Run:** `agent-spec lifecycle specs/client-retry.spec.md --code .`
- **Evidence:** `agent-spec-json:stdout`
- **Proves:** `passed >= 2 && failed == 0 && skipped == 0 && uncertain == 0`
```

This keeps “what the product must do” separate from “how one executor should
modify the repository.” Initially integrate through the stable CLI and pin an
accepted `agent-spec` version. Upstream shared parser or evidence APIs later if
the experiment proves useful.

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

## Delivery sequence

### Version 0.1 — static plan compiler

Implement only:

- strict Markdown grammar
- source-span diagnostics
- IDs and required sections
- dependency DAG
- write-scope overlap
- requirement coverage
- closed-world scope
- proof syntax validation
- context estimation
- contract fingerprinting
- fixtures and mutation tests

No command execution and no LLM-based rules.

### Version 0.2 — proof engine

Add:

- `probe`
- `verify`
- JUnit
- Cargo JSON
- SARIF
- Git scope verification
- non-vacuity checking
- strict argv execution
- timeout and output limits
- evidence receipts
- clean-worktree verification

### Version 0.3 — `/goal` controller

Add:

- Claude Code Stop hook
- Codex native `/goal` handoff and Stop hook
- Grok skill plus headless/ACP continuation controller
- one-plan-per-fresh-session scheduler
- runtime state machine
- actual context telemetry
- reconcile
- automatic blocking diagnostics

Only after these versions are evaluated should the tool consider parallel
agents, remote sandboxes, or broader orchestration.

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

## Source links

- [CodePlan — Microsoft Research](https://www.microsoft.com/en-us/research/publication/codeplan-repository-level-coding-using-llms-and-planning-2/)
- [Lost in the Middle — arXiv:2307.03172](https://arxiv.org/abs/2307.03172)
- [RULER — arXiv:2404.06654](https://arxiv.org/abs/2404.06654)
- [NoLiMa — arXiv:2502.05167](https://arxiv.org/abs/2502.05167)
- [SWE-RPG / unified issue-resolution benchmark — arXiv:2608.09072](https://arxiv.org/abs/2608.09072)
- [Claude Code goals](https://code.claude.com/docs/en/goal)
- [Claude Code hooks](https://code.claude.com/docs/en/hooks)
- [Codex: Follow a goal](https://learn.chatgpt.com/use-cases/follow-goals)
- [Codex hooks](https://learn.chatgpt.com/docs/hooks)
- [Codex developer commands](https://learn.chatgpt.com/docs/developer-commands?surface=cli)
- [Rig v0.42.0](https://github.com/0xPlaygrounds/rig/releases/tag/v0.42.0)
- [Rig agent concepts](https://rig.rs/docs/concepts/agent)
- [Rig AgentRunner concepts](https://rig.rs/docs/concepts/agentrunner)
- [Rig `AgentRun` source](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/run/mod.rs)
- [Rig post-release `rig-run` extraction](https://github.com/0xPlaygrounds/rig/blob/2b271b66ca21b5baa230e42589ca00184f43af59/crates/rig-run/src/run.rs)
- [Rig `AgentRunner` source](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/runner.rs)
- [Rig hooks source](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/hook.rs)
- [Rig evals-removal changelog](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-core/CHANGELOG.md#L481-L506)
- [Rig interactive-agent roadmap](https://github.com/0xPlaygrounds/rig/issues/2118)
- [Rig runner resume issue](https://github.com/0xPlaygrounds/rig/issues/2244)
- [Grok skills, plugins, and marketplaces](https://docs.x.ai/build/features/skills-plugins-marketplaces)
- [Grok hooks](https://docs.x.ai/build/features/hooks)
- [Grok headless and scripting](https://docs.x.ai/build/cli/headless-scripting)
- [Grok plan mode](https://docs.x.ai/build/features/plan-mode)
- [Grok permissions](https://docs.x.ai/build/features/permissions)
- [Grok Build goal state](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/goal_tracker.rs#L422-L636)
- [Grok Build goal evaluator](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/goal_evaluator.rs)
- [Grok Build verifier panel](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/goal_classifier.rs#L1870-L2217)
- [Grok 4.6](https://docs.x.ai/developers/grok-4-6)
- [OpenAI model catalog](https://developers.openai.com/api/docs/models)
- [The Working Set of a Coding Agent](https://arxiv.org/abs/2608.16630)
- [Coding Agents are Effective Long-Context Processors](https://arxiv.org/abs/2603.20432)
- [When and How Context Rot Appears in Coding Agents](https://arxiv.org/abs/2607.17937)
- [GitHub Spec Kit](https://github.com/github/spec-kit)
- [OpenSpec](https://github.com/Fission-AI/OpenSpec)
- [agent-spec](https://github.com/ZhangHanDong/agent-spec)
- [Agent Execution Harness](https://github.com/lordaeternus/agent-execution-harness)
- [comrak](https://docs.rs/comrak/latest/comrak/)
