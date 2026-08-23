# Recommended Rust architecture

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Recommended Rust architecture

Do not begin by writing a complete multi-agent orchestrator. Build the smallest
authoritative core first.

```text
crates/
├── plan-contract/
│   ├── Markdown parser
│   ├── typed PlanContract IR
│   ├── source spans
│   ├── lint rules
│   ├── dependency graph
│   └── context estimator
│
├── plan-proof/
│   ├── command policy
│   ├── JUnit adapter
│   ├── Cargo JSON adapter
│   ├── SARIF adapter
│   ├── Git diff adapter
│   └── evidence receipts
│
├── plan-hosts/
│   ├── Claude Code hook renderer
│   ├── Codex hook renderer
│   └── Grok skill and headless/ACP controller
│
└── planlint/
    └── CLI
```

Practical stack:

- `comrak` for CommonMark/GFM parsing and AST traversal
- `miette` for Rust-like diagnostics with source spans
- `petgraph` for dependency graphs and cycle detection
- `cargo_metadata` for Cargo metadata and JSON message processing
- `globset`, `ignore`, and `camino` for path handling
- `blake3` for fingerprints
- `serde` for generated receipts, never the canonical plan format
- `clap` for the CLI

Suggested commands:

```bash
# Static, side-effect-free compilation
planlint check roadmap/<slug>/plan/

# Resolve inputs and execute every proposed proof once
planlint probe roadmap/<slug>/plan/014-bounded-retry.md

# Record human acceptance and freeze the contract
planlint accept roadmap/<slug>/plan/014-bounded-retry.md

# Render the minimal context packet
planlint packet roadmap/<slug>/plan/014-bounded-retry.md

# Verify current changes
planlint verify roadmap/<slug>/plan/014-bounded-retry.md --worktree

# Reconcile plan state against repository evidence
planlint reconcile roadmap/<slug>/plan/

# Generate host integrations
planlint host install claude
planlint host install codex
planlint host install grok   # skill plus headless/ACP controller

# Explain a diagnostic
planlint explain PL041
```

Output formats:

```text
human compiler diagnostics
JSON
SARIF 2.1.0
compact agent diagnostics
```

Compact agent output contains only:

```text
diagnostic ID
file and span
observed fact
required fact
one suggested correction
```

This prevents the verifier from consuming excessive context.

