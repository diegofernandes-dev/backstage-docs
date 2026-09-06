# Eligibility window TOCTOU (bounded follow-up)

## Verdict

```text
eligibility-window-TOCTOU: PASS
```

Bounded follow-up under Accepted ADR-012. Does **not** authorize production rollout or a new Delivery milestone. Production remains **NO-GO**.

## Baselines

| Baseline | Value |
|---|---|
| Canonical docs at start (`backstage-docs` `origin/main`) | `4c55885deb6d138fe7c306d5de7d9024c2c70883` |
| Prior gate | [`adr-012-adoption-rereview.md`](./adr-012-adoption-rereview.md) — ADR-012 `ACCEPT`; named this follow-up |
| Implementation branch | `platform-devops-developer-portal` / `feat/delivery-mvp-slice` |
| Pre-change tip | `c2feb8ac122d952a560cb268d6b620265d57e22a` (E1) |
| Implementation SHA | `ee114cfac59d7aa91a190c188d9e634547534c8d` |

## What was proven

1. **`EligibilityService` consults `requestedWindow`** using ADR-009 half-open semantics `[startsAtUtc, endsAtUtc)`.
2. When authorization is `AUTHORIZED` but server time is outside the window → `DENY` / `OUTSIDE_WINDOW`.
3. Fresh evaluate near window close → `ALLOW`; evaluate again at `endsAtUtc` → `DENY` / `OUTSIDE_WINDOW` (TOCTOU).
4. Delivery re-checks eligibility on dispatch: prior `ALLOW` then `OUTSIDE_WINDOW` → `NOT_AUTHORIZED`, no provider promotion.

## Implementation `[code]`

| Change | Location |
|---|---|
| Window gate after AUTHORIZED | `packages/backend/src/modules/changeManagement/authorization/EligibilityService.ts` |
| Reason vocabulary | `OUTSIDE_WINDOW` on CM + Delivery `ExecutionEligibilityReason` |
| Clock injection for tests | `EligibilityServiceOptions.now` |
| Unit suite | `EligibilityService.test.ts` |
| Delivery dispatch TOCTOU | `DeliveryService.test.ts` — denies after prior ALLOW |

Pending authorization still returns `PENDING_AUTHORIZATION` (window not mis-reported as the primary reason).

## Regression tests

```text
CI=1 yarn test EligibilityService.test.ts DeliveryService.test.ts --watchAll=false
→ 25 passed (6 Eligibility window + 19 Delivery incl. window TOCTOU)
```

## Explicit non-goals (held)

- No production RBAC / credential hardening
- No new Delivery milestone
- No Argo SyncWindow mapping of GMUD windows
- No Change lifecycle `executing`/`completed` work

## Architecture implications

- Gate 5 (Eligibility / start) window clause is now evidenced for the ADR-009 contract.
- Production rollout remains separately **NO-GO** (authority blockers unchanged).
- No boundary regression: Change owns window; Delivery still consumes fresh eligibility over HTTP.

## Docs updated

- This record
- [`README.md`](./README.md) current-status pointer
- [`adr-012-adoption-rereview.md`](./adr-012-adoption-rereview.md) carried-gap note (window closed)

## Final report (compact)

```text
eligibility-window-TOCTOU: PASS
Docs baseline: 4c55885
Implementation pre: c2feb8a
Implementation SHA: ee114cfac59d7aa91a190c188d9e634547534c8d
Tests: 25 passed
Production rollout: NO_GO (unchanged)
STOP
```
