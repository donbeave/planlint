# Calling real coding-agent CLIs from Rig

Research cut: 2026-08-23. Commands and protocol surfaces were checked against
current official documentation and public source on this date.

## Result

Rig does not provide a generic adapter for another coding-agent CLI. Its model
provider and tool abstractions run a Rig-owned agent loop. To call a real
Claude Code, Codex, Grok Build, OpenCode, or Kimi Code process, the host must
own an adapter around that process or service.

```text
planlint controller
  -> Rig orchestration / policy
  -> backend adapter
  -> CLI child process, ACP agent, or HTTP server
  -> session events and tool progress
  -> deterministic verifier
  -> receipt and terminal goal state
```

The external agent remains the worker. The controller remains authoritative for
the accepted contract, working directory, allowed paths, timeout/cancellation,
retry policy, proof commands, changed-file checks, and `PASS`. Agent text,
session status, or a model-written “done” is never repository acceptance.

## Integration matrix

| Agent | One-shot / streaming | Long-lived control | Session continuation | Best Rig adapter |
| --- | --- | --- | --- | --- |
| Claude Code | `claude -p ... --output-format json` or `stream-json`; partial events require `--verbose --include-partial-messages` | No official ACP surface found; use CLI or the official Agent SDK through a sidecar | `--continue` or `--resume <id>` | Process adapter; JSONL parser; optional TypeScript/Python SDK sidecar |
| Codex | `codex exec --json ...` emits JSONL state events | `codex app-server` is bidirectional JSON-RPC over JSONL stdio; currently experimental | `codex exec resume <id>` or app-server `thread/resume` | Process adapter for stable batch work; app-server client for persistent sessions |
| Grok Build | `grok -p ... --output-format json` or `streaming-json` | `grok agent stdio` speaks ACP over stdin/stdout | `--session-id`, `--resume`, or `--continue` | ACP client for persistent work; process adapter for batch work |
| OpenCode | `opencode run ... --format json` | `opencode acp` speaks ACP; `opencode serve` exposes HTTP/OpenAPI plus SSE | `opencode run --session <id>` or `--continue`; HTTP sessions are explicit | HTTP client when a server is acceptable; otherwise ACP client |
| Kimi Code CLI (TypeScript) | `kimi -p ... --output-format stream-json` | `kimi acp` speaks ACP; `kimi web` exposes REST/WebSocket APIs | `--session <id>` or `--continue` | ACP client; use the official TypeScript SDK only through a Node sidecar |
| Other ACP agents | Agent-specific batch mode | Standard ACP subprocess interface | `session/load` only when the agent advertises it | Same ACP client after capability negotiation |

