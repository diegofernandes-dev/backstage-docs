# Golden Paths — implementation roadmap

> **Status:** Proposed roadmap  
> **Implementation authorization:** NO-GO until Slice 0 evidence gate is complete

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

**Current state: REQUIRED / BLOCKED by ADO access in this checkpoint.**

No later slice is authorized until Slice 0 is reviewed.

## 3. Slice 1 — brownfield registration-only vertical slice

### Objective

Prove the convergence model with the lowest-destructive-risk path: register an existing application into the canonical Catalog model without rewriting source, pipeline or runtime architecture.

### Why this is the recommended first implementation slice

A registration-only slice validates the hardest shared foundation — identity, ownership, System/Component semantics, repository annotations, permissions and post-registration validation — without coupling the first experiment to repository generation or central pipeline assumptions.

It also creates immediate value for the existing estate and prevents brownfield adoption from becoming secondary to greenfield scaffolding.

### Candidate scope

Subject to Slice 0 findings:

- one `Register Existing Application` Software Template/workflow;
- supported repository selector/input;
- owner/System/Component declaration using existing Catalog conventions;
- generation or proposal of minimal Catalog metadata;
- registration through the supported Catalog mechanism;
- post-task verification that the entity is resolvable with expected ownership/relations;
- tests for form/schema and orchestration behavior.

### Non-goals

- no automatic pipeline migration;
- no source rewrite;
- no automatic PR creation unless separately authorized after Slice 0;
- no repository policy mutation;
- no conformance engine;
- no retirement automation.

### Validation

- a representative existing repository can be registered safely;
- running the flow twice has defined/idempotent behavior;
- ownership/System/Component relations match the accepted Catalog model;
- failure leaves no misleading success state;
- no generated application or delivery logic is required.

### Gate

GO to Slice 2 only after the Catalog model and registration mechanism are proven against real estate examples.

## 4. Slice 2 — conformance assessment foundation

### Objective

Evaluate registered applications against a small, explainable platform profile without mutating repositories.

### Scope

Start with only a few high-confidence checks, for example:

- required Catalog identity/ownership;
- required repository annotation/link;
- supported delivery binding detection if Slice 0 provides a reliable contract;
- explicit UNKNOWN when evidence cannot be derived.

### Non-goals

- no generalized policy framework;
- no auto-remediation;
- no broad security scoring;
- no mass fleet migration.

### Validation

Findings must be deterministic, attributable to a profile/version and distinguish compliant/gap/unknown/not-applicable.

## 5. Slice 3 — first greenfield Golden Path

### Objective

Create one supported workload archetype using the already-proven Catalog and ownership model.

### Candidate

Choose the highest-demand, best-understood workload after Slice 0; do not assume API is correct before implementation inventory.

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

## 6. Slice 4 — platform capability composition

### Objective

Extract reusable building blocks proven by Slices 1–3 without prematurely building an internal template framework.

Candidates:

- Catalog metadata generation/normalization;
- repository bootstrap;
- pipeline binding;
- post-create validation;
- permission-aware repository operations.

Extraction is justified by repeated concrete use, not anticipation.

## 7. Slice 5 — PR-based modernization

### Objective

Turn selected brownfield assessment gaps into small reviewable repository changes.

### Prerequisites

- stable assessment model;
- verified ADO branch/policy behavior;
- reviewed mutation permissions;
- clear conflict/overwrite strategy.

### First candidate migrations

Prefer low-risk declarative changes such as Catalog normalization or thin pipeline binding before broad application-code modernization.

## 8. Slice 6 — multi-workload and monorepo expansion

### Objective

Support adding/registering multiple independently operated Components while preserving one coherent System/repository relationship.

This should extend a proven model rather than introduce the model before basic registration works.

## 9. Slice 7 — exceptions and day-2 governance

### Objective

Add governed deviations, expiry/review and recurring conformance evaluation.

This may require a shared platform ADR because exception semantics can span multiple Backstage workstreams.

## 10. Slice 8 — retirement workflow

### Objective

Provide a safe orchestration path for deprecation and retirement.

Deletion/decommission automation must be the last step, with explicit dependency and authority checks.

## 11. Cross-slice construction handoff

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

## 12. Current gate

**NO-GO for production implementation.**

Reason: the implementation source of truth has not been inspected during this checkpoint, so current Scaffolder, Catalog and pipeline behavior remains unverified.

**Recommended immediate next action:** complete Slice 0 against the actual Azure DevOps repository. After that review, the default recommendation is Slice 1 (brownfield registration-only) unless implementation evidence materially changes the ordering.
