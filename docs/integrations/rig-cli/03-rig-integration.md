# Rig integration point

[← Integrations index](../README.md) · [Rig CLI index](README.md)

> Preserved from the original Rig CLI adapter study.

## Rig integration point

Rig 0.42's typed `Tool` owns argument decoding, model-visible output, typed
errors, and a call receiving `ToolContext`. Register it with
`AgentBuilder::tool(...)`. Rig's default tool concurrency is one; callers can
opt into parallel calls with `tool_concurrency(n)`.

The model-facing schema should be narrow. Prefer one fixed tool instance per
backend and policy, such as `review_with_codex` or `implement_with_grok`. Do not
let the model provide an executable, arbitrary flags, environment variables,
raw working directory, or a vendor session ID.

```rust
use rig::tool::{Tool, ToolContext};
use serde::{Deserialize, Serialize};
use serde_json::json;

#[derive(Clone)]
struct CodingAgentTool {
    // Fixed executable, backend, workspace policy, permissions, and limits.
    supervisor: AgentSupervisor,
}

#[derive(Deserialize)]
struct WorkerArgs {
    instruction: String,
}

#[derive(Serialize)]
struct WorkerOutput {
    status: WorkerStatus,
    final_text: String,
    diagnostics: Vec<String>,
}

impl Tool for CodingAgentTool {
    const NAME: &'static str = "run_coding_agent";
    type Args = WorkerArgs;
    type Output = WorkerOutput;
    type Error = AgentToolError;

    fn description(&self) -> String {
        "Run the configured coding agent in the controller-owned workspace".into()
    }

    fn parameters(&self) -> serde_json::Value {
        json!({
            "type": "object",
            "properties": {
                "instruction": { "type": "string", "minLength": 1 }
            },
            "required": ["instruction"],
            "additionalProperties": false
        })
    }

    async fn call(
        &self,
        context: &mut ToolContext,
        args: Self::Args,
    ) -> Result<Self::Output, Self::Error> {
        let scope = context.require::<RunScope>()?.clone();
        let run = self.supervisor.run(&scope, args.instruction).await?;
        context.insert_result(run.receipt); // Host-only; not sent to the model.
        Ok(run.output)
    }
}

let agent = rig::agent::AgentBuilder::new(model)
    .tool(CodingAgentTool::new(config))
    .build();
```

This is a structural sketch; project types and constructors are omitted. The
important seam is `Tool::call`. `AgentSupervisor::run` must provide the bounded
I/O, cancellation, parsing, and cleanup described below. `RunScope` should be
injected by the host through `ToolContext`; it can carry the canonical
workspace, controller run ID, session-map key, and a host cancellation token
without exposing those controls to the model.

If `tool_concurrency` is raised above one, serialize calls sharing either a
workspace or external session. Two mutating workers must use separate Git
worktrees. A vendor session must never receive overlapping turns.

