# 2026 source refresh

[← Research index](../README.md) · [Review index](README.md)

> Preserved from the source-refresh review.

## Document context

Reviewed 2026-08-23. This document records what was checked before the design
archive was committed. Claims in the archive remain design inputs unless
explicitly marked as verified here.


## 2026 source refresh

The original conclusion still holds, with these material corrections and
additions:

- Markdown plans are established practice, not the product’s unique insight.
  GitHub Spec Kit and OpenSpec both use repository Markdown artifacts and
  multi-artifact planning workflows.
- Codex now documents native `/goal` and a project-local `Stop` hook. The
  initial integration can use either; it must follow the current Codex hook
  wire format instead of assuming Claude’s format.
- Grok Build now ships and publishes source for a native `/goal` harness. Older
  conclusions that require an outer ACP controller are superseded by the
  source-pinned trace in [Goal-execution research](../goal-execution/04-implementation-traces.md).
- Rig is a lower-level Rust runtime, not a native `/goal` host. Its current
  source supplies serializable run state and hooks, while
  its roadmap leaves durable session/run control and workflow orchestration to
  the host.
