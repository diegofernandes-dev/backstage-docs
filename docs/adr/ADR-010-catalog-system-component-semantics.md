# ADR-010: Canonical Catalog System/Component and repository semantics

## Status

Accepted (2026-09-01) — established during Golden Paths Slice 0 reconciliation.

## Context

The Golden Paths architecture proposed that System represents a business/product boundary and Component represents a deployable workload, with repository identity as metadata rather than entity identity. Slice 0 inspected the Azure DevOps implementation (`platform-devops-developer-portal`, branch `feat/ado-repo-governance`) and confirmed these semantics are already in use, but they were not yet recorded as a shared platform decision.

Without a canonical definition, greenfield templates, brownfield registration, corporate catalog promotion, and future conformance checks may diverge on what constitutes an application boundary versus a deployable unit.

## Decision

### System

A **System** represents a business/product/system boundary — not a repository and not a deployment environment.

In the current implementation:

- one System exists per Azure DevOps Team Project;
- Systems live in the corporate catalog repository (`platform-devops-idp-catalog/systems/`);
- Systems carry `metadata.annotations.azure.devops.com/project` with the ADO project slug;
- Components reference a System via `spec.system` using the Team Project slug.

### Component

A **Component** represents a deployable or independently operated software workload.

In the current implementation:

- each service repository typically maps to one Component;
- `spec.type` reflects workload kind (`service`, `website`, etc.);
- `spec.owner` references an Entra-ingested Catalog Group;
- `metadata.annotations.dev.azure.com/project-repo` links the Component to its ADO repository (`{teamProject}/{repository}`).

### Repository

A **repository is not a Catalog entity**. Repository identity is expressed through annotations and relations, not by conflating one repository with one Component kind.

This enables future support for monorepos where one repository contains multiple independently deployable Components.

### Ownership

- `spec.owner` on System and Component entities must reference a Catalog Group resolvable from Entra ID ingestion.
- The `TeamProjectPicker` Scaffolder field extension derives `groupRef` and `systemRef` from ADO Team Project visibility.
- Ownership is consistent across greenfield and brownfield paths.

### Platform metadata (complementary, not replacing Catalog)

Application platform intent is captured separately from Catalog presentation:

- `catalog-info.yaml` — Backstage entity descriptor (owner, system, type, lifecycle, annotations).
- `idp.platform.yaml` — platform manifest (archetype, stack, workloads, pipeline/deploy contracts).

Both are application-owned declarative artifacts. See implementation ADR 0014 in `platform-devops-developer-portal`.

### Monorepo and multi-workload (aspirational)

- One repository may contain multiple Components.
- `idp.platform.yaml` supports `spec.workloads[]` to declare workload intent within a single repository.
- Registration and template flows for multiple Components per repository are **not yet implemented** — this ADR establishes the semantic target only.

## Consequences

### Positive

- Greenfield and brownfield paths converge on the same Catalog vocabulary.
- Corporate catalog promotion (`idp:promote-catalog-pr`) and instance registration (`catalog:register`) produce compatible entities.
- Conformance assessment (`idpAssessor`) can evaluate Components against a stable identity model.

### Negative / follow-up

- Monorepo registration flows must be designed without breaking the one-repo-one-Component default.
- API and Resource entities are demonstrated in showcase examples but not enforced in golden-path templates.
- Domain kind is not used and is out of scope for this ADR.

## Alternatives considered

### One repository equals one Component (always)

Rejected. Cannot represent monorepos or multiple independently deployable workloads.

### System equals repository

Rejected. Team Projects represent organizational/product boundaries; repositories are deployment units within them.

### Repository as a Catalog Resource entity

Rejected for the primary application model. Repositories are better represented as annotations on Components. Infrastructure Resources (databases, caches) remain separate Catalog kinds for dependency modeling.

## References

- Golden Paths Slice 0 evidence: `docs/golden-paths/current-state.md`
- Implementation repo: `platform-devops-developer-portal` — `docs/adrs/0005-rbac-ownership-systems.md`, `docs/adrs/0013-brownfield-adoption-model.md`, `docs/adrs/0014-idp-platform-manifest.md`
- Corporate catalog structure: implementation `docs/catalog/platform-devops-idp-catalog/README.md`
