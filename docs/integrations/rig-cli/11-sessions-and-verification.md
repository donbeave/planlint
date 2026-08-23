# Sessions and verification

[← Integrations index](../README.md) · [Rig CLI index](README.md)

> Preserved from the original Rig CLI adapter study.

## Sessions, worktrees, and verification

The host owns a map from controller run ID to vendor session ID. Never use
global “continue latest” flags in concurrent automation; they race on ambient
state. Prefer explicit resume IDs and verify that the resumed session belongs
to the expected workspace. Fork when reusing context for a divergent attempt.

Default execution sequence:

1. Preflight binary, version, authentication, repository, clean baseline, and
   worktree.
2. Start or resume the controller-mapped session.
3. Send the immutable contract, current bounded diagnostic, and exact proof
   command. Do not send an unconstrained “keep going”.
4. Normalize progress and persist a bounded raw-event artifact.
5. Require the vendor's terminal contract and clean process/protocol exit.
6. Run deterministic verification outside the worker.
7. Compare the complete Git diff against the allowed path set.
8. Resume or fork only within the persisted retry budget.
9. Emit `PASS` only when verification and scope checks succeed.

Keep a mutating session bound to one worktree. If Rig is configured for
parallel tool calls, use a keyed lock or allocate separate worktrees. External
session state and repository state must move together.

