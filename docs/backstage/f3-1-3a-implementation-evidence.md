# F3.1.3a — Server-Authoritative Decision Command (implementation evidence)

- **Status:** IMPLEMENTATION PASS — not independently accepted; F3.1.3a is **not CLOSED**
- **Date:** 2026-09-21
- **Canonical docs baseline:** `backstage-docs@7a5bb442749c8185143a139c013811e3652bdd60`
- **Authority:** F3.1.3 ACCEPTED IMPLEMENTATION CONTRACT + `prompts/f3-1-3a-decision-command-implementation.md` (explicit launch)

This document records what was implemented and what was actually proven. It does
not close F3.1.3a, does not authorize F3.1.3b / F3.1.4 / F3.2, and does not
flip the committed runtime default to `LEDGER_REQUIRED`.

## 1. Implementation baseline

| Item | Value |
|---|---|
| Repository / branch | `platform-devops-developer-portal` / `feat/ado-repo-governance` |
| ADO baseline **before** (`az repos ref list`) | `22495229502dabf2d99588599a156d862c5114fa` |
| ADO SHA **after** (`az repos ref list`) | `6bad066d945d49feaf642313ec37467e2658dc3f` |
| Parent | Exact `2249522…` — direct child |
| Publication | Plain fast-forward `2249522..6bad066` (HTTPS; no force) |
| Source drift vs accepted SHA | **NONE** |
| Migrations | **NO** |
| Committed `newSubmissionAuthorizationMode` | `LEGACY_PRE_F3` |

**Publication transport note.** SSH `git fetch`/`push` to Azure DevOps failed
in-session (`remote: One or more errors occurred.`). Push used HTTPS with the
operator PAT. Remote tip was confirmed independently via `az repos ref list`
(`objectId` = `6bad066d945d49feaf642313ec37467e2658dc3f`). Repository `origin`
SSH URL is unchanged.

## 2. Source verification

Fetched `diegofernandes-dev/backstage-docs@main` at `7a5bb44` before editing.
Live ADO `feat/ado-repo-governance` tip was independently `2249522` via
`az repos ref list`. Working tree parent matched that SHA. Committed
`app-config.yaml` still pins `LEGACY_PRE_F3`. Laptop gitignored overlay
`app-config.local.yaml` remains `LEDGER_REQUIRED`.

No later source drift touched decision storage, ledger reads/writes, Change
lifecycle/index composition, permission/RBAC wiring, Catalog membership,
eligibility, or plugin routes.

## 3. Changed paths

```
packages/backend/config/rbac/rbac-policy.csv
packages/backend/src/modules/changeManagement/ChangeManagementService.ledgerSubmit.postgres.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.ledgerSubmit.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.ts
packages/backend/src/modules/changeManagement/architecture.test.ts
packages/backend/src/modules/changeManagement/authorization/AuthorizationLedgerRepository.ts
packages/backend/src/modules/changeManagement/authorization/DecisionCommandService.postgres.test.ts
packages/backend/src/modules/changeManagement/authorization/DecisionCommandService.test.ts
packages/backend/src/modules/changeManagement/authorization/DecisionCommandService.ts
packages/backend/src/modules/changeManagement/authorization/EligibilityService.test.ts
packages/backend/src/modules/changeManagement/authorization/EligibilityService.ts
packages/backend/src/modules/changeManagement/authorization/KnexAuthorizationLedgerRepository.test.ts
packages/backend/src/modules/changeManagement/authorization/KnexAuthorizationLedgerRepository.ts
packages/backend/src/modules/changeManagement/authorization/decisionMembership.test.ts
packages/backend/src/modules/changeManagement/authorization/decisionMembership.ts
packages/backend/src/modules/changeManagement/authorization/decisionTestHarness.ts
packages/backend/src/modules/changeManagement/authorization/sqlState.ts
packages/backend/src/modules/changeManagement/permissions.ts
packages/backend/src/modules/changeManagement/persistence/ChangeIndexRepository.ts
packages/backend/src/modules/changeManagement/persistence/KnexChangeIndexRepository.ts
packages/backend/src/modules/changeManagement/testHelpers.ts
packages/backend/src/modules/changeManagement/types.ts
packages/backend/src/modules/changeManagement/validation.ts
packages/backend/src/plugins/changeManagementPlugin.ts
plugins/change-management/src/api/ChangeManagementApi.ts
plugins/change-management/src/api/ChangeManagementClient.test.ts
plugins/change-management/src/api/ChangeManagementClient.ts
plugins/change-management/src/api/MockChangeManagementApi.ts
plugins/change-management/src/components/GmudCreatePage/GmudCreatePage.submit.test.tsx
plugins/change-management/src/components/GmudCreatePage/GmudCreatePage.test.tsx
plugins/change-management/src/components/GmudDetailPage/GmudDetailPage.test.tsx
plugins/change-management/src/components/GmudListPage/GmudListPage.test.tsx
plugins/change-management/src/components/GmudRouter/GmudRouter.test.tsx
plugins/change-management/src/model/types.test.ts
plugins/change-management/src/model/types.ts
```

