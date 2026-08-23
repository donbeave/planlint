# Strongest reusable practices

[← Research index](../README.md) · [Goal-execution index](README.md)

> Preserved from the original goal-execution research report.

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

