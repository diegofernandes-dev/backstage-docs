# F3.1.2b — Ledger-Governed Submission Integration (implementation evidence)

- **Status:** CLOSED / ACCEPTED IMPLEMENTED BASELINE — architecture/implementation acceptance **ACCEPT** ([`f3-1-2b-architecture-implementation-acceptance.md`](./f3-1-2b-architecture-implementation-acceptance.md))
- **Date:** 2026-09-20
- **Canonical docs baseline (start):** `backstage-docs@b3ea5eee58a68758d15233f311090555ac1f758c`
- **Authority:** F3.1.2 ACCEPTED IMPLEMENTATION CONTRACT + `prompts/f3-1-2b-ledger-submission-implementation.md` (explicit launch)

This document records what was implemented and what was actually proven. It does
not close F3.1.2b by itself, does not flip the committed runtime default to
`LEDGER_REQUIRED`, and does not authorize F3.1.3, F3.1.4, F3.2, or operational
cutover.

## 1. Implementation baseline

| Item | Value |
|---|---|
| Repository / branch | `platform-devops-developer-portal` / `feat/ado-repo-governance` |
| ADO baseline **before** (`az repos ref list`) | `f48dc825ab5d1d16fafc3f70ef772d28613df1a0` (accepted product-convergence tip) |
| ADO SHA **after** (`az repos ref list`) | `22495229502dabf2d99588599a156d862c5114fa` |
| Parent | Exact `f48dc82…` — direct child |
| Publication | Plain fast-forward `f48dc82..2249522` (HTTPS; no force) |
| Known product-convergence drift reconciled | **YES** |
| Source drift vs accepted plan | **EXPECTED_CONVERGENCE_ONLY** |
| Migrations | **NO** |

**Publication transport note.** SSH `git fetch`/`push` to Azure DevOps failed
in-session (`remote: One or more errors occurred.`). Push used HTTPS. Remote tip
was confirmed independently via `az repos ref list` (`objectId` =
`22495229502dabf2d99588599a156d862c5114fa`). Repository `origin` SSH URL is
unchanged.

## 2. Known drift reconciled

Complete diff `3b302ab..f48dc82` was inspected before editing. The only post-F3.1.1c
commit is product convergence `f48dc82`. It added the execution-eligibility read
route and Delivery surfaces and **did not** change `ChangeManagementService`
create/idempotency/ledger submission behavior.

F3.1.2b preserved:

- Catalog Component Deployments tab
- Delivery backend plugin/module
- `GET /changes/:changeId/execution-eligibility`
- Delivery RBAC/config
- `api:catalog/delivery` distinct extension identity
- additive change-management type/label exports
- Catalog and GMUD frontend behavior
- committed `newSubmissionAuthorizationMode: LEGACY_PRE_F3`

## 3. Changed paths

```
app-config.yaml
packages/backend/src/plugins/changeManagementPlugin.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.ts
packages/backend/src/modules/changeManagement/testHelpers.ts
packages/backend/src/modules/changeManagement/architecture.test.ts
packages/backend/src/modules/changeManagement/authorization/ledgerSubmission.ts
packages/backend/src/modules/changeManagement/authorization/ledgerSubmission.test.ts
packages/backend/src/modules/changeManagement/authorization/policy/registry.ts
packages/backend/src/modules/changeManagement/authorization/policy/registry.test.ts
packages/backend/src/modules/changeManagement/authorization/selector/config.ts
packages/backend/src/modules/changeManagement/authorization/selector/config.test.ts
packages/backend/src/modules/changeManagement/authorization/selector/types.ts
packages/backend/src/modules/changeManagement/authorization/selector/bootstrapAuthorization.ts
packages/backend/src/modules/changeManagement/authorization/selector/bootstrapAuthorization.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.recovery.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.integration.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.list.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.ledgerSubmit.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.ledgerSubmit.postgres.test.ts
```

20 files, `+2299 / −41`. No migrations. No frontend product changes. Delivery
domain/routes were not modified.

## 4. Behavior implemented

- Stored-mode-wins orchestration: config mode applies only to genuinely new
  reservations; existing reservations omit requested mode; repository explicit
  requested-mode mismatch remains `CONFLICT`.
