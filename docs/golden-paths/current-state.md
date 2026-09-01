# Golden Paths — current state

> **Workstream:** Software Templates / Golden Paths / Brownfield Adoption  
> **Architecture repository:** `diegofernandes-dev/backstage-docs@main`  
> **Implementation source of truth:** Azure DevOps `platform-devops-developer-portal`  
> **Assessment date:** 2026-09-01  
> **Implementation verification:** **BLOCKED — Azure DevOps repository not accessible from this checkpoint**

## 1. Purpose

This document records only evidence that can be verified at this architecture checkpoint. It deliberately separates:

1. architecture/documentation facts that are observable in `backstage-docs`; and
2. implementation facts that require inspection of the Azure DevOps repository.

A design described in this repository is not evidence that the capability exists in Backstage.

## 2. Verified platform/documentation facts

The canonical documentation bridge establishes the following authority split:

- Azure DevOps `platform-devops-developer-portal` answers **what is implemented**;
- `backstage-docs@main` answers **what should be implemented**, why, and what construction state was reviewed;
- the former `poc-teams-approval` bridge is historical evidence only;
- cross-workstream platform decisions belong under `docs/adr/`;
- Golden Path-specific architecture and construction handoffs belong under `docs/golden-paths/`.

The broader Backstage current-state document records Backstage 1.51.0, Microsoft Entra ID authentication/MS Graph ingestion, community RBAC with ownership on Systems, Azure DevOps integration, and AWS S3 as the production TechDocs path. Those statements belong to the shared platform snapshot and are not, by themselves, proof of Software Template or brownfield capabilities.

## 3. Golden Path implementation state — not yet verified

The following implementation questions remain **UNKNOWN** until the actual Azure DevOps repository is inspected. They must not be inferred from documentation or historical repositories.

| Area | Required evidence | State |
|---|---|---|
| Software Templates | template YAML files, registered template entities, custom actions | UNKNOWN |
| Scaffolder | installed modules/actions, permissions, task broker/configuration | UNKNOWN |
| Repository creation | ADO integration/action, repository defaults, branch/policy behavior | UNKNOWN |
| Pipeline bootstrap | generated pipeline files, central template references, service connections | UNKNOWN |
| Catalog model | actual `catalog-info.yaml`, processors/providers, entity relations | UNKNOWN |
| Ownership | Group/System/Component ownership mapping in source and catalog | UNKNOWN |
| Monorepo support | entity conventions and template behavior for multiple workloads | UNKNOWN |
| Multi-workload support | workload representation and deployment/pipeline contract | UNKNOWN |
| Brownfield registration | existing import/register flows, validation, discovery | UNKNOWN |
| Day-2 conformance | scheduled checks, scorecards, Tech Insights or equivalent | UNKNOWN |
| Exceptions | policy exception persistence/workflow | UNKNOWN |
| Retirement | deprecation/deletion lifecycle and downstream cleanup | UNKNOWN |

## 4. Evidence required from Azure DevOps before implementation

At minimum, the next implementation inspection must capture:

- repository branch and exact HEAD SHA;
- `packages/app` and `packages/backend` plugin/module registration relevant to Catalog and Scaffolder;
- `app-config*.yaml` Catalog locations/providers and Scaffolder integrations;
- existing `templates/`, `examples/`, or equivalent template assets;
- installed Backstage packages related to scaffolding, Azure DevOps, permissions and Catalog;
- custom Scaffolder actions and their authorization boundary;
- repository/pipeline creation code and any central pipeline-template contract;
- sample production `catalog-info.yaml` files or entity-generation conventions;
- RBAC/permission rules applied to template execution and registration;
- tests that prove template execution or brownfield registration behavior.

The evidence should be summarized here; source trees must not be copied into this documentation repository.

## 5. Architecture-relevant constraints already visible

Even without the ADO source, three constraints are already binding:

1. **Implementation and architecture have separate authorities.** Drift must become an explicit deviation, not an undocumented rewrite of the target design.
2. **Golden Paths and brownfield adoption are peers.** Existing applications cannot be forced through greenfield generation merely to obtain Catalog or governance value.
3. **Shared platform decisions require ADRs.** A workstream document may propose a model, but platform-wide ownership semantics or contracts must be promoted to a shared ADR only after the implementation landscape is verified.

## 6. Current-state conclusion

The documentation workstream is initialized, but the implementation baseline for Software Templates / Golden Paths / Brownfield Adoption is **not evidence-complete**.

Therefore:

- architecture may continue as a target/proposal with explicit assumptions;
- no capability may be labeled implemented from this checkpoint;
- no implementation slice may start until the ADO implementation repository is inspected and this current-state document is reconciled with its exact branch/SHA.

**Current implementation gate: NO-GO.**
