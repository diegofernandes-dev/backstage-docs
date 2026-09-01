# Golden Paths — current state

> **Workstream:** Software Templates / Golden Paths / Brownfield Adoption  
> **Architecture repository:** `diegofernandes-dev/backstage-docs@main`  
> **Implementation source of truth:** Azure DevOps `platform-devops-developer-portal`  
> **Assessment date:** 2026-09-01  
> **Slice 0 status:** **COMPLETE**  
> **Documentation checkpoint:** `backstage-docs@6706ad4` (published 2026-09-01)

## 1. Purpose

This document records evidence verified during Slice 0 (implementation inventory and architecture reconciliation). It separates:

1. architecture/documentation facts in `backstage-docs`; and
2. implementation facts verified in the Azure DevOps repository.

A design described in this repository is not evidence that the capability exists in Backstage.

## 2. Authority model

- Azure DevOps `platform-devops-developer-portal` answers **what is implemented**.
- `backstage-docs@main` answers **what should be implemented**, why, and what construction state was reviewed.
- The former `poc-teams-approval` bridge is historical evidence only.
- Cross-workstream platform decisions belong under `docs/adr/`.
- Golden Path-specific architecture and construction handoffs belong under `docs/golden-paths/`.

## 3. Implementation evidence inspected

### 3.1 Repository snapshot

| Field | Value |
|---|---|
| Repository | `platform-devops-developer-portal` (Azure DevOps) |
| Branch inspected | `feat/ado-repo-governance` |
| Slice 0 inspection baseline SHA | `6e28611e90455dfd56f583bdd132ee830d88f126` |
| Branch HEAD at docs publish | `be16ffb02b59d791f0086d8e5086e4428c82b90d` (GMUD F3.1 ledger; golden-path WIP unchanged) |
| `main` branch SHA (reference) | `86b2e03f1201decb23b15f13e24f87884e5e1bab` |
| Working-tree delta | ~39 uncommitted files on `feat/ado-repo-governance` (unchanged since Slice 0) |
| Backstage version | `1.51.0` (`backstage.json`) |
| Node | `22 \|\| 24` (`package.json` engines) |
| Package manager | Yarn `4.4.1` |

### 3.2 Two-layer inspection model

Slice 0 inspected **both** the committed baseline and the uncommitted working tree. The working tree contains the bulk of Golden Path and brownfield capabilities designed after the last commit.

| Layer | What exists |
|---|---|
| **Committed HEAD** | 4 greenfield .NET templates (`dotnet-minimal-api`, `dotnet-worker-service`, `dotnet-grpc-service`, `dotnet-cronjob`); 3 custom Scaffolder actions (`idp:catalog-register-pr`, `idp:ado-pipeline-create`, `idp:ado-repo-governance`); Scaffolder with Azure/GitHub/notifications; `TeamProjectPicker`; RBAC |
| **Working-tree delta** | Brownfield templates (`register-existing-application`, `modernize-application`); `angular-spa`; `idpAssessor` plugin + Platform tab; `catalogValidation` processor; 6 additional `idp:*` actions; `platform-pipeline-templates/`; golden-path alignment for `dotnet-minimal-api`; implementation ADRs 0013–0016; `templates/catalog-info.yaml` production bundle |

**Important:** capabilities marked as WIP below are present in the working tree but not yet committed or merged to `main`. They must not be treated as production-ready until reviewed, tested, merged, and (for templates) synced to the corporate catalog.

### 3.3 Paths inspected

| Area | Paths |
|---|---|
| Backend entry | `packages/backend/src/index.ts` |
| Scaffolder modules | `packages/backend/src/modules/idpProvisioner/`, `packages/backend/src/modules/dotnetNaming/` |
| Brownfield assessment | `packages/backend/src/modules/idpAssessor/`, `packages/backend/src/plugins/idpAssessorPlugin.ts` |
| Catalog validation | `packages/backend/src/modules/catalogValidation/` |
| Frontend | `packages/app/src/modules/scaffolder/`, `packages/app/src/modules/catalogEntityTabs/PlatformTab.tsx` |
| Templates | `templates/*/template.yaml`, `templates/catalog-info.yaml` |
| Pipeline contracts | `platform-pipeline-templates/` |
| Configuration | `app-config.yaml`, `app-config.production.yaml` |
| RBAC | `packages/backend/config/rbac/rbac-policy.csv`, `packages/backend/src/modules/templateExecutorRoleSeed.ts` |
| Scorecard rules | `config/scorecards/platform-adoption.yaml` |
| Corporate catalog docs | `docs/catalog/platform-devops-idp-catalog/README.md` |
| Implementation ADRs | `docs/adrs/0013-brownfield-adoption-model.md` through `0016-production-template-publishing.md` (WIP, uncommitted) |

## 4. Backstage baseline

### Major plugins and modules

