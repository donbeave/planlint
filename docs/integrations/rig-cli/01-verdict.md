# Rig CLI adapter verdict

[← Integrations index](../README.md) · [Rig CLI index](README.md)

> Preserved from the original Rig CLI adapter study.

## Document context

Research cut: 2026-08-23.


## Verdict

Rig does not natively turn another coding-agent CLI into a Rig agent. The
correct integration is a host-owned Rig `Tool` whose implementation supervises
the real agent process or protocol client.

Human takeover after a blocked `/goal` is achievable, but it is also a host
feature rather than a Rig feature. The host must persist the external session,
stop automated writes, release an exclusive session/worktree lease, and expose
a backend-specific attach or resume surface. Rig supplies the tool and hook
seams; it does not supply a terminal multiplexer, session broker, `/goal`
lifecycle, or handoff state machine.

```text
Rig agent
  -> typed Tool call
  -> host policy and process supervisor
  -> Claude / Codex / Grok / OpenCode / Kimi
  -> normalized worker result
  -> deterministic verifier outside the worker
```

Use two transport tiers:

1. **Process per turn:** the simplest common denominator. Spawn the CLI's
   documented headless command, parse JSON or JSONL, retain its session ID, and
   resume in a later process when needed.
2. **Long-lived protocol:** use ACP v1 for Grok Build, OpenCode, and Kimi Code.
   Codex has its own experimental app-server protocol. Claude Code has a
   proprietary duplex `stream-json` mode and official TypeScript/Python Agent
   SDKs, but no official ACP server.

Start with process-per-turn adapters. Add ACP when startup cost, permission
callbacks, rich progress, or cancellation justify a persistent connection.
Do not force all five products behind a fake common wire format.

The external agent remains a probabilistic worker. The controller remains
authoritative for the accepted contract, working directory, permissions,
timeout, retry policy, proof commands, changed-file checks, and `PASS`. Exit
code zero, an end-of-turn event, or agent text saying “done” proves only that
the worker turn ended.

