# Golden Paths — target architecture

> **Status:** Reconciled with ADO evidence (Slice 0 complete; template SoT checkpoint complete)  
> **Implementation:** Authorized — subject to preconditions in `current-state.md`  
> **Implementation verification:** `feat/ado-repo-governance@be16ffb` + uncommitted WIP  
> **Template SoT decision:** [ADR-011](../adr/ADR-011-software-template-source-of-truth.md) — HYBRID WITH SPLIT AUTHORING SOURCE

## 1. Objective

Provide one coherent Backstage application model reached through two first-class entry paths:

```text
Create New Application ─┐
                        ├─> Catalog + ownership + platform capability contract
Register Existing App ──┘
```

The two paths may differ in how much automation they perform, but they must converge on compatible Catalog semantics, ownership, platform capability metadata, conformance reporting and lifecycle states.

The governing principle is **Golden Path, not Golden Cage**.

## 2. Architectural boundaries

Backstage is the developer-facing orchestration and discovery plane. It must not become the implementation owner of every delivery capability.

| Concern | Preferred owner |
|---|---|
| Developer interaction, template selection, forms, task orchestration | Backstage Scaffolder |
| Catalog entity model and relations | Backstage Catalog contract |
| Application source code | application repository |
| Reusable CI/CD implementation | central platform pipeline/template repositories |
| Environment/deployment runtime implementation | platform delivery capabilities outside the generated app where possible |
| Template composition/versioning | platform-owned template assets |
| Conformance policy | platform contract + policy/conformance capability |
| Exceptions | explicit governed exception record, not a hidden template fork |

A Software Template should compose platform capabilities. It should not copy the internals of all platform capabilities into every generated repository.

## 3. Initial Golden Path portfolio

Do not start with one universal template containing a large matrix of conditionals. Start with a small portfolio organized around meaningful workload archetypes and compose shared building blocks.

Proposed initial portfolio, **validated by Slice 0**:

1. **Backend service/API** — `dotnet-minimal-api` (golden-path reference, WIP-aligned).
2. **Worker/background service** — `dotnet-worker-service` (POC, needs alignment).
3. **Frontend application** — `angular-spa` (WIP, near production-ready).
4. **Register existing application** — `register-existing-application` (WIP, brownfield registration).
5. **Improve platform alignment** — `modernize-application` (WIP, opt-in PR modernization).

Additional .NET archetypes (`dotnet-grpc-service`, `dotnet-cronjob`) exist as POC templates and should be aligned or deprecated.

Additional paths should be added from demonstrated product demand rather than by anticipating every possible stack.

## 4. Template decomposition

Use three layers.

### 4.1 Product template

The visible template selected by the developer. It owns:

- user intent and questions;
- supported variants;
- validation of required inputs;
- orchestration of reusable actions/fragments;
- final links/output shown to the developer.

It should remain understandable as a product workflow.

### 4.2 Reusable platform actions/components

Reusable implementation capabilities should be factored into tested, platform-owned building blocks, for example:

- repository bootstrap;
- Catalog descriptor generation;
- ownership/system relation normalization;
- central pipeline binding;
- documentation bootstrap;
- registration and post-create validation.

Custom Scaffolder actions are justified only when built-in actions cannot express the required governed behavior cleanly. Custom actions increase the plugin/API support surface and therefore require tests and explicit permission boundaries.

### 4.3 Central delivery capabilities

CI/CD logic, deployment policy, security checks and environment-specific behavior should remain centrally owned where the delivery platform already provides them.

Generated repositories should prefer a small, explicit binding to the central capability rather than receiving a copied implementation that immediately begins to drift.

## 5. Copied versus centrally owned

### Copy into an application repository when

- the artifact is intrinsically application-owned;
- developers must intentionally modify it as part of normal product development;
- version pinning or local review is important;
- the file is a thin declarative contract rather than an implementation framework.

Examples may include `catalog-info.yaml`, a thin pipeline entrypoint, application metadata and workload-specific source scaffolding.

### Keep centrally owned when

- consistency and fleet-wide remediation matter more than local customization;
- the capability is security/governance sensitive;
- duplication would produce upgrade debt;
- implementation changes should roll through a controlled platform contract.

Examples may include central pipeline stages/templates, policy evaluators, reusable deployment logic and organization-wide Scaffolder actions.

## 6. Centralized pipeline relationship

The Golden Path must bind applications to the delivery platform rather than reimplementing it.

Target pattern:

```text
application repo
  └─ thin pipeline declaration / versioned platform contract
       └─ central CI/CD capability
            ├─ build/test/security
            ├─ artifact publication
            ├─ deployment integration
            └─ governance controls
```

The binding must be versionable and reviewable. A central pipeline breaking change must not silently reinterpret every repository. The exact versioning mechanism is an implementation decision that must be reconciled with the existing ADO template model.

## 7. Catalog model

The Catalog must represent the product/application model independently of how the repository was created.

### Verified semantic rules (ADR-010)