| Plugin / module | Purpose |
|---|---|
| `@backstage/plugin-scaffolder-backend` + Azure/GitHub/notifications modules | Template execution |
| `idpProvisioner` backend module | Custom `idp:*` Scaffolder actions |
| `dotnetNaming` backend module | `idp:dotnet-project-name` action |
| `@backstage/plugin-catalog-backend` + MS Graph provider | Catalog ingestion |
| `catalogValidation` module (WIP) | Pre-process warnings on Components |
| `idpAssessor` plugin (WIP) | Brownfield repository assessment API |
| `adoProjectAccess` plugin | Team Project picker data |
| Community RBAC + `templateExecutorRoleSeed` | Scaffolder permission gating |
| Microsoft Entra auth + ownership resolver | Authentication and ownership |
| TechDocs (local dev; S3 in production per platform snapshot) | Documentation |
| GMUD `changeManagement` plugin | Change management (separate workstream) |

## 5. Scaffolder state

### Configuration

- Dev: file-based template locations in `app-config.yaml` (`catalog.locations` → `templates/*/template.yaml`).
- Production: URL location to `platform-devops-idp-catalog/templates/catalog-info.yaml` (ADR 0016, WIP config in `app-config.production.yaml`).
- Provisioner config: `idpProvisioner` block (org, pipeline tags, governance mode).
- Corporate catalog PR target: `idpCatalog` block.

### Custom Scaffolder actions

| Action ID | Layer | Purpose |
|---|---|---|
| `idp:catalog-register-pr` | HEAD | PR to corporate catalog to register a new Component |
| `idp:ado-pipeline-create` | HEAD | Create or reuse ADO YAML pipeline for a service repo |
| `idp:ado-repo-governance` | HEAD | Apply enterprise branch policies and build validation |
| `idp:register-existing-catalog-pr` | WIP | PR in existing repo adding `catalog-info.yaml` + `idp.platform.yaml` |
| `idp:adopt-platform-ci-pr` | WIP | PR replacing inline pipeline with thin central consumer |
| `idp:adopt-techdocs-pr` | WIP | PR adding `mkdocs.yml` and `docs/` skeleton |
| `idp:enrich-catalog-pr` | WIP | PR enriching catalog metadata and adoption stage |
| `idp:promote-catalog-pr` | WIP | PR to corporate catalog for org inventory promotion |
| `idp:platform-context` | WIP | Resolve org and pipeline contract versions from config |
| `idp:dotnet-project-name` | HEAD | Derive PascalCase .NET project name from kebab-case repo |

Built-in actions in use: `fetch:template`, `publish:azure`, `catalog:register`.

### Custom field extensions

- `TeamProjectPicker` (`packages/app/src/modules/scaffolder/TeamProjectPicker.tsx`) — fetches ADO Team Projects visible to the signed-in user; returns `{ name, groupRef, systemRef }`.

### Permission model

- `permission.enabled: true`; Scaffolder in RBAC `pluginsWithPermission`.
- `platform_admin` role: full Scaffolder access (`scaffolder.template.execute`, `scaffolder.action.execute`, etc.).
- `contributor` role: explicitly denied Scaffolder permissions.
- `template_executor` role: seeded at startup with Scaffolder + catalog create permissions; editable in `/rbac` UI.
- Sidebar Scaffolder nav gated on `scaffolder.template.execute`.
- ADO mutations run via service account/PAT in Scaffolder actions, not user ADO permissions.

### Effective Scaffolder architecture

```text
Developer (RBAC-gated)
  -> Scaffolder UI (TeamProjectPicker)
  -> Template YAML (dev: file locations / prod: corporate catalog URL)
  -> Built-in actions (fetch:template, publish:azure, catalog:register)
  -> Custom idp:* actions (repo governance, pipeline, catalog PRs)
  -> ADO (repo create, pipeline, branch policies, PRs)
  -> Catalog (instance register and/or corporate catalog PR)
```

## 6. Software Templates state

Seven templates exist in the working tree; four are committed at HEAD.

| Template | Layer | Stack / type | Maturity |
|---|---|---|---|
| `dotnet-minimal-api` | HEAD + WIP alignment | .NET 10 Minimal API / `service` | WIP: near production-ready (central CI, `idp.platform.yaml`) |
| `angular-spa` | WIP | Angular 19 SPA / `website` | WIP: near production-ready |
| `dotnet-worker-service` | HEAD | .NET 10 Worker / `service` | POC — inline CI, hardcoded org, no platform manifest |
| `dotnet-grpc-service` | HEAD | .NET 10 gRPC / `service` | POC — same gaps as worker |
| `dotnet-cronjob` | HEAD | .NET 10 console / `service` | POC — same gaps as worker |
| `register-existing-application` | WIP | Any stack / brownfield | WIP: registration-only PR flow |
| `modernize-application` | WIP | Existing repos / brownfield | WIP: opt-in PR modernization (promote URL placeholder gap) |

