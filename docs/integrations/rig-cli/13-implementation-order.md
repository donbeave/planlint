# Recommended implementation order

[← Integrations index](../README.md) · [Rig CLI index](README.md)

> Preserved from the original Rig CLI adapter study.

## Recommended implementation order

1. Implement the generic bounded process supervisor and fake-process tests.
2. Add batch adapters in this order: Codex, Claude, Grok, OpenCode, Kimi. Codex
   has the clearest typed terminal stream; Kimi needs the strongest external
   policy because print mode auto-approves.
3. Implement one ACP v1 client using the official Rust SDK; enable Grok,
   OpenCode, and Kimi through fixed command profiles and capability tests.
4. Add Codex app-server only when reverse requests or persistent processes are
   required.
5. Add Claude's official SDK sidecar only for SDK-only callbacks.
6. Add the blocked-goal handoff state machine first for Codex app-server, then
   implement only capability-tested equivalents for other backends.
7. Keep the contract parser, verifier, changed-path guard, retry budget, and
   receipt independent of every backend.

