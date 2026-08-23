# What source code establishes

[← Research index](../README.md) · [Goal-execution index](README.md)

> Preserved from the original goal-execution research report.

## What source code actually establishes

| Question | Codex | OpenHands | Claude Code | Rig | Cline | Grok Build |
| --- | --- | --- | --- | --- | --- | --- |
| Persistent goal state | Yes: SQL `thread_goals`. | Controller object during `run_goal`; conversation can persist separately. | Session condition; resume restores active goal. | No; `AgentRun` persists one run, not a goal. | Persistent task/mode, not goal. | Yes: serialized `GoalOrchestration`; active restores paused. |
| Separate completion evaluator | No; worker marks terminal status under a strong prompt. | Yes: second judge LLM. | Yes: separate fast model. | No native evaluator; hook or host can supply one. | No native goal evaluator located. | Yes: structured tool-free evaluator, then tool-using skeptic panel. |
| Fresh isolated executor context | No; same thread. | No; same conversation events. | No; same session conversation. | No; serialized run preserves conversation; host decides isolation. | No; same task/checkpoint context. | Worker stays in parent conversation; planner and verifiers are child sessions. |
| Independent command verifier | No. | No; judge reads transcript. | No; evaluator reads transcript. | No native; custom host can add one. | No. | Partly: independent agents inspect files/evidence and may run cheap checks; no deterministic verifier. |
| Built-in bounded goal retries | Budgets and terminal state, no inspected turn cap. | Yes: `max_iterations`. | Operational stop/clear rules; user may add turn/time clause. | Per-run `max_turns`; no goal-level cap. | Not applicable. | Yes: token budget, verifier cap, repeated-blocker and no-progress pauses. |
| Native plan approval | No. | No. | No. | No; host-defined. | Yes. | `/goal`: no. Separate `/plan`: yes. |

This matrix separates confirmed mechanisms from desired properties. In
particular, "fresh evaluator" does not mean "independent verification" when it
only reads a transcript. Grok's tool-using skeptic panel is stronger, but its
verdict remains model-mediated.

