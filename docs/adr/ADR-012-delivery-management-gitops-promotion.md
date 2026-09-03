# ADR-012 — Delivery Management, GitOps promotion, and Change boundary

- **Status:** Proposed — architecture spike required
- **Date:** 2026-09-03
- **Related:** [ADR-003](./ADR-003-provider-agnostic-change-management.md), [ADR-006](./ADR-006-change-management-backend-contract.md), [ADR-008](./ADR-008-multi-activity-change-execution-plan.md), [ADR-009](./ADR-009-change-authorization-model.md), [ADR-010](./ADR-010-catalog-system-component-semantics.md), [ADR-011](./ADR-011-software-template-source-of-truth.md)
- **Workstream:** [Delivery Management / GitOps promotion](../delivery/)

## Context

The original GMUD work began from an Azure DevOps production-stage bottleneck: after a business change had already been approved, DevOps still had to approve production pipeline environments individually. Early architecture naturally explored having Azure DevOps call Change Management at the production gate.

That direction exposed a deeper modeling problem. A GMUD is **not a pipeline and not a deployment**. A business change may include database work, network changes, manual procedures, multiple application deployments, or no software deployment at all. Conversely, software delivery exists in DEV/HML as well as PRD and needs a first-class operational experience even when no GMUD is required.

Keeping the pipeline as the primary control surface would also introduce avoidable infrastructure and lifecycle coupling: public inbound integration edges for Azure DevOps service checks, long-lived environment waits, or private agents blocked while humans approve. Those are valid integration options but poor domain boundaries.

A second observation is that a custom generic execution/state-machine service would risk becoming a new workflow engine. For Kubernetes delivery, existing tools already specialize in the hard parts: Git stores desired state, Argo CD reconciles it, and promotion controllers such as Kargo specialize in moving immutable release material through environments.

The platform therefore needs an explicit architecture boundary between **Change Management** and **Delivery Management** before F3 execution integration is designed.

## Decision direction

This ADR proposes the following target principle:

> **Change Management authorizes. Delivery promotes. Git declares. Argo CD reconciles. Kubernetes executes. Backstage composes the experience.**

Kargo is the first promotion-controller candidate to evaluate during the architecture spike. It is **not** accepted as a mandatory dependency by this ADR.

This ADR does not supersede ADR-009. It narrows how software-delivery execution should integrate with ADR-009's provider-neutral `ExecutionEligibility` contract.

## Domain boundaries

### Change Management

Owns business governance:

- Change identity and provider-neutral record contract;
- risk, classification, requested window, rollback plan and evidence;
- immutable `ExecutionPlan` / `ExecutionActivity` planning facts;
- authorization policy, requirements and decisions;
- `AuthorizationEvaluation` and `GovernanceEvaluation`;
- point-in-time `ExecutionEligibility` (`ALLOW` / `DENY`);
- accepted whole-change execution evidence and audit.

Change Management must not gain canonical fields such as:

```text
pipelineId
buildId
stageId
environmentId
argoApplication
kargoStage
freightId
gitopsCommit
deploymentId
```

Those are delivery/executor correlation facts.

### Delivery Management

Proposed new bounded context for software delivery:

- immutable release candidates and provenance;
- deployment targets;
- deployment/promotion requests;
- target-specific delivery policy;
- correlation to a Change and optional `ExecutionActivity` when governance is required;
- provider-neutral delivery status/projection;
- durable minimal execution evidence independent of provider retention;
- Backstage delivery APIs and UI projections.

Delivery may consult Change Management. Change Management must not orchestrate Delivery.

### Execution / promotion providers

Provider-specific adapters translate Delivery intent into a concrete mechanism. The first Kubernetes path under evaluation is:

```text
Delivery
  -> promotion provider (Kargo candidate)
  -> Git desired state
  -> Argo CD
  -> Kubernetes
```

The Delivery domain must not encode Kargo/Argo object names as canonical business semantics. A future provider could support MuleSoft, VM-based delivery, another Kubernetes CD mechanism, or another product without redesigning GMUD.

## Candidate domain vocabulary

The exact HTTP/schema design is deferred to the spike. The following concepts are architectural candidates, not authorized implementation types.

