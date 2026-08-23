# Confirmed implementation traces

[← Research index](../README.md) · [Goal-execution index](README.md)

> Preserved from the original goal-execution research report.

## Confirmed implementation traces

### Codex CLI

The current open-source Codex implementation is a persistent per-thread goal,
not a one-shot prompt wrapper.

```text
/goal <objective>
  -> SlashCommand::Goal creates GoalDraft and sends SetThreadGoalDraft
  -> state DB stores one thread_goals record for the thread
  -> goal extension exposes get_goal, create_goal, update_goal to model
  -> thread becomes idle
  -> GoalExtension.on_thread_idle()
  -> GoalRuntimeHandle.continue_if_idle()
  -> internal continuation prompt is injected
  -> start_turn_if_idle()
  -> model uses normal tools; marks complete or blocked
```

Confirmed source:

- [`SlashCommand::Goal` dispatch](https://github.com/openai/codex/blob/c9b19deb09c1841ce7acc33ddb96276030936a29/codex-rs/tui/src/chatwidget/slash_dispatch.rs#L832-L929)
  recognizes `clear`, `edit`, `pause`, and `resume`, otherwise builds a
  `GoalDraft` and submits `SetThreadGoalDraft`.
- [`thread_goals` SQL store](https://github.com/openai/codex/blob/c9b19deb09c1841ce7acc33ddb96276030936a29/codex-rs/state/src/runtime/goals.rs#L33-L78)
  persists `thread_id`, `goal_id`, objective, status, token budget, token use,
  elapsed time, and timestamps.
- [Goal tool schemas](https://github.com/openai/codex/blob/c9b19deb09c1841ce7acc33ddb96276030936a29/codex-rs/ext/goal/src/spec.rs#L9-L93)
  expose `get_goal`, `create_goal`, and `update_goal`. An unfinished goal
  prevents creation of another; the model can only set `complete` or `blocked`.
- [Idle lifecycle hook](https://github.com/openai/codex/blob/c9b19deb09c1841ce7acc33ddb96276030936a29/codex-rs/ext/goal/src/extension.rs#L148-L160)
  calls the goal runtime when a thread becomes idle.
- [`continue_if_idle`](https://github.com/openai/codex/blob/c9b19deb09c1841ce7acc33ddb96276030936a29/codex-rs/ext/goal/src/runtime.rs#L362-L439)
  reads active persisted state, creates a continuation steering item, and starts
  a new turn only if the thread is idle.
- [Continuation prompt](https://github.com/openai/codex/blob/c9b19deb09c1841ce7acc33ddb96276030936a29/codex-rs/ext/goal/templates/goals/continuation.md)
  re-injects the objective, usage, completion audit, and blocked audit as
  internal model context.

**Goal representation and context.** The objective, status, budget, time, and
token accounting are durable thread state. The continuation is injected into
the *same* thread, not a fresh isolated context. The prompt explicitly tells
the model to inspect current workspace state and perform a requirement-by-
requirement completion audit. It does not create a plan, a DAG, an immutable
acceptance contract, or a changed-path allowlist.

**Planning, execution, and tools.** `/goal` does not decompose work itself.
The model uses its normal tool surface. In fact, the lifecycle extension clears
the active turn's goal accounting in Plan mode, so planning and long-running
goal execution are separate concerns in the implementation.

**Retry and terminal state.** While status remains `Active`, the idle hook
starts another turn. A user can pause, resume, edit, or clear. The model may
mark the goal `complete` or `blocked`; the prompt forbids `blocked` until the
same blocker has repeated across three goal turns. Token budgets and usage
limits are system states. A turn error transitions an active goal to `Blocked`
in the goal runtime.

**Verification and scope.** The completion prompt requires real-state evidence
and commands, but completion is still a model tool call. No inspected goal code
runs a test, compiler, linter, or diff check independently. Therefore tests
only become completion evidence if the model runs and surfaces them; scope
constraints remain prompt-level unless a repository hook or tool enforces them.

**Human control.** Goal creation is explicit; users control pause, resume,
edit, and clear. Normal tool permissions still apply. Official Codex guidance
calls `/goal` a durable objective with a verifiable stopping condition and
requires one objective, declared inputs, proof commands, and checkpoints:
[Follow a goal](https://learn.chatgpt.com/use-cases/follow-goals).

### Claude Code

Claude Code's `/goal` is a session-scoped prompt-based Stop hook with a second,
small evaluator model. Its official documentation is the authoritative source;
the implementation is closed.

```text
/goal <condition>
  -> condition starts a turn immediately
  -> agent uses normal tools in same session
  -> end of each eligible turn
  -> small evaluator receives condition + surfaced conversation
  -> not met: evaluator reason becomes next-turn guidance
  -> met / impossible / cleared / unrecoverable error: terminal
```

Confirmed details from [Claude Code goals](https://code.claude.com/docs/en/goal):

- One condition is active per session. A replacement `/goal` replaces it.
  Resuming a session restores an active condition but resets its progress
  counters.
- The evaluator is a configured small fast model (Haiku by default on Claude
  API) and is separate from the worker. It returns `not yet met`, `met`, or
  `impossible`, with a reason.
- It sees conversation content, not filesystem state, and never calls tools.
  Tests and build output must therefore be run by the worker and surfaced in
  the transcript.
- It has no native planner, task graph, path allowlist, or deterministic
  verifier. The condition may *describe* a command and constraints, but that is
  prompt policy.
- The loop pauses if repeated evaluator turns make no progress without tool
  use. It clears for four unrecoverable failures (authentication, exhausted
  credit, unresolvable context overflow, unavailable model); transient failures
  leave it active. Background work defers evaluation and triggers check-ins.
- `/goal` does not change tool permission mode. Auto mode is needed for
  unattended tool use; workspace trust and enabled hooks are required.

**Unknown.** Official documentation does not describe an independent command
runner, automatic task decomposition, source-level state machine, or scope
checker behind `/goal`. Do not infer them from evaluator wording.

### OpenHands Software Agent SDK

OpenHands SDK provides a small, inspectable goal loop with an explicit
executor/controller/judge separation.

```text
run_goal(conversation, objective, judge_llm, max_iterations)
  -> GoalController.start() returns objective
  -> conversation.send_message(objective)
  -> conversation.run() executes normal agent/tool loop
  -> GoalController.on_run_finished(events)
  -> judge_goal(judge_llm, objective, events)
  -> complete: GoalOutcome(complete)
     capped:  GoalOutcome(capped)
     else:    send follow-up containing missing evidence; repeat
```

Confirmed source:

- [`run_goal`](https://github.com/OpenHands/software-agent-sdk/blob/9421149592da215066f58cb68cb04599d896ae74/openhands-sdk/openhands/sdk/conversation/goal/runner.py#L30-L57)
  is the outer I/O loop.
- [`GoalController.on_run_finished`](https://github.com/OpenHands/software-agent-sdk/blob/9421149592da215066f58cb68cb04599d896ae74/openhands-sdk/openhands/sdk/conversation/goal/controller.py#L78-L133)
  owns the objective, iteration counter, hard cap, and continue-versus-stop
  decision.
- [`judge_goal`](https://github.com/OpenHands/software-agent-sdk/blob/9421149592da215066f58cb68cb04599d896ae74/openhands-sdk/openhands/sdk/conversation/goal/judge.py#L43-L120)
  renders LLM-convertible conversation events, calls a separate LLM, and parses
  a strict JSON verdict. Invalid JSON becomes conservative `complete=False`.
- [Judge and follow-up prompts](https://github.com/OpenHands/software-agent-sdk/blob/9421149592da215066f58cb68cb04599d896ae74/openhands-sdk/openhands/sdk/conversation/goal/prompts.py)
  require evidence per objective requirement and direct the worker to inspect
  current state, run relevant commands, and preserve full objective scope.
- [Official goal-loop guide](https://docs.openhands.dev/sdk/guides/convo-goal)
  confirms `complete`/`capped`, default `max_iterations=10`, same-conversation
  history, and composition with an optional per-run critic.

**Goal representation and context.** The objective and iteration counter live
in `GoalController`, while all turns share the passed `Conversation` and its
event history. This is not a fresh or isolated execution. The judge deliberately
excludes the system prompt to reduce cost, but otherwise judges the rendered
transcript. The SDK separately supports an LLM summarizing condenser when
conversation size exceeds its configured event or token limits; that is context
management, not goal isolation.

**Planning and tool use.** `run_goal` invokes any configured agent and tools; it
does not plan or decompose. A critic may improve individual `conversation.run()`
calls, while goal loop decides whether another run starts. Test, compiler, and
linter commands remain agent-selected tools rather than goal-runtime gates.

**Retry, completion, and scope.** This is the only inspected native loop with a
hard, explicit audit-round cap and non-success terminal outcome (`capped`). It
also fails safely on an unparsable judge result. However, its independent judge
still evaluates transcript evidence, not a fresh filesystem or command runner.
There is no default changed-path allowlist, acceptance manifest, or requirement-
to-test binding.

**Human approval.** Goal API itself has no approval point. Tool confirmation,
security policy, workspace isolation, and persistence belong to the configured
conversation, not `GoalController`.

### Cline Plan and Act

Cline is an open-source implementation of **planner approval**, not a native
long-running goal evaluator. It is useful because it makes the planning /
execution boundary explicit.

```text
new task or Plan mode
  -> controller stores mode and uses plan-mode prompt
  -> agent reads / investigates, uses plan_mode_respond
  -> user manually toggles to Act
  -> Act-mode agent receives editing and command tools
  -> normal task completion or resumption
```

Confirmed source:

- [`planModeInstructions`](https://github.com/cline/cline/blob/be8b984d10d1ad0e9a3917e051ac697f592587d2/apps/vscode/src/core/prompts/responses.ts#L260-L265)
  permits gathering and discussion, then requires `plan_mode_respond`; it tells
  the agent that it cannot switch to Act itself.
- [`togglePlanActModeProto`](https://github.com/cline/cline/blob/be8b984d10d1ad0e9a3917e051ac697f592587d2/apps/vscode/src/core/controller/state/togglePlanActModeProto.ts#L13-L31)
  validates Plan/Act enum input and delegates transition to the controller.
- [Cline's Plan & Act documentation](https://github.com/cline/cline/blob/be8b984d10d1ad0e9a3917e051ac697f592587d2/docs/core-workflows/plan-and-act.mdx)
  confirms no file modifications or command execution in Plan mode and manual
  Act transition.

**Context and retry.** Task resumption uses the same task context and prompts
the agent to reassess potentially changed workspace state, then retry an
interrupted last step if needed. Checkpoints support restoration. This is
persistence and recovery, not fresh-context isolation.

**Verification, scope, and completion.** No goal evaluator, deterministic
completion gate, immutable plan contract, or task-DAG executor was located in
the inspected current source roots or official workflow documentation. Plan mode
is a strong human approval point, but proof commands remain agent actions and
scope must be guarded elsewhere.

**Unknown.** An absence in these inspected source roots does not prove no
extension/plugin can add a goal loop. It establishes only that the native Plan /
Act flow is not one.

### Rig framework

Rig is a Rust agent framework and runtime. The official site and docs describe
agents, tools, workflows, hooks, memory, and model-provider abstractions, but
not a native `/goal` command or durable goal controller. A source search of the
latest release, [v0.42.0](https://github.com/0xPlaygrounds/rig/releases/tag/v0.42.0)
at revision
[`d5a34986a1ad57f1e9c5984b82f8d7438ffc717e`](https://github.com/0xPlaygrounds/rig/tree/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e),
found no `/goal` parser, goal type, lifecycle, or controller. The closest
execution architecture is:

```text
host /goal <objective>
  -> external GoalController owns objective, plan, evidence, and terminal state
  -> AgentRunner drives one configured run
  -> rig::AgentRun advances CallModel -> CallTools -> Done
  -> AgentHook enforces tool policy and intercepts a tool-free model turn
  -> deterministic verifier decides PASS | CORRECTABLE | PLAN_GAP | BLOCKED
  -> hook accepts, retries with evidence, or stops; outer controller persists
  -> controller starts another run or persists/pauses the goal
```

Confirmed source:

- [Agent concepts](https://rig.rs/docs/concepts/agent) and [AgentRunner
  concepts](https://rig.rs/docs/concepts/agentrunner) describe a tool loop and
  the high-level/low-level split. The documented multi-turn setting is a
  per-prompt turn limit, not a goal loop.
- [`rig::AgentRun`](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/run/mod.rs#L1-L34)
  is a sans-IO, `Serialize`/`Deserialize` state machine. Its public steps are
  [`CallModel`, `CallTools`, and `Done`](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/run/mod.rs#L154-L179).
  It carries the accumulated conversation and raw provider responses, so
  persistence has sensitivity and growth costs; the source also warns that the
  format has no cross-version stability guarantee.
- Post-release `main` at
  [`2b271b66ca21b5baa230e42589ca00184f43af59`](https://github.com/0xPlaygrounds/rig/tree/2b271b66ca21b5baa230e42589ca00184f43af59)
  moves this state machine into a new internal `rig-run` crate but adds no goal
  lifecycle. The integration spelling above remains the released v0.42.0 API.
- [`AgentRunner`](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/runner.rs#L142-L220)
  is the configured execution path. It owns model/tool I/O, memory, tracing,
  and hooks. `max_turns` bounds model calls for one run, including model-turn
  hook retries; it is not a goal-round budget.
- [`AgentRun` normally finishes on an accepted tool-free model turn](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/run/mod.rs#L718-L955).
  Text therefore means run completion, not proven goal completion. An
  `on_model_turn_finished` hook may return `Retry` with verifier feedback or
  `Stop` instead, while `Continue` allows `Done`.
- [`AgentHook`](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/hook.rs#L584-L704)
  can steer model turns, and tool-call/result hooks can reject, rewrite, skip,
  or stop effects. This supports approval and scope gates, but the host tool or
  sandbox must enforce filesystem and command boundaries. Hook scratchpad
  state is run-scoped and is not serialized with `AgentRun`.
- The current [evals concepts page](https://rig.rs/docs/concepts/evals) is stale
  against v0.42.0: the release [deleted the unused evals module and
  `experimental` feature](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-core/CHANGELOG.md#L481-L506),
  and the [release feature list](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/Cargo.toml#L324-L385)
  contains no `experimental` flag. Current Rig supplies no built-in goal
  evaluator; use a host-defined deterministic verifier or evaluator agent.
- The [interactive-agent roadmap](https://github.com/0xPlaygrounds/rig/issues/2118)
  explicitly assigns workflow, permissions, storage, and orchestration to the
  host and lists steering, safe resume, durable sessions, and child-agent
  control as runtime work. [Runner-level resume](https://github.com/0xPlaygrounds/rig/issues/2244)
  remains a separate open issue: a serialized `AgentRun` can be hand-driven,
  but the high-level runner cannot yet accept a restored run while preserving
  its hooks, tool server, memory, and telemetry.

**Goal representation and context.** Rig has no native objective/status record.
`AgentRun` persists the run transcript and turn state, while `ConversationMemory`
is conversation history rather than a goal lifecycle. A controller must persist
the objective, accepted plan, fingerprints, verifier gaps, retry counters, and
terminal receipt separately. Hand-driven `AgentRun` resumption preserves the
serialized conversation; it does not create a fresh executor context. The
runner only loads memory before execution and appends it after `Done`; an append
failure is logged while the final response is still returned, so memory is not
a reliable goal checkpoint.

**Planning and approval.** No native goal planner or approval boundary was found.
The host can use Rig's tools, agents, and workflows, but human approval,
immutable plan state, allowed paths, and plan-to-proof bindings remain
controller responsibilities.

**Retry, failure, and verification.** `max_turns` bounds one run, and hooks can
retry or stop individual model turns. Neither primitive supplies an outer
corrective-attempt cap, no-progress policy, goal pause/resume status, or
completion receipt. A hook retry consumes the ordinary turn budget and has no
separate retry cap. Deterministic commands, changed-path checks, requirement
coverage, and classification of terminal non-success must remain external. A
model evaluator can route corrections but must not authorize repository
acceptance.

**Supported inference.** Rig is a strong substrate for a portable `/goal`
adapter: use `AgentRunner` for ordinary configured runs, `AgentHook` for
per-run policy and telemetry, `rig::AgentRun` only when the controller must
own mid-run persistence/tool approval, and `planlint` for the outer contract and
verifier. This is an adapter architecture, not a Rig feature.

**Unknown.** The roadmap may add run control or durable session APIs after this
revision. No inspected source establishes when those pieces will become a
native goal abstraction or whether a future host will expose `/goal` syntax.

### Grok Build

Grok Build's current open-source implementation is a native, durable,
multi-stage goal harness. Its official
[command guide](https://docs.x.ai/build/modes-and-commands) exposes `/goal`
with status, pause, resume, clear, and token-budget controls. It is
substantially different from Grok's ordinary prompt loop and from the
separately documented `/plan` mode.

```text
/goal <objective> [--budget <tokens>]
  -> slash parser creates GoalSet
  -> setup_goal creates and persists GoalOrchestration
  -> hidden planner subagent writes goal/plan.md
  -> normal worker turn executes with goal rules + plan pointer
  -> hidden evaluator classifies round: continue | candidate_complete | blocked
  -> continue: inject one synthetic next-step directive and run another turn
     candidate_complete: capture diff/evidence and run verifier panel
     blocked x3 with same key: pause for user action
  -> verifier achieved: complete
     verifier refuted: feed bounded gaps back and retry
     same gaps / cap / contradiction / unverifiable / infra: pause
```

Confirmed source at revision
[`07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8`](https://github.com/xai-org/grok-build/tree/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8):

- [Slash parsing](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/slash_commands.rs#L309-L360)
  recognizes `status`, `pause`, `resume`, `clear`, an objective, and an optional
  positive trailing token budget. The canonical command instruction also tells
  the worker to track steps, verify work, and reserve blocked status for a
  repeated blocker: [shared command text](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-tools-api/src/slash_commands.rs#L209-L257).
- [`GoalOrchestration`](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/goal_tracker.rs#L422-L636)
  stores objective, status, phase, token accounting, history, plan paths,
  baseline commit, verifier attempts and gaps, strategist state, and verifier
  session/model assignments. The state is persisted through
  [`PersistenceMsg::GoalModeState`](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/goal_orchestrator.rs#L29-L59).
- [`setup_goal`](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/acp_session_impl/goal.rs#L799-L859)
  captures token and Git baselines, creates state, runs the planner, and renders
  the worker rules. The
  [planner](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/goal_planner.rs#L438-L543)
  is a separate general-purpose subagent. It must write a non-empty plan file;
  planner transport, runtime, or missing-file failure is fail-closed and pauses
  the goal.
- The model-facing
  [`update_goal`](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-tools/src/implementations/grok_build/update_goal/mod.rs#L1-L79)
  accepts a progress message, `completed`, or `blocked_reason` and waits for a
  verdict-aware actor acknowledgement. A mid-turn completion claim is
  [deferred to turn end](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/acp_session_impl/goal.rs#L1549-L1781),
  so it nominates verification rather than bypassing it.
- The [planner contract](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/templates/goal_planner_prompt.md)
  selects `code-change`, `analysis`, or `research`; emits small outcome-based
  acceptance criteria, a verification plan, scope/non-goals, and an ordered
  checklist; and tells the planner not to modify the workspace outside its plan
  file. The original plan is separately snapshotted so later weakening can be
  shown to verifiers.
- At worker turn end,
  [`run_goal_round_end`](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/acp_session_impl/goal.rs#L1278-L1345)
  invokes a tool-free structured evaluator. The
  [evaluator](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/goal_evaluator.rs#L1-L167)
  receives the objective, plan, and at most 32 KiB of recent transcript; it
  returns strict JSON with `continue`, `candidate_complete`, or `blocked` plus
  evidence and one next step. Evaluation retries once; repeated failure pauses
  the goal as an infrastructure failure.
- Candidate completion enters the
  [adversarial verification stage](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/goal_classifier.rs#L1870-L2217).
  It captures a baseline-to-current Git diff, complete changed-file list, plan
  changes, final response, and captured evidence. It then runs up to five
  tool-using skeptic subagents, three by default. Skeptic 0 is a persistent
  reject-gatekeeper; cold skeptics run independently. Missing/malformed skeptic
  evidence becomes a refuting vote. A high-confidence skeptic-0 refutation is
  decisive; otherwise approval requires a strict majority of the cold panel.
- The [verifier prompt](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/templates/goal_verifier_prompt.md)
  tells skeptics to inspect current changed files and the implementer's tests
  and captured outputs, perform cheap real-path checks, detect plan weakening
  and test theater, and classify gaps as model-fixable, contradiction, or
  unverifiable. Verifiers may read/search/run checks but must not edit the
  workspace.
- [Outcome application](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/acp_session_impl/goal.rs#L241-L339)
  is harness-owned: achieved completes the goal; ordinary refutation stores
  gaps and continues; every non-fixable refutation pauses; repeated identical
  gaps pause early; and reaching the verifier-attempt cap pauses. The default
  cap is ten, while the identical-gap stall threshold is two
  ([classifier defaults](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/goal_classifier.rs#L37-L116),
  [stall state](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/goal_tracker.rs#L33-L48)).

**Goal representation and context.** One orchestration object is persisted with
the session. Worker continuation stays in the same parent conversation; stale
synthetic goal directives are pruned before the next is inserted. Planner and
skeptics are child sessions. On process restore, an active goal is deliberately
restored as user-paused rather than silently self-driving, and in-flight child
state is cleared
([restore logic](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/session/goal_tracker.rs#L721-L790)).

**Planning and approval.** Native goal planning is automatic and hidden; the
goal plan is not shown for human approval. This is distinct from Grok's
[interactive `/plan` mode](https://docs.x.ai/build/features/plan-mode), whose
preview requires approval and whose edit gate is independent of permissions.
The goal planner freezes an original baseline for later audit, but the current
plan remains editable and verifier-enforced rather than mechanically immutable.

**Execution, permissions, and scope.** The worker uses the normal session tool
loop. The traced goal setup does not switch permission mode; Grok's documented
[permission and sandbox controls](https://docs.x.ai/build/features/permissions)
remain separate. The harness captures the real Git diff and full changed-file
list and asks skeptics to reject fabricated or out-of-scope work. It does not
apply a closed-world allowed-path policy before tools run, so scope acceptance
is still model-mediated.

**Retry and failure semantics.** A token budget can terminate as
`BudgetLimited`. User actions provide status, pause, resume, and clear.
Evaluator `blocked` must repeat with the same stable key three times before the
goal pauses. Verifier refutations continue with bounded gap feedback; identical
gaps, cap exhaustion, contradictions, unverifiable requirements, planner
failure, evaluator failure, and verifier infrastructure failure pause rather
than pass. Resume resets per-attempt stall/cap counters while preserving the
objective and plan.

**Completion and verification.** This is true independent evidence review, not
just worker self-declaration: the worker/evaluator can only nominate a
candidate, and the harness applies the panel outcome. However, the panel is
still probabilistic. It audits files, tests, and captured outputs using agents;
it does not compile an immutable contract into deterministic predicates or
enforce an allowed-path set itself.

**Unknown.** Source establishes the implementation at the pinned revision, not
which feature flags, verifier models, or remote policy a particular installed
binary receives. No inspected source establishes a cryptographically immutable
plan, deterministic requirement-to-test binding, or transactional workspace
rollback as part of native `/goal`.

