# Contract inputs

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Inputs

- `crates/client/src/retry.rs#RetryPolicy`
- `crates/client/src/error.rs#ClientError`
- `crates/client/tests/retry.rs`
```

At `probe` time, resolve inputs and record:

- file hashes
- symbol locations
- byte count
- estimated tokens
- accepted commit
- expected tool-output budget

A broad glob may be permitted, but it must be expanded and frozen during
acceptance. If `crates/client/**` resolved to 41 files when accepted and later
resolves to 47, the plan becomes `STALE` until reviewed.

The runtime receipt records estimated and actual context use. Claude’s current
`/goal` status reports token spend, which can be captured for calibration.