- **System** represents a business/product/system boundary — implemented as one System per ADO Team Project.
- **Component** represents a deployable or independently operated software component/workload.
- repository identity is metadata/annotation (`dev.azure.com/project-repo`), not automatically the Component identity;
- ownership resolves to Catalog Groups ingested from Entra ID and remains meaningful across greenfield and brownfield paths;
- generated and registered applications use the same metadata vocabulary (`catalog-info.yaml`, `idp.platform.yaml`, adoption-stage annotation).

See [ADR-010](../adr/ADR-010-catalog-system-component-semantics.md) for the shared platform decision.

## 8. Monorepos

A repository is not the cardinality boundary for Catalog Components.

For a monorepo:

- one repository may contain multiple Components;
- shared repository metadata should not force a fake single deployable Component;
- each independently deployable/operable workload should be representable separately;
- template and registration flows should allow one repository to contribute multiple entity descriptors or one descriptor containing multiple entities, according to the verified Catalog ingestion convention.

The platform should avoid duplicating repository-level bootstrap operations for every Component in the same monorepo.

## 9. Multi-workload applications

An application may contain API, worker, scheduled workload or frontend elements. Do not encode all of them as booleans inside one mega-template.

Prefer a composition model where:

1. the application/system is established once;
2. workloads are added as Components with an explicit workload type;
3. each workload binds to supported delivery capabilities;
4. ownership and system relations remain consistent;
5. the developer can add another workload later without regenerating the entire application.

This gives the greenfield flow a path to evolve into day-2 self-service rather than making the initial template the only automation event in the application's lifetime.

## 10. Template and platform contract evolution

Software Templates are an entry mechanism; the durable contract must outlive the template execution.

Use explicit version dimensions for:

- template product version where materially useful;
- generated application/platform contract version;
- central pipeline contract/version;
- optional policy/conformance profile version.

Do not depend on a hidden assumption that the latest template equals the state of every previously created application.

Evolution rules:

- additive compatible changes should not require mass regeneration;
- breaking contract changes require migration guidance and a bounded compatibility period;
- fleet upgrades should be performed through PR-based modernization or central capability upgrades, not by re-running the creation template;
- conformance should evaluate the effective repository/catalog state, not the historical template version alone.

## 11. Greenfield creation experience

Target developer flow:

```text
Choose Golden Path
  -> provide product/application identity
  -> choose owner/system/workload intent
  -> validate naming and supported options
  -> create/bootstrap repository or target existing approved repository
  -> generate application-owned contract files
  -> bind central delivery capabilities
  -> register Catalog entities
  -> validate resulting entity/repository/pipeline bindings
  -> return links and explicit next actions
```

The form should ask for intent, not implementation trivia that the platform can derive safely.

A successful Scaffolder task is not sufficient. Post-create validation must prove that the generated/registered resources are actually consumable.

## 12. Brownfield entry path

Brownfield adoption is not "run the greenfield template against an old repository".

The first goal is visibility and ownership, followed by progressively stronger platform integration.

Target flow:

```text
Register Existing Application
  -> identify repository and owner
  -> discover/declare system + components
  -> validate Catalog contract
  -> assess platform capabilities and deviations
  -> register without forced rewrite
  -> produce conformance/adoption plan
  -> optionally open modernization PRs
```

Details are defined in `adoption-model.md`.

## 13. Day-2 conformance and governance

Golden Paths only create lasting value if the platform can detect drift after creation.

Conformance should evaluate observable state such as:

- required Catalog metadata and ownership;
- supported pipeline/platform contract binding;
- required repository governance controls when observable;
- supported workload metadata;
- required documentation/operational links;
- declared exceptions and expiry.

Conformance must distinguish:

- compliant;
- partially adopted / informational gap;
- explicitly excepted;
- non-compliant actionable drift;
- not applicable.

Do not make template execution history the conformance oracle.

## 14. Exceptions

An exception is a governed state, not an unofficial template branch.

At minimum, an exception should carry:

- subject (System/Component/repository as appropriate);
- violated or deferred capability/policy;
- rationale;
- owner;
- approver/authority according to the future governance model;
- creation and expiry/review date;
- remediation or retirement expectation.

Exception mechanics are cross-workstream governance and may require a future shared ADR.

## 15. Retirement

Retirement must be explicit and should avoid accidental deletion automation in the first implementation phases.

Target lifecycle:

1. mark Component/System as deprecated/retiring;
2. validate dependents, ownership and operational state;
3. remove or disable delivery bindings through controlled changes;
4. archive/decommission external resources in their owning systems;
5. remove Catalog registration only after the entity is no longer required for dependency/audit discovery.

The Scaffolder may eventually orchestrate retirement workflows, but deletion is not symmetrical with creation and must have stronger safety controls.

## 16. Rejected alternatives

### One mega-template for every stack/workload

Rejected because conditional complexity grows into an internal framework and makes testing, ownership and evolution difficult.

### Copy all pipeline/deployment implementation into every new repository

Rejected because it creates immediate fleet drift and weakens central governance/remediation.

### Force brownfield applications through a rewrite before Catalog registration

Rejected because it turns the Golden Path into a Golden Cage and delays useful ownership/discovery value.

### Treat one repository as one Component

