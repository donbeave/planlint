# Calling real coding-agent CLIs from Rig

Research cut: 2026-08-23.

## Verdict

Rig does not natively turn another coding-agent CLI into a Rig agent. The
correct integration is a host-owned Rig `Tool` whose implementation supervises
the real agent process or protocol client.

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

## Evidence and versions

The command parsers and event contracts were checked without making paid model
calls. Installed binaries:

| CLI | Checked version |
| --- | --- |
| Claude Code | 2.1.233 |
| Codex CLI | 0.149.0 |
| Grok Build | 1.0.5 |
| OpenCode | 1.18.15 |
| Kimi Code (TypeScript) | 0.36.1 |

Public source was pinned during the review:

| Project | Revision |
| --- | --- |
| Rig | tag `v0.42.0`, `d5a34986a1ad57f1e9c5984b82f8d7438ffc717e` |
| Codex | `c9b19deb09c1841ce7acc33ddb96276030936a29` |
| Grok Build | `07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8` |
| OpenCode | `3a31c4ea801915c0b050df4b3842997ea62b6e93` |
| Kimi Code | `368b4b7400228028006c9b0d5789fcced85f75aa` |

The exact authenticated output of every provider still belongs in
version-pinned integration tests. This research establishes the documented and
source-defined contracts; it does not substitute fabricated live transcripts
for those tests.

## Rig integration point

Rig 0.42's typed `Tool` owns argument decoding, model-visible output, typed
errors, and a call receiving `ToolContext`. Register it with
`AgentBuilder::tool(...)`. Rig's default tool concurrency is one; callers can
opt into parallel calls with `tool_concurrency(n)`.

The model-facing schema should be narrow. Prefer one fixed tool instance per
backend and policy, such as `review_with_codex` or `implement_with_grok`. Do not
let the model provide an executable, arbitrary flags, environment variables,
raw working directory, or a vendor session ID.

```rust
use rig::tool::{Tool, ToolContext};
use serde::{Deserialize, Serialize};
use serde_json::json;

#[derive(Clone)]
struct CodingAgentTool {
    // Fixed executable, backend, workspace policy, permissions, and limits.
    supervisor: AgentSupervisor,
}

#[derive(Deserialize)]
struct WorkerArgs {
    instruction: String,
}

#[derive(Serialize)]
struct WorkerOutput {
    status: WorkerStatus,
    final_text: String,
    diagnostics: Vec<String>,
}

impl Tool for CodingAgentTool {
    const NAME: &'static str = "run_coding_agent";
    type Args = WorkerArgs;
    type Output = WorkerOutput;
    type Error = AgentToolError;

    fn description(&self) -> String {
        "Run the configured coding agent in the controller-owned workspace".into()
    }

    fn parameters(&self) -> serde_json::Value {
        json!({
            "type": "object",
            "properties": {
                "instruction": { "type": "string", "minLength": 1 }
            },
            "required": ["instruction"],
            "additionalProperties": false
        })
    }

    async fn call(
        &self,
        context: &mut ToolContext,
        args: Self::Args,
    ) -> Result<Self::Output, Self::Error> {
        let scope = context.require::<RunScope>()?.clone();
        let run = self.supervisor.run(&scope, args.instruction).await?;
        context.insert_result(run.receipt); // Host-only; not sent to the model.
        Ok(run.output)
    }
}

let agent = rig::agent::AgentBuilder::new(model)
    .tool(CodingAgentTool::new(config))
    .build();
```

This is a structural sketch; project types and constructors are omitted. The
important seam is `Tool::call`. `AgentSupervisor::run` must provide the bounded
I/O, cancellation, parsing, and cleanup described below. `RunScope` should be
injected by the host through `ToolContext`; it can carry the canonical
workspace, controller run ID, session-map key, and a host cancellation token
without exposing those controls to the model.

If `tool_concurrency` is raised above one, serialize calls sharing either a
workspace or external session. Two mutating workers must use separate Git
worktrees. A vendor session must never receive overlapping turns.

## Compatibility matrix

