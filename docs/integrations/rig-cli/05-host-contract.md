# Normalized host contract

[← Integrations index](../README.md) · [Rig CLI index](README.md)

> Preserved from the original Rig CLI adapter study.

## Normalized host contract

Keep vendor-specific events behind a small controller interface:

```text
preflight(BackendSpec) -> CapabilitySnapshot
start_or_resume(RunScope) -> AgentSession
prompt(session, instruction) -> EventStream + TurnResult
cancel(session) -> Result
close(session) -> Result
```

Suggested terminal model:

```text
WorkerStatus =
    Completed       # vendor turn completed; acceptance not implied
  | GoalBlocked     # native/host goal stopped with a resumable blocker
  | AwaitingHuman   # permission, question, authentication, or policy decision
  | Failed          # vendor reported a terminal turn failure
  | PermissionDenied
  | TimedOut
  | Cancelled
  | ProtocolError   # malformed/missing/contradictory terminal frame
  | ProcessError    # spawn, signal, or non-protocol exit failure
```

Suggested normalized result:

```text
TurnResult {
  backend
  backend_version
  status
  final_text
  session_handle       # controller alias; raw vendor ID stays host-side
  usage                # optional and vendor-qualified
  stop_reason          # optional raw vendor value
  diagnostics
  raw_event_artifact   # sensitive, retention-controlled reference
}
```

Do not erase absence into zero. Missing cost means unknown, not free. Missing
usage means unreported, not zero tokens. Preserve unknown event types so a new
CLI version does not crash a forward-compatible parser, while still requiring
the known terminal contract for the pinned major/version.