35 files, `+2880 / −163`. No migrations. No approve/reject UI. Delivery routes
were not modified. `app-config.yaml` was not modified.

## 4. Behavior implemented

HTTP contract:

```text
POST /api/change-management/changes/:changeId/rounds/:roundNumber/requirements/:requirementId/decisions
```

User credentials only. `Idempotency-Key` required (missing/blank →
`VALIDATION_ERROR` / 400). First commit HTTP 201; exact replay HTTP 200 with
the original `decisionId` / `decidedAt` / `commandHash` and no duplicate audit.

Canonical command hash:

```text
sha256Canonical({
  changeId, roundNumber, requirementId, outcome,
  reason: reason ?? null,
  comment: comment ?? null,
  cabMeetingRef: cabMeetingRef ?? null,
})
```

Authority:

- `individual`: `change-management.change.authorization.decide` **and** exact
  `actor.userEntityRef == principalSnapshot.resolvedPrincipalRef`.
- `cab` / `authority`: dedicated `change-management.change.authorization.cab.record`
  **and** live Catalog `memberOf` (relations + spec, prefix-agnostic) via
  `decisionMembership.ts` (not under `authorization/selector/`).
- `platform_admin` does **not** receive `cab.record`. CSV binds
  `role:default/change_cab_recorder` to
  `group:default/cloud_azure_devops_platform_devops`.

Transaction:

- Catalog / permission / domain proof **before** the writer transaction.
- One caller-owned Knex transaction.
- `change_index FOR UPDATE` (PostgreSQL; SQLite still serializes via the same
  lock method).
- trx-aware ledger reads (`findRound`, `findCurrentRound`, `listRequirements`,
  `listAuditEvents`, `findDecisionByRequirement`, `findDecisionByIdempotency`).
- Immutable `ApprovalDecision` + `change.authorization.decision_recorded`.
- Derived `AuthorizationEvaluation`; at most one
  `authorization_reached` or `round_rejected` milestone.
- Mandatory pre rejection projects `change_index.status = rejected`.
- Unique-loser path rolls back then re-observes under a new lock.
- Post-execution requirements without completion evidence fail closed
  (`CONFLICT` / `execution_completion_required`) with zero writes.

Reads:

- `GET /changes/:changeId` overlays **status only** from `change_index` onto
  the provider detail.
- Eligibility no longer fabricates a sandbox Round. Missing-round /
  `LEDGER_REQUIRED` without a current round is `NO_LEDGER_ROUND`.

Frontend: `ChangeStatus` includes `rejected` (`Rejeitada`). Optional
`recordDecision` client exists for product/API tooling. No approve/reject
buttons and no authorization read-model UI.

## 5. Permission / RBAC bindings

| Permission | Action | Bound to |
|---|---|---|
| `change-management.change.authorization.decide` | update | `role:default/contributor`, `role:default/platform_admin` |
| `change-management.change.authorization.cab.record` | update | `role:default/change_cab_recorder` only |
| `change-management.change.resubmit` | — | **absent** |

Startup with `policyFileReload: true` loaded the new CSV policies, including
`g, group:default/cloud_azure_devops_platform_devops, role:default/change_cab_recorder`.

## 6. Automated proofs

| Gate | Result |
|---|---|
| Change Management SQLite (`--testPathPatterns changeManagement`) | **PASS** — 36 suites passed, 1 skipped (`CatalogPrincipalResolver.live` without live Catalog env), 389 tests passed, 4 skipped |
| PostgreSQL D1–D6 + F3.1.2 submission/ledger postgres | **PASS** — 3 suites / 15 tests |
| D1 exact replay | **PASS** |
| D2 concurrent duplicate approval | **PASS** |
| D3 approve vs reject concurrency | **PASS** |
| D4 authorization milestone concurrency | **PASS** |
| D5 membership / Catalog unavailable | **PASS** |
| D6 rejection atomicity / lifecycle | **PASS** |
| GMUD frontend | **PASS** — 13 suites / 60 tests |
| Catalog + Deployments tab (`catalogEntityTabs\|App.test\|Sidebar`) | **PASS** — 7 suites / 22 tests |
| Delivery backend | **PASS** — 1 suite / 26 tests |
| Lint (`yarn lint:all`) | **PASS** |
| Build (`yarn build:all`) | **PASS** |
| TypeScript baseline | **UNCHANGED** — exactly 5 historical dual-package Knex errors in `changeManagementPlugin.ts`; zero new errors |

