# Golden Paths — implementation roadmap

> **Status:** Active roadmap (Slice 0 complete)  
> **Implementation authorization:** GO — subject to preconditions in `current-state.md`

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

## 3. Slice 1 — land and harden brownfield vertical slice

### Objective

Land the brownfield registration flow already implemented in uncommitted WIP: commit, review, test, and validate end-to-end against a real legacy repository.

Slice 0 found that `register-existing-application`, `idp:register-existing-catalog-pr`, `idpAssessor`, and the Platform tab already exist in the working tree. This slice is **not** greenfield construction — it is review, merge, and production readiness.

### Why this remains the recommended first implementation slice

A registration-only slice validates the hardest shared foundation — identity, ownership, System/Component semantics, repository annotations, permissions and post-registration validation — without coupling the first experiment to repository generation or central pipeline assumptions.

### Scope (revised after Slice 0)

- commit and PR-review WIP on `feat/ado-repo-governance`;
- validate `register-existing-application` template and `idp:register-existing-catalog-pr` action;
- end-to-end test: register one real legacy repo → merge PR → catalog import → Platform tab assessment;
- execute first production template sync to `platform-devops-idp-catalog/templates/`;
- fix `modernize-application` promote URL placeholder;
- document runbook in corporate catalog README.

### Non-goals

- no automatic pipeline migration in Slice 1;
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

GO to Slice 1.5 and Slice 2 only after the brownfield registration mechanism is proven against real estate examples.

## 4. Slice 1.5 — align legacy greenfield templates and production publishing

### Objective

Close maturity gaps found in Slice 0 without blocking brownfield delivery.

### Scope

- align `dotnet-worker-service`, `dotnet-grpc-service`, and `dotnet-cronjob` to the `dotnet-minimal-api` golden-path pattern (central CI, `idp.platform.yaml`, `idp:platform-context`), **or** mark them deprecated in favor of `dotnet-minimal-api` + `modernize-application`;
- automate or document template sync CI from portal repo to corporate catalog (ADR 0016);
- verify production template discovery via `platform-devops-idp-catalog/templates/catalog-info.yaml`.

### Gate

May proceed in parallel with Slice 1 after WIP is committed.

## 5. Slice 2 — conformance assessment foundation

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

## 6. Slice 3 — first greenfield Golden Path (harden)

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

## 7. Slice 4 — platform capability composition

### Objective

Extract reusable building blocks proven by Slices 1–3 without prematurely building an internal template framework.

Candidates:

- Catalog metadata generation/normalization;
- repository bootstrap;
- pipeline binding;
- post-create validation;
- permission-aware repository operations.

Extraction is justified by repeated concrete use, not anticipation.

## 8. Slice 5 — PR-based modernization (harden)

### Objective

Harden the `modernize-application` template and its PR actions already implemented in WIP.

### Prerequisites

- Slice 1 registration proven;
- stable assessment model (Slice 2);
- verified ADO branch/policy behavior;
- reviewed mutation permissions;
- clear conflict/overwrite strategy;
- promote URL placeholder fixed.

### First candidate migrations

Prefer low-risk declarative changes such as Catalog normalization or thin pipeline binding before broad application-code modernization.

## 9. Slice 6 — multi-workload and monorepo expansion

### Objective

Support adding/registering multiple independently operated Components while preserving one coherent System/repository relationship.

This should extend a proven model rather than introduce the model before basic registration works.

## 10. Slice 7 — exceptions and day-2 governance

### Objective

Add governed deviations, expiry/review and recurring conformance evaluation.

This may require a shared platform ADR because exception semantics can span multiple Backstage workstreams.

## 11. Slice 8 — retirement workflow

### Objective

Provide a safe orchestration path for deprecation and retirement.

Deletion/decommission automation must be the last step, with explicit dependency and authority checks.

## 12. Cross-slice construction handoff

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

## 13. Current gate

**GO FOR IMPLEMENTATION** — subject to preconditions in `current-state.md` section 15.

**Recommended immediate next action:** Slice 1 — land and harden the brownfield vertical slice (commit WIP, E2E validation, production template sync). Slice 1.5 may proceed in parallel.