- Committed default remains `changeManagement.authorization.newSubmissionAuthorizationMode: LEGACY_PRE_F3`.
- `LEDGER_REQUIRED` is exercised only through controlled test wiring.
- Startup `AuthorizationRuntime` is injected into `ChangeManagementService`
  with `KnexAuthorizationLedgerRepository` on the existing database.
- CAB-safe policy `default-change-authorization@2026-09-19.1` is evaluated
  exactly once per attempt that still lacks Round 1.
- `requirementId = requirementRole` with fail-closed duplicate-role validation
  at policy registration.
- Emergency same-person A/B is `CONFLICT` / `details.reason=separation_of_duty`
  with no Round, requirement, audit, finalize, or idempotency completion.
- DevelopmentProvider path uses one caller-owned Knex transaction for provider
  create + Round 1 + requirements + required audit + index.finalize +
  idempotency.complete.
- External provider create remains outside the platform transaction and
  converges by canonical `changeId`.
- Healthy Round-1 uniqueness/serialization losers roll back, re-read outside
  the transaction, and return `{ changeId, status: submitted }` when coherent.

Lifecycle remains `submitted`. F3.1.2b materializes authorization requirements
only.

## 5. Mandatory rollback correctness gate

Before rollback to **any binary that predates F3.1.2 ledger submission support**,
operators must run:

```sql
SELECT COUNT(*)
FROM change_idempotency
WHERE authorization_mode = 'LEDGER_REQUIRED'
  AND state = 'pending';
```

Required result:

```text
0
```

If nonzero, rollback to the older binary is **FORBIDDEN** until those logical
submissions are completed/drained through supported F3.1.2 recovery.

**B1 — verified old-binary threat model.** At pre-F3.1.2 source (`188d8e9` /
product tip `f48dc82` before this commit), `ChangeManagementService.createChange`
called `reserve` without an explicit mode, copied `reserved.authorizationMode`,
and finalized through the legacy provider/index/complete path with no Round.
A pending `LEDGER_REQUIRED` reservation could therefore become discoverable
without AuthorizationRound 1 on that older binary. That is why the zero-count
rule above is mandatory.

## 6. Tests and gates

| Gate | Result |
|---|---|
| Focused F3.1.2b SQLite ledgerSubmit | PASS |
| Focused helper/registry/config/architecture | PASS |
| Full Change Management module (SQLite + disposable PostgreSQL 15) | PASS — 33 suites / 348 tests (4 skipped, pre-existing) |
| PostgreSQL C1/C2 authoritative concurrency | PASS |
| PostgreSQL C3 different-payload | PASS |
| SQLite C1/C3/C4 | PASS |
| F3.1.2a recovery / canonical snapshot regressions | PASS |
| F3.1.1 policy/selector/publication regressions | PASS |
| Deployments frontend (`catalogEntityTabs`) + Delivery backend + GMUD frontend | PASS — 19 suites / 104 tests |
| Lint (`yarn lint:all`) | PASS |
| `yarn build:all` | PASS |
| TypeScript baseline | IDENTICAL — exactly 5 historical dual-package Knex errors in `changeManagementPlugin.ts`; zero new errors |

PostgreSQL proof used a disposable local PostgreSQL 15 on `127.0.0.1:55432`
(`CHANGE_MANAGEMENT_TEST_POSTGRES_URL=postgresql://postgres@127.0.0.1:55432/change_management_test`).
No production database was used.

## 7. Explicitly not done

- Operational cutover of committed `newSubmissionAuthorizationMode` to `LEDGER_REQUIRED`
- F3.1.3 decision commands
- F3.1.4 authorization read/RBAC
- F3.2 CAB autonomy / skipCab / grants / CAB Workbench
- Teams
- Production rollout
- Migrations
- Delivery/Kargo/Argo redesign

## 8. Next gate

Independent F3.1.2b architecture/implementation acceptance review completed with **ACCEPT**.

F3.1.2b is CLOSED / ACCEPTED IMPLEMENTED BASELINE at ADO `2249522`.
F3.1.2 is CLOSED / ACCEPTED IMPLEMENTED SUBMISSION BASELINE.

Operational LEDGER_REQUIRED cutover remains a later explicit activation step. Recommended next gate: author a narrow LEDGER_REQUIRED activation/cutover checkpoint before F3.1.3 implementation.
