# Goal execution in coding agents

Research cut: 2026-08-23. This report studies the execution mechanism, not
marketing claims. Source-pinned open-source links identify the revision read.

## Evidence labels

- **Confirmed**: explicit official documentation or inspected source code.
- **Supported inference**: a proposed integration that follows documented
  primitives, but is not a product feature.
- **Unknown**: not documented or not established by the inspected source.

## Bottom line

`/goal` is not one architecture. Claude Code implements a continuation hook;
Codex implements durable goal state plus model-driven completion; OpenHands
wraps a normal run with a judge; current Grok Build implements a multi-stage
planner, worker, evaluator, and adversarial verifier harness.

Grok is the strongest native architecture found. It automatically writes a
structured plan, evaluates every worker round, and sends candidate completion
to tool-using verifier subagents. Its verifier still consists of LLMs applying
a model-written plan and evidence packet. It is materially stronger than a
transcript-only judge, but it is not a deterministic command-level acceptance
authority.

No surveyed product makes an immutable human-approved contract, closed-world
changed-path policy, and deterministic requirement-to-proof mapping
authoritative by default. An external verifier remains necessary for
predictable coding work.

## Architecture comparison

| Agent | What changes from a normal prompt | Architecture | Authoritative completion | Bounded retry | Planning / approval |
| --- | --- | --- | --- | --- | --- |
| Codex | Keeps one thread active and starts another turn when it becomes idle. | Persistent thread goal + lifecycle extension + model tools + hidden continuation prompt. | Model calls `update_goal(complete)` after a prompt-driven audit. | Optional token budget; model may mark `blocked`; no fixed turn cap in the inspected goal loop. | No planner is required or enforced by `/goal`. |
| Claude Code | A separate fast model evaluates after each turn and may start the next one. | Session-scoped prompt-based Stop hook. | Evaluator's transcript-only verdict. | Stops on `met`, `impossible`, no-progress safeguard, clear, or specified unrecoverable errors. | No native planning stage; permission mode is unchanged. |
| OpenHands SDK | An outer driver re-runs a conversation after an independent judge says evidence is missing. | `GoalController` + `run_goal` driver + judge LLM. | Judge verdict over transcript evidence. | Explicit `max_iterations`, terminal `complete` or `capped`. | No planner in `run_goal`; it composes with any agent or critic. |
| Cline | Plan mode is read-oriented; Act mode performs edits and commands after a manual toggle. | Persistent task/controller state with mode-specific prompts and models. | Agent's normal completion; no native goal evaluator located. | Task-resumption prompt asks agent to reassess and retry an interrupted step; no goal-level cap found. | Manual Plan-to-Act transition. |
| Grok Build | Creates durable goal state, runs a hidden planner, then evaluates and continues worker rounds automatically. | Persisted state machine + plan file + worker + structured evaluator + adversarial verifier panel. | Verifier quorum; only an achieved aggregate transitions the goal to complete. | Optional token budget; evaluator blocker requires 3 identical rounds; default 10 verifier attempts; identical verifier gaps auto-pause after 2 occurrences. | Hidden goal plan is not human-approved. Normal permissions stay separate; `/plan` remains the explicit approval mode. |

Cline is included as a planning/approval comparison, not as a claim that it
offers a native `/goal` equivalent.

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

## What source code actually establishes

| Question | Codex | OpenHands | Claude Code | Cline | Grok Build |
| --- | --- | --- | --- | --- | --- |
| Persistent goal state | Yes: SQL `thread_goals`. | Controller object during `run_goal`; conversation can persist separately. | Session condition; resume restores active goal. | Persistent task/mode, not goal. | Yes: serialized `GoalOrchestration`; active restores paused. |
| Separate completion evaluator | No; worker marks terminal status under a strong prompt. | Yes: second judge LLM. | Yes: separate fast model. | No native goal evaluator located. | Yes: structured tool-free evaluator, then tool-using skeptic panel. |
| Fresh isolated executor context | No; same thread. | No; same conversation events. | No; same session conversation. | No; same task/checkpoint context. | Worker stays in parent conversation; planner and verifiers are child sessions. |
| Independent command verifier | No. | No; judge reads transcript. | No; evaluator reads transcript. | No. | Partly: independent agents inspect files/evidence and may run cheap checks; no deterministic verifier. |
| Built-in bounded goal retries | Budgets and terminal state, no inspected turn cap. | Yes: `max_iterations`. | Operational stop/clear rules; user may add turn/time clause. | Not applicable. | Yes: token budget, verifier cap, repeated-blocker and no-progress pauses. |
| Native plan approval | No. | No. | No. | Yes. | `/goal`: no. Separate `/plan`: yes. |

This matrix separates confirmed mechanisms from desired properties. In
particular, "fresh evaluator" does not mean "independent verification" when it
only reads a transcript. Grok's tool-using skeptic panel is stronger, but its
verdict remains model-mediated.

## Strongest reusable practices

1. **Separate continuation from completion.** Codex, Claude Code, OpenHands,
   and Grok all re-enter after a normal worker run. Grok goes further: a
   tool-free evaluator nominates completion and a separate verifier panel owns
   the terminal verdict.
2. **Make non-success terminal.** OpenHands returns `capped`; Claude reports
   `impossible` and clears unrecoverable states; Codex has `blocked`, budget,
   and usage states; Grok distinguishes user, backoff, no-progress,
   infrastructure, blocked, and budget pauses. Never reinterpret one as
   success.
3. **Use transcript evaluators only for routing.** Claude, OpenHands, and Grok's
   first evaluator show the pattern. They can decide whether another worker
   round or deeper verification is warranted; they cannot prove repository
   state from a transcript.
4. **Give the verifier real workspace access, but keep deterministic gates.**
   Grok's skeptics inspect changed files, tests, and captured output, which is
   stronger than transcript review. Their quorum is still probabilistic. Run
   mechanical scope and proof predicates before any LLM judge.
5. **Keep a human planning boundary.** Cline and Grok `/plan` make review before
   edits explicit; Grok `/goal`'s hidden plan does not. Freeze one approved
   contract before execution and reject later weakening or expansion.
6. **Provide compact correction feedback.** OpenHands feeds the judge's
   `missing` field back to the worker. Codex reinjects completion and blocked
   rules each continuation. Grok persists bounded verifier gaps and a single
   evaluator next step. External diagnostics should be equally compact.
7. **Bound automatic recovery.** OpenHands has a hard cap; Claude has a
   no-progress safeguard; Grok combines token, verifier-attempt, repeated-gap,
   and repeated-blocker limits. Cap correction attempts per gate.
8. **Treat scope as executable policy.** No surveyed native goal owns a
   changed-path allowlist or requirement-to-proof graph. A verifier must compare
   `git diff` against the accepted plan and reject unlisted work rather than
   asking the agent to improvise a broader solution.
9. **Use isolated contexts deliberately.** Worker loops retain their parent
   conversation even when planner/verifier subagents are fresh. Clean executor
   sessions or worktrees are an additional controller policy.

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

## Research limits

- Closed-source Claude Code conclusions are limited to current official
  documentation and reproducible documented behaviour, not internal source.
- Open-source links are pinned revisions read on 2026-08-23. They may differ
  from later released binaries.
- Grok Build source is public, but installed binaries can differ by release,
  feature flag, remote policy, and configured verifier models.
- No claim here says an LLM evaluator or verifier quorum proves repository
  state. Transcript-only judgment is weakest; tool-using model verification is
  stronger; deterministic predicates remain the acceptance authority.
