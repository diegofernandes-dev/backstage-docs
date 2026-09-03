# Delivery Management / GitOps Promotion — Architecture Spike

- **Status:** Planned architecture spike; implementation not authorized
- **Opened:** 2026-09-03
- **Shared ADR:** [ADR-012](../adr/ADR-012-delivery-management-gitops-promotion.md)
- **Implementation baseline relevant to Change:** ADO `d3c0751` for F3.1.1a, as recorded by the canonical GMUD current state at spike opening

## Objective

Determine whether a dedicated Delivery Management boundary, with Kargo + Git + Argo CD as the first Kubernetes provider hypothesis, gives the platform a simpler and safer software-delivery architecture than:

- holding Azure DevOps environments/jobs open while humans approve;
- exposing Change draft/eligibility APIs primarily for Azure DevOps callbacks;
- embedding deployment lifecycle inside GMUD;
- building a generic execution/state-machine controller from scratch.

The spike must challenge the proposal. Its purpose is not to prove Kargo is correct.

## Questions to answer

### Domain

1. Can `ReleaseCandidate`, `DeploymentTarget`, `DeploymentRequest`, and `ChangeBinding` remain provider-neutral and materially simpler than a generic workflow model?
2. Can one Change govern multiple deployments without making deployment a Change lifecycle/status field?
3. Can Delivery bind exact release fingerprint + target + optional Change activity so an authorization cannot be replayed for another artifact/target?
4. Can ADR-009 whole-Change lifecycle evidence coexist with ADR-008 multi-activity planning without introducing per-activity workflow status?
5. What is the minimum durable Delivery evidence needed for audit after Kargo/Argo history is pruned?

### CI / release provenance

6. Can CI end after producing trusted immutable release material, with promotion fully decoupled?
7. For Azure DevOps-produced releases, what authoritative server-side data proves source branch, source commit, build result and artifact identity?
8. What release eligibility policy supports current legacy branch diversity without leaking branch names into Change/Delivery environment semantics?
9. What provenance/signature/attestation hooks should the model preserve even if supply-chain enforcement is deferred?

### Branching

10. Validate protected trunk-based development with short-lived PR branches as the new-project default.
11. Identify legitimate exceptions requiring short-lived release branches or maintenance branches.
12. Compare GitOps desired-state layouts:
    - one protected branch + target directories;
    - stage-specific branches (`stage/dev`, `stage/hml`, `stage/prd`).
13. Verify that neither GitOps layout recreates Git Flow semantics in application source repositories.
14. Define a migration posture for repositories currently using `main`, `release/*`, `hotfix/*`, ad-hoc flows, or no organization-wide standard.

### Kargo

15. Confirm current CRD/API maturity and upgrade strategy are acceptable.
16. Verify Warehouse/Freight/Stage/Promotion semantics against the proposed ReleaseCandidate/DeploymentTarget model.
17. Prove production Stage RBAC can prevent ordinary developers/pipelines from bypassing Change eligibility.
18. Verify Azure DevOps Git integration, registry discovery, credentials, private-network requirements and secret handling.
19. Exercise duplicate Promotion requests, concurrent requests to the same Stage, Git conflicts and controller restarts.
20. Validate verification behavior and how Delivery projects `PROMOTING`, `HEALTHY`, `VERIFIED`, `FAILED` without copying provider state as a second authority.
21. Determine retention/history behavior and what evidence must be persisted outside Kargo.
22. Evaluate rollback capabilities, but keep automatic PRD rollback disabled until governance policy is explicit.
23. Measure operational footprint/HA/observability enough to determine whether Kargo reduces or merely relocates platform complexity.

### Argo CD / GitOps

24. Prove normal deployment through Git desired-state change + Argo reconciliation, not imperative pipeline sync.
25. Validate AppProject/RBAC/sourceRepo/destination restrictions for production.
26. Test self-heal as drift remediation to the existing authorized desired state.
27. Test prune/delete guardrails and critical-resource deletion behavior.
28. Prove recovery if Argo misses webhook/notification events; polling/reconciliation must recover.
29. Define success for the POC: initially `Synced + Healthy`; optionally add smoke verification for `Verified`.
30. Verify rollback returns Git desired state as well as runtime state.

### Change integration

