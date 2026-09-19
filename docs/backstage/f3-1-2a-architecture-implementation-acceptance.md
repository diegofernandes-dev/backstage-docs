# F3.1.2a — Canonical Change Architecture / Implementation Acceptance

- **Status:** CLOSED / ACCEPTED IMPLEMENTED BASELINE
- **Date:** 2026-09-19
- **Verdict:** `ACCEPT`
- **Implementation:** Azure DevOps `platform-devops-developer-portal@ccee1e1676a2763e68880e5383ce1e5e48742843`, branch `feat/ado-repo-governance`
- **Expected parent (verified):** accepted F3.1.1b baseline `188d8e9cc43423f3644b3cacfb9849257838a583`
- **Documentation review baseline:** `backstage-docs@a3c5c8b8299eb5ca5ba22ab13f8443bc52d9abcb` (`origin/main` at review start)
- **Authority:** F3.1.2 ACCEPTED IMPLEMENTATION CONTRACT §7 / §20 A1–A4 + [`f3-1-2-final-architecture-rereview.md`](./f3-1-2-final-architecture-rereview.md) + [`f3-1-2a-implementation-evidence.md`](./f3-1-2a-implementation-evidence.md) + `prompts/f3-1-2a-canonical-change-implementation.md` + this independent review

```text
F3.1.2a architecture/implementation acceptance: ACCEPT
F3.1.2a: CLOSED / ACCEPTED IMPLEMENTED BASELINE
Accepted ADO SHA: ccee1e1676a2763e68880e5383ce1e5e48742843
F3.1.1c implementation: still requires separate explicit authorization
F3.1.2b implementation: NO-GO until F3.1.1c is implemented and independently accepted
```

---

## 1. Status / verdict

F3.1.2a architecture/implementation acceptance: **ACCEPT**.

All mandatory gates **G1–G8 PASS**. The published slice at `ccee1e1` closes the double-`buildChange()` defect without authorization/ledger scope leakage. Acceptance does **not** authorize F3.1.1c, F3.1.2b, Round creation, or `LEDGER_REQUIRED` cutover.

ADO implementation was **not modified** by this review.

---

## 2. Reviewed baselines

### Docs baseline

| Item | Value |
|---|---|
| Repo | `diegofernandes-dev/backstage-docs` |
| Branch | `main` |
| `origin/main` SHA at review | `a3c5c8b8299eb5ca5ba22ab13f8443bc52d9abcb` |
| Prompt | `prompts/f3-1-2a-architecture-implementation-acceptance.md` |

Canonical authority files read: F3.1.2 plan, final architecture re-review, F3.1.2a implementation evidence, F3.1.2a implementation prompt, current-state, implementation-progress, prompts/README.

### Independent ADO source verification: **YES**

| Check | Result |
|---|---|
| Fresh HTTPS `git ls-remote` tip | `ccee1e1676a2763e68880e5383ce1e5e48742843` (`refs/heads/feat/ado-repo-governance`) |
| Local fetch of `feat/ado-repo-governance` | tip equals `ccee1e1` |
| Parent of reviewed commit | **YES** — exact `188d8e9cc43423f3644b3cacfb9849257838a583` |
| Complete diff `188d8e9..ccee1e1` inspected | **YES** |
| Isolated worktree at exact SHA | `/tmp/f312a-acceptance-review` |
| Later tip drift beyond `ccee1e1` | **NONE** |

---

## 3. Candidate lineage and scope

```text
188d8e9 (F3.1.1b ACCEPTED)
  └─ ccee1e1 (F3.1.2a candidate — this review)
```

Diff `188d8e9..ccee1e1`: **3 files**, `+151 / −2`.

```text
packages/backend/src/modules/changeManagement/ChangeManagementService.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.recovery.test.ts
```

Matches the acceptance prompt's expected production/test surface exactly. No unexpected production scope expansion.

**Confirmed untouched:** authorization policy/selector/runtime, ledger repository, idempotency repository semantics, `authorization_mode` / `newSubmissionAuthorizationMode`, migrations, routes, frontend, Delivery, RBAC, plugin wiring beyond the service method under review.

---

## 4. Mandatory gate matrix (G1–G8)

