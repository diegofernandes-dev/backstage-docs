# Delivery Management / GitOps promotion

## Status

**MVP vertical slice implemented and demo-hardened (CONDITIONAL PASS). Production rollout NOT authorized.**

The MVP vertical slice (ReleaseCandidate → DEV/HML/PRD promotion via Kargo/Argo, gated by a real GMUD authorization decision) was implemented and independently verified end-to-end — see [`mvp-vertical-delivery-slice.md`](./mvp-vertical-delivery-slice.md) (`CONDITIONAL PASS`) and [`mvp-demo-hardening.md`](./mvp-demo-hardening.md) (`CONDITIONAL PASS`, demo-reliability fixes). ADR-012 remains **Proposed** — the implementation exists as an accepted MVP checkpoint, not as an accepted production architecture.

Shared decision record: [ADR-012 — Delivery Management, GitOps promotion, and Change boundary](../adr/ADR-012-delivery-management-gitops-promotion.md).

Current GMUD implementation state remains authoritative in [`../backstage/current-state.md`](../backstage/current-state.md). At the documentation baseline that opened this workstream, F3.1.1a was implemented/published at ADO `d3c0751`; F3.1.1b and F3.1.2+ were not authorized.

## Why this workstream exists

The GMUD architecture correctly established that a Change is a governed business change, not an Azure DevOps pipeline. The next architectural step is to prevent software delivery from being forced back into the GMUD lifecycle merely because production deployment is one execution mechanism.

Delivery therefore becomes a separate platform concern:

```text
Change Management
  "may this governed change execute?"

Delivery Management
  "what release should move to what target, and what is its delivery state?"

Execution provider
  "make the target converge"
```

For Kubernetes, the first provider hypothesis is Kargo for promotion plus Git and Argo CD for reconciliation.

## Current architectural invariants

1. **GMUD != deployment.** A Change may include multiple deployments, manual work, DB/network work, or no deployment.
2. **Deployment != pipeline.** Azure DevOps may produce a release or initiate a request; it is one integration source.
3. **Change Management authorizes; it does not orchestrate Delivery.**
4. **Delivery owns Change correlation from its side.** Do not add ADO/Kargo/Argo/deployment IDs to canonical Change.
5. **Build once, promote immutable material.** DEV/HML/PRD should receive the same ReleaseCandidate fingerprint unless a new release is intentionally created.
6. **Branch is provenance/policy, not environment routing.** Do not model `develop -> DEV`, `release -> HML`, `main -> PRD` as the target architecture.
7. **Git is desired-state authority for GitOps-managed targets.** Argo reconciles; normal CI/CD does not imperatively `argocd app sync`.
8. **Kargo is a candidate, not an authority.** Its approval mechanisms must not become a duplicate business approval layer.
9. **Backstage composes the UX; it is not the Kubernetes runtime authority.**
10. **No generic workflow engine.** If Kargo is rejected, build only the thin Delivery behavior that remains necessary.

## Candidate user experience

### Component context

```text
Catalog -> Component -> Deployments
```

Answers: "How is this component deployed?"

Show current/desired release by target, sync/health/verification, recent delivery activity, and related Change when present.

### Global Delivery workbench

```text
Deployments
```

Answers: "What is happening with my/team/platform deliveries?"

Use one shared deployment-detail route for both entry points. GMUD remains a separate governance workbench and may show related delivery projections.

## Proposed production flow

```text
trusted CI
  -> immutable ReleaseCandidate
  -> DEV promotion/verification
  -> HML promotion/verification
  -> request PRD
  -> Delivery policy says CHANGE_REQUIRED
  -> create/select GMUD
  -> ChangeBinding pins Change/activity + release fingerprint + target
  -> fresh Change eligibility
  -> ALLOW
  -> production promotion dispatch
  -> Git desired state
  -> Argo CD reconcile
  -> health/verification
  -> durable Delivery result projection
```

The pipeline may already have ended long before PRD promotion. A developer should follow CD status in Backstage Delivery, not by keeping an Azure DevOps agent/environment waiting.

## Branching direction

For new Golden Path application repositories, evaluate protected trunk-based development with short-lived PR branches:

```text
feature/bugfix/chore -> PR -> protected main -> trusted CI -> immutable release
```

No permanent `develop`; no source branch per environment. Short-lived release/hotfix branches remain possible exceptions when product/release needs justify them.

