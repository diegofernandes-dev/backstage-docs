# Backstage platform documentation

This directory is the active architecture and construction documentation set for the Backstage platform.

## Authority

| Question | Source of truth |
|---|---|
| What is implemented? | Azure DevOps `platform-devops-developer-portal` source code |
| What should be implemented? | `diegofernandes-dev/backstage-docs@main` ADRs and normative contracts |
| What was reviewed/built at each checkpoint? | Workstream construction/handoff documents in this repository |
| Historical Teams/ADO POC evidence | Legacy `diegofernandes-dev/poc-teams-approval` archive only |

A disagreement between code and documentation is a **deviation**. Do not silently change architecture documents to match implementation.

## Shared architecture

[`adr/`](./adr/) contains decisions that affect the Backstage platform across workstreams. Always fetch `main` before assigning the next ADR number; do not assume an ADR number from a stale checkout.

Current accepted GMUD authorization model: [ADR-009](./adr/ADR-009-change-authorization-model.md).

Current proposed Delivery/GitOps direction: [ADR-012](./adr/ADR-012-delivery-management-gitops-promotion.md) — **PROPOSED / ARCHITECTURE SPIKE REQUIRED; no Delivery implementation is authorized by that ADR.**

## Workstreams

### GMUD / Change Management

- [Current state](./backstage/current-state.md)
- [Construction progress](./backstage/implementation-progress.md)
- [F3 authorization review](./architect-review-f3-change-authorization.md)
- [Create UI contract](./ui/gmud-create-screen.md)
- [My Changes UI contract](./ui/gmud-my-changes-screen.md)
- [Detail UI contract](./ui/gmud-detail-screen.md)

### Delivery Management / GitOps promotion

Use [`delivery/`](./delivery/) for the proposed software-delivery workstream that separates release/promotion/deployment concerns from GMUD governance.

The current direction is intentionally a **spike, not an implementation commitment**:

- Change Management authorizes a business change; it does not own deployment orchestration.
- Delivery manages release candidates, deployment targets, promotion/deployment requests, and their operational projection.
- Argo CD is the preferred Kubernetes reconciliation candidate.
- Kargo is the first promotion-controller candidate to evaluate before building a custom controller.
- Application branching is upstream release provenance/policy, not an environment-selection mechanism.
- New application repositories should be evaluated against protected trunk-based development with short-lived PR branches and build-once/promote-many artifacts.

See:

- [Delivery workstream current direction](./delivery/README.md)
- [Architecture spike plan](./delivery/architecture-spike.md)
- [ADR-012](./adr/ADR-012-delivery-management-gitops-promotion.md)

### Golden Paths / Software Templates / Brownfield adoption

Use [`golden-paths/`](./golden-paths/) for architecture, current-state assessment, application-template strategy, legacy/brownfield adoption, implementation roadmap, and future construction handoffs. Shared decisions that constrain the wider Backstage platform belong in `adr/` rather than being hidden in a workstream document.

- Slice 0 (2026-09-01): implementation inventory complete.
- Template source-of-truth checkpoint (2026-09-01): [ADR-011](./adr/ADR-011-software-template-source-of-truth.md) — **HYBRID WITH SPLIT AUTHORING SOURCE** (`platform-software-templates`).
- Shared Catalog semantics: [ADR-010](./adr/ADR-010-catalog-system-component-semantics.md).
- **Gates:** brownfield hardening **GO** (Slice 1B); production template publishing **NO-GO** until Slice T0.

## Documentation protocol

Before architecture or implementation work:

1. fetch `origin/main` from `backstage-docs`;
2. compare local HEAD with `origin/main`;
3. read the relevant workstream current state and shared ADRs;
4. inspect ADO source before claiming implementation state;
5. after a reviewed implementation checkpoint, update the workstream construction log with both ADO SHA and docs SHA.

Do not copy application source, generated application code, secrets, pipelines, or operational repository trees into this documentation repository.

## Migration

The active Backstage bridge migrated from `diegofernandes-dev/poc-teams-approval` on 2026-09-01, using legacy commit `fe4f8073f2a8785673e32ce51e5f70b7c322ad68` as the final import baseline. See [migration record](./migration-2026-09-01.md).
