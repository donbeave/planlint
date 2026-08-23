# Safe batch profiles

[← Integrations index](../README.md) · [Rig CLI index](README.md)

> Preserved from the original Rig CLI adapter study.

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

