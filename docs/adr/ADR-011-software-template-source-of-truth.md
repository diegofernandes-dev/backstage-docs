# ADR-011: Software Template source of truth and production discovery

## Status

Accepted (2026-09-01) — established during Golden Paths template source-of-truth checkpoint.

## Context

Golden Paths Slice 0 documented a **CURRENT HYBRID** model: Software Templates authored in `platform-devops-developer-portal/templates/` and synced to `platform-devops-idp-catalog/templates/` for production discovery (implementation ADR 0016, WIP).

That model decouples **runtime discovery** from portal deployment but leaves three problems unresolved:

1. **Authoring ownership** — product teams maintaining Golden Paths still require write access to the Backstage application repository.
2. **Duplication risk** — two copies of the same template artifacts with a sync step that had not been executed at checkpoint time.
3. **Responsibility conflation** — the corporate catalog repository (`platform-devops-idp-catalog`) mixes organizational entity inventory (Systems, Components) with Software Template distribution.

Slice 0 also established that Scaffolder custom actions are privileged trust-boundary code (repository creation, pipeline registration, governance, catalog PRs). Software Templates orchestrate those actions but are not equivalent to action implementation. They require review at a different boundary.

The platform already splits central pipeline implementation into `platform-pipeline-templates` with semver git tags (implementation ADR 0015). Software Templates should follow the same separation principle.

## Decision

Adopt **HYBRID WITH SPLIT AUTHORING SOURCE**.

### Separate concerns explicitly

| Concern | Mechanism |
|---|---|
| **Authoring source** | `platform-software-templates` (new Azure DevOps repository) |
| **Publishing / distribution** | Semver git tags on `platform-software-templates` |
| **Runtime discovery** | Backstage `catalog.locations` URL pointing to `templates/catalog-info.yaml` in the template repo, tag-pinned in production |

### Authoritative repository boundaries

| Asset | Authoritative repository |
|---|---|
| Backstage frontend/backend | `platform-devops-developer-portal` |
| Scaffolder custom actions | `platform-devops-developer-portal` |
| Custom field extensions (e.g. `TeamProjectPicker`) | `platform-devops-developer-portal` |
| Software Template YAML + skeletons | `platform-software-templates` |
| Template production distribution | `platform-software-templates` (direct URL discovery; no catalog mirror) |
| Central pipeline implementation | `platform-pipeline-templates` |
| Corporate Catalog entities (Systems, Components) | `platform-devops-idp-catalog` |
| Application platform manifest schema | Shared spec (implementation ADR 0014); generated copies in application repos |

### Corporate catalog is not a template registry

`platform-devops-idp-catalog` remains responsible for **organizational entity inventory only**. Software Templates must not be synced into that repository as a permanent source of truth.

Production Backstage discovers templates via a direct, tag-pinned URL to `platform-software-templates`, using the same Azure DevOps URL location pattern already used for corporate catalog ingestion.

### Compatibility contract

Templates depend on stable platform contracts, not portal internal paths:

- Scaffolder action IDs (`idp:platform-context`, `idp:register-existing-catalog-pr`, etc.)
- Custom UI field extensions (`TeamProjectPicker`)
- Runtime configuration resolved by actions (e.g. pipeline contract tags via `idpProvisioner` app-config)
- Relative `fetch:template` content paths within each template directory

Breaking changes to action inputs require coordinated portal release and template bundle update. A minimal compatibility note (template bundle tag → minimum portal version) is sufficient; a formal matrix is required only when breaking action changes occur.

### Versioning dimensions

| Dimension | Mechanism |
|---|---|
| Template product version | Semver git tags on `platform-software-templates` (e.g. `templates-v1.0.0`) |
| Scaffolder action contract | Portal release; document action catalog and schemas |
| Application/platform contract | `idp.platform.yaml` `apiVersion: idp.company/v1alpha1` |
| Pipeline contract | Git tags on `platform-pipeline-templates` (`dotnet-ci-1.0.0`, etc.) |

### Security model

```text
Template YAML (reviewed orchestration)
  → registered Scaffolder actions only (portal trust boundary, RBAC-gated)
    → service integrations (ADO, catalog PRs)
```

Splitting templates from portal action code **reduces** the risk of casual privileged-code edits alongside template YAML. Template authors cannot introduce new privileged behavior without a new action registered in the portal.

Template YAML is not harmless — it orchestrates repository creation and governance — and requires security review independent of portal code changes.

### Publishing and discovery

| Environment | Discovery mechanism |
|---|---|
| Local development | URL location to template repo branch, or local clone override |
| Production | URL location to `templates/catalog-info.yaml` with semver tag pin |
| Promotion | PR review + git tag on `platform-software-templates` |
| Rollback | Re-pin production location to prior tag |

Do not use portal → corporate catalog sync as the permanent publishing mechanism.

## Consequences

### Positive

- Golden Path maintainers can contribute without write access to the Backstage application repository.
- Eliminates duplicate template copies and sync drift between portal and corporate catalog.
- Aligns Software Templates with the existing `platform-pipeline-templates` separation pattern.
- Corporate catalog scope remains focused on entity inventory.
- Production template releases are independent of portal deployment.

### Negative / follow-up

- **Slice T0** must create `platform-software-templates`, move `templates/` from the portal repo, update discovery configuration, and supersede implementation ADR 0016 before first production template publishing.
- Dev ergonomics require either URL-based discovery or a local clone of the template repo during portal development.
- Implementation ADR 0017 in `platform-devops-developer-portal` should document the local supersession of ADR 0016 when T0 is executed.

### Impact on Slice 1

- **GO** for brownfield hardening (portal actions, assessor, Platform tab) — independent of template SoT.
- **NO-GO** for production template publishing (catalog sync) until Slice T0 completes.

## Alternatives considered

### KEEP TOGETHER (templates only in portal repo)

Rejected. Couples template changes to portal deployment and prevents independent Golden Path ownership.

### CURRENT HYBRID (portal authoring + catalog sync for discovery)

Rejected as permanent architecture. Solves runtime discovery but not authoring ownership; introduces unnecessary duplication. The catalog sync path had not been executed and would cement the wrong boundary if treated as permanent.

### FULL SPLIT (templates and actions in separate repos)

Rejected. Scaffolder custom actions and field extensions must remain in the portal repository as privileged runtime/trust-boundary code tightly coupled to Backstage releases.

## References

- Golden Paths checkpoint evidence: `docs/golden-paths/current-state.md`
- Target architecture: `docs/golden-paths/architecture.md` section 18
- Implementation roadmap: `docs/golden-paths/implementation-roadmap.md` (Slice T0)
- [ADR-010](./ADR-010-catalog-system-component-semantics.md) — Catalog semantics
- Implementation ADR 0016 (superseded authoring/distribution portions) — `platform-devops-developer-portal`
- Implementation ADR 0015 — central pipeline consumer contract
