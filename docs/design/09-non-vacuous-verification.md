# Non-vacuous verification

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Verification must be non-vacuous

A simple verification command is necessary but not sufficient.

Weak gates include:

```text
cargo test exits 0
npm test succeeds
the project builds
```

They can pass when:

- the package name no longer resolves as expected
- a test filter matches zero tests
- all matching tests are ignored
- the command checks the wrong workspace
- expected output was never generated
- a test command succeeds without exercising the changed entry point

A strong gate has three parts:

```text
command
+ structured evidence source
+ proof predicate
```

Example:

```markdown
- **Run:** `mise run test-client-retry`
- **Evidence:** `junit:target/nextest/ci/junit.xml`
- **Proves:** `tests >= 3 && failures == 0`
```

First proof adapters:

1. `junit` — test count, failures, skipped tests, named cases.
2. `cargo-json` — packages, targets, compiler artifacts, diagnostics.
3. `sarif` — static-analysis findings.
4. `git-diff` — changed, created, deleted, and renamed paths.
5. `file` — path existence, count, hash, and exact-content predicates.
6. `command-json` — repository-specific tools that emit a documented JSON schema.

Arbitrary terminal-text regexes should be lower-assurance fallback only.
Structured reports are more stable.

The command runner declares:

```text
executable
arguments
working directory
timeout
output limit
network policy
environment allowlist
```

Strict mode rejects shell pipes, redirections, command substitution, and
undeclared variables. A human may explicitly opt into shell execution for a
trusted repository; it is not the default.