### Greenfield flow (golden-path templates)

`idp:platform-context` → `idp:dotnet-project-name` (dotnet only) → `fetch:template` → `publish:azure` → `idp:ado-pipeline-create` → `idp:ado-repo-governance` → `catalog:register`.

### Brownfield flow

`register-existing-application`: `idp:register-existing-catalog-pr` only (no repo creation, no pipeline mutation).

`modernize-application`: conditional `idp:adopt-platform-ci-pr`, `idp:adopt-techdocs-pr`, `idp:enrich-catalog-pr`, `idp:promote-catalog-pr`.

### Template decomposition observations

- No universal mega-template; seven discrete product templates share reusable `idp:*` actions.
- Significant duplication across three legacy .NET POC templates (inline CI, hardcoded `diegolab`, no `idp.platform.yaml`).
- Reusable platform actions are correctly centralized in `idpProvisioner`; product questions remain in template YAML.

### Production template publishing

ADR 0016 (WIP) defines a HYBRID model: templates authored in `platform-devops-developer-portal/templates/`, synced to `platform-devops-idp-catalog/templates/` for production discovery. **Sync has not been executed** — the corporate catalog repo has no `templates/` folder yet.

## 7. Catalog state

### Entity semantics (verified)

| Kind | Implementation | Architecture alignment |
|---|---|---|
| **System** | One per ADO Team Project in `platform-devops-idp-catalog`; `metadata.annotations.azure.devops.com/project` | **MATCH** — System = product/TP boundary |
| **Component** | Primary application entity; `spec.type: service/website`; `dev.azure.com/project-repo` annotation | **MATCH** — deployable/operable workload |
| **Group** | Entra ID ingestion (allowlisted security groups) + `spec.owner` on entities | **MATCH** |
| **API / Resource** | Showcase examples only (`examples/showcase/`) | **PARTIAL** — not enforced in golden paths |
| **Domain** | Not used | N/A |

See shared [ADR-010](../adr/ADR-010-catalog-system-component-semantics.md) for canonical platform-wide semantics.

### Ownership

- Components and Systems use `spec.owner` referencing Entra-ingested Groups.
- `TeamProjectPicker` derives `groupRef` and `systemRef` from ADO Team Project visibility.
- `catalogValidation` processor (WIP) warns on missing `spec.owner` and `spec.system`.

### Monorepo and multi-workload

- `idp.platform.yaml` supports `spec.workloads[]` (WIP schema).
- No template or registration flow yet supports multiple Components per repository.
- **DEVIATION** — schema exists; flows do not.

### Catalog locations

- Dev: file locations for templates, examples, RBAC seed; URL to corporate catalog root.
- Prod: URL to corporate catalog root + URL to templates bundle (ADR 0016).

## 8. Pipeline relationship

### Central pipeline model (ADR 0015, WIP)

```text
application repo / azure-pipelines.yml  (thin consumer, ~15 lines)
  -> platform-devops/platform-pipeline-templates@refs/tags/{dotnet-ci-1.0.0|angular-ci-1.0.0}
```

| Item | Value |
|---|---|
| Central repo (ADO) | `platform-devops/platform-pipeline-templates` |
| Local dev mirror | `platform-pipeline-templates/` in portal repo (WIP) |
| Tags | `dotnet-ci-1.0.0`, `angular-ci-1.0.0` |
| Config | `idpProvisioner.pipelineTemplates` in app-config |
| Brownfield generator | `pipelineConsumerYaml.ts` (WIP) |
| Enforcement | `blockPipelineFileChanges: true` in governance config |

### Implementation status

- **MATCH (WIP):** `dotnet-minimal-api` and `angular-spa` use thin consumer pattern.
- **DEVIATION (HEAD):** `dotnet-worker-service`, `dotnet-grpc-service`, `dotnet-cronjob` still copy inline pipeline YAML.

## 9. Brownfield state

### Implemented (WIP, uncommitted)

| Capability | Evidence |
|---|---|
| Registration-only template | `templates/register-existing-application/template.yaml` |
| PR-based catalog + platform manifest | `idp:register-existing-catalog-pr` action |
| Progressive modernization template | `templates/modernize-application/template.yaml` |
| Repository assessment | `idpAssessor` plugin + `assessRepository.ts` |
| Platform tab UI | `PlatformTab.tsx` on `kind:component` |
| Adoption stages | ADR 0013 — `discovered` → `cataloged` → ... → `golden-path` |
| Catalog validation warnings | `catalogValidationProcessor.ts` |
| Scorecard rule definitions | `config/scorecards/platform-adoption.yaml` |

### Brownfield flow (designed)

