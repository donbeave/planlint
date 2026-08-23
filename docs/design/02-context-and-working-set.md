# Context and working set

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

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

