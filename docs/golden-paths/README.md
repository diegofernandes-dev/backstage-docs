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

## Architecture checkpoint — template SoT decision complete (2026-09-01)

Slice 0 inspected `platform-devops-developer-portal` on branch `feat/ado-repo-governance` at baseline `6e28611`, plus ~39 uncommitted WIP files containing brownfield templates, assessor, catalog validation, and golden-path alignment.

Template source-of-truth checkpoint inspected ADO at `be16ffb` (same golden-path WIP delta). Documentation baseline: `backstage-docs@3a6dc50`.

Key findings:

- Greenfield Scaffolder foundation exists at HEAD (4 .NET templates, 3 `idp:*` actions).
- Golden Path and brownfield capabilities are substantially implemented in uncommitted WIP.
- Template repository boundary: **HYBRID WITH SPLIT AUTHORING SOURCE** — [ADR-011](../adr/ADR-011-software-template-source-of-truth.md).
- Catalog System/Component semantics confirmed — see [ADR-010](../adr/ADR-010-catalog-system-component-semantics.md).

**Current gates:**

| Track | Gate |
|---|---|
| Brownfield hardening (Slice 1B) | **GO** |
| Production template publishing | **NO-GO** until Slice T0 |

Do not sync templates to `platform-devops-idp-catalog/templates/`.

## Workstream documents

- [`current-state.md`](./current-state.md) — evidence-based assessment with ADO implementation evidence.
- [`architecture.md`](./architecture.md) — target greenfield + brownfield architecture, reconciled with ADO evidence.
- [`adoption-model.md`](./adoption-model.md) — brownfield registration, assessment, progressive adoption stages, PR-based modernization.
- [`implementation-roadmap.md`](./implementation-roadmap.md) — reviewable slices and gates; Slice T0 + Slice 1B are next.
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
