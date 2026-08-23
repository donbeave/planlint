# Configuration and trust boundaries

[← Integrations index](../README.md) · [Rig CLI index](README.md)

> Preserved from the original Rig CLI adapter study.

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