### `ReleaseCandidate`

An immutable description of releasable software material, for example:

```text
releaseId
componentRef
sourceRepositoryRef
sourceCommit
sourceBranch (provenance, not environment routing)
buildRef
artifacts[] / artifact fingerprint
provenance / attestation refs
createdAt
```

The important invariant is **build once, promote the same immutable material**. DEV/HML/PRD must not silently rebuild different artifacts for the same logical release.

A release caller's branch/artifact claims are not trusted merely because they appear in a request. Where Azure DevOps is the CI authority, an adapter should resolve authoritative run/source/artifact metadata server-side.

### `DeploymentTarget`

Provider-neutral identity for a place to deploy, not merely a string such as `prd`:

```text
targetKey
componentRef
environmentClass  // e.g. development, homologation, production
```

Provider configuration may map a target to Kargo Stage, Argo Application, cluster, namespace, Git path/branch, etc. Those mappings remain outside the canonical Delivery object.

This allows future shapes such as:

```text
payments-api/dev
payments-api/hml
payments-api/prd-core
payments-api/prd-dmz
```

without embedding infrastructure topology into Change Management.

### `DeploymentRequest`

Represents the user/system request to promote a specific release to a specific target. It is not itself a GMUD and not necessarily an active technical deployment.

A request may require no Change (typical DEV/HML MVP) or may be blocked on a Change eligibility decision (typical PRD MVP).

### `ChangeBinding`

Correlation owned from the Delivery side, conceptually:

```text
changeId
activityId?              // when a Change execution activity represents this deployment
releaseFingerprint
deploymentTargetKey
```

The binding prevents the same production authorization from being ambiguously reused for unrelated release material or another target.

MVP relationship constraint:

```text
Change            1 -> N DeploymentRequests
DeploymentRequest 0..1 -> Change
```

One Change may govern multiple deployments. One deployment request is governed by at most one Change in the MVP.

## Production authorization binding

ADR-009 requires target/context correlation during eligibility but deliberately deferred the transport and correlation vocabulary. Delivery makes that requirement concrete.

A production eligibility request must be bound to the **actual intended execution**, not merely to `componentRef` or a branch name. The spike must define a stable fingerprint over at least:

```text
Change / round evidence
+ optional ExecutionActivity
+ ReleaseCandidate fingerprint
+ DeploymentTarget
```

Example:

```text
CHG-42 authorizes payments-api release ABC -> prd-core
```

must not automatically authorize:

```text
payments-api release XYZ -> prd-core
```

or:

```text
payments-api release ABC -> another production target
```

The binding belongs to Delivery/execution context; it must not turn provider-specific deployment IDs into canonical Change fields.

## Multi-activity Changes

ADR-008 deliberately models `ExecutionActivity` as planned work, not workflow nodes, and rejects per-activity lifecycle/status. That decision remains intact.

Therefore a successful software deployment may emit evidence related to an activity, but it **must not automatically complete a multi-activity Change**. For example:

```text
CHG-42
  ACT-1 database migration
  ACT-2 deploy payments-api -> DEP-100 succeeds
  ACT-3 network change still pending
```

`DEP-100` success is execution evidence for ACT-2; it is not proof that CHG-42 is complete.

The architecture spike must reconcile ADR-009's whole-Change `executing/completed` milestones with multi-activity execution evidence without converting ADR-008 activities into workflow tasks. A simple single-activity Change may support safe automatic whole-change completion as a later optimization.

## Branching and release eligibility

Branching is important, but it is an **upstream source/release concern**, not the Delivery environment-routing model.

### New application repositories — proposed default

Preferred direction for Golden Paths / new repositories:

> **Protected trunk-based development using short-lived PR branches.**

Expected characteristics:

- protected `main` / trunk;
- PR required; no direct developer push to trunk;
- short-lived `feature/*`, `bugfix/*`, `chore/*` (naming is illustrative, not domain semantics);
- required CI/security/quality checks;
- trunk stays releasable;
- no permanent `develop` branch;
- no permanent environment branches in the application source repository;
- `release/*` only as a justified, short-lived exception when product/release characteristics require it;
- `hotfix/*` is a short-lived repair branch merged back to trunk, not a separate production authority.