The old Python `MoonshotAI/kimi-cli` repository is being wound down. “Kimi
Code TypeScript” means the current [`MoonshotAI/kimi-code`](https://github.com/MoonshotAI/kimi-code)
repository and its [`@moonshot-ai/kimi-code-sdk`](https://github.com/MoonshotAI/kimi-code/tree/main/packages/node-sdk)
package.

## Per-agent call surfaces

### Claude Code

The official non-interactive entry point is the Agent SDK-compatible CLI:

```text
claude -p "<contract and task>" --output-format stream-json \
  --verbose --include-partial-messages
```

Use `json` when only the final structured result is needed. Use
`stream-json` for progress, tool events, and cancellation-aware monitoring.
The result includes session metadata; resume with `claude --resume <id>` or
continue the latest local session with `claude --continue`.

Claude Code documents official TypeScript and Python Agent SDKs, but not a
Rust SDK or ACP server. Therefore a Rust host should spawn `claude -p`, or run
a small SDK sidecar when it needs SDK-only hooks or callbacks. Do not pretend
that Claude's CLI JSONL format is ACP.

### Codex

For a bounded batch run:

```text
codex exec --json "<contract and task>"
codex exec resume <session-id> --json "<next bounded instruction>"
```

The official `--json` mode is newline-delimited state events. `codex exec
resume` continues an exec session by ID or can select the latest session.

For a persistent controller, launch:

```text
codex app-server
```

The app-server protocol is JSON-RPC-shaped JSONL over stdio. The lifecycle is
`initialize` → `initialized` → `thread/start` or `thread/resume` → `turn/start`
→ streamed notifications → `turn/completed`; interrupt with `turn/interrupt`.
The server can also send approval, permission, MCP, or elicitation requests to
the client. Rig must answer those requests or deliberately reject them.

The app-server is currently marked experimental. Pin the Codex version and
generate its matching TypeScript or JSON Schema artifacts when implementing a
client; do not hand-copy an unpinned schema.

### Grok Build

Headless mode is directly scriptable:

```text
grok -p "<contract and task>" --output-format streaming-json --cwd <repo>
grok -p "<next bounded instruction>" --resume <session-id> \
  --output-format streaming-json
```

`json` returns one final object; `streaming-json` emits newline-delimited
events. Named sessions use `--session-id`; existing sessions use `--resume` or
`--continue`.

For a long-lived integration, launch `grok agent stdio`. This is an ACP agent
server. The ACP client performs initialization, creates a session, sends
`session/prompt`, collects `session/update` chunks, handles client callbacks,
and can send `session/cancel`.

### OpenCode

The simplest batch path is:

```text
opencode run --format json --dir <repo> "<contract and task>"
```

OpenCode's JSON run format is intended for automation. Its session flags allow
continuation with `--session <id>` or `--continue`.

For direct programmatic control, start:

```text
opencode serve --hostname 127.0.0.1 --port 4096
```

The server publishes OpenAPI at `/doc`, session creation and message APIs,
abort/permission endpoints, and an SSE event stream. A Rig HTTP adapter can
create a session, post a message, consume events, abort the session, and then
query the diff before verification. Protect non-loopback servers with the
documented basic-auth environment variables.

`opencode acp` is the alternative stdio integration. It speaks ACP using
newline-delimited JSON. Use it when the same generic ACP client should also
drive Grok and Kimi.

### Kimi Code CLI (TypeScript)

The current TypeScript CLI supports machine-readable output:

```text
kimi -p "<contract and task>" --output-format stream-json
kimi --continue -p "<next bounded instruction>" \
  --output-format stream-json
```

The official CLI reference also exposes `kimi acp` and `kimi web`. `kimi acp`
is the preferred Rust integration: Kimi Code is launched as a child ACP agent
and reuses its existing local authentication. `kimi web` is a local
REST/WebSocket service and publishes OpenAPI/AsyncAPI descriptions.

The repository also publishes a TypeScript SDK. A Rust process should not
embed Node implicitly; if SDK use is required, run a version-pinned Node
sidecar and give it an explicit IPC contract. For direct CLI orchestration,
ACP avoids that extra runtime.

## ACP from Rust

ACP is the useful common protocol, not a universal replacement for every CLI.
The official protocol uses JSON-RPC 2.0. A normal client flow is:

```text
spawn agent command with stdin/stdout pipes
  -> initialize
  -> authenticate when required
  -> session/new
  -> session/prompt
  <- session/update notifications
  <-> permission, filesystem, and terminal callbacks
  -> session/cancel when the controller stops the run
  <- session/prompt result with stop reason
```

The official Rust SDK is [`agent-client-protocol`](https://github.com/agentclientprotocol/rust-sdk).
Use its stable protocol surface for the first adapter. Treat draft ACP v2 as
opt-in and version-pinned. Rig is an ACP client in this arrangement; the CLI
is the ACP agent. Do not implement the agent role unless planlint itself is
being exposed to editors.

ACP has an important consequence: an agent may ask the client to read/write
files, run terminals, or resolve permissions. A client that only sends prompts
but does not implement the negotiated callbacks may start successfully and
then fail on the first tool call. Decide explicitly whether the external CLI
owns its own workspace tools or whether Rig proxies them.

## Rig adapter contract

Keep the backend-specific wire code behind one narrow controller-facing
interface:

```text
start(spec: BackendSpec) -> AgentSession
prompt(session, instruction) -> EventStream + TurnResult
cancel(session) -> Result
close(session) -> Result
```

`BackendSpec` should include the executable or URL, argument vector, exact
working directory, environment allowlist, model/permission flags, timeout,
and protocol version. Persist only the external session identifier and the
controller state in the goal record; treat raw transcripts as sensitive
artifacts with explicit retention.

Required lifecycle:

1. Preflight the binary/version, authentication, repository, and worktree.
2. Start a session or resume the recorded session ID.
3. Send the immutable contract, current bounded diagnostic, and exact proof
   command. Do not send an unconstrained “keep going” prompt.
4. Parse events into normalized progress, tool, text, error, and terminal
   records. Preserve raw events for debugging, but never interpret agent text
   as proof.
5. Cancel on controller timeout or terminal policy. Kill the whole process
   group/job object so child shells do not survive the adapter.
6. Run deterministic verification and changed-path checks outside the agent.
7. Resume or start a fresh corrective session only within the persisted retry
   budget. A fresh session must receive the contract and diagnostic again.
8. Emit `PASS` only after the external verifier succeeds and the final diff
   matches the allowed path set.

## Security and reliability rules

- Spawn with an argument vector. Never interpolate the prompt into `sh -c`.
- Use a disposable worktree or an explicit repository root. Validate the
  agent's actual cwd before accepting any change.
- Do not default to `--always-approve`, `--yolo`, `--auto`, or dangerous
  sandbox bypass flags. Make approval policy a controller configuration and
  record it in the receipt.
- Pass only required environment variables. Keep API keys out of prompts,
  captured stdout, and persisted event logs.
- Enforce allowed paths before tool execution where the protocol permits it,
  then re-check the complete Git diff after the process exits.
- Bound wall time, model turns, corrective rounds, output bytes, and child
  process count. Treat malformed JSONL, auth failure, protocol mismatch,
  process exit, and orphaned children as terminal infrastructure failures.
- Pin CLI versions in integration tests. CLI flags, JSONL event shapes, ACP
  capabilities, and Codex app-server schemas can change independently.

## Recommended implementation order

1. Build a generic Rust process runner: exact cwd, piped stdio, cancellation,
   process-group cleanup, stderr capture, timeout, and exit classification.
2. Add narrow JSONL adapters for Claude Code, Codex `exec`, Grok headless,
   OpenCode `run`, and Kimi Code `-p`. This covers batch execution without
   implementing agent callbacks.
3. Add one ACP client using the official Rust SDK. Enable Grok, OpenCode, Kimi,
   and other ACP agents through command configuration and capability checks.
4. Add OpenCode HTTP and Codex app-server adapters only when persistent
   bidirectional control is needed. Both expose richer control than a one-shot
   process, but both require more version-specific schema handling.
5. Keep planlint's contract parser, deterministic verifier, changed-path
   guard, retry budget, and receipt outside every agent-specific adapter.

## Official sources

- [Claude Code headless and scripting](https://code.claude.com/docs/en/headless)
- [Claude Code CLI reference](https://code.claude.com/docs/en/cli-usage)
- [Claude Code sessions](https://code.claude.com/docs/en/sessions)
- [Codex developer command reference](https://learn.chatgpt.com/docs/developer-commands?surface=cli)
- [Codex CLI source README](https://github.com/openai/codex/blob/main/codex-rs/README.md)
- [Codex app-server protocol](https://github.com/openai/codex/blob/main/codex-rs/app-server/README.md)
- [Grok headless and scripting](https://docs.x.ai/build/cli/headless-scripting)
- [Grok CLI reference](https://docs.x.ai/build/cli/reference)
- [OpenCode CLI](https://dev.opencode.ai/docs/cli)
- [OpenCode server API](https://dev.opencode.ai/docs/server/)
- [OpenCode ACP](https://dev.opencode.ai/docs/acp)
- [Kimi Code CLI reference](https://www.kimi.com/code/docs/en/kimi-code-cli/reference/kimi-command)
- [Kimi Code IDE/ACP integration](https://www.kimi.ai/help/kimi-code/cli-ides)
- [Kimi Code TypeScript repository](https://github.com/MoonshotAI/kimi-code)
- [Agent Client Protocol overview](https://github.com/agentclientprotocol/agent-client-protocol/blob/main/docs/protocol/v1/overview.mdx)
- [Agent Client Protocol Rust SDK](https://github.com/agentclientprotocol/rust-sdk)
