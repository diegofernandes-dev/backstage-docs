# ADR-012 Adoption Gate — Independent Architecture Decision

## Role

Act as an independent Principal Platform Architect / Staff+ reviewer.

Your job is **not** to implement new Delivery features and **not** to optimize for accepting ADR-012.

Your job is to decide, from the executed evidence now present in the canonical documentation and implementation, whether ADR-012 has earned an architecture status change or must remain Proposed.

This is an architecture/adoption gate, not another implementation milestone.

## Canonical sources

Before doing anything:

1. `git fetch origin main` in `diegofernandes-dev/backstage-docs`.
2. Record the exact `origin/main` SHA used for the review.
3. Read the latest canonical state, especially:
   - `docs/adr/ADR-012-delivery-management-gitops-promotion.md`
   - `docs/delivery/README.md`
   - `docs/delivery/d0-architecture-review.md`
   - `docs/delivery/d1-kargo-fit-evaluation.md`
   - `docs/delivery/mvp-vertical-delivery-slice.md`
   - `docs/delivery/mvp-demo-hardening.md`
   - `docs/delivery/branching-release-control.md`
   - relevant ADR-006 / ADR-008 / ADR-009 material
   - current Backstage/Delivery implementation state referenced by the docs.
4. Where implementation evidence is needed, inspect the real implementation repository/branch and the exact SHAs referenced by canonical docs. Do not infer implementation facts from documentation alone when direct inspection is available.
5. If the implementation repository cannot be accessed, explicitly mark implementation-dependent conclusions as documentation-derived and do not claim independent code verification.

The latest canonical state always wins over assumptions embedded in this prompt.

## Current checkpoint context

The following prior checkpoints have already executed and must be treated as evidence, not as work to repeat:

- D0: `ACCEPT_CONDITIONAL_PASS` with carried production-hardening gaps.
- D1: `KARGO_FIT` with qualified adoption scope.
- MVP vertical delivery slice: `CONDITIONAL PASS`.
- MVP demo hardening: `CONDITIONAL PASS`.

The demonstrated path includes, in sandbox form:

```text
ReleaseCandidate
  -> DEV
  -> HML
  -> PRD request
  -> CHANGE_REQUIRED
  -> GMUD / Change binding
  -> ExecutionEligibility DENY / ALLOW
  -> Kargo
  -> Git desired state
  -> Argo CD
  -> Kubernetes
  -> Backstage projection
```

ADR-012 is still `Proposed` and must not be changed merely because the MVP works.

## Decision to make

Evaluate whether ADR-012 should now be classified as one of:

### `ACCEPT_ADR_012`

The architecture direction has been sufficiently proven to become the accepted target architecture. Remaining gaps are production implementation/hardening tasks and do not challenge the architectural decision itself.

### `ACCEPT_ADR_012_WITH_EXPLICIT_CARRIED_GAPS`

The architecture direction is sufficiently proven to accept, but acceptance must carry a short, explicit set of non-negotiable production gates that remain open before rollout.

Use this only if the unresolved items are genuinely implementation/hardening concerns and not missing proof of a core architectural claim.

### `REMAIN_PROPOSED`

The architecture is promising and the MVP is useful, but one or more ADR-012 architecture-spike gates remain materially unproven such that accepting it for production implementation would be premature.

If this is the verdict, identify the **smallest possible evidence checkpoint** needed to resolve the uncertainty. Do not propose a broad new research phase.

### `REWORK_REQUIRED`

Executed evidence contradicts a core ADR-012 premise or reveals boundary/tooling problems substantial enough that the direction itself needs revision before more implementation.

This should require concrete contradictory evidence, not generic production-hardening concerns.

## Mandatory review against ADR-012 architecture-spike gates

ADR-012 defines 12 minimum gates before it may move to Accepted for production implementation.

Evaluate every one independently as:

```text
PROVEN
PARTIALLY_PROVEN
NOT_PROVEN
CONTRADICTED
```

For each gate, cite the exact evidence and explain what remains missing.

The 12 gates are:

1. **Pure delivery path** — trusted immutable release -> DEV -> HML -> Git -> Argo -> Healthy/Verified with no GMUD.
2. **Provider comparison** — Kargo materially reduces custom orchestration complexity and fallback is understood.
3. **Backstage projection** — component deployment experience works without Backstage becoming runtime authority.
4. **Production binding** — exact release + target + Change/activity context is bound; authorization cannot be reused for different material/target.
5. **Eligibility/start** — fresh ALLOW precedes dispatch/start evidence and TOCTOU/window semantics are sound enough for the architectural contract.
6. **Concurrency/idempotency** — duplicate requests and same-target concurrent promotions behave deterministically.
7. **Failure/recovery** — missed callbacks/read-back, controller restart, sync/verification failure, and Git conflict behavior recover or fail visibly.
8. **Security** — production Git/Kargo/Argo write authority cannot be bypassed by an ordinary squad pipeline/user.
9. **Audit** — durable Delivery facts survive provider history cleanup or the architecture clearly proves provider retention is not the audit authority.
10. **Multi-activity semantics** — a deployment does not incorrectly complete a multi-activity Change.
11. **Rollback/break-glass** — production policy is explicitly defined even where automation remains deferred.
12. **Branching** — release eligibility is independent from environment routing, and the GitOps desired-state layout decision is sufficiently resolved for adoption.

