# Architecture comparison

[← Research index](../README.md) · [Goal-execution index](README.md)

> Preserved from the original goal-execution research report.

## Architecture comparison

| Agent | What changes from a normal prompt | Architecture | Authoritative completion | Bounded retry | Planning / approval |
| --- | --- | --- | --- | --- | --- |
| Codex | Keeps one thread active and starts another turn when it becomes idle. | Persistent thread goal + lifecycle extension + model tools + hidden continuation prompt. | Model calls `update_goal(complete)` after a prompt-driven audit. | Optional token budget; model may mark `blocked`; no fixed turn cap in the inspected goal loop. | No planner is required or enforced by `/goal`. |
| Claude Code | A separate fast model evaluates after each turn and may start the next one. | Session-scoped prompt-based Stop hook. | Evaluator's transcript-only verdict. | Stops on `met`, `impossible`, no-progress safeguard, clear, or specified unrecoverable errors. | No native planning stage; permission mode is unchanged. |
| OpenHands SDK | An outer driver re-runs a conversation after an independent judge says evidence is missing. | `GoalController` + `run_goal` driver + judge LLM. | Judge verdict over transcript evidence. | Explicit `max_iterations`, terminal `complete` or `capped`. | No planner in `run_goal`; it composes with any agent or critic. |
| Rig | Provides reusable run-control primitives; host code must define `/goal`. | `rig::AgentRun` sans-IO state machine + `AgentRunner` + typed `AgentHook`; host-supplied verifier. | A tool-free model turn normally ends a run; a hook or outer controller must withhold goal success until proof passes. | Per-run `max_turns` and hook retry/stop actions; serialized `AgentRun` can pause/resume, but high-level runner resume remains an open issue. | No native goal planning or approval; host-defined. |
| Cline | Plan mode is read-oriented; Act mode performs edits and commands after a manual toggle. | Persistent task/controller state with mode-specific prompts and models. | Agent's normal completion; no native goal evaluator located. | Task-resumption prompt asks agent to reassess and retry an interrupted step; no goal-level cap found. | Manual Plan-to-Act transition. |
| Grok Build | Creates durable goal state, runs a hidden planner, then evaluates and continues worker rounds automatically. | Persisted state machine + plan file + worker + structured evaluator + adversarial verifier panel. | Verifier quorum; only an achieved aggregate transitions the goal to complete. | Optional token budget; evaluator blocker requires 3 identical rounds; default 10 verifier attempts; identical verifier gaps auto-pause after 2 occurrences. | Hidden goal plan is not human-approved. Normal permissions stay separate; `/plan` remains the explicit approval mode. |

Cline is included as a planning/approval comparison, not as a claim that it
offers a native `/goal` equivalent.