| Agent | Batch command | Machine output and terminal condition | Resume | Long-lived control | Prompt transport |
| --- | --- | --- | --- | --- | --- |
| Claude Code | `claude -p` | `json`: one result object. `stream-json`: JSONL ending in `type: "result"`. Require that terminal result **and** exit 0. | `--resume <id>`; `--continue`; optional `--fork-session` | Proprietary duplex stream JSON or official Agent SDK sidecar | stdin is supported and capped at 10 MB |
| Codex | `codex exec --json -` | JSONL: `thread.started`, item events, then `turn.completed`; failures use `turn.failed` or `error`. Require terminal event and exit 0. | `codex exec resume <id> -`; `fork` also exists | Experimental `codex app-server` JSONL protocol | stdin when prompt is absent or `-` |
| Grok Build | `grok -p` | `json`: one result object. `streaming-json`: JSONL ending in `type: "end"`; failures emit `error` and nonzero. | `--resume <id>` or `--continue`; `--session-id` creates a **new** UUID session only | `grok agent stdio` speaks ACP | no piped prompt; use `--prompt-file` or an argv value |
| OpenCode | `opencode run --format json` | Normalized JSONL events. No dedicated success event: the command exits after `session.status=idle`; `session.error` produces an `error` line and exit 1. Require clean EOF and exit 0. | `--session <id>` or `--continue`; optional `--fork` | `opencode acp`; alternatively `serve` plus HTTP/SSE | stdin is supported and appended to positional text |
| Kimi Code TypeScript | `kimi -p --output-format stream-json` | Assistant/tool JSONL; success writes a final `role: "meta", type: "session.resume_hint"` line, then exits 0. Turn errors exit nonzero. | `--session <id>` or `--continue` | `kimi acp`; alternatively `kimi web` REST/WebSocket | print prompt is an argv value; ACP avoids argv exposure |