GitHub Flow-style collaboration is compatible with this direction; the architectural property is trunk-based integration and short-lived PR branches, not branding the process with one workflow name.

### Branch does not select environment

Reject this coupling as the target architecture:

```text
develop -> DEV
release -> HML
main    -> PRD
```

Environment promotion operates on immutable `ReleaseCandidate`s:

```text
main/trusted source -> CI -> release ABC
                         -> DEV
                         -> HML
                         -> PRD
```

The same release fingerprint is promoted. Merge, release creation, promotion, and deployment are distinct events.

### Legacy/brownfield repositories

Lack of an organization-wide branch standard is **not a POC blocker**. A `ReleaseEligibilityPolicy` boundary may support repository-specific source rules during migration, for example current trusted `main`, `release/*`, or `hotfix/*` conventions, while persisting source provenance in the immutable release record.

Those temporary source rules must not leak into Change Management or become a permanent `if branch == prd` deployment model.

### GitOps repository branch strategy

Two viable GitOps layouts remain intentionally open for the spike:

1. one protected branch with stage/environment directories;
2. stage-specific desired-state branches (for example `stage/dev`, `stage/hml`, `stage/prd`).

The current hypothesis favors stage-specific branches for strong stage isolation and Kargo fit, but this ADR does **not** accept that layout before the POC compares operational simplicity, branch protection, review/audit, promotion conflict handling, and Argo/Kargo behavior.

GitOps stage branches are desired-state snapshots; they are not Git Flow development branches.

## Argo CD boundary

For Kubernetes targets, Argo CD is the preferred reconciler candidate.

Proposed rules:

- CI/Delivery should not use `argocd app sync` as the normal deployment contract;
- Delivery/promotion changes Git desired state;
- Argo CD pulls/reconciles the declared state;
- Git is the desired-state authority;
- Argo Application health/sync is provider operational state, not canonical Change status;
- Argo AppProjects, Kubernetes RBAC, Git permissions, and admission controls form defense in depth.

### Drift/self-heal

A live-cluster drift correction back to an already-authorized Git state is conceptually remediation, not a new desired-state change. The production POC should therefore evaluate `selfHeal: true` with direct cluster mutation strongly restricted.

### Prune/delete

Deletion has larger blast radius. Production prune behavior must be explicitly tested/guarded; critical resource deletion may require confirmation or admission/policy controls. This is not a GMUD approval duplication mechanism.

## Kargo hypothesis

Kargo is the first candidate to evaluate for continuous promotion because its concepts align with the missing layer between CI artifacts and Argo CD:

```text
Warehouse -> Freight -> Stage -> Promotion -> Git -> Argo CD
```

The platform must **not** use Kargo approval as a second business approval authority. If PRD promotion is gated, Change Management remains the authority and Delivery invokes/permits the Kargo promotion only after eligibility is `ALLOW`.

The spike must explicitly validate:

- API/CRD maturity and upgrade posture;
- operational complexity and HA requirements;
- Azure DevOps Git/registry integration needed by the organization;
- RBAC, especially `promote` authority for production Stages;
- stage/promotion concurrency semantics;
- Git write/retry/conflict behavior;
- verification model;
- history/retention limits;
- rollback behavior;
- observability and Backstage integration surface;
- whether the product materially reduces custom platform code.

If Kargo fails these gates, the fallback is a **thin Delivery controller/provider**, not a generic workflow/state-machine engine.

## Delivery policy by target

The MVP policy may be intentionally simple:

```text
DEV -> Change not required
HML -> Change not required
PRD -> Change required
```

But this must be represented as Delivery policy, not as a hardcoded semantic that every production execution forever requires one GMUD. Future categories may include pre-authorized operational changes or other governed paths.

## GMUD creation UX

Do not introduce persistent `ChangeDraft` merely so a deployment request can pre-create a GMUD.

Preferred MVP:

```text
DeploymentRequest -> CHANGE_REQUIRED
                 -> Backstage offers "Create GMUD"
                 -> /gmud/new?deploymentRequest=...
                 -> pre-fill safe delivery context
                 -> user completes risk/window/rollback/execution plan
                 -> POST /changes
                 -> create ChangeBinding
```

