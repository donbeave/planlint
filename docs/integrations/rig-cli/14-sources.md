# Sources

[← Integrations index](../README.md) · [Rig CLI index](README.md)

> Preserved from the original Rig CLI adapter study.

## Sources

### Rig and ACP

- [Rig 0.42 typed `Tool`](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/tool/mod.rs)
- [Rig 0.42 `AgentBuilder::tool`](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/builder.rs)
- [Rig 0.42 tool concurrency](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/runner.rs)
- [Rig 0.42 tool-result hook stop action](https://github.com/0xPlaygrounds/rig/blob/d5a34986a1ad57f1e9c5984b82f8d7438ffc717e/crates/rig-agent/src/agent/hook.rs#L1086-L1116)
- [ACP v1 overview](https://agentclientprotocol.com/protocol/v1/overview)
- [ACP v1 initialization](https://agentclientprotocol.com/protocol/v1/initialization)
- [ACP v1 prompt turns](https://agentclientprotocol.com/protocol/v1/prompt-turn)
- [ACP v1 permission calls](https://agentclientprotocol.com/protocol/v1/tool-calls)
- [ACP v1 cancellation](https://agentclientprotocol.com/protocol/v1/cancellation)
- [ACP session resume](https://agentclientprotocol.com/rfds/session-resume)
- [Official ACP Rust SDK](https://github.com/agentclientprotocol/rust-sdk)

### Claude Code

- [Headless mode](https://code.claude.com/docs/en/headless)
- [CLI reference](https://code.claude.com/docs/en/cli-usage)
- [Permission modes](https://code.claude.com/docs/en/permission-modes)
- [Permissions](https://code.claude.com/docs/en/permissions)
- [Sessions](https://code.claude.com/docs/en/sessions)
- [Background agents and `claude attach`](https://code.claude.com/docs/en/agent-view)
- [Remote Control](https://code.claude.com/docs/en/remote-control)
- [Environment variables](https://code.claude.com/docs/en/env-vars)

### Codex

- [Noninteractive mode](https://developers.openai.com/codex/noninteractive)
- [Pinned `codex exec` parser](https://github.com/openai/codex/blob/c9b19deb09c1841ce7acc33ddb96276030936a29/codex-rs/exec/src/cli.rs)
- [Pinned `codex exec` events](https://github.com/openai/codex/blob/c9b19deb09c1841ce7acc33ddb96276030936a29/codex-rs/exec/src/exec_events.rs)
- [Pinned headless approval default](https://github.com/openai/codex/blob/c9b19deb09c1841ce7acc33ddb96276030936a29/codex-rs/exec/src/lib.rs)
- [Pinned app-server protocol](https://github.com/openai/codex/blob/c9b19deb09c1841ce7acc33ddb96276030936a29/codex-rs/app-server/README.md)
- [Pinned app-server goal methods and statuses](https://github.com/openai/codex/blob/c9b19deb09c1841ce7acc33ddb96276030936a29/codex-rs/app-server-protocol/src/protocol/v2/thread.rs#L768-L863)

### Grok Build

- [Headless and scripting](https://docs.x.ai/build/cli/headless-scripting)
- [CLI reference](https://docs.x.ai/build/cli/reference)
- [Permissions](https://docs.x.ai/build/features/permissions)
- [Sandbox](https://docs.x.ai/build/features/sandbox)
- [Pinned headless contract](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-pager/docs/user-guide/14-headless-mode.md)
- [Pinned ACP agent mode](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-pager/docs/user-guide/15-agent-mode.md)
- [Pinned session management](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-pager/docs/user-guide/17-sessions.md)
- [Pinned goal update event](https://github.com/xai-org/grok-build/blob/07b2f7144fd5c5c9d3dd1966937a87852d2dbdb8/crates/codegen/xai-grok-shell/src/extensions/notification.rs#L926-L1034)

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
- [Local server and API](https://moonshotai.github.io/kimi-code/en/guides/server.html)
- [Pinned CLI reference source](https://github.com/MoonshotAI/kimi-code/blob/368b4b7400228028006c9b0d5789fcced85f75aa/docs/en/reference/kimi-command.md)
- [Pinned headless goal contract](https://github.com/MoonshotAI/kimi-code/blob/368b4b7400228028006c9b0d5789fcced85f75aa/apps/kimi-code/src/cli/goal-prompt.ts#L5-L102)
- [Pinned print runner](https://github.com/MoonshotAI/kimi-code/blob/368b4b7400228028006c9b0d5789fcced85f75aa/apps/kimi-code/src/cli/run-prompt.ts)
- [Pinned stream renderer](https://github.com/MoonshotAI/kimi-code/blob/368b4b7400228028006c9b0d5789fcced85f75aa/apps/kimi-code/src/cli/prompt-render.ts)
