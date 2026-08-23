# Goal execution in coding agents

Research cut: 2026-08-23. This report studies the execution mechanism, not
marketing claims. Source-pinned open-source links identify the revision read.

## Evidence labels

- **Confirmed**: explicit official documentation or inspected source code.
- **Supported inference**: a proposed integration that follows documented
  primitives, but is not a product feature.
- **Unknown**: not documented or not established by the inspected source.

## Bottom line

`/goal` is primarily a **continuation policy**: it prevents an agent from
giving control back after an ordinary turn. It is not, by itself, a planner,
scope guard, or deterministic verifier.

The strongest native architecture found is OpenHands SDK's explicit
planner/executor-independent judge split: a controller drives the loop, and a
second LLM judges evidence after every run. Codex has the strongest persistent
goal state and completion-audit prompt. Claude Code has a fresh evaluator and
well-defined operational failure behaviour. Cline and Grok demonstrate a
separate, valuable pattern: human approval of a plan before editing.

None of these products makes command-level verification, changed-path scope,
or requirement coverage authoritative by default. An external verifier remains
necessary for predictable coding work.

## Architecture comparison

| Agent | What changes from a normal prompt | Architecture | Authoritative completion | Bounded retry | Planning / approval |
| --- | --- | --- | --- | --- | --- |
| Codex | Keeps one thread active and starts another turn when it becomes idle. | Persistent thread goal + lifecycle extension + model tools + hidden continuation prompt. | Model calls `update_goal(complete)` after a prompt-driven audit. | Optional token budget; model may mark `blocked`; no fixed turn cap in the inspected goal loop. | No planner is required or enforced by `/goal`. |
| Claude Code | A separate fast model evaluates after each turn and may start the next one. | Session-scoped prompt-based Stop hook. | Evaluator's transcript-only verdict. | Stops on `met`, `impossible`, no-progress safeguard, clear, or specified unrecoverable errors. | No native planning stage; permission mode is unchanged. |
| OpenHands SDK | An outer driver re-runs a conversation after an independent judge says evidence is missing. | `GoalController` + `run_goal` driver + judge LLM. | Judge verdict over transcript evidence. | Explicit `max_iterations`, terminal `complete` or `capped`. | No planner in `run_goal`; it composes with any agent or critic. |
| Cline | Plan mode is read-oriented; Act mode performs edits and commands after a manual toggle. | Persistent task/controller state with mode-specific prompts and models. | Agent's normal completion; no native goal evaluator located. | Task-resumption prompt asks agent to reassess and retry an interrupted step; no goal-level cap found. | Manual Plan-to-Act transition. |
| Grok | Plan mode requires a reviewed plan before edit tools run; skills can expose custom slash commands. | Plan-mode guard, hooks, skills, and resumable sessions. | No documented native `/goal` evaluator or state machine. | No native goal loop found. | Plan preview requires human approval; edit guard has a shell-write caveat. |

The last two rows are useful comparisons, not claims that Cline or Grok offer a
native `/goal` equivalent.

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

OpenHands SDK provides a small, inspectable goal loop with a clearer
planner/executor/verifier separation than the other native implementations.

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

### Grok CLI

Current Grok documentation supplies usable components, but does not document a
built-in `/goal` that persistently evaluates completion across turns.

Confirmed details:

