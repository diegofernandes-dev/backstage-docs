# F3.1.2a — Canonical Change Construction (implementation evidence)

- **Status:** IMPLEMENTED / PUBLISHED — architecture/implementation acceptance **ACCEPT** ([`f3-1-2a-architecture-implementation-acceptance.md`](./f3-1-2a-architecture-implementation-acceptance.md))
- **Date:** 2026-09-19
- **Canonical docs baseline (start):** `backstage-docs@b36c725b8355cf377ac5d4ac3f6afd7fd7f27778`
- **Authority:** ADR-006/007/008 + F3.1.2 ACCEPTED IMPLEMENTATION CONTRACT §7 / §20 A1–A4 + `prompts/f3-1-2a-canonical-change-implementation.md` (explicit launch)

This document records what was implemented and what was actually proven. Independent
acceptance closed F3.1.2a as the accepted implemented baseline. It does not authorize
F3.1.1c or F3.1.2b and introduces no authorization/ledger behavior.

## 1. Implementation baseline

| Item | Value |
|---|---|
| Repository / branch | `platform-devops-developer-portal` / `feat/ado-repo-governance` |
| ADO baseline **before** (HTTPS `git ls-remote` + ADO REST API) | `188d8e9cc43423f3644b3cacfb9849257838a583` |
| ADO SHA **after** (HTTPS `git ls-remote` + ADO REST API) | `ccee1e1676a2763e68880e5383ce1e5e48742843` |
| Parent | Exact `188d8e9…` — direct child |
| Publication | Plain fast-forward `188d8e9..ccee1e1` (no force) |
| Source drift on F3.1.2a surface | **NONE** |
| Delivery branch `feat/delivery-mvp-slice` | **Not used, not merged, not read from** |

Work was done in an isolated worktree (`implement/f3-1-2a-canonical-change`) off
the live tip `188d8e9`, not in the Delivery working tree.

**Publication transport note.** SSH to Azure DevOps failed in-session
(`remote: One or more errors occurred.`). The push used HTTPS with the existing
osxkeychain credential helper. Remote tip was confirmed independently via
`git ls-remote` and `az repos ref list` (`objectId` =
`ccee1e1676a2763e68880e5383ce1e5e48742843`). Repository `origin` SSH URL is
unchanged.

## 2. Changed paths

```
packages/backend/src/modules/changeManagement/ChangeManagementService.ts            (M)
packages/backend/src/modules/changeManagement/ChangeManagementService.test.ts        (M — A1/A2)
packages/backend/src/modules/changeManagement/ChangeManagementService.recovery.test.ts (M — A3)
```

3 files changed, 151 insertions, 2 deletions. Scope audit against `188d8e9`
found **only** these paths.

Untouched: authorization policy/selector/runtime, `AuthorizationLedgerRepository`,
idempotency repository semantics, `authorization_mode`, migrations, routes,
frontend, Delivery, RBAC, `changeManagementPlugin.ts`, providers.

## 3. Required production behavior implemented

**New logical submission**

1. Parse / reserve / validate / claim `changeId` (unchanged F2 flow).
2. If no pending index row exists: call `buildChange()` **exactly once**,
   `insertPending` with that object, then re-read the durable index row.
3. Pass `indexRecord.snapshot` into `finalizeCreate` (provider + finalize +
   idempotency complete). Never call `buildChange()` a second time.

**Recovery with an existing pending index**

1. Re-read the durable pending Change snapshot.
2. Reuse it — **do not** call `buildChange()`; **do not** regenerate
   `activityId` / `createdAt`.
3. Finalize using that recovered snapshot.

`POST /changes` remains legacy/F2: still reserves `LEGACY_PRE_F3`, still creates
**no** `AuthorizationRound`.

## 4. Mandatory proofs

| ID | Result | How proven |
|---|---|---|
| A1 — single build | **PASS** | `jest.spyOn(…buildChange)` expects exactly one call on a new logical create |
| A2 — provider/index equality | **PASS** | Deep `toEqual` between finalized index snapshot and `FakeChangeManagementProvider` stored Change, including `changeId`, `createdAt`, activity IDs/order, execution plan, requested window |
| A3 — recovery stability | **PASS** | Seeded pending index + pending idempotency; retry proves `buildChange` call count = 0 and original seeded `createdAt` / `activityId` values preserved into provider + finalized index |
| A4 — F2 regressions | **PASS** | Existing create / recovery / integration / list suites remain green |

## 5. Validation

| Gate | Result |
|---|---|
| Focused F3.1.2a + service suites | 4 suites / 56 tests PASS |
| Full Change Management module | 28 passed suites (+1 skipped live Catalog), **287 tests PASS**, 4 skipped |
| SQLite | PASS (in-process `better-sqlite3`) |
| PostgreSQL | Disposable `postgres:16-alpine`; `authorization/postgres.test.ts` executed and PASS; container removed |
| `yarn workspace backend lint` | PASS |
| `yarn workspace backend build` | PASS |
| TypeScript baseline | **Set-identical** to `188d8e9`: 5 pre-existing duplicate-Knex errors in `changeManagementPlugin.ts` at `(88,61)`, `(98,64)`, `(99,64)`, `(100,60)` `TS2345` and `(101,11)` `TS2322` — byte-identical error lines base vs candidate |
| `--forceExit` | **Not used** |

## 6. Hard non-goals confirmed

- AuthorizationRuntime / AuthorizationLedgerRepository / AuthorizationRound /
  ApprovalRequirement: **not wired**
- `authorization_mode` / `newSubmissionAuthorizationMode`: **unchanged**
- Idempotency repository semantics: **unchanged**
- Policy/selector files: **unchanged**
- Migrations: **none**
- F3.1.1c / F3.1.2b: **not started**

## 7. Gate

```text
F3.1.2a implementation: PASS
F3.1.2a architecture/implementation acceptance: ACCEPT
Single canonical Change build: PASS
Pending recovery reuses canonical snapshot: PASS
Provider/index snapshot equality: PASS
Authorization/ledger behavior added: NO
Migrations added: NO
Implementation commit: ccee1e1676a2763e68880e5383ce1e5e48742843
Remote ADO tip: ccee1e1676a2763e68880e5383ce1e5e48742843
F3.1.2a: CLOSED / ACCEPTED IMPLEMENTED BASELINE
F3.1.1c: still requires separate explicit authorization
F3.1.2b: NO-GO
```