| Gate | Result | Concise basis |
|---|---|---|
| **G1** Single canonical construction | **PASS** | `buildChange()` runs only inside `if (!indexRecord)` before `insertPending`. Finalize uses `indexRecord.snapshot`. A1 spy proves exactly one call on a new logical create. |
| **G2** Durable snapshot recovery authority | **PASS** | Existing pending index skips the build block; finalize always reads `indexRecord.snapshot`. A3 spy proves `buildChange` call count = 0 and seeded `createdAt` / `activityId` preserved into provider + finalized index. |
| **G3** Provider/index structural identity | **PASS** | A1/A2 use deep `toEqual` over the whole Change snapshot (plus explicit `createdAt`, execution plan, activity IDs, requested window). Not field-cherry-picked. |
| **G4** F2 behavior preserved | **PASS** | Service still defaults/reserves `LEGACY_PRE_F3`; no Round path; architecture guard still forbids `createRound(` in service source; create/recovery/integration/list suites green. |
| **G5** Hard non-goals | **PASS** | Diff contains no AuthorizationRuntime/ledger wiring, Round/requirement creation, mode cutover, policy/selector edits, CAB/RBAC/route/frontend/Delivery/migration. |
| **G6** Failure/recovery semantics | **PASS** | After pending insert, durable snapshot is re-read and reused; concurrent insert race still re-reads winner; no second construction that can diverge provider vs index; no new silent swallow/recovery loop. |
| **G7** Tests and proof quality | **PASS** | Independent re-run at exact SHA (see §5). Do not rely solely on implementation evidence. |
| **G8** Diff minimality | **PASS** | Production delta is the smallest reuse fix (~20 lines / 2 deletions). Tests target A1–A3 invariants only; no new framework. |

**Gates: 8 / 8 PASS.**

---

## 5. Independent proof re-run

Executed in detached worktree at `ccee1e1` (linked to an existing `node_modules` tree; no ADO source edits):

| Gate | Result |
|---|---|
| Focused A1/A2/A3 suites (`ChangeManagementService.test.ts` + `.recovery.test.ts`) | **30/30 PASS** |
| Full Change Management module + disposable Postgres 16 | **28 suites PASS** (1 skipped live Catalog), **287 tests PASS**, 4 skipped |
| SQLite path | **PASS** (in-process) |
| PostgreSQL | **PASS** — disposable `postgres:16-alpine`; `authorization/postgres.test.ts` executed; container removed |
| `yarn workspace backend lint` | **PASS** |
| `yarn workspace backend build` | **PASS** |
| TypeScript baseline vs `188d8e9` | **SET-IDENTICAL** — five pre-existing `changeManagementPlugin.ts` Knex duplicate-type errors at `(88,61)`, `(98,64)`, `(99,64)`, `(100,60)` `TS2345` and `(101,11)` `TS2322` |
| `--forceExit` | **Not used** |

---

## 6. Decision

```text
F3.1.2a architecture/implementation acceptance: ACCEPT
F3.1.2a: CLOSED / ACCEPTED IMPLEMENTED BASELINE
Accepted ADO SHA: ccee1e1676a2763e68880e5383ce1e5e48742843
```

No material scope leakage or source contradiction remains against the accepted F3.1.2a contract.

---

## 7. Final report

```text
Docs baseline reviewed: a3c5c8b8299eb5ca5ba22ab13f8443bc52d9abcb
ADO commit reviewed: ccee1e1676a2763e68880e5383ce1e5e48742843
Parent verified: YES
Independent ADO source verification: YES
Single canonical build gate: PASS
Durable recovery snapshot gate: PASS
Provider/index identity gate: PASS
F2 regression gate: PASS
Hard non-goals gate: PASS
Test/proof gate: PASS
Diff minimality gate: PASS
F3.1.2a architecture/implementation acceptance: ACCEPT
ADO implementation modified by review: NO
```

---

## 8. STOP

```text
STOP
ADO implementation modified: NO
F3.1.1c: still requires separate explicit authorization
F3.1.2b: NO-GO until F3.1.1c is implemented and independently accepted
No F3.1.3 / F3.1.4 / F3.2 implementation
```