A persistent editable draft lifecycle would add autosave, ownership, expiry, abandonment, editing and submission semantics and should require a separate decision if later needed.

## Delivery UI/UX

Backstage should expose two views over one Delivery domain:

### Component contextual view

`Catalog -> Component -> Deployments`

Answers: **"How is this component deployed?"**

Typical content:

- current/desired release per target;
- sync/health/verification projection;
- recent deployments/promotions;
- actions such as deploy/promote/request PRD where authorized;
- related Change when one exists.

### Global Delivery workbench

Top-level `Deployments` / Delivery page.

Answers: **"What is happening with my deliveries / the platform now?"**

Typical filters:

- mine/team/all by permission;
- running/waiting/failed/healthy;
- component, target/environment;
- Change-required / waiting-authorization.

Both routes use one shared Deployment detail page. Change/GMUD retains a separate governance workbench and may show related deployments as a read projection; neither UI object owns the other's lifecycle.

## Authority matrix

| Question | Proposed authority |
|---|---|
| What source/build produced the release? | CI / trusted release provenance |
| What immutable material is promoted? | Delivery `ReleaseCandidate` |
| Which target is requested? | Delivery `DeploymentTarget` / request |
| Is the business production change authorized? | **Change Management** |
| Is this exact execution eligible now? | **Change Management eligibility over Delivery-supplied governed context** |
| What is the GitOps desired state? | **Git** |
| What promotion is being performed? | Delivery provider / Kargo candidate |
| What should Kubernetes converge to? | Argo CD reading Git |
| What is the live runtime state? | Kubernetes |
| Is Argo synced/healthy? | Argo CD |
| Did post-deploy verification pass? | Verification provider / Delivery projection |
| What should the developer see? | Backstage composed projection |

## Time/window semantics

ADR-009 correctly separates `ALLOW` from execution start. The delivery architecture must define the start boundary to avoid time-of-check/time-of-use ambiguity.

Preferred MVP hypothesis:

> The Change window governs **dispatch/start acceptance**, not the total duration of reconciliation.

For the Kargo candidate this may map to an accepted production Promotion after a fresh eligibility check. The exact evidence contract is not accepted until the spike proves the sequence and failure modes.

Do not translate each GMUD window into an Argo SyncWindow. Argo SyncWindows are better suited to platform-wide freeze/maintenance guardrails.

## Idempotency, concurrency and recovery

These are production gates even if Kargo handles part of them:

- the same logical deployment request must not create duplicate promotions;
- only one active production promotion should mutate a given `DeploymentTarget` at a time unless explicitly proven safe;
- Git conflicts must be retried safely; never force-push desired-state history as a normal path;
- provider callbacks/notifications are accelerators, not sole truth; reconciliation/read-back must recover missed events;
- durable Delivery evidence must survive Kargo/Argo object retention/cleanup.

If custom code writes Git directly, Git+database behavior must be treated as a saga with an idempotent correlation identity. Prefer using a proven promotion controller so the platform does not recreate this machinery without need.

## Rollback

Rollback is a desired-state change and must not be hand-waved as an implementation detail.

Hypothesis to evaluate:

- rollback explicitly described in the approved Change rollback plan may later be treated as pre-authorized compensation under defined constraints;
- automatic production rollback is **deferred** until this policy is explicit;
- DEV/HML may experiment with automatic rollback/verification independently;
- Git should remain the desired-state history; imperative Argo rollback must not leave Git declaring the failed state.

## Break-glass and control-plane availability

Backstage UI availability must not be a prerequisite for emergency recovery. If Change/Delivery becomes a production control plane, architecture must define:

- HA expectations for backend control services;
- a tightly privileged break-glass path;
- mandatory identity/audit evidence;
- reconciliation back into canonical records after emergency action;
- no ordinary developer bypass of GMUD/Delivery controls.

Break-glass is an operational resilience requirement, not an alternate happy path.

## Audit and retention