Do not lower these gates simply because a stakeholder demo is now possible.

## Questions the review must challenge

### Architecture boundary

Confirm whether the implementation still preserves:

> Change Management authorizes. Delivery promotes. Git declares. Argo CD reconciles. Kubernetes executes. Backstage composes the experience.

Specifically challenge whether:

- Delivery has leaked provider-specific Kargo/Argo semantics into canonical Change objects;
- Change Management has become an execution/orchestration engine;
- Backstage has become runtime authority;
- Kargo or Git PR approval has accidentally become a second business approval authority;
- `ExecutionActivity` has drifted into workflow-task semantics.

Any such drift is architecture-significant.

### Kargo adoption

Separate these questions:

1. Is Kargo a good fit for the currently proven Kubernetes promotion path?
2. Is Kargo mandatory to the architecture?
3. Are the current RBAC/credential gaps Kargo disqualifiers or ordinary production-hardening gates?
4. Does the implementation still preserve a provider boundary so Kargo can be replaced without redesigning Change Management?

Do not equate `KARGO_FIT` with unconditional production readiness.

### Security gaps

Re-review the carried authority gaps, including at minimum:

- Argo CD controller Kubernetes authority currently broader than acceptable production scope;
- Git writer/reconciler authority separation not yet production-grade;
- Kargo/Argo authorization defense-in-depth versus actual Kubernetes RBAC capability boundaries;
- protection against ordinary squad pipelines/users bypassing the intended path.

Classify each as either:

```text
ARCHITECTURE_BLOCKER
PRODUCTION_ADOPTION_BLOCKER
PRODUCTION_HARDENING_FOLLOW_UP
```

This distinction is critical. An ADR can potentially be architecturally Accepted while production rollout remains NO-GO.

### Demo hardening versus architecture proof

Do not count demo reliability work as proof of unrelated architecture gates.

For example:

- unit tests do not prove production authority separation;
- fixing Kargo no-op behavior does not prove same-target concurrent mutation safety;
- a working single Change does not automatically prove multi-activity semantics;
- durable database rows existing in an MVP do not automatically prove retention/cleanup behavior unless that behavior was actually demonstrated or is structurally guaranteed.

## Decision rules

### ADR status and rollout status are separate

The final review must state **two independent decisions**:

```text
ADR-012 architecture status
Production rollout gate
```

Examples:

```text
ADR-012: ACCEPTED
Production rollout: NO-GO pending security/hardening gates
```

or:

```text
ADR-012: REMAIN_PROPOSED
Production rollout: NO-GO
```

Do not use an open production-hardening item as automatic proof that the architecture itself is invalid. Conversely, do not accept the ADR merely because the architecture appears sensible.

### Evidence bar for acceptance

If one of the 12 explicit ADR gates is still materially `NOT_PROVEN`, acceptance requires a clear argument for why the missing proof is no longer architecture-decision-relevant and can safely become a downstream production gate.

If you cannot make that argument from evidence, keep the ADR Proposed.

### No implementation in this checkpoint

This prompt does **not** authorize:

- production RBAC changes;
- credential redesign;
- additional Kargo/Argo installation work;
- new Backstage features;
- global Delivery Workbench implementation;
- Teams/CAB implementation;
- new F3 horizontal slices;
- automatic rollback;
- break-glass implementation;
- multi-cluster work;
- arbitrary refactoring;
- accepting ADR-012 before the review reaches that conclusion.

If missing proof is discovered, document the smallest follow-up checkpoint only.

## Required output

Produce a canonical architecture review document, preferably:

```text
docs/delivery/adr-012-adoption-review.md
```

It must contain:

1. exact docs baseline SHA;
2. implementation baseline SHA(s) actually inspected;
3. overall verdict;
4. the 12-gate evidence matrix;
5. architectural boundary assessment;
6. Kargo fit/adoption assessment;
7. authority/security classification;
8. carried gaps and their severity;
9. independent ADR-012 status decision;
10. independent production rollout GO/NO-GO decision;
11. if needed, exactly one smallest next evidence checkpoint;
12. explicit statement of what was **not** executed in this review.

If and only if the evidence justifies an ADR status transition:

- update `docs/adr/ADR-012-delivery-management-gitops-promotion.md` status accordingly;
- update `docs/adr/README.md` and relevant Delivery current-state/README references;
- make the carried production gates explicit without rewriting historical evidence.

If the verdict is `REMAIN_PROPOSED`, do **not** edit the ADR status merely to show progress.

## Final response format

Return a concise final report containing:

```text
Verdict:
ADR-012 status:
Production rollout:
12 gates: X PROVEN / Y PARTIALLY_PROVEN / Z NOT_PROVEN / N CONTRADICTED
Architecture blockers:
Production adoption blockers:
Production hardening follow-ups:
Smallest next checkpoint (if any):
Docs commit SHA:
Implementation SHA(s) inspected:
STOP
```

## STOP gate

After the review, canonical documentation update, and final report:

**STOP.**

Do not implement the next checkpoint, production hardening, or additional Delivery functionality.