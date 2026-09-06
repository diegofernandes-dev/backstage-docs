# E1 — Multi-activity binding and same-target concurrency

## Verdict

```text
E1 verdict: PASS
```

ADR-012 status is **unchanged (Proposed)**. This checkpoint does **not** accept ADR-012 and does **not** authorize production rollout or the next Delivery milestone.

## Baselines

| Baseline | Value |
|---|---|
| Canonical docs at E1 start (`diegofernandes-dev/backstage-docs` `origin/main`) | `d0eed6ac38f1e5229cff2e0e9913305e28cd9d7d` |
| Execution contract | [`prompts/e1-multi-activity-concurrency.md`](../../prompts/e1-multi-activity-concurrency.md) |
| Reproducibility contract | [`prompts/e1-commit-and-adr012-rereview.md`](../../prompts/e1-commit-and-adr012-rereview.md) |
| Implementation baseline (pre-E1 tip) | `platform-devops-developer-portal` / `feat/delivery-mvp-slice` / `50ed1b0b72fdf3ddc5152fb47b22f20114238332` |
| Implementation after E1 (committed) | `platform-devops-developer-portal` / `feat/delivery-mvp-slice` / `c2feb8ac122d952a560cb268d6b620265d57e22a` |
| Prior gate | [`adr-012-adoption-review.md`](./adr-012-adoption-review.md) — `REMAIN_PROPOSED` / production **NO-GO** |
| Subsequent re-review | [`adr-012-adoption-rereview.md`](./adr-012-adoption-rereview.md) |

## What E1 authorized

Prove three architecture claims only:

1. Activity-scoped Delivery binding (`changeId` + `activityId` + release fingerprint + target).
2. Multi-activity completion boundary (one successful deployment does not complete the Change).
3. Deterministic same-target concurrency for distinct release material.

## Implementation summary `[code]`

| Change | Location |
|---|---|
| `ChangeBinding.activityId` required | `packages/backend/src/modules/delivery/types.ts` |
| Migration `activity_id` | `packages/backend/migrations/delivery/20260906200000_binding_activity_id.cjs` |
| Membership validation via user-on-behalf `GET /changes/:changeId` | `packages/backend/src/modules/delivery/changeLookup/ChangeActivityClient.ts` |
| Bind API requires `{ changeId, activityId }` | `packages/backend/src/plugins/deliveryPlugin.ts` |
| Same-target busy claim (`claimDispatch`) | `DeliveryService.dispatch` + Knex/Fake repositories |
| Deployments tab bind UX shows activity identity + non-completion hint | `packages/app/src/modules/catalogEntityTabs/DeploymentsTab.tsx` |
| Regression suite | `DeliveryService.test.ts` — **18** tests passing (was 15; +E1.1/E1.2/E1.3) |

Provider/Kargo/Argo identifiers remain on Delivery request/target/projection only. Canonical Change types were **not** extended with deployment/provider IDs. No ADR-008 activity lifecycle fields were added.

---

## E1.1 — Activity-scoped Delivery binding

### Evidence

- **[code]** `bindChange` validates `activityId ∈ change.executionPlan.activities` through `ChangeActivityClient` (HTTP, user-on-behalf ACL preserved).
- **[code]** Binding row stores `change_id` + `activity_id`; release fingerprint and target remain correlated via `DeploymentRequest` → `ReleaseCandidate` + `DeploymentTarget`.
- **[code]** Tests:
  - valid `changeId + activityId` binding persists;
  - nonexistent activity / activity of another Change → `VALIDATION_ERROR`;
  - rebinding different activity → `CONFLICT` (immutable binding);
  - identical rebind is idempotent.

### Negative checks

| Check | Result |
|---|---|
| `activityId` not on bound Change | Fail closed (`VALIDATION_ERROR`) |
| Activity belonging only to another Change | Fail closed when asserted against wrong Change |
| Silent reuse of prior binding for different activity | Rejected (`CONFLICT`) |
| Provider IDs leaked into canonical Change | Not introduced |