- [Skills](https://docs.x.ai/build/features/skills-plugins-marketplaces) are
  Markdown instruction folders; user-invocable skills appear as
  `/<skill-name>` slash commands. A custom `/project-goal` can therefore expose
  a handoff convention, but is not automatically an execution loop.
- [Plan mode](https://docs.x.ai/build/features/plan-mode) explores and drafts a
  plan before edits. The review surface requires a human to approve, request
  changes, comment, or quit; auto-approve does not skip it. The documented
  caveat matters: Plan mode gates edit tools, not shell redirection.
- [Hooks](https://docs.x.ai/build/features/hooks) provide lifecycle events.
  `PreToolUse` is the only blocking event; `Stop` is passive, and stdout for
  passive events is ignored. A Stop hook can record a verifier result but cannot
  turn it into a continuation prompt.
- [Headless sessions](https://docs.x.ai/build/cli/headless-scripting) support
  `--session-id`, `--resume`, `--continue`, JSON output, and
  `--always-approve`. Sessions are stored under `~/.grok/sessions`.
- [Permissions](https://docs.x.ai/build/features/permissions) distinguish
  permission approval from sandboxing; deny rules and `PreToolUse` hooks still
  apply in always-approve mode.

**Supported inference: portable Grok goal controller.** A project may combine
a slash skill, a frozen plan contract, `grok -p` / `grok --resume`, and an
external verifier. After every run, the controller, not a passive Stop hook,
reads verifier state. It resumes only for a correctable, bounded failure and
stops on `PASS`, `PLAN_GAP`, `STALE`, `BLOCKED`, or retry cap. This uses
documented primitives; it is not native Grok behaviour.

**Unknown.** No official source examined states that Grok's internal agent loop
has a durable objective record, separate goal evaluator, fixed retry policy,
or direct native `/goal` syntax. Do not claim any of these without a source.

## What source code actually establishes

| Question | Codex | OpenHands | Claude Code | Cline | Grok |
| --- | --- | --- | --- | --- |
| Persistent goal state | Yes: SQL `thread_goals`. | Controller object during `run_goal`; conversation can persist separately. | Session condition; resume restores active goal. | Persistent task/mode, not goal. | No native goal state documented. |
| Separate completion evaluator | No; worker marks terminal status under a strong prompt. | Yes: second judge LLM. | Yes: separate fast model. | No native goal evaluator located. | No native evaluator documented. |
| Fresh isolated executor context | No; same thread. | No; same conversation events. | No; same session conversation. | No; same task/checkpoint context. | No native goal loop to assess. |
| Independent command verifier | No. | No; judge reads transcript. | No; evaluator reads transcript. | No. | No. |
| Built-in bounded goal retries | Budgets and terminal state, no inspected turn cap. | Yes: `max_iterations`. | Operational stop/clear rules; user may add turn/time clause. | Not applicable. | Not applicable. |
| Native plan approval | No. | No. | No. | Yes. | Yes. |

This matrix separates confirmed mechanisms from desired properties. In
particular, "fresh evaluator" does not mean "independent verification" when it
only reads a transcript.

## Strongest reusable practices

1. **Separate continuation from completion.** Codex, Claude Code, and OpenHands
   all re-enter after a normal worker run. That removes a common failure mode:
   the worker deciding it is done before evidence exists.
2. **Make non-success terminal.** OpenHands returns `capped`; Claude reports
   `impossible` and clears unrecoverable states; Codex has `blocked`, budget,
   and usage states. A controller must never reinterpret a cap or blocker as
   success.
3. **Use a fresh evaluator only as a backstop.** Claude and OpenHands demonstrate
   an independent model judging worker output. Their own source proves its
   limitation: both judge transcript evidence, not actual repository state.
4. **Keep a human planning boundary.** Cline and Grok make review before edits
   explicit. For a bounded change, freeze that approved plan before execution;
   do not let later turns silently expand it.
5. **Provide compact correction feedback.** OpenHands feeds the judge's
   `missing` field back to the worker. Codex reinjects completion and blocked
   rules each continuation. An external verifier should likewise return only
   failed gate, observed fact, required fact, and one correction target.
6. **Bound automatic recovery.** OpenHands' hard cap is the cleanest native
   example. Claude's no-progress stop is another guard. A portable controller
   should cap correction attempts per gate, not continue indefinitely.
7. **Treat scope as executable policy.** No surveyed native goal owns a
   changed-path allowlist or requirement-to-proof graph. A verifier must compare
   `git diff` against the accepted plan and reject unlisted work rather than
   asking the agent to improvise a broader solution.
8. **Use isolated contexts deliberately.** The surveyed native goal loops retain
   their normal conversation context. Fresh executor sessions or worktrees are
   an additional controller policy, not a property implied by `/goal`.

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
2. Make the goal condition name exact final command(s), expected proof, allowed
   paths, and a fixed turn/time cap. Claude's evaluator can only see what the
   worker surfaces, so require final verifier output in the transcript.
3. Install a project Stop hook that runs the external verifier. It should allow
   completion only for `PASS`, provide compact diagnostics for one bounded
   correction, and record terminal non-success rather than looping.
4. Use Auto mode only after reviewing plan and tool permissions. On cap,
   impossible, or failed verifier, clear/replan rather than append unrelated
   instructions to an old session.

### Codex

1. Run planning separately, freeze the contract, then create one native
   `/goal` for its execution. The source shows goal accounting is not active for
   Plan-mode turns.
2. Let native goals supply persistence, progress accounting, pause/resume, and
   continuation. Do not treat `update_goal(complete)` as the acceptance
   authority; it is a model-mediated terminal transition.
3. Add a `Stop` hook that invokes the external verifier. On correctable failure,
   return a compact continuation prompt; use `stop_hook_active` and a verifier
   retry counter to prevent recursive continuation. On terminal non-success,
   persist receipt and allow stop.
4. Verify frozen inputs and changed paths before final `PASS`. If needed work
   lies outside the contract, emit `PLAN_GAP`, not an autonomous redesign.

### Grok

1. Use native Plan mode for review, but save the approved plan as a frozen
   repository artifact. Treat its shell-redirection caveat as a reason to add
   external diff and sandbox checks.
2. Implement a project skill for the small execution handoff only. Do not call
   that skill a native `/goal`.
3. Use an outer controller: create/resume one headless session, run verifier
   after every run, resume only after a correctable diagnostic, and stop on a
   terminal receipt. A `Stop` hook may log the receipt; it cannot continue work.
4. Use `PreToolUse` deny hooks and permission/sandbox policy for preventive
   safety, while verifier enforces scope and completion after execution.

## Research limits

- Closed-source Claude Code and Grok conclusions are limited to current official
  documentation and reproducible documented behaviour, not internal source.
- Open-source links are pinned revisions read on 2026-08-23. They may differ
  from later released binaries.
- No claim here says an LLM evaluator proves repository state. Where the source
  shows transcript-only judgment, that limitation is intentional and central to
  the recommendations.