PostgreSQL runtime used for D1–D6: disposable Homebrew **PostgreSQL 15** on
`127.0.0.1:55432` (`CHANGE_MANAGEMENT_TEST_POSTGRES_URL=postgresql://diegofernandes@127.0.0.1:55432/change_management_test`).
Docker/Rancher were not running, so PostgreSQL 16 was not started. Authoritative
concurrency proofs D1–D6 still executed against PostgreSQL, not SQLite.

## 7. Live non-production product proof

Isolated operator-laptop `yarn start`. Loaded
`app-config.yaml, app-config.local.yaml`. Startup:

```text
newSubmissionAuthorizationMode LEDGER_REQUIRED
policy default-change-authorization@2026-09-19.1
selector bundle selector-bundle-dev@2026-09-07.1
contentDigest 6a0c2fb4f30037603848cb72700e77d60f68cf35a25e6db055c3690780d28b94
```

Authenticated product identity: `user:default/diego.fernandes_outlook.com`.
Live Catalog membership (not fabricated):
`group:default/cloud_azure_devops_platform_devops` via both `relations.memberOf`
and `spec.memberOf`.

### Happy path — `CHG-2026-000003`

Preferred accepted target still had Round 1 primary+CAB and **zero** F3
decisions before this checkpoint.

| Step | Result |
|---|---|
| Before eligibility | `DENY` / `PENDING_AUTHORIZATION` |
| Primary `approved` | HTTP **201**, evaluation `PENDING`, lifecycle `submitted`, evidence `kind=individual` |
| CAB `approved` by current Catalog member with `cab.record` | HTTP **201**, evaluation `AUTHORIZED`, lifecycle still `submitted`, `actingAuthorityRef=group:default/cloud_azure_devops_platform_devops`, evidence `kind=authority_membership` / `membershipSource=catalog` |
| Exact primary replay | HTTP **200**, same `decisionId` `5b64d801-cdef-4288-9523-07420c838033` and `decidedAt` `2026-09-21T01:23:53.158Z` |
| After eligibility | `DENY` / `OUTSIDE_WINDOW` (window `[2026-09-29T01:00:00.000Z, 2026-09-29T02:00:00.000Z)`) |
| Ledger | 2 decisions, 2 `decision_recorded`, **exactly one** `authorization_reached`, still **one** Round 1, status `submitted` |

Missing `Idempotency-Key` against the same route: HTTP **400**
`VALIDATION_ERROR` / `idempotencyKey=required`.

### Rejection path — disposable `CHG-2026-000005`

Created live under `LEDGER_REQUIRED`. Did **not** reject `CHG-2026-000003`.

| Step | Result |
|---|---|
| Create | HTTP 201, `CHG-2026-000005`, status `submitted` |
| Reject primary with reason | HTTP **201**, evaluation `REJECTED`, `changeStatus=rejected` |
| Detail overlay | `rejected` |
| List | `Rejeitada` |
| Eligibility | `DENY` / `REJECTED` |
| Ledger | 1 decision, 1 `decision_recorded`, 1 `round_rejected`, 1 Round, **no** `authorization_reached` |
| `POST .../resubmit` | HTTP **404** |
| `POST .../rounds` | HTTP **404** |

### Eligibility safety

`CHG-2026-000002` (`LEGACY_PRE_F3`, no Round): `DENY` / `NO_LEDGER_ROUND`.
No sandbox Round was fabricated.

### Product surfaces

| Surface | Result |
|---|---|
| `/gmud` | reachable; `CHG-2026-000003` Submetida; `CHG-2026-000005` Rejeitada |
| `/gmud/CHG-2026-000003` | reachable; Submetida; no approve/reject buttons |
| `/gmud/CHG-2026-000005` | reachable; Rejeitada; no approve/reject buttons |
| Catalog `/` | reachable |
| Deployments tab | `/catalog/default/component/idp-showcase-api/deployments` visible |
| Delivery reads | HTTP 200 on namespaced release-candidates / deployments / events |

No F3.1.4 authorization UI. No CAB Workbench. No membership was fabricated.

## 8. Explicit non-goals proven absent

- No `change-management.change.resubmit`
- No Round 2 / `DevelopmentProvider.replaceCurrent`
- No migrations
- No committed `LEDGER_REQUIRED` default
- No Teams / execution lifecycle / production cutover
- No Kargo/Argo/GitOps mutation

## 9. Next gate

Independent **F3.1.3a architecture/implementation acceptance review**.

```text
F3.1.3a implementation: PASS
F3.1.3a CLOSED: NO
F3.1.3b prompt authoring/implementation: NO-GO
F3.1.4/F3.2: NO-GO
Production cutover: NOT AUTHORIZED
```