### Result

**PASS**

---

## E1.2 — Multi-activity completion boundary

### Evidence

- **[code]** Harness: multi-activity Change plan (`act-deploy`, `act-manual`); bind only `act-deploy`; dispatch ALLOW; projection `Succeeded` → Delivery request `succeeded`.
- **[code]** Asserted before/after: Change `status` remains `submitted`; both activities remain plan facts; activities have no `status`/`completed`/`state` fields.
- **[code]** `refreshProjection` mirrors Delivery terminal status only — does not call Change Management write/completion APIs.
- **[ui]** GMUD detail (`GmudDetailPage`) renders `STATUS_LABELS.submitted` ("Submetida") and lists activities as plan items without per-activity completion chips. Deployments tab, when request is `succeeded`, shows: bound activity identity plus explicit copy that deployment success does **not** complete the Change.

### Change state before/after

```text
before: status=submitted; activities=[act-deploy, act-manual] (plan facts)
after:  status=submitted; activities=[act-deploy, act-manual] (unchanged)
Delivery request: succeeded; binding.activityId=act-deploy
```

### Result

**PASS** — explicit boundary demonstrated; not safe-by-omission alone.

---

## E1.3 — Same-target concurrency

### Concurrent inputs `[code]`

Two distinct ReleaseCandidates (`rc-a` / `rc-b`) → same PRD `DeploymentTarget` → two governed requests bound and dispatched via `Promise.all`.

### Observed behavior

```text
exactly one winner (status=dispatched, provider.dispatch ×1)
+ second request safely rejected (CONFLICT / target busy)
+ loser remains awaiting_authorization (no provider promotion)
```

Enforced by `claimDispatch`: target busy check + CAS inside a short transaction / synchronous fake claim. No force-push path; no silent double mutation; no lock-manager subsystem.

### Final desired/runtime state

- **[code]** At most one `dispatched` request per target at a time.
- **[live-system]** Sandbox cluster reachable (`d1-prd` Argo Application Synced/Healthy; prior Kargo promotions visible in `d1-sandbox`). **No new live dual-Freight PRD mutation was executed** in this checkpoint — concurrency invariant is closed at the Delivery dispatch gate before provider create (sufficient for the E1 claim of deterministic safe behavior without silent double mutation).

### Result

**PASS**

---

## Backstage UI/browser verification

| Item | Observation |
|---|---|
| Running UI | Frontend `http://localhost:3000` → HTTP 200; `yarn start` active |
| Screens | Component → Deployments tab (source + live permission evaluations for `delivery.deployment.read` in running backend logs); GMUD detail activity list |
| Activity identity | Deployments bind form requires Change ID + Activity ID; bound view shows both |
| Change completion implication | Succeeded deployment shows explicit non-completion copy; GMUD status remains whole-Change `submitted` only |
| Concurrency UX | CONFLICT surfaces as dispatch error message (no second successful mutation presented) |
| Narrow UI fix | Activity ID field + bound activity display + non-completion hint only |

Browser screenshots were not captured during the original E1 execution pass; post-commit re-review captured Deployments/Platform/GMUD observations (see Committed-SHA verification below).

---

## Optional window TOCTOU

```text
performed / not performed: not performed
reason: EligibilityService does not consult requestedWindow; adding window TOCTOU would expand beyond E1 into eligibility framework work. Carried to subsequent ADR-012 adoption review.
```

---

## Regression tests

```text
yarn test src/modules/delivery/DeliveryService.test.ts
→ 18 passed (PRD gate, E1.1, E1.2, E1.3, idempotency, projection, listDeployments)
```

Existing Delivery / demo-hardening invariants preserved.

---

## Committed-SHA verification (reproducibility)

