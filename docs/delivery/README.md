# Delivery Management / GitOps promotion

## Status

**P1 residual closure: `CONDITIONAL_PASS`.** See [`p1-residual-closure.md`](./p1-residual-closure.md) (docs baseline `7458d23`; implementation `platform-devops-developer-portal@b08e7b2`, unchanged). Live Argo/Kargo Git credentials cut over to the two non-human Entra service principals with a proven, automated `*/30 * * * *` token-refresh CronJob; human PATs are no longer active in either controller path. The Kargo no-op promotion guard (broken on v1.11.4) was fixed and both `hml`/`prd` stages recovered from `Errored`. Full negative-authority matrix re-proven on the final live state with fully isolated kubeconfigs. Regression suites green (32 + 8 tests). One residual gap: the durable-GitOps-steady-state PR (`#81`, superseding an unmergeable `#80`) is `Active`, correctly awaiting human approval per branch policy — `d1-prd` still reconciles through the broad destination until it merges. **Ready for separate production-adoption review: NO. Production rollout remains NO-GO.**

**Deployments UX v2 responsive polish: `PASS`.** See [`deployments-ux-v2-responsive-polish.md`](./deployments-ux-v2-responsive-polish.md) (docs baseline `1b181ef`; implementation `platform-devops-developer-portal@b08e7b2` on `feat/delivery-mvp-slice`). Model composition (env+GMUD, history+events) from MUI `lg`; env card stack only below `md`; densified entity masthead; no ADR-012 / backend domain changes. **Production rollout remains NO-GO.**

**Deployments UX v2 implementation: `CONDITIONAL_PASS`.** See [`deployments-ux-v2-implementation.md`](./deployments-ux-v2-implementation.md) (implementation `0163a49`).

**P1 production authority hardening: `CONDITIONAL_PASS`.** See [`p1-production-authority-hardening.md`](./p1-production-authority-hardening.md) (docs baseline `e64435e`; implementation `platform-devops-developer-portal@ee114cf` + one config-only change). All four authority properties (Git writer/reconciler separation, Argo/K8s least privilege, Argo control-object governance, squad/pipeline bypass resistance) are `PROVEN` with real positive and negative execution evidence, including one confirmed live bypass (`ado-agent-cluster-admin`) found and remediated during the checkpoint. Named residual gaps: live Argo/Kargo credential swap + token-refresh automation not built; regression suite not re-executed (environment issue); two GitOps PRs awaiting merge approval. **Production rollout remains NO-GO** pending a separate adoption review — do not treat this checkpoint as that review.

**Eligibility window TOCTOU: `PASS`.** See [`eligibility-window-toctou.md`](./eligibility-window-toctou.md) (implementation `platform-devops-developer-portal@ee114cf`). ADR-009 half-open `requestedWindow` is enforced in `EligibilityService`; Delivery dispatch re-check fail-closes on `OUTSIDE_WINDOW`.

**ADR-012 adoption re-review (post-E1): `ACCEPT`. Production rollout: NO-GO.** See [`adr-012-adoption-rereview.md`](./adr-012-adoption-rereview.md) (docs baseline `f298e61`; E1 implementation SHA `platform-devops-developer-portal@c2feb8a` on `feat/delivery-mvp-slice`). Architecture Accepted 2026-09-06; production authority gaps remain **NO-GO**. Do **not** treat ADR acceptance as production rollout or as authorization to start the next Delivery milestone without an explicit checkpoint.

**E1 evidence checkpoint: `PASS` (reproducible).** See [`e1-multi-activity-concurrency.md`](./e1-multi-activity-concurrency.md) — committed at `c2feb8a`; regression re-verified from that SHA (18 Delivery + 4 tab-loader tests).

**Prior ADR-012 adoption gate: `REMAIN_PROPOSED`.** See [`adr-012-adoption-review.md`](./adr-012-adoption-review.md) (docs baseline `fd19d8a`; implementation inspected at `50ed1b0`). Historical 3 / 7 / 2 tally; named E1 as next checkpoint — E1 executed and re-reviewed above.

Prior product checkpoints (historical evidence, not superseded): MVP vertical slice [`mvp-vertical-delivery-slice.md`](./mvp-vertical-delivery-slice.md) (`CONDITIONAL PASS`) and demo hardening [`mvp-demo-hardening.md`](./mvp-demo-hardening.md) (`CONDITIONAL PASS`). Those prove a sandbox path; they do not by themselves close ADR-012.

Shared decision record: [ADR-012 — Delivery Management, GitOps promotion, and Change boundary](../adr/ADR-012-delivery-management-gitops-promotion.md) — **Accepted** (production rollout separately **NO-GO**).

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

**Current status: P1 authority hardening `CONDITIONAL_PASS`; ADR-012 `ACCEPT` / Accepted; production rollout NO-GO; next Delivery implementation milestone NO-GO pending explicit authorization.** See [`p1-production-authority-hardening.md`](./p1-production-authority-hardening.md), [`eligibility-window-toctou.md`](./eligibility-window-toctou.md), [`adr-012-adoption-rereview.md`](./adr-012-adoption-rereview.md), and [`e1-multi-activity-concurrency.md`](./e1-multi-activity-concurrency.md). Remaining production path is a separate, explicit production-adoption review — not automatic from this pointer.

The sections below are preserved as the historical record of how the workstream reached the MVP/demo-hardening/E1/re-review point — architecture spike, then D0, then D1, then the MVP vertical slice, then demo hardening, then the first adoption review, then E1, then the post-E1 re-review. Do not treat the original "GO: architecture spike only" gate below as current; it reflects the state before D0/D1/MVP execution.

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
gate invariants (previously zero). ADR-012 remained **Proposed**.

The ADR-012 adoption gate was subsequently executed as an independent architecture review (not an
implementation milestone). Result: **`REMAIN_PROPOSED`** — see
[`adr-012-adoption-review.md`](./adr-012-adoption-review.md). Material architecture-spike gaps remain
(notably multi-activity semantics `NOT_PROVEN`, plus partial proof on activity binding, window
TOCTOU, concurrency, and GitOps layout). Production authority gaps remain **production adoption
blockers**. ADR-012 status was **not** changed. Production rollout and the next Delivery
implementation milestone remain **NO-GO**.

E1 (multi-activity binding and same-target concurrency) was subsequently authorized and executed.
Result: **`PASS`** — see [`e1-multi-activity-concurrency.md`](./e1-multi-activity-concurrency.md).
Activity-scoped `ChangeBinding` (`changeId` + `activityId`), multi-activity non-completion boundary,
and same-target `claimDispatch` concurrency were proven with regression coverage and committed at
`c2feb8a`. The post-E1 independent ADR-012 adoption re-review then returned **`ACCEPT`** — see
[`adr-012-adoption-rereview.md`](./adr-012-adoption-rereview.md). Production rollout remains **NO-GO**.
The eligibility-window-TOCTOU bounded follow-up was subsequently executed (`PASS` — see
[`eligibility-window-toctou.md`](./eligibility-window-toctou.md)). Do not start production
hardening or the next Delivery milestone from this pointer alone.
