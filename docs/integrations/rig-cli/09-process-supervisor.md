# Process supervisor

[← Integrations index](../README.md) · [Rig CLI index](README.md)

> Preserved from the original Rig CLI adapter study.

## Process supervisor requirements

`tokio::process::Command` is necessary but not sufficient:

1. Resolve the configured executable to an approved absolute path and record
   its version before accepting work.
2. Canonicalize the workspace and prove it is beneath an admitted root. Never
   accept `~`, `/`, an unresolved environment variable, or a model-provided
   path.
3. Spawn an argument vector with an explicit cwd. Never use a shell.
4. Pipe stdin, stdout, and stderr. Drain stdout and stderr concurrently with
   byte caps; an unbounded `wait_with_output` can exhaust memory, while reading
   only one pipe can deadlock the child.
5. Parse UTF-8 JSONL incrementally with a per-line limit and a total event
   limit. Preserve unknown event types; reject malformed required frames.
6. Enforce wall time, idle time, maximum turns, output bytes, and process count.
7. On cancellation or timeout, request protocol cancellation when available,
   close stdin, send graceful termination, wait briefly, then force-kill the
   entire POSIX process group or Windows Job Object. `kill_on_drop(true)` alone
   does not guarantee grandchildren die.
8. Wait for process exit after the terminal event. A valid event followed by a
   nonzero exit is still failure.
9. Capture bounded stderr separately. Never merge logs into the machine JSONL
   parser.
10. Return normalized diagnostics and a host-only receipt. Raw transcripts can
    contain source, prompts, credentials, command output, and reasoning; apply
    explicit encryption, access, and retention policy.

The supervisor should classify retryability. A provider rate limit or Codex
overload may be retryable within policy. Malformed protocol, invalid cwd,
permission-policy violation, authentication failure, or changed-path breach is
not an automatic model retry.

