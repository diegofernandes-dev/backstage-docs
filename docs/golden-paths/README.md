# Golden Paths / Software Templates / Brownfield Adoption

This folder is the documentation home for the Backstage application-onboarding workstream.

## Scope

The workstream covers two entry paths that should converge on a coherent platform model:

- **Greenfield:** Software Templates / Golden Paths for new applications.
- **Brownfield:** registration, assessment, progressive adoption, modernization, exceptions, and retirement of existing applications.

The goal is a Golden Path, not a Golden Cage: new applications should start close to the supported platform standard, while existing applications can obtain Catalog/governance/platform value without requiring an immediate rewrite.

## Authority model

- Azure DevOps `platform-devops-developer-portal` is authoritative for what is implemented.
- `diegofernandes-dev/backstage-docs@main` is authoritative for reviewed architecture, contracts, decisions, and construction handoffs.
- Shared decisions that affect the overall Backstage platform belong in `docs/adr/`.
- Workstream-specific reasoning and handoffs belong here.

## Expected documents

The architect may refine names, but the workstream should maintain clear equivalents of:

- `current-state.md` — evidence-based assessment of the existing portal/templates/Catalog model;
- `architecture.md` — target Golden Path and brownfield architecture;
- `adoption-model.md` — progressive legacy onboarding/modernization model;
- `implementation-roadmap.md` — small reviewable implementation slices;
- `implementation-progress.md` — implementation handoffs once coding is authorized.

Do not create production templates in this repository. Template/source implementation belongs in the Azure DevOps implementation repository; this repo holds architecture and construction documentation only.

## Working protocol

Before each checkpoint, fetch and compare `origin/main`. Read the root README, `docs/README.md`, relevant shared ADRs, and this workstream's latest documents. Inspect the actual Azure DevOps implementation before stating what exists.

Each implementation handoff must record the ADO branch/SHA inspected or changed, the `backstage-docs` baseline/final SHA, tests/validation, deviations, unresolved questions, and an explicit next gate.
