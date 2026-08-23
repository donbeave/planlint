# Test strategy

[← Integrations index](../README.md) · [Rig CLI index](README.md)

> Preserved from the original Rig CLI adapter study.

## Test strategy

Build three layers:

1. **Pure parser fixtures:** one fixture per checked CLI version for success,
   vendor-declared failure, missing terminal event, unknown event, malformed
   JSON, invalid UTF-8, oversized line, and truncated stream.
2. **Fake executable tests:** scripts/binaries that emit events slowly, flood
   stdout and stderr together, hang, ignore graceful termination, spawn a child,
   exit nonzero after a success frame, and request permission. Prove timeouts,
   output caps, process-tree cleanup, and fail-closed policy.
3. **Authenticated smoke tests:** opt-in tests behind explicit environment
   flags. Check each pinned real binary with a no-write prompt, a controlled
   write inside a disposable worktree, cancellation, resume by exact ID, and a
   denied operation. Record version and redact artifacts.

Add handoff race tests around these layers. Prove that attach material is not
issued before terminal acknowledgement, a stale lease generation cannot send a
prompt or answer an approval, automation cannot reacquire while a human owns
the lease, a disconnect does not imply return of control, and post-handoff
verification observes every human repository change.

Also test Rig behavior with `tool_concurrency(1)` and above one. Prove a shared
workspace/session cannot run concurrently and separate worktrees can.