Legacy repositories are not blocked from the spike. Release eligibility can temporarily be repository-specific while source branch/commit provenance is snapshotted into the release.

GitOps repository layout is deliberately undecided until the spike compares:

- one protected branch + stage directories;
- stage-specific desired-state branches such as `stage/dev`, `stage/hml`, `stage/prd`.

The latter is the current hypothesis, not an accepted standard.

## Gate

**Current status (post demo-hardening): MVP vertical slice implemented, demo-hardened, CONDITIONAL PASS. Production rollout, ADR-012 acceptance, and the next Delivery milestone remain NO-GO pending separate, explicit authorization.**

The sections below are preserved as the historical record of how the workstream reached this point — architecture spike, then D0, then D1, then the MVP vertical slice, then demo hardening. Do not treat the original "GO: architecture spike only" gate below as current; it reflects the state before D0/D1/MVP execution.

**Original spike gate — GO:** architecture spike only.

**Original spike gate — NO-GO:** Delivery backend/frontend implementation, Kargo adoption, Argo production integration, F3.1.2 execution integration based on an ADO-specific contract, or organization-wide branching enforcement.

The spike plan is [`architecture-spike.md`](./architecture-spike.md).

D0 has been executed. Result: **CONDITIONAL PASS** — see
[`d0-gitops-reconciliation-evidence.md`](./d0-gitops-reconciliation-evidence.md).

The architecture review of that evidence is complete — see
[`d0-architecture-review.md`](./d0-architecture-review.md). D0 is
**ACCEPT_CONDITIONAL_PASS**; both open authority findings (broad Argo controller RBAC, and
unseparated Git write / Argo read credentials) are classified **PRODUCTION_HARDENING_GAP**.
D1 is **GO_D1_WITH_CARRIED_GAPS** — authorized to be planned under the carried constraints
recorded in the review, notably that D1 must exercise a representative protected Git
desired-state mutation path. ADR-012 remains Proposed.

D1 has been executed. Result: **KARGO_FIT** (qualified — see the exact adoption scope) — see
[`d1-kargo-fit-evaluation.md`](./d1-kargo-fit-evaluation.md). Kargo materially removes real
promotion machinery (artifact discovery, protected-branch PR-gated Git mutation with proven
controller-restart recovery, Job-backed verification) and its operational footprint is
justified by what it removes, provided the platform does not expect full automation onto a
protected branch — Kargo's own mutation path there is PR-gated, requiring human review, not
push-gated. Both D0-carried authority gaps remain open and unresolved. Git writer/reconciler
identity separation was partially advanced (Kargo was given its own distinct PAT) but not
resolved (the PAT is still tied to the same human ADO account; a true non-human writer
identity was not achievable in the sandbox). ADR-012 remains **Proposed**; production adoption,
GMUD/Change integration, F3.1.2, and Backstage Delivery UI implementation remain **NO-GO**
pending a separate, explicit authorization for the next vertical MVP slice.

The MVP vertical slice was subsequently authorized and executed. Result: **CONDITIONAL PASS** —
see [`mvp-vertical-delivery-slice.md`](./mvp-vertical-delivery-slice.md). The full contract flow
(ReleaseCandidate → DEV → HML → PRD `CHANGE_REQUIRED` → GMUD → DENY → real authorization decision →
ALLOW → Kargo → Git → Argo CD → Kubernetes → Backstage `Deployments` tab) was exercised live and
independently verified against raw cluster/Git/ledger state. Carried gaps: no dedicated test
coverage for `delivery` at that time, the Kargo no-op-PR template limitation (worked around via
baseline seeding, not fixed), and the D0-carried production RBAC/credential-separation gaps.
ADR-012 remained **Proposed**.

Demo-hardening was subsequently authorized and executed. Result: **CONDITIONAL PASS** — see
[`mvp-demo-hardening.md`](./mvp-demo-hardening.md). Three real demo blockers were found and fixed:
a clean checkout of the demo SHA did not build (committed code referenced never-committed files),
the Kargo no-op-PR gap was closed for real (a narrow, imperative Stage-template guard, on top of
the existing baseline-seeding reset), and a 15-test regression suite now covers the Delivery PRD
gate invariants (previously zero). ADR-012 remains **Proposed**. Production rollout, ADR-012
acceptance, and the next Delivery milestone remain **NO-GO** pending a separate, explicit
authorization.