31. Define the exact governed execution-context fingerprint passed to `ExecutionEligibility`.
32. Prove release ABC + target PRD cannot reuse the same binding for release XYZ or another target.
33. Define the execution-start evidence boundary. Preferred hypothesis: accepted production Promotion after a fresh `ALLOW` check.
34. Test the time-window edge case where eligibility occurs seconds before the window closes.
35. Define what happens if eligibility is `ALLOW` but dispatch fails before start evidence is accepted.
36. Define cancellation before dispatch vs. cancellation after dispatch without pretending cancellation stops already-running infrastructure.
37. Demonstrate that a successful deployment of one activity does not complete a multi-activity Change.
38. Decide how a simple one-activity deployment Change may eventually auto-complete without becoming the general model.

### Resilience / security

39. Define HA expectations for the Delivery and Change backend control path.
40. Define a privileged break-glass path for emergency recovery when Backstage UI/control components are unavailable.
41. Prove break-glass produces durable actor/time/target/release evidence and requires later reconciliation.
42. Protect production GitOps write paths so ordinary squad credentials cannot push desired state directly.
43. Validate that Kargo/Argo credentials have narrow blast radius by stage/project/target.
44. Define replay/idempotency identity for DeploymentRequest and production dispatch.
45. Confirm notifications/webhooks are accelerators only; read-back/reconciliation is authoritative.

### Backstage UX

46. Build or mock the Component `Deployments` tab: current/desired release by target, health, recent deployments, related Change.
47. Build or mock the top-level Delivery/Deployments workbench: mine/team/all, waiting/running/failed/healthy, target and Change filters.
48. Use one shared Deployment detail page from both entry points.
49. Prove Change detail can show related deployments as a read projection without owning their lifecycle.
50. Verify community Argo/Kargo plugin compatibility with the platform's Backstage New Frontend System before committing to any plugin; prefer a provider-backed internal Delivery UI if compatibility is weak.

## POC sequence

### POC D0 — pure GitOps reconciliation

No GMUD and no custom Delivery state machine.

```text
known immutable image digest
  -> controlled Git desired-state update
  -> Argo CD
  -> sandbox cluster
  -> Synced + Healthy
  -> Git revert / recovery
```

Prove AppProject restrictions, health, self-heal, prune posture and private connectivity.

### POC D1 — promotion controller fit

```text
artifact source
  -> Kargo Warehouse/Freight
  -> DEV Stage
  -> HML Stage
  -> Git
  -> Argo
  -> verification
```

Compare Kargo behavior against the amount of custom orchestration that would otherwise be required.

**Exit:** either `KARGO_FIT` with documented constraints, or `KARGO_NO_FIT` with a precise thin-controller fallback scope.

### POC D2 — Release/Delivery projection

Introduce the minimum platform-owned Delivery model/projection only for the POC:

```text
ReleaseCandidate
DeploymentTarget
DeploymentRequest
provider correlation
```

No GMUD yet. Prove Backstage Component view + shared detail and durable minimal audit evidence.

### POC D3 — production Change binding

```text
DeploymentRequest(PRD)
  -> CHANGE_REQUIRED
  -> create/select Change
  -> ChangeBinding(change/activity + release fingerprint + target)
  -> authorization
  -> eligibility
```

Do not create persistent Change drafts. A blocked DeploymentRequest may deep-link to `/gmud/new` with safe prefill context.

### POC D4 — full vertical slice

```text
trusted CI/release
  -> DEV
  -> HML
  -> PRD request
  -> GMUD
  -> authorization
  -> fresh ALLOW
  -> dispatch/start evidence
  -> Kargo/Git/Argo
  -> Healthy/Verified
  -> Delivery result
  -> related evidence visible from Change
```

This is the first point at which the architecture can be considered end-to-end validated.

## Comparative acceptance matrix

Score Kargo and thin-custom-controller alternatives against the same criteria; do not optimize for confirming the preferred tool.

| Criterion | Kargo candidate | Thin custom controller |
|---|---|---|
| Promotion semantics already implemented | Evaluate | Must implement |
| Git conflict/retry/idempotency burden | Evaluate | Must design/build |
| Verification integration | Evaluate | Must design/build/integrate |
| Argo awareness | Evaluate | Must integrate |
| Azure DevOps compatibility | Evaluate | Under our control |
| Operational footprint | Evaluate | Evaluate |
| API maturity/upgrade risk | Evaluate | Internal ownership risk |
| RBAC/production isolation | Evaluate | Must implement |
| Backstage integration effort | Evaluate | Under our control |
| Audit retention | Supplement required | Must implement |
| Vendor/tool lock-in | Evaluate behind provider boundary | Internal framework risk |
| Risk of becoming generic workflow engine | Lower if used narrowly | High unless scope is aggressively constrained |