| Step | Result |
|---|---|
| Isolation | Pathspec commit of Delivery/E1 files only; brownfield/adoption WIP left uncommitted |
| Pre-commit | `yarn test src/modules/delivery/DeliveryService.test.ts` → **18 passed**; `packages/app` `catalogEntityTabs/index.test.ts` → **4 passed** |
| Commit | `c2feb8ac122d952a560cb268d6b620265d57e22a` — `feat(delivery): E1 activity-scoped binding and same-target claimDispatch` |
| Post-commit (from committed SHA) | Same suites → **18 passed** / **4 passed** |
| E1 working-tree scope | Clean for Delivery/E1 paths after commit |

### UI/browser (post-commit, supplemental)

| Item | Observation |
|---|---|
| Running UI | `http://localhost:3000` HTTP 200; Component → Deployments for `idp-showcase-api` |
| Bound Change / non-completion | PRD shows bound `CHG-2026-000001` and copy that deployment success does not complete the Change (legacy row may show empty Activity until rebound) |
| Platform tab | Loads without regression (Platform adoption facts) |
| GMUD detail | `CHG-2026-000001` status **Submetida**; activities listed as plan items |
| Screenshots | Captured in agent browser session during re-review (Deployments, Platform, GMUD) |

## Residual E1 gaps

1. ~~E1 implementation delta not yet committed~~ — closed at `c2feb8a`.
2. Live dual-Freight Git/Kargo PRD mutation not re-driven (invariant enforced pre-provider).
3. Window TOCTOU still open (optional adjacent; omitted) — carried into ADR re-review classification.
4. Change lifecycle still `submitted`-only — ADR-009 `executing`/`completed` milestones remain a later Change Management concern; E1 proves Delivery does not falsely complete multi-activity Changes.

---

## Architecture implications

- ADR-008 boundary held: activities remain plan facts; no workflow-task drift.
- ADR-012 binding vocabulary closed for `activityId` on Delivery-owned `ChangeBinding`.
- Same-target exclusive active mutation is now a Delivery invariant (busy target → `CONFLICT`).
- No architecture rework signal; no need for a new ADR from this checkpoint.

---

## Recommended next gate

Independent **ADR-012 adoption re-review** was authorized and executed — see [`adr-012-adoption-rereview.md`](./adr-012-adoption-rereview.md). Do **not** start production hardening or the next Delivery milestone from the E1 handoff alone.

---

## Docs updated

- Created this record: `docs/delivery/e1-multi-activity-concurrency.md`
- Updated with committed SHA `c2feb8a` and committed-SHA verification
- Updated `docs/delivery/README.md` current-state pointer
- ADR-012 status updated by the subsequent re-review (`ACCEPT`) — see [`adr-012-adoption-rereview.md`](./adr-012-adoption-rereview.md)

---

## Final report (compact)

```text
E1 verdict: PASS
Docs baseline SHA (E1 start): d0eed6ac38f1e5229cff2e0e9913305e28cd9d7d
Implementation baseline SHA: 50ed1b0b72fdf3ddc5152fb47b22f20114238332
E1 committed SHA: c2feb8ac122d952a560cb268d6b620265d57e22a

E1.1 activity-scoped binding: PASS
E1.2 multi-activity completion boundary: PASS
E1.3 same-target concurrency: PASS (Delivery claimDispatch; live dual-Freight not re-driven)

Backstage UI/browser: Deployments bound Change + non-completion copy; Platform OK; GMUD Submetida
Optional window TOCTOU: not performed (EligibilityService has no window consult)
Regression tests (committed SHA): 18 passed (+ 4 tab loader)
Residual: window TOCTOU; ADR-009 lifecycle still deferred; live dual-Freight not re-driven
Architecture implications: boundaries held; no REWORK_SIGNAL
Recommended next gate: see adr-012-adoption-rereview.md
Docs updated: e1-multi-activity-concurrency.md committed-SHA evidence
STOP
```