Rejected because it cannot represent monorepos or multiple independently deployable workloads coherently.

### Use historical template version as day-2 compliance truth

Rejected because applications and central capabilities evolve after scaffolding.

## 17. Slice 0 reconciliation summary

| Concern | Result |
|---|---|
| Catalog System/Component semantics | **MATCH** — see ADR-010 |
| ADO repository/pipeline creation | **MATCH** — `publish:azure`, `idp:ado-pipeline-create` |
| Central pipeline contract | **PARTIAL** — thin consumer in WIP for 2/5 greenfield templates |
| Scaffolder actions and permissions | **MATCH** — 10 custom `idp:*` actions (WIP), RBAC gating |
| Catalog locations | **MATCH** — dev file locations, prod URL to corporate catalog |
| Monorepo conventions | **DEVIATION** — schema only, no flows |
| Conformance capability | **PARTIAL** — assessor foundation in WIP |
| Repository policies | **MATCH** — `idp:ado-repo-governance` with hybrid mode |

Full evidence in `current-state.md`.

## 18. Software Template repository boundary

**Recommendation: HYBRID WITH SPLIT AUTHORING SOURCE** — see [ADR-011](../adr/ADR-011-software-template-source-of-truth.md).

Template source-of-truth checkpoint (2026-09-01) superseded the Slice 0 **CURRENT HYBRID** recommendation (portal authoring + corporate catalog sync). The catalog-sync model decoupled runtime discovery but did not decouple authoring ownership and introduced an unnecessary second copy.

### Three explicit concerns

| Concern | Mechanism |
|---|---|
| **Authoring source** | `platform-software-templates` (dedicated Azure DevOps repository) |
| **Publishing / distribution** | Semver git tags on `platform-software-templates` |
| **Runtime discovery** | Backstage `catalog.locations` URL to `templates/catalog-info.yaml`, tag-pinned in production |

### Authoritative boundaries

| Asset | Owner repository | Release model |
|---|---|---|
| Backstage portal + Scaffolder actions | `platform-devops-developer-portal` | Portal deploy |
| Custom field extensions | `platform-devops-developer-portal` | Portal deploy |
| Template YAML + skeletons (authoring) | `platform-software-templates` | Semver git tags |
| Template production distribution | `platform-software-templates` | Direct URL discovery (no catalog mirror) |
| Pipeline implementation | `platform-pipeline-templates` | Git tag semver (`dotnet-ci-1.0.0`, etc.) |
| Corporate catalog entities | `platform-devops-idp-catalog` | PR-based promotion |

`platform-devops-idp-catalog` is **not** a Software Template registry. It holds organizational entity inventory (Systems, Components) only.

### Ownership rationale

- Scaffolder custom actions are trust-boundary code and must ship with the portal.
- Product teams maintaining Golden Paths must be able to contribute **without write access to the Backstage application repository**. Path-based CODEOWNERS inside the portal repo are insufficient — portal CI, dependency upgrades, and RBAC remain in the blast radius.
- Pipeline implementation is already in a separate repository with versioned tags; Software Templates follow the same pattern.

### Versioning

- **Template product version:** semver git tags on `platform-software-templates` (e.g. `templates-v1.0.0`).
- **Scaffolder action contract:** portal release; document action IDs and input schemas.
- **Application/platform contract version:** `idp.platform.yaml` `contracts.pipeline` and adoption-stage annotation.
- **Central pipeline contract version:** git tags on `platform-pipeline-templates`.

### CI validation model

| Pipeline | Validates |
|---|---|
| Portal build/test/deploy | Backstage app, Scaffolder actions, backend modules |
| Template repo CI | Template schema, YAML validity, bundle integrity, optional Scaffolder dry-run |
| Pipeline template release | Central CI template correctness, tag creation |

Templates must not share the portal release path in production.

### Dependency direction

```text
Scaffolder actions (portal) ──orchestrate──> Template YAML (platform-software-templates)
Template YAML ──generates──> Application-owned artifacts (catalog-info, idp.platform.yaml, thin pipeline)
Application artifacts ──bind to──> Central pipeline templates (tagged)
```

### Rejected alternatives

- **KEEP TOGETHER** (templates only in portal repo): couples template changes to portal deploy; blocks independent Golden Path ownership.
- **CURRENT HYBRID** (portal authoring + catalog sync): runtime discovery OK; authoring ownership and duplication risk remain. **Not permanent.**
- **FULL SPLIT** (actions also separate): rejected — actions and field extensions remain portal-coupled.
- **Mega-template:** rejected — seven discrete templates with shared actions is the correct decomposition.

### Migration impact

**Slice T0** (not yet executed): create `platform-software-templates`, move `templates/` from portal, update dev/prod discovery URLs, validate E2E, supersede implementation ADR 0016. Do **not** populate `platform-devops-idp-catalog/templates/`.

## 19. Remaining open questions

- Monorepo registration flow for multiple Components per repository.
- Production authentication for brownfield PR actions (PAT vs service principal).
- Whether legacy .NET POC templates should be aligned or deprecated.
- Exact dev ergonomics for local template iteration (URL vs clone) after T0.
