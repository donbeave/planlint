# Minimal proof of concept

[← Integrations index](../README.md) · [Linear index](README.md)

> Preserved from the original Linear integration study.

## Minimal proof of concept

Build the smallest vertical slice before committing to Linear as a product
dependency:

1. Add three Markdown plan items with stable IDs, one parent, one child, and
   one blocking edge.
2. Implement `planlint linear sync --project ...` using the GraphQL SDK/API.
3. Run it twice and prove it creates no duplicates.
4. Verify the Linear issue shows the Markdown projection, parent/sub-issue,
   blocker edge, milestone, and source commit link.
5. Create a test branch and PR containing the Linear ID; prove the PR appears
   on the issue and status automation behaves as configured.
6. Run the verifier; post a receipt comment and set the controlled verified
   signal only on pass.
7. Edit an operational field in Linear and confirm the local manifest/cache
   updates without changing the contract file.
8. Edit the managed block in Linear and confirm the adapter reports drift and
   does not overwrite Git.
9. Test a failed webhook, duplicate delivery, GraphQL partial error, expired
   token, rate limit, missing team/project, dangling dependency, and cycle.

Acceptance for the POC:

- repeatable sync is idempotent;
- the same stable plan ID always resolves to the same Linear issue;
- no Linear UI status can bypass `planlint verify`;
- PR, review, check, and merge state are visible from the issue;
- dependency edges are visible in Linear and validated in Git;
- cloud edits cannot silently rewrite the canonical contract;
- loss of Linear access does not prevent local verification;
- the adapter can export enough metadata to rebuild the projection.

The POC must also record the live schema revision and the exact scopes needed
for each mutation. This is the remaining API-level unknown; it is not a reason
to make Linear the canonical contract store before testing.

If the POC passes, Linear is a good backend for the **operational graph**. If
the requirement changes to “edit contracts in a web UI and reproduce them
exactly as Git files,” Linear should not be the canonical store; build a
dedicated web editor backed by Git or introduce an explicit reviewed import
step.

