# Rig CLI adapter research

[← Integrations index](../README.md)

Research on invoking real coding-agent CLIs and long-lived protocols from a
Rig host. Start with the host contract, then choose batch or protocol mode.

## Decision path

1. [Verdict](01-verdict.md) — recommended boundary.
2. [Evidence and versions](02-evidence-and-versions.md) — pinned assumptions.
3. [Rig integration point](03-rig-integration.md) — where the adapter sits.
4. [Compatibility matrix](04-compatibility-matrix.md) — backend choices.
5. [Normalized host contract](05-host-contract.md) — common adapter shape.

## Execution modes

- [Human takeover](06-human-takeover.md)
- [Safe batch profiles](07-batch-profiles.md)
- [Long-lived protocols](08-long-lived-protocols.md)

## Safety and delivery

- [Process supervisor](09-process-supervisor.md)
- [Configuration and trust boundaries](10-trust-boundaries.md)
- [Sessions and verification](11-sessions-and-verification.md)
- [Test strategy](12-test-strategy.md)
- [Implementation order](13-implementation-order.md)
- [Sources](14-sources.md)
