# Human takeover

[← Integrations index](../README.md) · [Rig CLI index](README.md)

> Preserved from the original Rig CLI adapter study.

## Human takeover when a goal blocks

### Required ownership model

Treat handoff as a transfer of write ownership, not as a second client casually
joining a live worker. One controller-issued lease must cover both the vendor
session and its bound worktree:

```text
Automated
  -> Blocked                 # durable goal state and reason captured
  -> HandoffReady            # turn is idle; automation writer lease released
  -> HumanAttached           # human owns prompts, approvals, and repository writes
  -> ResumePending           # human explicitly returns control
  -> Automated               # lease reacquired; state and diff revalidated
  -> Verify
  -> Complete | Blocked
```

The controller may remain connected as a read-only event subscriber while the
human owns the lease. It must not send a prompt, answer a permission request,
resume the goal, or start deterministic cleanup concurrently. “Multiple clients
supported” does not mean “multiple writers to one session are safe.”

A handoff record should contain only controller-owned references:

```text
Handoff {
  handoff_id
  backend
  session_handle           # opaque alias, not a model-selected raw ID
  canonical_workspace
  worktree_id
  config_profile
  blocked_reason
  last_terminal_event
  repository_head
  diff_fingerprint
  lease_generation
  attach_kind              # local command, authenticated URL, or protocol endpoint
  expires_at
}
```

Keep raw session IDs, bearer tokens, sockets, and authenticated URLs out of the
Rig model transcript. Resolve `handoff_id` to attach material only in an
authenticated operator API or CLI. An attach token should be short-lived and
scoped to one handoff when the backend permits it.

### Blocked-to-handoff sequence

1. Detect a native goal-blocked event or a host verifier's `BLOCKED` result.
   Keep goal blockage separate from authentication failure, malformed protocol,
   timeout, crash, and deterministic verification failure.
2. Stop automatic continuation. If a prompt is still running, request native
   cancellation and wait for its terminal acknowledgement; never expose the
   worktree while the worker can still mutate it.
3. Persist the exact session ID, cwd, worktree, configuration root, backend
   version, goal snapshot, Git `HEAD`, full changed-file set, and bounded event
   cursor.
4. Atomically transition the lease from `Automation` to `Human`. Only then
   return `GoalBlocked { handoff_id, reason }` from the adapter and stop the
   outer Rig goal controller.
5. Let the operator attach through the backend surface below. The human may
   answer the blocker, inspect state, change files, and resume the native goal.
6. Require an explicit “return to automation” action. A terminal disconnect is
   not proof that the human finished.
7. Reacquire the lease, confirm that the session still maps to the same cwd and
   worktree, recompute `HEAD` and the complete diff, refresh the event cursor,
   and rerun deterministic verification. Resume automation only after these
   checks pass.

If the process crashes before its session is durably recorded, or the backend
cannot reload the session with the same tools and roots, exact handoff is not
available. Create a new session from a sanitized transcript and repository
checkpoint and label that as a fork, not a resume.

### Backend attach surfaces

| Backend | Recommended human handoff | Important boundary |
| --- | --- | --- |
| Claude Code | After a print/SDK turn settles, run `claude --resume <session-id>` with the same cwd and `CLAUDE_CONFIG_DIR`. A session launched under Claude's background supervisor can instead use `claude attach <job-id>`; Remote Control is another authenticated interactive surface. | Background job IDs and conversation session IDs are different handle types. Live attach requires the background/Remote Control architecture from session start; it cannot be retrofitted onto an arbitrary `claude -p` child. |
| Codex | Prefer `codex app-server --listen ws://127.0.0.1:<port>` and let the Rig adapter use JSON-RPC. On handoff, stop sending turns and run `codex --remote <ws-url> resume <thread-id>`. Process-per-turn fallback is `codex resume <thread-id>`. | Use a loopback or authenticated socket and one prompt/approval writer. Generate protocol types from the pinned binary. |
| Grok Build | After the blocked goal reaches its persisted paused state, run `grok --cwd <workspace> --resume <session-id>`. An ACP client can reconnect with `session/load` or `session/resume` when advertised; leader/dashboard mode can expose resident sessions when both clients use the same leader. | Scripts must use the ID, not “continue latest.” Preserve `GROK_HOME`; a restored active goal is intentionally paused rather than silently continuing. |
| OpenCode | Keep `opencode serve` on loopback with `OPENCODE_SERVER_PASSWORD`; attach with `opencode attach <url> --dir <workspace> --session <session-id>`. The server is explicitly multi-client and exposes session status, abort, permission, prompt, and event APIs. | Multi-client transport still needs the controller's one-writer lease. OpenCode has no native `/goal`; the host decides `Blocked`. |
| Kimi Code | Either run `kimi --session <session-id>` after the headless process exits, or use `kimi web --no-open` and open its authenticated browser UI for the same session. | Preserve `KIMI_CODE_HOME`. The web URL contains a bearer credential; never place it in model output or logs. |

ACP stdio is not itself a terminal-sharing mechanism. It is one client/agent
connection. Stable ACP now has capability-negotiated `session/resume`, while
older clients commonly use `session/load`; call only what the agent advertised.
For a stdio handoff, quiesce and disconnect the automation client, then let the
human-facing client resume or load the session. Re-send the exact cwd, MCP
servers, additional roots, and policy inputs required by that backend.

### Detecting `/goal` blockage

There is no portable `/goal` wire event. Each adapter must normalize its own
source, and only an explicit native or controller state transition should
produce `GoalBlocked`:

| Backend | Reliable blocked signal |
| --- | --- |
| Claude Code | Claude's session goal is a continuation condition, not the portable status model used here. Define blockage in the host controller from an explicit user/permission need, a no-progress rule, or a verifier result; do not scrape prose for the word “blocked.” |
| Codex | App-server notification `thread/goal/updated` whose `goal.status` is `blocked`; `thread/goal/get` provides the durable snapshot. These goal methods are non-experimental in the inspected protocol even though app-server itself remains experimental CLI surface. |
| Grok Build | Vendor ACP session update `goal_updated` with `status: "blocked"` and its optional `pause_message`. Wait for the associated terminal turn update before releasing the lease. |
| OpenCode | No native goal state. The host-defined goal controller emits `Blocked` after its own repeated-blocker or verifier policy. Normal `session.error` remains worker failure. |
| Kimi Code | Headless `kimi -p "/goal ..." --output-format stream-json` emits a final `type: "goal.summary"` with `status: "blocked"` and exits `3`. Interactive/web goal state can later be resumed with `/goal resume`. |

For Codex specifically, the app-server path is the cleanest match for this
requirement: Rig invokes a typed tool, the adapter owns one durable thread,
`thread/goal/updated` provides the blocked transition, and the native TUI can
resume that same thread through the app-server socket. Rig should not remain
inside a pending `Tool::call` during human work. The tool returns the handoff
handle, an `AgentHook` stops the current run before the Rig model can issue
another worker call, an outer controller persists the paused run, and a new Rig
run starts after the operator returns control. This avoids depending on
high-level Rig runner resume, which is not available in Rig 0.42.

