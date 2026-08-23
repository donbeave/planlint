# planlint

`planlint` is a proposed Rust plan-contract compiler for deterministic
verification of AI coding-agent work.

The core idea:

```text
accepted specification
        ↓
pure-Markdown plan item
        ↓
Rust static checker
        ↓
one fresh /goal execution
        ↓
deterministic verifier
        ↓
evidence receipt
        ↓
DONE or explicit failure state
```

This repository currently stores the research and design basis. It does not
yet claim to implement the compiler.

## Documentation

- [Research archive](docs/research.md) — complete design proposal and source links.
- [Research review](docs/research-review.md) — checked claims, corrections, and open verification items.

## License

Apache License 2.0. See [LICENSE](LICENSE).