```text
register-existing-application
  -> PR: catalog-info.yaml + idp.platform.yaml (stage: cataloged)
  -> merge + catalog import
  -> Platform tab (idp-assessor assessment)
  -> modernize-application (opt-in PRs per capability)
```

**Register ≠ Migrate** is implemented by design: registration does not alter pipelines or application code.

## 10. Progressive conformance evidence

Deterministic evidence derivable today (WIP assessor + validation):

- `catalog-info.yaml` presence with `spec.owner` and `spec.system`
- `dev.azure.com/project-repo` annotation
- `idp.company/platform-adoption-stage` annotation
- `idp.platform.yaml` with `contracts.pipeline`
- Central CI detection in `azure-pipelines.yml` (`usesCentralPipelineTemplate()`)
- TechDocs structure (`mkdocs.yml`, `docs/`)
- Dockerfile, OpenAPI, Helm file presence (assessor facts)

Not yet implemented: scheduled re-evaluation, exception persistence, fleet-wide scorecard engine, Tech Insights integration.

## 11. Template repository boundary

**Recommendation: HYBRID** (see `architecture.md` section 18 and implementation ADR 0016).

| Asset | Owner repository |
|---|---|
| Backstage portal + Scaffolder actions | `platform-devops-developer-portal` |
| Template YAML (authoring) | `platform-devops-developer-portal/templates/` |
| Template YAML (production discovery) | `platform-devops-idp-catalog/templates/` (sync required) |
| Pipeline implementation | `platform-pipeline-templates` (separate ADO repo) |
| Corporate catalog entities | `platform-devops-idp-catalog` |

## 12. Architecture reconciliation summary

| Concern | Result |
|---|---|
| Catalog model (System/Component) | **MATCH** |
| Templates as orchestration, not durable contract | **MATCH** (WIP) |
| Pipeline thin consumer binding | **DEVIATION** (partial — 2 of 5 greenfield templates aligned) |
| Ownership (Entra Groups) | **MATCH** |
| Monorepo support | **DEVIATION** (schema only) |
| Multi-workload | **PARTIAL** |
| Brownfield register → assess → adopt | **DEVIATION** (implemented in WIP, unlanded) |
| Template repository boundary | **MATCH** (HYBRID per ADR 0016 WIP) |
| Day-2 conformance | **PARTIAL** (assessor foundation, no engine) |
| Production template publishing | **DEVIATION** (config ready, sync not executed) |
| Greenfield portfolio | **MATCH** (with maturity gaps on 3 .NET POCs) |

## 13. Deviations requiring follow-up

1. Brownfield and golden-path capabilities exist only in uncommitted working tree on `feat/ado-repo-governance`.
2. Production template sync to `platform-devops-idp-catalog/templates/` not executed.
3. Three .NET POC templates not aligned to central CI / `idp.platform.yaml` pattern.
4. `modernize-application` promote step has placeholder `repoContentsUrl`.
5. Monorepo / multi-Component registration flow not implemented.
6. Implementation ADRs 0013–0016 uncommitted; pending merge to implementation repo.

## 14. Unresolved questions

1. When will automated CI sync replace manual template publishing to corporate catalog?
2. Should legacy .NET POC templates be aligned or deprecated in favor of `dotnet-minimal-api` + `modernize-application`?
3. What is the production authentication model for brownfield PR actions — continued PAT or service principal/managed identity?
4. How should monorepo registration declare multiple Components — one descriptor or multiple?

## 15. Current-state conclusion

Slice 0 is **evidence-complete**. The ADO implementation contains a functional greenfield Scaffolder foundation at HEAD and a substantially more complete Golden Path / brownfield model in uncommitted WIP.

**Implementation gate: GO FOR IMPLEMENTATION** — subject to preconditions:

1. Commit and PR-review WIP on `feat/ado-repo-governance`.
2. Execute first production template sync to corporate catalog.
3. End-to-end validation of `register-existing-application` against a real legacy repository.
4. Fix `modernize-application` promote URL placeholder.

**Recommended next slice:** land and harden the brownfield vertical slice (not build from scratch). See `implementation-roadmap.md`.

## 16. Documentation checkpoint record

| Field | SHA |
|---|---|
| `backstage-docs` baseline (pre-Slice 0) | `c0ef3b4d5dfcce5da005d563e4c36fbb48bbb933` |
| `backstage-docs` Slice 0 publish | `f6260c7657ba53207e83bef5542124b3d9581206` |
| `backstage-docs` at last refresh | `6706ad4442dd30435d0b49bfa7ac5a502d7e3426` |
| ADO Slice 0 inspection baseline | `6e28611e90455dfd56f583bdd132ee830d88f126` |
| ADO branch HEAD at last refresh | `be16ffb02b59d791f0086d8e5086e4428c82b90d` |

Shared ADR created: [ADR-010](../adr/ADR-010-catalog-system-component-semantics.md).
