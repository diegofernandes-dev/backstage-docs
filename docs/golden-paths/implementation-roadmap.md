# Golden Paths — implementation roadmap

> **Status:** Active roadmap (Slice 0 complete; template SoT checkpoint complete)  
> **Implementation authorization:** GO for brownfield hardening — NO-GO for production template publishing until Slice T0

## 1. Roadmap principles

Implementation must proceed in small independently reviewable slices. Each slice has explicit prerequisites, non-goals, validation and a stop gate.

Do not begin production template implementation from this roadmap alone.

## 2. Slice 0 — implementation inventory and architecture reconciliation

### Objective

Establish an evidence-complete baseline from Azure DevOps `platform-devops-developer-portal` and reconcile this target architecture against the real Backstage implementation.

### Scope

- inspect exact ADO branch and HEAD SHA;
- inventory Catalog providers/processors and entity conventions;
- inventory Scaffolder plugin/modules/actions and permissions;
- locate existing Software Templates, examples and template registrations;
- identify repository/pipeline creation integrations;
- document central pipeline contract/versioning mechanism;
- inspect relevant tests and representative `catalog-info.yaml` usage;
- identify monorepo/multi-workload conventions already present;
- update `current-state.md` with evidence;
- record deviations between implementation and proposed architecture.

### Non-goals

- no production template creation;
- no Scaffolder code changes;
- no Catalog model migration;
- no pipeline changes;
- no repository policy changes.

### Validation

The checkpoint must record:

- ADO repository;
- ADO branch;
- exact ADO SHA;
- evidence for every UNKNOWN row in `current-state.md` or an explicit unresolved item;
- architecture deviations;
- whether any proposed Catalog semantics require a shared ADR;
- revised recommendation for Slice 1.

### Gate

**COMPLETE** — inspected `feat/ado-repo-governance@6e28611` plus uncommitted WIP. See `current-state.md`.

## 3. Slice T0 — template source boundary foundation

### Objective

Establish the authoritative Software Template repository boundary per [ADR-011](../adr/ADR-011-software-template-source-of-truth.md) before any production template publishing is treated as permanent.

### Scope

- create ADO repository `platform-software-templates` with CODEOWNERS, branch policies, and CI skeleton;
- move `templates/` from `platform-devops-developer-portal` (preserve git history where practical);
- update dev and production `catalog.locations` to URL → template repo bundle (dev: branch; prod: semver tag);
- add template-repo CI: YAML/schema validation, bundle integrity, optional Scaffolder dry-run;
- validate all seven templates E2E on staging;
- supersede implementation ADR 0016 authoring/distribution portions (new ADR 0017 in portal repo);
- remove portal `templates/` only after successful cutover;
- **do not** populate `platform-devops-idp-catalog/templates/`.

### Non-goals

- no brownfield action changes;
- no Scaffolder action refactoring;
- no corporate catalog entity migration.

### Validation

- production Backstage discovers templates from `platform-software-templates` via tag-pinned URL;
- no duplicate template copies in corporate catalog;
- template maintainers can contribute without portal repo write access.

### Gate

**BLOCKS** permanent production template publishing. May proceed in parallel with Slice 1B.

## 4. Slice 1B — land and harden brownfield vertical slice

### Objective

Land the brownfield registration flow already implemented in uncommitted WIP: commit, review, test, and validate end-to-end against a real legacy repository.

Slice 0 found that `register-existing-application`, `idp:register-existing-catalog-pr`, `idpAssessor`, and the Platform tab already exist in the working tree. This slice is **not** greenfield construction — it is review, merge, and production readiness.

### Why this remains the recommended first implementation slice

A registration-only slice validates the hardest shared foundation — identity, ownership, System/Component semantics, repository annotations, permissions and post-registration validation — without coupling the first experiment to repository generation or central pipeline assumptions.

### Scope (revised after Slice 0 and template SoT checkpoint)

- commit and PR-review WIP on `feat/ado-repo-governance`;
- validate `register-existing-application` template and `idp:register-existing-catalog-pr` action;
- end-to-end test: register one real legacy repo → merge PR → catalog import → Platform tab assessment;
- fix `modernize-application` promote URL placeholder;
- document brownfield runbook.

### Non-goals

- no production template publishing (deferred to post-T0);
- no automatic pipeline migration in Slice 1B;
- no source rewrite;
- no repository policy mutation beyond existing governance actions;
- no fleet-wide conformance engine;
- no retirement automation.

### Validation

- a representative existing repository can be registered safely;
- running the flow twice has defined/idempotent behavior;
- ownership/System/Component relations match the accepted Catalog model;
- failure leaves no misleading success state;
- no generated application or delivery logic is required.

### Gate

