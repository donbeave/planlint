# Confirmed claims

[← Research index](../README.md) · [Review index](README.md)

> Preserved from the source-refresh review.

## Confirmed claims

- **CodePlan:** Microsoft Research reports 5 of 7 repositories passing validity
  checks, while the equivalent non-planning baselines passed none.
- **Long-context limits:** *Lost in the Middle* reports positional degradation;
  RULER evaluates 17 models and reports that only half maintained satisfactory
  performance at 32K; NoLiMa v3 evaluates 13 models claiming at least 128K and
  reports that 11 fell below half of their short-context baseline at 32K.
- **Coding-agent context:** *The Working Set of a Coding Agent* reports that
  required-fact availability, not token spend, determined work supported by
  those facts. The agent-skills audit reports a large clean/long-context gap on
  one task, no gap on another, and better results from a detailed external
  checklist than a generic self-check. The long-context-processors paper shows
  coding agents can use filesystem and executable tools over corpora far larger
  than a prompt window. Together these support complete, explicit working sets;
  they do not establish a universal token threshold.
- **SWE-RPG:** the August 10, 2026 preprint reports an average resolved rate of
  31.5% and identifies recovery of implicit requirements as a major difficulty.
- **Claude Code:** current goal documentation says `/goal` uses a separate
  evaluator, that the evaluator judges surfaced conversation content rather
  than independently running commands or reading files, and that Stop hooks can
  run deterministic scripts. The hooks reference documents `decision: "block"`
  for Stop hooks and a `stop_hook_active` loop guard.
- **Codex:** current official documentation says `/goal` is a persistent,
  verifiable stopping-condition workflow. It also documents lifecycle hooks,
  including `Stop`; a hook can return `{ "decision": "block", "reason":
  "..." }` to continue with the reason as a new continuation prompt, and it
  provides `stop_hook_active` for loop control. The Codex wire format is
  host-specific and must not be copied from Claude.
- **Grok Build:** current documentation and public source now expose native
  `/goal`. The inspected implementation persists `GoalOrchestration`, creates a
  hidden structured plan, evaluates each worker round, and sends candidate
  completion to an adversarial tool-using skeptic panel. It bounds verifier
  attempts and repeated-gap/blocker loops and pauses on infrastructure,
  contradiction, unverifiable, no-progress, and budget outcomes. The native
  plan is not human-approved, verifier authority remains model-mediated, and no
  closed-world changed-path rule is mechanically enforced.
- **Rig:** current official docs and source expose no native `/goal` command or
  goal lifecycle. `rig::AgentRun` is a serializable sans-IO
  `CallModel`/`CallTools`/`Done` state machine; `AgentRunner` and `AgentHook`
  provide configured execution and per-run steering. An accepted tool-free
  model turn ends a run; there is no built-in goal evaluator. A durable goal
  controller, plan approval, outer retry loop, repository verifier, and
  terminal receipt remain host responsibilities. High-level runner resume is
  still an open roadmap issue. The official evals page is stale: v0.42.0
  deleted the evals module and `experimental` feature.
- **Real CLI integration:** Rig must use a host-owned process, ACP, or HTTP
  adapter. Claude Code exposes non-interactive JSON/JSONL and resume; Codex
  exposes `exec` JSONL plus an experimental bidirectional app-server; Grok,
  OpenCode, and Kimi Code expose ACP subprocesses; OpenCode and Kimi also
  expose local HTTP/WebSocket services. The adapter must handle cancellation,
  permissions/callbacks, session IDs, and process cleanup; agent output is not
  acceptance proof.
- **Claude Code context:** current goal documentation says the condition is
  restored on resume but turn/time/token baselines reset; `/goal` continues the
  same session rather than creating a fresh execution context. Background work
  defers evaluation until its output is surfaced.
- **Markdown precedent:** GitHub Spec Kit documents separate specification,
  plan, and task artifacts; OpenSpec documents plain-Markdown requirements,
  scenarios, design, and tasks.
- **agent-spec:** current project documentation describes a Rust intent compiler
  with requirements, Task Contracts, boundaries, explicit test bindings,
  deterministic gates, lifecycle verification, trace, and archive behavior.
- **Agent Execution Harness:** current project documentation describes narrow
  tasks, typed evidence, strict command allowlists, compact handoffs, scope
  guards, and explicit verification. It uses JSON operational plans and is not
  the proposed canonical-Markdown Rust architecture.
- **Codex source:** the public Rust repository confirms a native `Goal` slash
  command, persisted thread goal API, goal tool, internal goal steering, idle
  continuation, lifecycle statuses, and separate user/system controls for
  pause/resume/budgets. The inspected goal path does not expose an independent
  test/linter/compiler verifier or changed-path scope gate.
- **OpenHands source:** the SDK includes a `run_goal` example that wraps a
  normal conversation in an independent judge-LLM audit loop with a hard
  iteration cap and `complete`/`capped` outcomes. It keeps work in the same
  conversation history; the example does not provide a closed-world scope
  guard or deterministic command runner.
- **SWE-agent source:** `RetryAgent` bounds retries by cost/configuration,
  saves trajectories, and hard-resets the environment between attempts.
  `DefaultAgent` bounds corrective model re-queries after formatting,
  blocked-action, policy, and shell-syntax failures.
- **Deterministic harness sources:** Agent Execution Harness implements typed
  run-state transitions, evidence/claim/rollback completion checks, and a
  Git-based scope guard. `agent-spec` implements Markdown contract lifecycle
  verification, explicit test binding, and non-passing `skip`/`uncertain`
  semantics. Neither is itself a native `/goal` controller.
- **Rust stack:** the proposed crates are plausible ecosystem choices. Exact
  versions must be selected and pinned during implementation, not inferred from
  this research archive.

