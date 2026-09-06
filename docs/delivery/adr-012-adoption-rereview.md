# ADR-012 Adoption Re-review (post-E1)

## Review baselines

| Baseline | Value |
|---|---|
| Canonical docs (`diegofernandes-dev/backstage-docs` `origin/main` at re-review start) | `f298e6130a0426d79c33cbe71cbf44e254d30308` |
| Review contract | [`prompts/e1-commit-and-adr012-rereview.md`](../../prompts/e1-commit-and-adr012-rereview.md) |
| Prior adoption review | [`adr-012-adoption-review.md`](./adr-012-adoption-review.md) — `REMAIN_PROPOSED` (3 / 7 / 2) |
| E1 evidence | [`e1-multi-activity-concurrency.md`](./e1-multi-activity-concurrency.md) — `PASS` |
| Implementation pre-commit SHA | `platform-devops-developer-portal` / `feat/delivery-mvp-slice` / `50ed1b0b72fdf3ddc5152fb47b22f20114238332` |
| E1 committed SHA (inspected) | `platform-devops-developer-portal` / `feat/delivery-mvp-slice` / `c2feb8ac122d952a560cb268d6b620265d57e22a` |
| ADR under review at start | [`ADR-012`](../adr/ADR-012-delivery-management-gitops-promotion.md) — **Proposed** |

This re-review does **not** optimize for accepting ADR-012 because E1 passed. Architecture acceptance and production rollout remain separate decisions.

## Overall verdict

```text
ADR-012 verdict: ACCEPT
ADR-012 architecture status: Accepted (2026-09-06; this re-review)
Production rollout gate: NO-GO
```

E1 closed the previously architecture-decision-blocking gaps on activity-scoped binding, multi-activity completion boundary, and same-target Delivery concurrency. Remaining open items are either (a) bounded follow-ups under a now-stable contract (window TOCTOU eligibility wiring; GitOps layout as configuration under Option A invariants) or (b) production adoption / hardening blockers that do **not** invalidate the architecture direction.

Production rollout remains **NO-GO** while authority gaps stay open.

## Committed-SHA verification (Phase A)

| Check | Result |
|---|---|
| E1 isolation from brownfield WIP | Pathspec commit of 10 Delivery files only; brownfield dirt left uncommitted |
| Pre-commit tests | `DeliveryService.test.ts` **18 passed**; `catalogEntityTabs/index.test.ts` **4 passed** |
| Post-commit tests from `c2feb8a` | Same suites: **18 passed** / **4 passed** |
| E1 working-tree scope | Clean for Delivery/E1 paths after commit |

## 12-gate evidence matrix

| # | Gate | Score | Evidence | What E1 changed | Remains / gap class |
|---|---|---|---|---|---|
| 1 | Pure delivery path | **PROVEN** | **[docs+prior-run]** MVP/D1 DEV→HML without GMUD. **[code]** `@c2feb8a` non-Change targets still dispatch without eligibility. | None material | Supply-chain provenance hardening — **production-hardening follow-up** |
| 2 | Provider comparison | **PROVEN** | **[docs+prior-run]** D1 `KARGO_FIT` + thin fallback recorded | None | Footprint cost — not architecture reopen |
| 3 | Backstage projection | **PROVEN** | **[code]** Deployments tab projects binding/request/provider; server re-evaluates eligibility. **[ui]** Component → Deployments + Platform tabs observed live on running UI | Narrow activity bind UX + non-completion copy | Global workbench — **product follow-up** |
| 4 | Production binding | **PROVEN** | **[code]** `ChangeBinding` requires `changeId` + `activityId`; migration `activity_id`; `ChangeActivityClient` membership via user-on-behalf Change read; request pins RC + target; dispatch refuses unbound / non-ALLOW. **[ui]** Bind form requires Change ID + Activity ID; PRD shows bound Change + non-completion copy (pre-E1 row may show empty Activity until rebound) | Closed prior missing `activityId` | Provider-specific IDs remain outside canonical Change (held). Residual empty-Activity display on legacy rows is data carry, not contract gap — **hardening/product** |
| 5 | Eligibility / start | **PARTIALLY_PROVEN** | **[code]** Live eligibility before `claimDispatch`; fail-closed DENY. **[docs+prior-run]** MVP DENY-before / ALLOW-after. **[code]** `EligibilityService.evaluate` still does **not** consult `requestedWindow` | None for window | Window TOCTOU undemonstrated — classified **bounded follow-up under Accepted ADR** (see Challenge D), not architecture reopen |
| 6 | Concurrency / idempotency | **PROVEN** | **[code]** `claimDispatch`: target busy + CAS before provider; concurrent distinct RCs → exactly one winner + `CONFLICT`/`target_busy`; same RC+target idempotent; dispatched short-circuit | Closed same-target exclusive active mutation at Delivery gate | Live dual-Freight Git conflict not re-driven — **additional confidence only** (architecture claim closed pre-provider) |
| 7 | Failure / recovery | **PARTIALLY_PROVEN** | **[docs+prior-run]** D1 restart recovery; MVP Errored→`failed`; D0 Git revert. **[code]** pull-based projection | None | Callback/ingress at scale; adversarial Git retry — **production-hardening follow-up** |
| 8 | Security | **NOT_PROVEN** | **[docs+prior-run]** D0/D1 authority gaps unchanged (broad Argo RBAC; Git writer/reconciler not production-separated; ordinary-squad bypass unproven) | None (by design — out of scope) | **Production adoption blockers** (see security classification). Not treated as architecture-direction invalidation |
| 9 | Audit | **PARTIALLY_PROVEN** | **[code]** Durable RC/target/request/binding (now with `activity_id`)/projection/eligibility decision id | Durable `activityId` on binding | Retention vs provider history; desired-state Git revision as first-class audit field — **production-hardening follow-up** |
| 10 | Multi-activity semantics | **PROVEN** | **[code]** Multi-activity harness: bind one activity; Succeeded mirrors Delivery only; Change `submitted` unchanged; activities remain plan facts without status fields. **[ui]** GMUD detail status `Submetida`; activities listed as plan items. Deployments non-completion copy | Closed prior `NOT_PROVEN` | ADR-009 `executing`/`completed` lifecycle remains Change Management follow-up — **product/CM follow-up**, not Delivery boundary reopen |
| 11 | Rollback / break-glass | **PARTIALLY_PROVEN** | **[docs]** Hypotheses + D0 Git revert capability | None | Explicit production operational policy / break-glass procedure — **production-hardening / ops follow-up** (architecture allows deferred automation) |
| 12 | Branching | **PROVEN** | **[docs+prior-run]** Immutable digest promotion; Option A (`stages/{dev,hml,prd}` on one protected line) exercised. **[code]** RC identity is digest/Freight | None | Option B not built — classified **not required for ADR acceptance** (layout is implementation/configuration under documented invariants). Org-wide app-repo branching unenforced — acceptable |