**GO** for Slice 1B. Proceed to Slice 2 only after brownfield registration is proven against real estate examples. Production template publishing requires Slice T0 completion.

## 5. Slice 1.5 — align legacy greenfield templates and production publishing

### Objective

Close maturity gaps found in Slice 0 without blocking brownfield delivery.

### Scope

- align `dotnet-worker-service`, `dotnet-grpc-service`, and `dotnet-cronjob` to the `dotnet-minimal-api` golden-path pattern (central CI, `idp.platform.yaml`, `idp:platform-context`), **or** mark them deprecated in favor of `dotnet-minimal-api` + `modernize-application`;
- automate template release CI on `platform-software-templates` (semver tags);
- verify production template discovery via tag-pinned URL to `platform-software-templates/templates/catalog-info.yaml`.

### Gate

Requires Slice T0 complete. May proceed in parallel with Slice 1B after T0.

## 6. Slice 2 — conformance assessment foundation

### Objective

Evaluate registered applications against a small, explainable platform profile without mutating repositories.

### Scope

Slice 0 found substantial WIP foundation (`idpAssessor`, `catalogValidation`, `config/scorecards/platform-adoption.yaml`). This slice hardens and validates that foundation:

- required Catalog identity/ownership;
- required repository annotation/link;
- central pipeline contract detection (`usesCentralPipelineTemplate()`);
- explicit UNKNOWN when evidence cannot be derived;
- Platform tab UX for actionable recommendations.

### Non-goals

- no generalized policy framework;
- no auto-remediation;
- no broad security scoring;
- no mass fleet migration.

### Validation

Findings must be deterministic, attributable to a profile/version and distinguish compliant/gap/unknown/not-applicable.

## 7. Slice 3 — first greenfield Golden Path (harden)

### Objective

Harden one supported workload archetype using the already-proven Catalog and ownership model.

### Candidate (confirmed by Slice 0)

`dotnet-minimal-api` is the golden-path reference implementation (WIP-aligned). `angular-spa` is a secondary candidate. Slice 3 focuses on production readiness, not initial creation.

### Scope

- one product template;
- minimal application-owned source/metadata scaffold;
- Catalog registration;
- central pipeline binding using the verified ADO contract;
- post-create verification;
- permissions and tests.

### Non-goals

- no mega-template;
- no second/third stack just for portfolio completeness;
- no broad policy mutation framework.

## 8. Slice 4 — platform capability composition

### Objective

Extract reusable building blocks proven by Slices 1–3 without prematurely building an internal template framework.

Candidates:

- Catalog metadata generation/normalization;
- repository bootstrap;
- pipeline binding;
- post-create validation;
- permission-aware repository operations.

Extraction is justified by repeated concrete use, not anticipation.

## 9. Slice 5 — PR-based modernization (harden)

### Objective

Harden the `modernize-application` template and its PR actions already implemented in WIP.

### Prerequisites

- Slice 1B registration proven;
- stable assessment model (Slice 2);
- verified ADO branch/policy behavior;
- reviewed mutation permissions;
- clear conflict/overwrite strategy;
- promote URL placeholder fixed.

### First candidate migrations

Prefer low-risk declarative changes such as Catalog normalization or thin pipeline binding before broad application-code modernization.

## 10. Slice 6 — multi-workload and monorepo expansion

### Objective

Support adding/registering multiple independently operated Components while preserving one coherent System/repository relationship.

This should extend a proven model rather than introduce the model before basic registration works.

## 11. Slice 7 — exceptions and day-2 governance

### Objective

Add governed deviations, expiry/review and recurring conformance evaluation.

This may require a shared platform ADR because exception semantics can span multiple Backstage workstreams.

## 12. Slice 8 — retirement workflow

### Objective

Provide a safe orchestration path for deprecation and retirement.

Deletion/decommission automation must be the last step, with explicit dependency and authority checks.

## 13. Cross-slice construction handoff

Once implementation begins, create `docs/golden-paths/implementation-progress.md` and record for every checkpoint:

1. ADO repository;
2. ADO branch;
3. ADO implementation SHA;
4. `backstage-docs` baseline SHA;
5. `backstage-docs` resulting SHA;
6. decisions applied;
7. implementation areas changed;
8. tests;
9. functional validation;
10. deviations;
11. unresolved questions;
12. explicit GO / NO-GO.

## 14. Current gate

| Track | Gate |
|---|---|
| Brownfield hardening (Slice 1B) | **GO** |
| Production template publishing | **NO-GO** until Slice T0 |
| Template source boundary (Slice T0) | **GO** — blocks publishing |

**Recommended immediate actions:** Slice T0 and Slice 1B in parallel. Do not sync templates to corporate catalog.