The old Python `MoonshotAI/kimi-cli` is not the target here. “Kimi Code
TypeScript” means the current [`MoonshotAI/kimi-code`](https://github.com/MoonshotAI/kimi-code)
CLI.

## Normalized host contract

Keep vendor-specific events behind a small controller interface:

```text
preflight(BackendSpec) -> CapabilitySnapshot
start_or_resume(RunScope) -> AgentSession
prompt(session, instruction) -> EventStream + TurnResult
cancel(session) -> Result
close(session) -> Result
```

Suggested terminal model:

```text
WorkerStatus =
    Completed       # vendor turn completed; acceptance not implied
  | Failed          # vendor reported a terminal turn failure
  | PermissionDenied
  | TimedOut
  | Cancelled
  | ProtocolError   # malformed/missing/contradictory terminal frame
  | ProcessError    # spawn, signal, or non-protocol exit failure
```

Suggested normalized result:

```text
TurnResult {
  backend
  backend_version
  status
  final_text
  session_handle       # controller alias; raw vendor ID stays host-side
  usage                # optional and vendor-qualified
  stop_reason          # optional raw vendor value
  diagnostics
  raw_event_artifact   # sensitive, retention-controlled reference
}
```

Do not erase absence into zero. Missing cost means unknown, not free. Missing
usage means unreported, not zero tokens. Preserve unknown event types so a new
CLI version does not crash a forward-compatible parser, while still requiring
the known terminal contract for the pinned major/version.

## Safe batch argv profiles

The examples below are argument vectors, not shell snippets. Build them with
`tokio::process::Command`; do not concatenate or pass them through `sh -c`.

### Claude Code

Read-only review profile:

```text
claude
  -p
  --safe-mode
  --output-format stream-json
  --verbose
  --permission-mode dontAsk
  --allowedTools Read Glob Grep
  --disallowedTools Bash Edit Write WebFetch WebSearch
  --max-turns 12
```

Write the prompt to stdin and close stdin. Parse the `system/init` metadata and
terminal `result`; retain `session_id`. `--no-session-persistence` is useful
only for deliberately stateless calls and conflicts with later resume.

`--safe-mode` is the sealed profile: it disables project/user customizations
and makes the controller pass required context explicitly. A repo-aware profile
may omit it only after reviewing the repository configuration.

For a mutating profile, keep `dontAsk` but provide explicit allow and deny
rules through a controller-owned settings file or fixed flags. Print mode
silently ignores a settings file that fails validation, so validate generated
settings before launch and keep the outer sandbox authoritative. Never use
`--dangerously-skip-permissions` outside an independent OS/container sandbox.
Claude print mode skips the workspace trust dialog, so the host must admit
only reviewed workspaces.

Use `--json-schema` when the worker's final answer must have a known shape.
Claude places the validated value in `structured_output`; this structures the
report, not the repository proof.

### Codex

Read-only review profile:

```text
codex exec
  --json
  --sandbox read-only
  --cd <canonical-workspace>
  -
```

Write the prompt to stdin. For controlled implementation work, change the
sandbox to `workspace-write`. Do not use `danger-full-access` or
`--dangerously-bypass-approvals-and-sandbox` as an ordinary adapter profile.

Headless `codex exec` defaults to `AskForApproval::Never`; it does not wait for
an interactive approval response. Therefore the sandbox and fixed config are
the enforcement boundary. Parse the first `thread.started.thread_id`, the last
completed `agent_message`, and exactly one terminal `turn.completed`. A
`turn.failed`, fatal `error`, missing terminal event, or nonzero exit fails the
tool call.

`--output-schema <file>` constrains the final answer. `--output-last-message
<file>` is convenient for humans but creates another file lifecycle; parsing
JSONL directly keeps the adapter's output contract in one place.

### Grok Build

Read-only review profile:

```text
grok
  --no-auto-update
  --cwd <canonical-workspace>
  --sandbox read-only
  --output-format streaming-json
  --permission-mode dontAsk
  --tools read_file,grep,list_dir
  --max-turns 12
  --prompt-file <private-temporary-file>
```

Grok does not consume piped stdin as prompt text. Create the prompt file with
owner-only permissions, pass its explicit path, and remove it after the process
has exited. `--session-id` is only for creating a new client-chosen UUID; use
`--resume` for an existing session.

For mutation, use a controller-owned custom sandbox profile extending Grok's
`workspace` profile, plus fixed `--allow` and `--deny` rules under `dontAsk`.
The built-in profile is named `workspace`, not `workspace-write`. `--allow` is
not a closed allowlist by itself; unmatched calls fall through to the
permission mode. `dontAsk` makes those unmatched calls fail closed. Avoid
`--yolo`/`bypassPermissions` unless a separate sandbox is the actual security
boundary.

Grok's built-in sandbox profiles can warn and continue without enforcement if
the platform cannot apply them. An explicitly requested valid custom profile
with kernel-enforced denies fails closed. In either case, inspect startup
diagnostics and keep the controller's outer sandbox authoritative. Also note
that built-in `read-only` blocks writes to the workspace but can read outside
it; use deny paths or an outer sandbox to protect host secrets.

Parse `text`, tool, usage, and unknown events until the mandatory final `end`.
An `end` stop reason such as `max_turn_requests`, `refusal`, or `cancelled` is a
completed protocol exchange but not a successful worker result.

### OpenCode

Batch profile:

```text
opencode run
  --pure
  --format json
  --dir <canonical-workspace>
```

`--pure` disables external plugins for the sealed profile. Write the prompt to
stdin with no positional message. JSON events include
`type`, `timestamp`, `sessionID`, and event-specific data. Important types are
`tool_use`, `step_start`, `step_finish`, `text`, `reasoning`, and `error`.

OpenCode permissions are permissive by default. In noninteractive mode,
questions, plan entry, and plan exit are denied; other permission prompts are
auto-rejected unless `--auto` is set. That is not a read-only sandbox. Supply a
controller-owned config with explicit `permission` rules and run the process in
an external OS/container sandbox. `--auto` is acceptable only when explicit
denies and the outer sandbox already constrain it.

For repeated process calls, `opencode serve` plus `opencode run --attach
http://127.0.0.1:<port>` avoids repeated server and MCP startup. Bind to
loopback and set `OPENCODE_SERVER_PASSWORD`; do not expose an unauthenticated
server.

### Kimi Code TypeScript

Batch profile:

```text
kimi
  -p <instruction>
  --output-format stream-json
```

Use `--session <id>` to resume. The final `meta` line of a successful run has
`type: "session.resume_hint"` and `session_id`; assistant messages before it
contain the final text. Tool progress and diagnostic output remain on stderr.

Kimi print mode forcibly uses its `auto` permission policy, so regular tool
approvals are handled automatically. Static deny rules still apply, but `-p`
cannot be combined with `--auto`, `--yolo`, or `--plan` to change this. Kimi
has no documented CLI filesystem sandbox. A safe host must
use a controller-owned `config.toml` with first-match deny rules **and** an
external OS/container sandbox. For example, a review profile should deny
`Write`, `Edit`, `Bash`, agent spawning, and network tools before any broader
allow rule.

The print prompt is a command-line argument and may be visible to local process
inspection. Use `kimi acp` when prompts may contain secrets or when a durable
session is already needed. Set `KIMI_CODE_NO_AUTO_UPDATE=1` for a pinned
automation environment.

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

## Process supervisor requirements

`tokio::process::Command` is necessary but not sufficient:

1. Resolve the configured executable to an approved absolute path and record
   its version before accepting work.
2. Canonicalize the workspace and prove it is beneath an admitted root. Never
   accept `~`, `/`, an unresolved environment variable, or a model-provided
   path.
3. Spawn an argument vector with an explicit cwd. Never use a shell.
4. Pipe stdin, stdout, and stderr. Drain stdout and stderr concurrently with
   byte caps; an unbounded `wait_with_output` can exhaust memory, while reading
   only one pipe can deadlock the child.
5. Parse UTF-8 JSONL incrementally with a per-line limit and a total event
   limit. Preserve unknown event types; reject malformed required frames.
6. Enforce wall time, idle time, maximum turns, output bytes, and process count.
7. On cancellation or timeout, request protocol cancellation when available,
   close stdin, send graceful termination, wait briefly, then force-kill the
   entire POSIX process group or Windows Job Object. `kill_on_drop(true)` alone
   does not guarantee grandchildren die.
8. Wait for process exit after the terminal event. A valid event followed by a
   nonzero exit is still failure.
9. Capture bounded stderr separately. Never merge logs into the machine JSONL
   parser.
10. Return normalized diagnostics and a host-only receipt. Raw transcripts can
    contain source, prompts, credentials, command output, and reasoning; apply
    explicit encryption, access, and retention policy.

The supervisor should classify retryability. A provider rate limit or Codex
overload may be retryable within policy. Malformed protocol, invalid cwd,
permission-policy violation, authentication failure, or changed-path breach is
not an automatic model retry.

## Configuration and trust boundaries

Each CLI reads more than its command line:

| CLI | User state override | Important limitation |
| --- | --- | --- |
| Claude | `CLAUDE_CONFIG_DIR` | Project `.claude` files still apply; macOS credentials may use Keychain |
| Codex | `CODEX_HOME` | Project instructions/config and the chosen sandbox still matter |
| Grok | `GROK_HOME` | Project `.grok`, compatible Claude settings, hooks, and plugins may apply after trust |
| OpenCode | `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_CACHE_HOME`, `XDG_STATE_HOME`; optional `OPENCODE_CONFIG_DIR` | Custom config layers over other config; project/agent rules and plugins still require review |
| Kimi | `KIMI_CODE_HOME` | Generic user `.agents` resources remain under the real OS home; project config still applies |

Use a dedicated, pre-authenticated automation profile rather than the user's
interactive state. Give the child a documented environment allowlist, but keep
the platform variables and toolchain paths its shell tools genuinely need.
Never place secrets in prompts or argv when a stdin/protocol path exists.

Repository instructions, hooks, skills, MCP servers, and plugins can alter
behavior or execute code. Config-root isolation does not neutralize project
configuration. Admit trusted repositories only, disable unused extension
sources where the CLI supports it, and rely on an outer sandbox for the final
filesystem/network boundary.

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

Also test Rig behavior with `tool_concurrency(1)` and above one. Prove a shared
workspace/session cannot run concurrently and separate worktrees can.

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
6. Keep the contract parser, verifier, changed-path guard, retry budget, and
   receipt independent of every backend.

## Sources

### Rig and ACP

- [Rig 0.42 typed `Tool`](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/tool/mod.rs)
- [Rig 0.42 `AgentBuilder::tool`](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/builder.rs)
- [Rig 0.42 tool concurrency](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/runner.rs)
- [ACP v1 overview](https://agentclientprotocol.com/protocol/v1/overview)
- [ACP v1 initialization](https://agentclientprotocol.com/protocol/v1/initialization)
- [ACP v1 prompt turns](https://agentclientprotocol.com/protocol/v1/prompt-turn)
- [ACP v1 permission calls](https://agentclientprotocol.com/protocol/v1/tool-calls)
- [ACP v1 cancellation](https://agentclientprotocol.com/protocol/v1/cancellation)
- [Official ACP Rust SDK](https://github.com/agentclientprotocol/rust-sdk)

### Claude Code

- [Headless mode](https://code.claude.com/docs/en/headless)
- [CLI reference](https://code.claude.com/docs/en/cli-usage)
- [Permission modes](https://code.claude.com/docs/en/permission-modes)
- [Permissions](https://code.claude.com/docs/en/permissions)
- [Sessions](https://code.claude.com/docs/en/sessions)
- [Environment variables](https://code.claude.com/docs/en/env-vars)

### Codex

- [Noninteractive mode](https://developers.openai.com/codex/noninteractive)
- [Pinned `codex exec` parser](https://github.com/openai/codex/blob/c9b19deb09c1841ce7acc33ddb96276030936a29/codex-rs/exec/src/cli.rs)
- [Pinned `codex exec` events](https://github.com/openai/codex/blob/c9b19deb09c1841ce7acc33ddb96276030936a29/codex-rs/exec/src/exec_events.rs)
- [Pinned headless approval default](https://github.com/openai/codex/blob/c9b19deb09c1841ce7acc33ddb96276030936a29/codex-rs/exec/src/lib.rs)
- [Pinned app-server protocol](https://github.com/openai/codex/blob/c9b19deb09c1841ce7acc33ddb96276030936a29/codex-rs/app-server/README.md)

### Grok Build

- [Headless and scripting](https://docs.x.ai/build/cli/headless-scripting)
- [CLI reference](https://docs.x.ai/build/cli/reference)
- [Permissions](https://docs.x.ai/build/features/permissions)
- [Sandbox](https://docs.x.ai/build/features/sandbox)
- [Pinned headless contract](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-pager/docs/user-guide/14-headless-mode.md)
- [Pinned ACP agent mode](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-pager/docs/user-guide/15-agent-mode.md)

### OpenCode

- [CLI](https://opencode.ai/docs/cli/)
- [Permissions](https://opencode.ai/docs/permissions/)
- [Server](https://opencode.ai/docs/server/)
- [Pinned `run` implementation](https://github.com/anomalyco/opencode/blob/3a31c4ea801915c0b050df4b3842997ea62b6e93/packages/opencode/src/cli/cmd/run.ts)
- [Pinned ACP command](https://github.com/anomalyco/opencode/blob/3a31c4ea801915c0b050df4b3842997ea62b6e93/packages/opencode/src/cli/cmd/acp.ts)
- [Pinned ACP permission handling](https://github.com/anomalyco/opencode/blob/3a31c4ea801915c0b050df4b3842997ea62b6e93/packages/opencode/src/acp/permission.ts)

### Kimi Code TypeScript

- [CLI reference](https://www.kimi.com/code/docs/en/kimi-code-cli/reference/kimi-command)
- [ACP reference](https://www.kimi.com/code/docs/en/kimi-code-cli/reference/kimi-acp)
- [Configuration files](https://www.kimi.com/code/docs/en/kimi-code-cli/configuration/config-files)
- [Environment variables](https://www.kimi.com/code/docs/en/kimi-code-cli/configuration/env-vars)
- [Pinned CLI reference source](https://github.com/MoonshotAI/kimi-code/blob/368b4b7400228028006c9b0d5789fcced85f75aa/docs/en/reference/kimi-command.md)
- [Pinned print runner](https://github.com/MoonshotAI/kimi-code/blob/368b4b7400228028006c9b0d5789fcced85f75aa/apps/kimi-code/src/cli/run-prompt.ts)
- [Pinned stream renderer](https://github.com/MoonshotAI/kimi-code/blob/368b4b7400228028006c9b0d5789fcced85f75aa/apps/kimi-code/src/cli/prompt-render.ts)
