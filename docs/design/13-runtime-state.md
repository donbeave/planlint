# Runtime state

[← Documentation home](../README.md) · [Design index](README.md)

> Preserved from the original design research archive.

## Runtime state is separate from the plan

The Markdown plan is immutable after acceptance. Do not put mutable checkboxes
or current status inside the plan contract.

Use a separate state machine:

```text
TODO
  ↓
CLAIMED
  ↓
CANDIDATE
  ↓
VERIFIED
  ↓
DONE
```

Failure states:

```text
BLOCKED   — environment or precondition failure
PLAN_GAP  — correct implementation requires unapproved work
STALE     — plan or starting inputs changed
FAIL      — proof failed within the permitted correction loop
REJECTED  — human explicitly chose not to execute it
```

Only the Rust controller transitions state. The agent reports evidence but
cannot mark itself `DONE`.

Generated receipts may be JSON because they are output artifacts, not the
authored contract:

```json
{
  "plan_id": "PLAN-014",
  "contract_hash": "blake3:...",
  "base_commit": "4f12a8c",
  "head_commit": "98ca301",
  "status": "verified",
  "changed_paths": [
    "crates/client/src/retry.rs",
    "crates/client/tests/retry.rs"
  ],
  "checks": [
    {
      "id": "ACC-1",
      "tests": 3,
      "failures": 0,
      "evidence_hash": "blake3:..."
    }
  ],
  "unexpected_paths": 0
}
```

Bind the receipt to:

```text
accepted plan hash
+ accepted starting commit
+ resulting commit
+ changed paths
+ exact commands
+ structured evidence hashes
```

This makes the result replayable and auditable.

