# Golden Paths / Software Templates / Brownfield Adoption

This folder is the documentation home for the Backstage application-onboarding workstream.

## Scope

The workstream covers two first-class entry paths that converge on a coherent platform model:

- **Greenfield:** Software Templates / Golden Paths for new applications.
- **Brownfield:** registration, assessment, progressive adoption, modernization, exceptions, and retirement of existing applications.

The governing principle is **Golden Path, not Golden Cage**: new applications should start close to the supported platform standard, while existing applications can obtain Catalog/governance/platform value without requiring an immediate rewrite.

## Authority model

- Azure DevOps `platform-devops-developer-portal` is authoritative for what is implemented.
- `diegofernandes-dev/backstage-docs@main` is authoritative for reviewed architecture, contracts, decisions, and construction handoffs.
- Shared decisions that affect the overall Backstage platform belong in `docs/adr/`.
- Workstream-specific reasoning and handoffs belong here.

## Architecture checkpoint — 2026-09-01

The target architecture and brownfield adoption model are now documented, but implementation readiness is intentionally blocked because the Azure DevOps implementation source of truth could not be inspected from this checkpoint.

Current gate: **NO-GO for production implementation until Slice 0 implementation inventory/reconciliation is completed against the actual ADO repository.**

No shared ADR was created in this checkpoint. Proposed platform-wide Catalog semantics remain proposals until current production conventions are verified.

## Workstream documents

- [`current-state.md`](./current-state.md) — evidence-based assessment; implementation-specific Golden Path capabilities are currently marked UNKNOWN pending ADO inspection.
- [`architecture.md`](./architecture.md) — target greenfield + brownfield architecture, template decomposition, central ownership boundaries, Catalog semantics, monorepos, multi-workload, versioning, conformance, exceptions and retirement.
- [`adoption-model.md`](./adoption-model.md) — brownfield registration, assessment, progressive adoption stages, PR-based modernization, exceptions and retirement.
- [`implementation-roadmap.md`](./implementation-roadmap.md) — reviewable slices and gates; Slice 0 is evidence reconciliation and the default first production slice after that gate is registration-only brownfield adoption.
- `implementation-progress.md` — create only when implementation is explicitly authorized and begins.

## Target model

```text
Create New Application ─┐
                        ├─> Catalog + ownership + platform capability contract
Register Existing App ──┘
```

A Software Template is an orchestration/onboarding mechanism, not the lasting compliance oracle. Day-2 conformance must evaluate observable application/platform state against a versioned contract/profile.

## Working protocol

Before each checkpoint:

1. fetch and compare `origin/main`;
2. read the root README, `docs/README.md`, `docs/adr/README.md`, the shared ADRs relevant to the proposal, and the latest workstream documents;
3. inspect the actual Azure DevOps implementation before stating that a template, action, Catalog convention, pipeline integration or permission exists;
4. record code/documentation disagreement as a deviation rather than rewriting architecture to justify drift.

Each implementation handoff must record the ADO branch/SHA inspected or changed, the `backstage-docs` baseline/final SHA, tests/validation, deviations, unresolved questions, and an explicit next gate.

Do not create production templates in this repository. Template/source implementation belongs in Azure DevOps; this repository holds architecture and construction documentation only.
