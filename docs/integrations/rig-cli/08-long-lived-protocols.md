# Long-lived protocols

[← Integrations index](../README.md) · [Rig CLI index](README.md)

> Preserved from the original Rig CLI adapter study.

## Long-lived protocols

### ACP v1: Grok, OpenCode, and Kimi

ACP is the real shared protocol for these three agents. Use the official Rust
crate [`agent-client-protocol`](https://docs.rs/agent-client-protocol/latest/agent_client_protocol/)
and implement the **client** role.

```text
spawn CLI with piped stdin/stdout
  -> initialize with protocolVersion 1 and honest client capabilities
  <- chosen version, agent capabilities, auth methods
  -> authenticate when required
  -> session/new { cwd: absolute path, mcpServers: [] }
  -> session/prompt
  <- session/update notifications
  <-> permission / filesystem / terminal reverse requests
  <- session/prompt result with stopReason
```

Commands:

```text
grok agent stdio
opencode acp --pure --cwd <workspace>
kimi acp
```

ACP requires capability negotiation. Omitted capabilities are unsupported.
All protocol paths are absolute. A client that advertises filesystem or
terminal callbacks must implement them; otherwise omit those capabilities and
fail unsupported reverse requests explicitly.

For permission requests, match a controller rule to the offered option kinds.
Default to a reject option. Never select `allow_always` merely because the
agent offered it. On cancellation, send `session/cancel`; resolve outstanding
permission requests with the ACP `cancelled` outcome, wait a bounded grace
period, then terminate the process tree if it does not settle.

The terminal condition is the `session/prompt` response and its `stopReason`,
not the last `session/update`. Message chunks with the same optional
`messageId` belong together. Treat extensions under vendor prefixes as
optional capabilities, not portable ACP behavior.

Vendor differences remain:

- Grok supports ACP directly and exposes xAI-specific extensions. Its session
  `_meta.yoloMode` is vendor-specific; do not use it as the portable policy.
- OpenCode starts an embedded HTTP server behind its ACP process. Its ACP
  permission handler offers allow-once, allow-always, and reject choices and
  fails closed when no handler is available.
- Kimi ACP can issue permission, filesystem, and session updates, but shell
  commands execute locally inside the Kimi process rather than through an ACP
  terminal reverse call. The outer sandbox is still required.

Draft ACP v2 is out of scope for the first adapter. Negotiate stable v1 and pin
the Rust SDK version.

### Codex app-server

`codex app-server --listen stdio://` is an experimental bidirectional JSONL
protocol. The lifecycle is:

```text
initialize -> initialized
thread/start | thread/resume | thread/fork
turn/start
item and delta notifications
turn/completed
```

Interrupt with `turn/interrupt`. The server can send approval and elicitation
requests back to the client. It can report overload as JSON-RPC error `-32001`;
retry only with bounded exponential backoff and jitter.

Generate JSON Schema or TypeScript definitions from the exact installed
binary (`codex app-server generate-json-schema ...`) and test them with that
binary. Do not hand-copy `main` branch types into a client pinned to another
release. A third-party `codex-acp` adapter exists, but adding it changes the
trust and version boundary; it is not the baseline design.

### Claude duplex stream / SDK

Claude can keep a bidirectional stream open with `claude -p --input-format
stream-json --output-format stream-json --verbose`. That wire is
Claude-specific. If the integration needs permission callbacks, hooks, or
durable in-process control, use a small version-pinned official Agent SDK
sidecar instead of guessing the wire. Process-per-turn plus `--resume` remains
simpler for a Rust-only first implementation.

