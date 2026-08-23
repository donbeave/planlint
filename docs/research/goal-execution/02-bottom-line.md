# Bottom line

[← Research index](../README.md) · [Goal-execution index](README.md)

> Preserved from the original goal-execution research report.

## Bottom line

`/goal` is not one architecture. Claude Code implements a continuation hook;
Codex implements durable goal state plus model-driven completion; OpenHands
wraps a normal run with a judge; current Grok Build implements a multi-stage
planner, worker, evaluator, and adversarial verifier harness.

Rig is a lower-level Rust runtime, not a coding-agent product with a native
`/goal` controller. Its current source exposes a serializable run state machine,
per-run hooks, and bounded model/tool execution; the host must supply the
objective record, verifier, outer continuation loop, plan, and terminal
authority.

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