Do not rely on Argo/Kargo CR history forever. Delivery should durably retain the minimum evidence needed to explain a deployment even if provider records are pruned, likely including:

```text
deploymentRequestId
release fingerprint
deployment target
requestedBy/requestedAt
changeId/activityId when present
providerKey + externalRef
desired-state revision/commit
start/completion timestamps
outcome
```

This is intentionally smaller than the Change authorization ledger. Detailed schema/retention duration is deferred.

## Supply-chain posture

The Delivery boundary is a natural enforcement point for immutable artifacts and future provenance/signature checks.

Preferred direction:

```text
trusted CI -> immutable release/artifact fingerprint
           -> provenance/signature/attestation validation
           -> Change authorization when required
           -> Delivery promotion
           -> Git desired state
           -> Argo
           -> cluster admission controls
```

Supply-chain implementation is not part of this ADR's immediate spike, but the ReleaseCandidate model must not prevent it.

## Explicit non-goals

This proposal does **not** authorize:

- a generic workflow engine or arbitrary state-machine framework;
- embedding ADO/Kargo/Argo identifiers in canonical Change;
- turning `ExecutionActivity` into executable workflow tasks;
- a second approval authority in Kargo, Argo, Azure DevOps, Teams or Git PRs;
- persistent Change drafts;
- automatic production rollback;
- multi-cluster progressive-delivery frameworks beyond what the spike needs;
- direct implementation of Kargo/Argo integration in the existing GMUD F3 slices;
- a final organization-wide Git branching mandate for all legacy repositories.

## Architecture spike gate

Before this ADR may move to Accepted for production implementation, the Delivery spike must prove at minimum:

1. **Pure delivery path:** trusted immutable release -> DEV -> HML -> Git -> Argo -> Healthy/Verified, with no GMUD.
2. **Provider comparison:** Kargo materially reduces custom orchestration complexity; document fallback if it does not.
3. **Backstage projection:** component deployment view plus shared deployment detail can be composed without making Backstage the runtime authority.
4. **Production binding:** a PRD request binds exact release + target + Change/activity context and cannot reuse an authorization for different material.
5. **Eligibility/start:** fresh `ALLOW` precedes accepted dispatch/start evidence; window TOCTOU behavior is demonstrated.
6. **Concurrency/idempotency:** duplicate requests and same-target concurrent promotions behave deterministically.
7. **Failure/recovery:** lost callbacks, provider/controller restart, failed sync, failed verification, and Git conflicts recover or fail visibly.
8. **Security:** production Git/Kargo/Argo write authority cannot be bypassed by an ordinary squad pipeline/user.
9. **Audit:** durable Delivery facts survive provider history cleanup.
10. **Multi-activity semantics:** a deployment does not incorrectly mark a multi-activity Change completed.
11. **Rollback/break-glass:** production policy is explicitly documented even if automation is deferred.
12. **Branching:** application release eligibility is proven independent of environment routing; compare GitOps branch/layout options.

## Roadmap impact

- F3.1.1a remains a valid, orthogonal Change authorization foundation; this ADR does not invalidate its published implementation.
- F3.1.2 execution integration must **not** be designed as an Azure DevOps-specific pipeline gate before the Delivery spike resolves the governed execution-context contract.
- F3.1.1b selector work is technically orthogonal, but no implementation priority/authorization is implied by this ADR; continue only under its own explicit checkpoint.
- No Delivery implementation is authorized by this proposal.

## Non-normative research references

- OpenGitOps principles: https://opengitops.dev/
- Argo CD automated sync: https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/
- Argo CD projects: https://argo-cd.readthedocs.io/en/stable/user-guide/projects/
- Argo CD sync windows: https://argo-cd.readthedocs.io/en/stable/user-guide/sync_windows/
- Kargo core concepts: https://docs.kargo.io/user-guide/core-concepts
- Kargo stages/promotions: https://docs.kargo.io/user-guide/how-to-guides/working-with-stages
- Kargo verification: https://docs.kargo.io/user-guide/how-to-guides/verification
- Kargo access controls: https://main.docs.kargo.io/user-guide/security/access-controls
- Trunk-based release branching guidance: https://trunkbaseddevelopment.com/branch-for-release/
