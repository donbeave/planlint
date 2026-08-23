# Combining with agent-spec

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Combining with `agent-spec`

Do not fork all of `agent-spec` immediately.

Use this division:

```text
agent-spec
    validates the behavioral/specification contract

planlint
    validates the bounded execution contract

host continuation adapters
    provide native /goal, hook, or outer-controller persistence

Tailrocks reconcile
    validates package-level completion and state
```

A plan acceptance criterion can invoke `agent-spec` directly:

```markdown
### ACC-4: Behavioral contract

- **Run:** `agent-spec lifecycle specs/client-retry.spec.md --code .`
- **Evidence:** `agent-spec-json:stdout`
- **Proves:** `passed >= 2 && failed == 0 && skipped == 0 && uncertain == 0`
```

This keeps “what the product must do” separate from “how one executor should
modify the repository.” Initially integrate through the stable CLI and pin an
accepted `agent-spec` version. Upstream shared parser or evidence APIs later if
the experiment proves useful.

