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

Use the [documentation home](docs/README.md). It routes the material by
question into three layers:

- [Design](docs/design/README.md) — current product synthesis and delivery path.
- [Research](docs/research/README.md) — checked evidence, comparisons, and
  qualifications.
- [Integrations](docs/integrations/README.md) — Linear and Rig CLI adapter
  studies.

## License

Apache License 2.0. See [LICENSE](LICENSE).
