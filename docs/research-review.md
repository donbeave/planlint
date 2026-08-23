# Research review

Reviewed 2026-08-23. This document records what was checked before the design
archive was committed. Claims in the archive remain design inputs unless
explicitly marked as verified here.

## Confirmed claims

- **CodePlan:** Microsoft Research reports 5 of 7 repositories passing validity
  checks, while the equivalent non-planning baselines passed none.
- **Long-context limits:** *Lost in the Middle* reports positional degradation;
  RULER evaluates 17 models and reports that only half maintained satisfactory
  performance at 32K; NoLiMa evaluates models claiming at least 128K and reports
  that 10 of 12 fell below half of their short-context baseline at 32K.
- **SWE-RPG:** the August 10, 2026 preprint reports an average resolved rate of
  31.5% and identifies recovery of implicit requirements as a major difficulty.
- **Claude Code:** current goal documentation says `/goal` uses a separate
  evaluator, that the evaluator judges surfaced conversation content rather
  than independently running commands or reading files, and that Stop hooks can
  run deterministic scripts.
- **Grok:** current documentation supports Markdown skills exposed as slash
  commands, lifecycle hooks, and Claude Code compatibility. The checked page
  does not document a built-in `/goal` equivalent.
- **agent-spec:** current project documentation describes a Rust intent compiler
  with requirements, Task Contracts, boundaries, explicit test bindings,
  deterministic gates, lifecycle verification, trace, and archive behavior.
- **Agent Execution Harness:** current project documentation describes narrow
  tasks, typed evidence, strict command allowlists, compact handoffs, scope
  guards, and explicit verification. It uses JSON operational plans and is not
  the proposed canonical-Markdown Rust architecture.
- **Rust stack:** the proposed crates are plausible ecosystem choices. Exact
  versions must be selected and pinned during implementation, not inferred from
  this research archive.

## Claims kept qualified

### The 150K threshold

No checked source establishes 150K tokens as a universal smart-context limit.
The archive therefore treats `150K` as a configurable initial profile and
calibration target, not a scientific threshold.

### Codex Stop-hook schema

The checked Codex source supports `/goal` as a durable objective with a
verifiable stopping condition. The cited page did not expose the specific
`decision: "block"` Stop-hook schema claimed in the original research. Keep
that adapter detail as an implementation hypothesis until an exact official
Codex hook reference is pinned.

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
7. [Codex: Follow a goal](https://learn.chatgpt.com/use-cases/follow-goals)
8. [Grok skills, plugins, and marketplaces](https://docs.x.ai/build/features/skills-plugins-marketplaces)
9. [agent-spec documentation](https://docs.rs/crate/agent-spec/latest)
10. [agent-spec repository](https://github.com/ZhangHanDong/agent-spec)
11. [Agent Execution Harness](https://github.com/lordaeternus/agent-execution-harness)
12. [comrak documentation](https://docs.rs/comrak/latest/comrak/)

## Bottom line

The concept is credible. The strongest defensible claim is not that a fixed
context size makes agents smart. It is that explicit, bounded plans plus
deterministic, non-vacuous evidence can make acceptance more reliable than
transcript-only completion judgments.
