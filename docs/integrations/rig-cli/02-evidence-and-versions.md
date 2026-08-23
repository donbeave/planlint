# Evidence and versions

[← Integrations index](../README.md) · [Rig CLI index](README.md)

> Preserved from the original Rig CLI adapter study.

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

