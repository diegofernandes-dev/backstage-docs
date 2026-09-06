# ADR-012 Adoption Gate — Independent Architecture Review

## Review baselines

| Baseline | Value |
|---|---|
| Canonical docs (`diegofernandes-dev/backstage-docs` `origin/main`) | `fd19d8a4a34e4d8dd311fa47e7de8df2247a03bb` |
| Review contract | [`prompts/adr-012-adoption-gate.md`](../../prompts/adr-012-adoption-gate.md) |
| ADR under review | [`docs/adr/ADR-012-delivery-management-gitops-promotion.md`](../adr/ADR-012-delivery-management-gitops-promotion.md) — status **Proposed** at review start |
| Implementation repo / branch / SHA | `platform-devops-developer-portal` / `feat/delivery-mvp-slice` / `50ed1b0b72fdf3ddc5152fb47b22f20114238332` |
| MVP slice commit (included in demo-hardened tip) | `244df910d3790fb9d2044d4d38b5ec59d2dd2804` |
| GitOps sandbox inspected (layout corroboration) | local `d0-gitops-sandbox` at `b196b7b67dd58f36f69969eb085348d4ad8bd509` (`stages/{dev,hml,prd}` on a single desired-state line) |

Prior checkpoints treated as evidence (not re-executed):

| Checkpoint | Verdict | Canonical record |
|---|---|---|
| D0 | `ACCEPT_CONDITIONAL_PASS` | [`d0-architecture-review.md`](./d0-architecture-review.md) |
| D1 | `KARGO_FIT` (qualified) | [`d1-kargo-fit-evaluation.md`](./d1-kargo-fit-evaluation.md) |
| MVP vertical slice | `CONDITIONAL PASS` | [`mvp-vertical-delivery-slice.md`](./mvp-vertical-delivery-slice.md) |
| MVP demo hardening | `CONDITIONAL PASS` | [`mvp-demo-hardening.md`](./mvp-demo-hardening.md) |

Independent code inspection was performed against `50ed1b0` for Delivery and Change eligibility surfaces. Claims that depend only on prior sandbox cluster/Git runs and were not re-executed live in this review are marked **[docs+prior-run]**. Claims verified in source at the implementation SHA are marked **[code]**.

## Overall verdict

```text
Verdict: REMAIN_PROPOSED
ADR-012 architecture status: Proposed (unchanged)
Production rollout gate: NO-GO
```

The MVP and demo-hardening checkpoints sufficiently demonstrate that the proposed **boundary shape** can host a working sandbox path. They do **not** close enough of ADR-012's twelve explicit architecture-spike gates for an Accepted production-architecture decision. Remaining gaps include architecture-decision-relevant proof (multi-activity semantics; incomplete production-binding/`activityId` contract; window TOCTOU; same-target concurrent mutation), not only ordinary hardening.

This review deliberately does **not** optimize for accepting ADR-012 merely because a stakeholder demo is possible.

## 12-gate evidence matrix

