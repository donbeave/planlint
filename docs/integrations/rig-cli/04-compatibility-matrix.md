# Compatibility matrix

[← Integrations index](../README.md) · [Rig CLI index](README.md)

> Preserved from the original Rig CLI adapter study.

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

