# Decision

[← Integrations index](../README.md) · [Linear index](README.md)

> Preserved from the original Linear integration study.

## Decision

Adopt Linear experimentally as a **projection and observability backend** for
planlint. Do not replace local Markdown contracts. The first implementation
should be one-way contract projection plus controlled operational-state
feedback, with GitHub’s native integration handling PR visibility and Linear’s
webhooks/API handling synchronization.

This gives the requested UI/UX—hierarchy, dependencies, owners, agent activity,
progress, and PRs—without weakening the core design: acceptance remains
versioned, local, deterministic, and independently verifiable.

