# Risks and controls

[← Integrations index](../README.md) · [Linear index](README.md)

> Preserved from the original Linear integration study.

## Risks and controls

- **Two masters:** field-level ownership, managed blocks, hashes, and drift
  detection.
- **Cloud dependency:** local verifier and Git source remain fully usable
  without Linear.
- **API drift/limits:** typed SDK, narrow filtered queries, cursor pagination,
  webhook-driven updates, retries with backoff, and GraphQL error inspection.
- **Webhook spoof/replay:** verify raw-body HMAC and timestamp; deduplicate by
  delivery ID.
- **Over-sharing:** least-privilege OAuth/API scopes; read-only MCP where
  possible; never store secrets in Linear descriptions or agent activity.
- **False completion:** only receipts from deterministic commands may produce
  the verified signal.
- **Graph mismatch:** keep typed dependency semantics and cycle policy in the
  Markdown contract/compiler; use Linear’s native edges for visualization.
- **Vendor lock-in:** isolate Linear behind an adapter and keep stable IDs,
  source files, and receipts in Git.

