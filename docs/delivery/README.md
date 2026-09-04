# Delivery Management / GitOps promotion

## Status

**Architecture direction only — implementation NOT authorized.**

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

**GO:** architecture spike only.

**NO-GO:** Delivery backend/frontend implementation, Kargo adoption, Argo production integration, F3.1.2 execution integration based on an ADO-specific contract, or organization-wide branching enforcement.

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