| # | Gate | Score | Evidence | Still missing |
|---|---|---|---|---|
| 1 | Pure delivery path | **PROVEN** | **[docs+prior-run]** MVP/D1: same Freight/digest promoted DEV→HML without GMUD; Git→Argo→Healthy path. **[code]** `requiresChange: false` targets dispatch without eligibility (`DeliveryService.dispatch`, tests “DEV/HML … without any eligibility check”). | Production-grade provenance/signature chain (hardening, not this gate’s core claim). |
| 2 | Provider comparison | **PROVEN** | **[docs+prior-run]** D1 `KARGO_FIT`: Warehouse/Freight, PR-gated protected-branch mutation, Job verification, restart recovery; thin-alternative comparison and fallback understanding recorded. | Footprint remains real; does not reopen the fit question for this gate. |
| 3 | Backstage projection | **PROVEN** | **[code]** Component `Deployments` tab projects target/request/binding/provider state; dispatch UX is affordance only — server re-evaluates eligibility. Provider projection is read-only (`ProviderProjection` comment; `refreshProjection` mirrors terminal phases only). Global Delivery workbench absent but not required to prove “composes without becoming runtime authority.” | Global workbench (product follow-up). |
| 4 | Production binding | **PARTIALLY_PROVEN** | **[docs+prior-run]** MVP: unbound PRD → `CHANGE_REQUIRED` / no Promotion CR; bind Change; DENY then ALLOW. **[code]** `ChangeBinding` pins `deploymentRequestId` + `changeId`; request pins `releaseCandidateId` + `deploymentTargetId`; dispatch refuses without binding / without `ALLOW`. | **`activityId` is not bound** (`ChangeBinding` / DB binding row have no activity field). ADR-012 candidate vocabulary and gate text require Change/**activity** context. Authorization reuse across material is prevented for distinct RCs, but activity-scoped binding is unproven. |
| 5 | Eligibility / start | **PARTIALLY_PROVEN** | **[code]** Live `EligibilityClient` HTTP call before CAS+provider; fail-closed DENY paths; decision id recorded on dispatch. **[docs+prior-run]** MVP DENY-before / ALLOW-after with negative “no Promotion CR” proof. | **Window TOCTOU not demonstrated.** `EligibilityService.evaluate` does not consult `requestedWindow`; no evidence of ALLOW-near-window-close then deny-on-dispatch. Sandbox policy is a static single-requirement file, not F3.1.x richer shapes. |
| 6 | Concurrency / idempotency | **PARTIALLY_PROVEN** | **[code]** Same RC+target returns same request; already-dispatched short-circuits; provider lookup by deployment-request annotation; CAS status transition. **[docs+prior-run]** D1 duplicate no-op promotions; demo hardening closed no-op-PR Errored path. | **Same-target concurrent mutation with distinct Freights / Git push conflict not exercised.** No exclusive “one active PRD mutation per target” lock beyond per-request CAS. |
| 7 | Failure / recovery | **PARTIALLY_PROVEN** | **[docs+prior-run]** D1 controller kill mid-promotion recovered via CR durability; MVP visible Errored→`failed` mirroring without runtime disturbance; D0 Git revert. **[code]** Projection refresh is pull-based (callbacks not sole truth). | Missed-callback architecture at scale unproven; adversarial Git conflict retry policy unproven; public ingress/callback architecture explicitly carried as gap. |
| 8 | Security | **NOT_PROVEN** | **[docs+prior-run]** D0/D1: AppProject guardrails useful but controller ClusterRole cluster-admin-equivalent; Git writer vs reconciler not production-separated (distinct PAT still human-account-tied); Application/AppProject CRs applied imperatively; Kargo Argo RBAC broad capability with code-level skip. Sandbox PR policy blocks unprotected push on desired-state branch — not equivalent to production least-privilege proof. | Production write-authority separation and ordinary-squad bypass resistance remain unproven. |
| 9 | Audit | **PARTIALLY_PROVEN** | **[code]** Durable tables for RC, target, request, binding, projection, eligibility decision id, timestamps, outcome via status. Delivery does not rely solely on Kargo CR names as SoT. | Retention/cleanup vs provider history **not demonstrated**. Durable row lacks first-class `activityId` and desired-state Git revision as ADR-012’s minimum evidence sketch requires. Advanced retention listed as carried gap in MVP docs. |
| 10 | Multi-activity semantics | **NOT_PROVEN** | **[code]** Delivery never mutates Change completion / `executing` milestones (no activity completion API usage). Binding has no `activityId`. **[docs+prior-run]** MVP/demo records contain **zero** multi-activity exercises. | Must prove a multi-activity Change where one deployment success is activity evidence only and **does not** complete the whole Change, without turning ADR-008 activities into workflow tasks (ADR-012 § Multi-activity Changes; spike item 37). Safe-by-omission is not proof. |
| 11 | Rollback / break-glass | **PARTIALLY_PROVEN** | **[docs]** ADR-012 documents rollback hypotheses (Git as history; automatic PRD rollback deferred) and break-glass requirements list. D0 proved technical Git revert. Branching refinement describes hotfix lineage. | **Production policy is not yet an accepted, explicit operational policy** — still hypothesis / deferred automation. Break-glass path, HA expectations, and mandatory emergency audit/reconciliation are requirements, not proven procedures. |
| 12 | Branching | **PARTIALLY_PROVEN** | **[docs+prior-run]** MVP promotes immutable digest, not branch-as-environment. D1 exercised **Option A** (one protected branch + `stages/{dev,hml,prd}`). **[code]** RC identity is digest/Freight, not source branch. | Option B not exercised; ADR still leaves GitOps layout undecided for adoption; organization-wide app-repo branching unenforced (acceptable) but layout decision not “sufficiently resolved” as an accepted standard. |

### Gate tally

```text
PROVEN: 3
PARTIALLY_PROVEN: 7
NOT_PROVEN: 2
CONTRADICTED: 0
```

Gates **8** and **10** are material `NOT_PROVEN`. Gate **10** is architecture-decision-relevant (Change/Delivery completion boundary). Gate **8** is primarily a production-authority proof gap (see security classification) but remains an explicit ADR-012 spike gate and therefore blocks unconditional acceptance without a documented argument that it is no longer architecture-decision-relevant — this review does **not** make that argument for gate 10, and does not treat gate 8 as closed.

## Architectural boundary assessment

Invariant under challenge:

> Change Management authorizes. Delivery promotes. Git declares. Argo CD reconciles. Kubernetes executes. Backstage composes the experience.

| Risk | Finding |
|---|---|
| Delivery leaking Kargo/Argo into canonical Change | **Not observed.** Change types remain free of pipeline/Kargo/Argo fields. Correlation annotations live on Kargo Promotion metadata (`idp.laborit.io/deployment-request-id`, optional `change-id`). Provider names live on Delivery request/projection only. **[code]** |
| Change becoming execution/orchestration engine | **Not observed.** Change supplies eligibility over HTTP; Delivery owns dispatch/CAS/provider call. `EligibilityService` is deliberately outside `ChangeManagementService` submission path. **[code]** |
| Backstage as runtime authority | **Not observed.** UI cannot bypass server eligibility; Argo/Kargo remain execution substrate. **[code]** |
| Kargo / Git PR as second business approval | **Boundary held in design.** PR wait is Git mutation policy, not GMUD substitute (D1). Residual UX risk: `Succeeded` after human PR merge can look “fully automated” unless Delivery distinguishes PR-gated wait — operational clarity gap, not a second GMUD. **[docs+prior-run]** |
| `ExecutionActivity` drifted into workflow tasks | **Not introduced by Delivery**, but also **not integrated**: no `activityId` binding, no multi-activity evidence path. Risk is under-specification, not contradictory workflow-engine drift. **[code]** |

No boundary collapse sufficient for `REWORK_REQUIRED` was found. The direction remains coherent; proof is incomplete.

## Kargo fit / adoption assessment

| Question | Answer |
|---|---|
| Good fit for the proven Kubernetes promotion path? | **Yes, qualified** — per D1 `KARGO_FIT` and MVP reuse of that path. |
| Mandatory to the architecture? | **No.** ADR-012 and D1 keep a provider boundary; thin writer remains a documented fallback with higher custom orchestration cost. |
| Are current RBAC/credential gaps Kargo disqualifiers? | **No** — they are production adoption / hardening blockers, not proof that Kargo cannot fit the promotion niche. |
| Provider boundary preserved? | **Yes** at MVP code shape (`KargoK8sProvider` adapter; Delivery domain types remain provider-neutral aside from target adapter fields). |

`KARGO_FIT` ≠ unconditional production readiness and ≠ ADR-012 Accepted.

## Authority / security classification

| Gap | Classification | Rationale |
|---|---|---|
| Argo CD controller Kubernetes authority broader than acceptable production scope (cluster-admin-equivalent in sandbox; no destination-scoped cluster secret topology proven) | **PRODUCTION_ADOPTION_BLOCKER** | Does not invalidate Git→Argo→K8s architecture (mitigations exist). Blocks production rollout until least-privilege topology is demonstrated with negative checks. |
| Git writer / reconciler identity separation not production-grade (distinct PAT still human-tied; AppProject/Application CRs outside governed Git) | **PRODUCTION_ADOPTION_BLOCKER** | Mandatory production security property; unproven in sandbox. |
| Kargo→Argo broad RBAC capability vs code-level authorization skip | **PRODUCTION_HARDENING_FOLLOW_UP** (elevates to adoption blocker if relied on as sole control) | Defense-in-depth incomplete; pair with Kubernetes RBAC scoping. |
| Ordinary squad pipeline/user bypass of intended path | **PRODUCTION_ADOPTION_BLOCKER** until non-human writer + scoped reconciler + control-object governance proven | Sandbox branch policy helps but is not a complete production authority story. |

None of the above is classified **ARCHITECTURE_BLOCKER** on present evidence: the architecture direction remains compatible with least-privilege topologies that were structurally excluded by the sandbox, not rejected by experiment.

## Carried gaps and severity

### Architecture-decision-relevant (keep ADR Proposed)

1. Multi-activity Change / deployment completion semantics (gate 10) — **architecture spike open**.
2. Production binding missing `activityId` (gate 4 partial) — contract incomplete vs ADR vocabulary.
3. Eligibility window TOCTOU (gate 5 partial) — ADR-009 window semantics not joined to Delivery dispatch.
4. Same-target concurrent promotion / Git conflict determinism (gate 6 partial).
5. GitOps layout adoption decision (gate 12 partial) — Option A exercised only.

### Production adoption blockers (ADR may later Accept with explicit carry; rollout remains NO-GO)

1. Argo/Kubernetes least-privilege execution topology with negative denial proof.
2. Non-human Git writer identity separated from reconciler read identity; governed Application/AppProject mutation path.
3. Demonstrated resistance to ordinary squad bypass of the governed promotion path.

### Production hardening follow-ups

1. Supply-chain provenance/signature/admission beyond digest correlation.
2. Audit retention duration and provider-history independence drill.
3. Callback/ingress architecture as accelerator only.
4. HA/DR for control-plane services.
5. Automatic production rollback automation (policy first).
6. Global Delivery workbench UX.
7. Richer F3.1.x authorization policy shapes beyond sandbox static policy.

## Independent ADR-012 status decision

```text
ADR-012: REMAIN_PROPOSED
```

Rationale: at least one explicit architecture-spike gate (**multi-activity semantics**) remains materially `NOT_PROVEN`, and several others remain only `PARTIALLY_PROVEN` on architecture-relevant claims (activity binding, window TOCTOU, concurrency, GitOps layout). The MVP’s usefulness does not convert those into downstream-only hardening items without an evidence-backed argument this review cannot honestly make.

**ADR status file edits:** none. [`ADR-012-delivery-management-gitops-promotion.md`](../adr/ADR-012-delivery-management-gitops-promotion.md) remains **Proposed — architecture spike required**.

## Independent production rollout decision

```text
Production rollout: NO-GO
```

Independent of ADR status: even a future Accepted ADR must not authorize production rollout while PRODUCTION_ADOPTION_BLOCKER authority gaps remain open.

## Smallest next evidence checkpoint

**Name:** `E1 — multi-activity binding and same-target concurrency`

**Objective (narrow):** close the smallest architecture-decision uncertainty blocking acceptance consideration — not a broad research phase and not production hardening.

**Must prove, in one checkpoint:**

1. A Change with ≥2 `ExecutionActivity` items; Delivery binds **`changeId` + `activityId`** together with release fingerprint + target.
2. Successful deployment evidence for activity A does **not** mark the whole Change completed / does not imply remaining activities done (assert against Change ledger/milestones; no ADR-008 workflow-task drift).
3. Two concurrent production-oriented promotion attempts against the **same** `DeploymentTarget` with different release material (or an induced Git conflict) behave deterministically (exactly-one winner or visible safe failure; no silent double-mutate / force-push).

**Out of scope for E1:** Argo RBAC redesign, credential redesign, break-glass implementation, automatic rollback, multi-cluster, global workbench, organization-wide branching mandates.

**Optional adjacent (only if cheap inside the same harness):** one window TOCTOU case (ALLOW then window expiry → dispatch DENY). If not included, it remains an explicit carry into the subsequent acceptance review.

After E1, re-run an ADR-012 adoption gate review; do not auto-Accept.

## What this review did **not** execute

- No production RBAC / credential / cluster topology changes.
- No new Kargo/Argo installation or sandbox re-drive of the full GMUD→PRD live path.
- No new Backstage features, Delivery Workbench, Teams/CAB, F3 horizontal slices.
- No automatic rollback or break-glass implementation.
- No multi-cluster work.
- No ADR-012 status transition.
- No next Delivery implementation milestone.
- No rewrite of historical D0/D1/MVP evidence documents beyond current-state pointers in Delivery/ADR READMEs.

## Final report (compact)

```text
Verdict: REMAIN_PROPOSED
ADR-012 status: Proposed (unchanged)
Production rollout: NO-GO
12 gates: 3 PROVEN / 7 PARTIALLY_PROVEN / 2 NOT_PROVEN / 0 CONTRADICTED
Architecture blockers: none identified (direction intact; proof incomplete)
Production adoption blockers: Argo K8s least-privilege; Git writer/reconciler separation + control-object governance; ordinary-squad bypass resistance
Production hardening follow-ups: supply-chain admission; audit retention drill; callbacks/HA; auto-rollback automation; workbench UX; richer authz policy
Smallest next checkpoint: E1 — multi-activity binding and same-target concurrency
Docs commit SHA: fd19d8a4a34e4d8dd311fa47e7de8df2247a03bb
Implementation SHA(s) inspected: platform-devops-developer-portal@50ed1b0b72fdf3ddc5152fb47b22f20114238332 (feat/delivery-mvp-slice); gitops layout corroboration d0-gitops-sandbox@b196b7b67dd58f36f69969eb085348d4ad8bd509
STOP
```