## Branch-control spike detail

Branch strategy must be evaluated as a source/release policy, not CD routing.

### Greenfield hypothesis

```text
protected main
  <- short-lived PR branches

on accepted merge:
trusted CI
  -> immutable ReleaseCandidate
```

Minimum controls to evaluate for Golden Paths:

- PR required;
- direct push denied;
- required CI/security/quality checks;
- review policy appropriate to repository risk;
- stale/failed checks cannot produce a release-eligible candidate;
- source commit and branch stored as provenance;
- merge to main does not imply automatic PRD promotion.

### Brownfield compatibility

Create a documented `ReleaseEligibilityPolicy` concept for the spike, not necessarily a new generic policy engine. It may initially express repository-specific accepted source refs while the organization converges.

Example migration data, not canonical domain:

```text
repo A -> main only
repo B -> main | release/* | hotfix/*
repo C -> legacy rule under explicit exception
```

Every resulting `ReleaseCandidate` still carries authoritative provenance. Delivery promotes the candidate, never a mutable branch reference.

### GitOps layout comparison

Evaluate both with the same release promoted through DEV/HML/PRD:

**Option A — one branch, directories**

```text
main/
  apps/payments/dev/
  apps/payments/hml/
  apps/payments/prd/
```

**Option B — stage branches**

```text
stage/dev
stage/hml
stage/prd
```

For each measure:

- branch/ruleset complexity;
- writer permissions;
- Kargo native fit;
- Argo targetRevision mapping;
- conflict behavior;
- audit readability;
- promotion diff clarity;
- rollback/revert behavior;
- human accidental-write risk;
- scaling to many components/targets.

Do not decide by analogy with Git Flow; desired-state branches have different semantics.

## Failure scenarios that must be demonstrated

| Failure | Required behavior |
|---|---|
| Same release/request submitted twice | One logical DeploymentRequest / deterministic replay |
| Caller lies about branch/artifact | Trusted adapter/provenance wins; request rejected or corrected |
| Controller restarts waiting for Change | Recovers from durable request/binding |
| Approval happens while Delivery is offline | Fresh state is observed after recovery |
| Window closes before dispatch | No production dispatch/start accepted |
| `ALLOW` returned but dispatch fails | No false execution-start evidence; safe retry requires fresh rules as defined |
| Two PRD requests target same app/target | Serialized/rejected/queued deterministically |
| Git push conflicts | Safe retry/re-read; no normal force-push |
| Kargo/Argo callback lost | Reconciliation/read-back recovers state |
| Argo sync fails | Delivery visibly fails; Change authorization is not rewritten as rejected |
| Argo becomes Healthy then controller restarts | Final state is reconstructable |
| Verification fails | Delivery fails; rollback behavior follows explicit policy |
| Provider history is pruned | Durable Delivery evidence still explains the deployment |
| One deployment in multi-activity GMUD succeeds | Whole Change does not incorrectly complete |
| Backstage UI unavailable | Backend/process can recover; break-glass exists for genuine emergency |
| Developer pushes directly toward production desired state | Prevented by Git/Kargo/Argo/RBAC boundaries |

## Explicit STOP conditions

Stop the spike and return to architecture review if any of the following occurs:

- making Kargo a business approval authority appears necessary;
- canonical Change must gain Kargo/Argo/ADO deployment fields for the flow to work;
- the only workable model maps source branches directly to environments;
- Kargo requires broader production credentials than the proposed provider boundary can safely permit;
- Delivery requires a general DAG/rules/workflow language to support the first vertical slice;
- multi-activity Change lifecycle cannot be reconciled without mutating ADR-008 activity semantics;
- the POC cannot bind authorization to an immutable release + target strongly enough to prevent replay/substitution;
- provider state cannot be reconciled after controller/event loss;
- operational footprint exceeds the value compared with a narrowly scoped custom provider.

## Gate after the spike

The architecture reviewer must produce one of:

```text
GO_KARGO
GO_THIN_DELIVERY_PROVIDER
REWORK_REQUIRED
NO_GO
```

Only a `GO_*` decision may move ADR-012 from Proposed toward Accepted and authorize an implementation plan.

Until then:

- no Kargo/Argo production implementation is authorized;
- no Delivery plugin/backend implementation is authorized;
- no organization-wide branch standard is declared by this workstream;
- F3.1.2 must not hardwire Azure DevOps pipeline/environment concepts into Change Management.