### Gate tally

```text
PROVEN: 7
PARTIALLY_PROVEN: 4
NOT_PROVEN: 1
CONTRADICTED: 0
```

Prior tally was 3 / 7 / 2. Material lifts: gates **4**, **6**, **10** (and **12** reclassified with explicit Option B argument).

## Required challenge areas

### A. Activity-scoped production binding

Committed code binds:

```text
changeId + activityId + release identity (via DeploymentRequest → ReleaseCandidate)
+ DeploymentTarget (via DeploymentRequest → DeploymentTarget)
```

Membership is validated against Change `executionPlan.activities` over HTTP (ACL preserved). Immutable binding rejects silent reuse for a different activity. Provider/Kargo/Argo identifiers remain on Delivery request/projection only.

**Sufficient for architecture?** Yes for the ADR vocabulary of Change/activity + release fingerprint + target. Authorization reuse across unrelated material remains prevented by request↔RC↔target pinning plus binding immutability. Residual: pre-E1 DB rows may have empty `activity_id` default until rebound — operational carry, not a contract hole for new binds.

### B. Multi-activity completion boundary

E1 proved the claim by assertion, not omission alone: after Succeeded projection, Change status and activity plan facts are unchanged; Delivery has no Change completion write path. ADR-008 activities did not drift into workflow tasks/status machines.

### C. Same-target concurrency

`claimDispatch` exclusivity before provider create is **architecturally sufficient** for ADR acceptance: deterministic exactly-one active mutation (or visible `CONFLICT`) with no silent double-promote. Absence of a live dual-Freight PRD experiment is additional confidence, not an architecture gap — the invariant is enforced at the Delivery domain boundary where double mutation would otherwise be admitted.

### D. Window TOCTOU

```text
performed / not performed: not performed (intentionally)
EligibilityService: no requestedWindow consult [code]
```

**Classification:** bounded follow-up under an **Accepted** ADR — **not** an architecture-decision blocker.

Rationale: ADR-009 already Accepted owns window semantics; Delivery already performs fresh eligibility immediately before dispatch (proven). The missing piece is wiring `requestedWindow` into `EligibilityService` (Change Management eligibility completeness) plus a TOCTOU evidence case. No competing architectural alternative for window ownership. Do **not** treat this as license to skip the follow-up — production start paths that depend on windows remain incomplete until wired and proven.

### E. Security vs architecture acceptance

Reaffirmed: least-privilege Argo/Kubernetes authority, non-human Git writer vs reconciler separation, and ordinary-squad bypass resistance remain **PRODUCTION_ADOPTION_BLOCKER**s. They do **not** prove the Git→Argo→K8s / Delivery boundary direction is wrong. Therefore ADR may Accept while production rollout stays **NO-GO**. No production RBAC/credential work was performed in this checkpoint.

### F. GitOps layout

Option A has been exercised. Option B was not built. **Option B comparison is not required to accept ADR-012.** Concrete repository layout remains an implementation/configuration choice governed by invariants (immutable release fingerprint; Git desired-state authority; no branch-as-environment routing). ADR text may later refine “undecided layout” language to “Option A proven; alternatives allowed if invariants hold” without reopening the decision.

