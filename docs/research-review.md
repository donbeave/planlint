# Research review

Reviewed 2026-08-23. This document records what was checked before the design
archive was committed. Claims in the archive remain design inputs unless
explicitly marked as verified here.

## 2026 source refresh

The original conclusion still holds, with two material corrections:

- Markdown plans are established practice, not the product’s unique insight.
  GitHub Spec Kit and OpenSpec both use repository Markdown artifacts and
  multi-artifact planning workflows.
- Codex now documents native `/goal` and a project-local `Stop` hook. The
  initial integration can use either; it must follow the current Codex hook
  wire format instead of assuming Claude’s format.

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
- **Grok:** current documentation supports Markdown skills exposed as slash
  commands, lifecycle hooks, Claude Code compatibility, headless sessions, and
  ACP. It does not document a built-in `/goal` equivalent. Its `Stop` event is
  passive, so automatic continuation requires an outer headless/ACP controller.
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

## Claims kept qualified

### The 150K threshold

No checked source establishes 150K tokens as a universal smart-context limit.
The archive therefore treats `150K` as a configurable initial profile and
calibration target, not a scientific threshold.

The current model catalogs also show why a fixed threshold is the wrong
abstraction: OpenAI lists 1.05M context for GPT-5.6 and 400K for GPT-5.2-Codex;
xAI lists 500K for Grok 4.6. Recent studies still report
working-set/coherence failures in long coding-agent trajectories. Capacity is
not a quality guarantee.

### Tailrocks integration

Tailrocks workflow semantics are supplied project context, not independently
verified by the public sources checked here. They are recorded as an existing
workflow that `planlint` should compile and enforce, not as an external fact.

## Sources checked

1. [CodePlan — Microsoft Research](https://www.microsoft.com/en-us/research/publication/codeplan-repository-level-coding-using-llms-and-planning-2/)
2. [Lost in the Middle — arXiv:2307.03172](https://arxiv.org/abs/2307.03172)
3. [RULER — arXiv:2404.06654](https://arxiv.org/abs/2404.06654)
4. [NoLiMa — arXiv:2502.05167](https://arxiv.org/abs/2502.05167)
5. [SWE-RPG / unified issue-resolution benchmark — arXiv:2608.09072](https://arxiv.org/abs/2608.09072)
6. [Claude Code goals](https://code.claude.com/docs/en/goal)
7. [Claude Code hooks](https://code.claude.com/docs/en/hooks)
8. [Codex: Follow a goal](https://learn.chatgpt.com/use-cases/follow-goals)
9. [Codex hooks](https://learn.chatgpt.com/docs/hooks)
10. [Codex developer commands](https://learn.chatgpt.com/docs/developer-commands?surface=cli)
11. [Grok skills, plugins, and marketplaces](https://docs.x.ai/build/features/skills-plugins-marketplaces)
12. [Grok hooks](https://docs.x.ai/build/features/hooks)
13. [Grok headless and scripting](https://docs.x.ai/build/cli/headless-scripting)
14. [Grok 4.6](https://docs.x.ai/developers/grok-4-6)
15. [agent-spec documentation](https://docs.rs/crate/agent-spec/latest)
16. [agent-spec repository](https://github.com/ZhangHanDong/agent-spec)
17. [Agent Execution Harness](https://github.com/lordaeternus/agent-execution-harness)
18. [comrak documentation](https://docs.rs/comrak/latest/comrak/)
19. [Codex slash dispatch](https://github.com/openai/codex/blob/main/codex-rs/tui/src/chatwidget/slash_dispatch.rs)
20. [Codex goal runtime](https://github.com/openai/codex/blob/main/codex-rs/ext/goal/src/runtime.rs)
21. [Codex goal steering](https://github.com/openai/codex/blob/main/codex-rs/ext/goal/src/steering.rs)
22. [OpenHands goal completion loop](https://github.com/OpenHands/software-agent-sdk/blob/main/examples/01_standalone_sdk/54_goal_completion_loop.py)
23. [SWE-agent retry implementation](https://github.com/SWE-agent/SWE-agent/blob/main/sweagent/agent/agents.py)
24. [Agent Execution Harness runner](https://github.com/lordaeternus/agent-execution-harness/blob/main/src/core/runner.ts)
25. [Agent Execution Harness finish checks](https://github.com/lordaeternus/agent-execution-harness/blob/main/src/core/finish-check.ts)
26. [Agent Execution Harness scope guard](https://github.com/lordaeternus/agent-execution-harness/blob/main/src/core/scope-guard.ts)
27. [agent-spec lifecycle source](https://github.com/ZhangHanDong/agent-spec/blob/main/src/spec_gateway/lifecycle.rs)

### Additional current sources

- [OpenAI model catalog](https://developers.openai.com/api/docs/models)
- [The Working Set of a Coding Agent](https://arxiv.org/abs/2608.16630)
- [Coding Agents are Effective Long-Context Processors](https://arxiv.org/abs/2603.20432)
- [When and How Context Rot Appears in Coding Agents](https://arxiv.org/abs/2607.17937)
- [GitHub Spec Kit](https://github.com/github/spec-kit)
- [OpenSpec](https://github.com/Fission-AI/OpenSpec)

## Bottom line

The concept is credible. The strongest defensible claim is not that a fixed
context size makes agents smart, nor that Markdown planning is new. It is that
explicit, bounded Markdown plan contracts plus deterministic, non-vacuous
evidence can make acceptance more reliable than transcript-only completion
judgments, while remaining portable across host agents.