## Boundary regression review

Invariant:

> Change Management authorizes. Delivery promotes. Git declares. Argo CD reconciles. Kubernetes executes. Backstage composes the experience.

| Risk | Finding |
|---|---|
| Provider IDs in canonical Change | **Not observed** at `c2feb8a` |
| Change orchestrates Delivery | **Not observed** — eligibility HTTP only |
| Backstage as runtime authority | **Not observed** — UI affordance only; **[ui]** Deployments/Platform/GMUD render |
| Second business approval in Kargo/Git PR/Argo | **Boundary held** (PR = mutation policy) |
| `ExecutionActivity` workflow-task drift | **Not observed** — plan facts only |
| Provider-neutral Delivery domain | **Held** (`KargoK8sProvider` adapter) |

```text
Architecture boundary regression: NONE
```

## Authority / security classification (unchanged substance)

| Gap | Classification |
|---|---|
| Argo controller Kubernetes authority broader than production scope | PRODUCTION_ADOPTION_BLOCKER |
| Git writer / reconciler identity separation not production-grade | PRODUCTION_ADOPTION_BLOCKER |
| Ordinary squad bypass resistance unproven | PRODUCTION_ADOPTION_BLOCKER |
| Kargo→Argo broad RBAC vs code-level skip | PRODUCTION_HARDENING_FOLLOW_UP |

## Carried gaps after ACCEPT

### Bounded follow-ups under Accepted ADR (not architecture reopen)

1. Eligibility window TOCTOU — wire `requestedWindow` into `EligibilityService` + evidence case.
2. Desired-state Git revision / audit retention drill.
3. Explicit production rollback / break-glass operational policy text.
4. Pre-E1 empty `activity_id` legacy binding rows (rebind or migrate).

### Production adoption blockers (rollout NO-GO)

1. Argo/Kubernetes least-privilege topology with negative denial proof.
2. Non-human Git writer separated from reconciler; governed Application/AppProject path.
3. Demonstrated resistance to ordinary squad bypass.

### Production hardening / product follow-ups

Supply-chain admission; callback/ingress architecture; HA/DR; automatic rollback automation; global Delivery workbench; richer F3.1.x policy shapes; ADR-009 Change lifecycle milestones (`executing`/`completed`).

## Independent ADR-012 status decision

```text
ADR-012: ACCEPT
```

Domain contracts and boundaries are sufficiently resolved for implementation to continue without reopening fundamental Change/Delivery/Git/Argo/K8s/Backstage authorities. Residual gaps are carried explicitly and do not change the architecture decision itself.

**ADR status file:** updated to Accepted with this re-review as reference. Production rollout remains separately **NO-GO**.

## Independent production rollout decision

```text
Production rollout: NO_GO
```

No new factual evidence closed authority gaps in this checkpoint; hardening was explicitly out of scope.

## Smallest next evidence checkpoint

Not required to keep ADR Proposed. Recommended **bounded follow-up** (not a new Delivery milestone authorization):

**Name:** `eligibility-window-TOCTOU` — Change Management `EligibilityService` consults `requestedWindow`; prove ALLOW-near-close then DENY on Delivery dispatch. Do not start production hardening or a broad Delivery milestone from this handoff alone.

## What this review did not execute

- No window/TOCTOU implementation.
- No production RBAC / credential / cluster topology changes.
- No Option B GitOps layout build.
- No break-glass / automatic rollback implementation.
- No next Delivery milestone.
- No live dual-Freight PRD re-drive.

## Final report (compact)

```text
Phase A — E1 reproducibility
Docs baseline SHA: f298e6130a0426d79c33cbe71cbf44e254d30308
Implementation pre-commit SHA: 50ed1b0b72fdf3ddc5152fb47b22f20114238332
E1 committed SHA: c2feb8ac122d952a560cb268d6b620265d57e22a
Working tree: clean for E1 scope (brownfield WIP remains uncommitted)
Tests: DeliveryService 18 passed; catalogEntityTabs 4 passed (pre + post commit)
UI/browser evidence: Deployments (bound Change + non-completion copy); Platform tab loads; GMUD CHG-2026-000001 status Submetida + plan activity

Phase B — ADR-012 re-review
Gate tally:
PROVEN: 7
PARTIALLY_PROVEN: 4
NOT_PROVEN: 1
CONTRADICTED: 0

ADR-012 verdict: ACCEPT
Production rollout: NO_GO

Window TOCTOU classification: bounded follow-up under Accepted ADR
Security classification: production adoption blockers (rollout NO-GO)
GitOps layout classification: Option A sufficient; Option B not required for acceptance
Architecture boundary regression: NONE

Canonical docs updated: e1-multi-activity-concurrency.md; adr-012-adoption-rereview.md; delivery/README.md; adr/README.md; ADR-012 status
Next smallest gate (if required): eligibility-window-TOCTOU (bounded follow-up; not ADR reopen)
STOP
```
