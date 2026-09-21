# F3.1.3b — Architecture / Implementation Acceptance

- **Status:** CLOSED / ACCEPTED IMPLEMENTED BASELINE
- **Date:** 2026-09-21
- **Verdict:** `ACCEPT`
- **Implementation:** Azure DevOps `platform-devops-developer-portal@5e70d8818f55072d8568b5eb15bae754abb943d1`, branch `feat/ado-repo-governance`
- **Expected parent (verified):** F3.1.3a accepted baseline `6bad066d945d49feaf642313ec37467e2658dc3f`
- **Documentation review baseline:** `backstage-docs@eaa218e1a0739e3477641764c000053bb1b1599f` (`origin/main` at review start)
- **Authority:** F3.1.3 ACCEPTED IMPLEMENTATION CONTRACT + ADR-014 + [`f3-1-3b-implementation-evidence.md`](./f3-1-3b-implementation-evidence.md) + `prompts/f3-1-3b-resubmission-new-round-implementation.md` + `prompts/f3-1-3b-architecture-implementation-acceptance.md` + F3.1.3a acceptance + this independent review

```text
F3.1.3b architecture/implementation acceptance: ACCEPT
F3.1.3b: CLOSED / ACCEPTED IMPLEMENTED BASELINE
Accepted ADO SHA: 5e70d8818f55072d8568b5eb15bae754abb943d1
F3.1.3: CLOSED / ACCEPTED IMPLEMENTED DECISION + RESUBMISSION BASELINE
F3.1.4 planning/prompt authoring: GO
F3.1.4 implementation: still requires separate explicit authorization
F3.2 implementation: NO-GO
Production cutover: NOT AUTHORIZED
```

This review did **not** author the F3.1.4 prompt. ADO source, Round/Decision/Audit facts, laptop overlay, and production were not modified.

---

## 1. Status / verdict

F3.1.3b architecture/implementation acceptance: **ACCEPT**.

All mandatory gates **G1–G23 PASS**. The live post-Round-2 `round_not_terminal` retarget is **not** treated as R10. R10 is independently proven from source and tests on a **rejected-round** path (`VALIDATION_ERROR` / `change_identity_mismatch`, zero durable mutation) in both SQLite and PostgreSQL 16.14.

The published slice at `5e70d88` is a faithful realization of the accepted F3.1.3b + ADR-014 rejected-Change same-`changeId` resubmission contract. The committed runtime default remains `LEGACY_PRE_F3`.

ADO implementation was **not modified** by this review.

---

## 2. Reviewed baselines

### Docs baseline

| Item | Value |
|---|---|
| Repo | `diegofernandes-dev/backstage-docs` |
| Branch | `main` |
| `origin/main` SHA at review | `eaa218e1a0739e3477641764c000053bb1b1599f` |
| Prompt | `prompts/f3-1-3b-architecture-implementation-acceptance.md` |

Canonical authority files read: F3.1.3 implementation plan, F3.1.3 revised-plan re-review, F3.1.3a acceptance, F3.1.3b implementation evidence, F3.1.3b implementation prompt, this acceptance prompt, ADR-007, ADR-008, ADR-009, ADR-013, ADR-014, current-state, implementation-progress, prompts/README.

### Independent ADO source verification: **YES**

| Check | Result |
|---|---|
| Independent `az repos ref list` `objectId` | `5e70d8818f55072d8568b5eb15bae754abb943d1` (`refs/heads/feat/ado-repo-governance`) |
| Local inspected SHA | `5e70d8818f55072d8568b5eb15bae754abb943d1` |
| Parent of reviewed commit | **YES** — exact `6bad066d945d49feaf642313ec37467e2658dc3f` |
| Complete diff `6bad066..5e70d88` inspected | **YES** — 26 files, `+2447 / −27` |
| Later tip drift beyond `5e70d88` | **NONE** |

SSH `git fetch` to Azure DevOps remains broken in-session. Remote tip was confirmed independently via `az repos ref list`. Local `feat/ado-repo-governance` already contained the exact candidate as a direct child of `6bad066`. Working tree: untracked `.vscode/` only. This review created no ADO commit.

---

## 3. Candidate lineage and scope

```text
6bad066 (F3.1.3a CLOSED / ACCEPTED IMPLEMENTED BASELINE)
  └─ 5e70d88 (F3.1.3b candidate — this review)
```

Diff `6bad066..5e70d88`: **26 files**, `+2447 / −27`. Backend Change Management, RBAC CSV, template-executor seed, plugin route, architecture/tests only. No migrations. No `app-config.yaml` change. No frontend plugin change. No Delivery/Kargo/Argo/GitOps mutation. `targetContextResolver.ts` only **exports** existing helpers; create-path resolution is unchanged. `ChangeManagementService` still calls `materializeLedgerRound1` only.

**Confirmed untouched:** Catalog Deployments tab identity, Delivery module, committed `newSubmissionAuthorizationMode: LEGACY_PRE_F3`, historical policies, selector-bundle identity, F3.1.3a decision route.

---

## 4. Gate results

| Gate | Result |
|---|---|
| G1 Lineage / scope | **PASS** |
| G2 Resubmission HTTP contract | **PASS** — `POST /changes/:changeId/resubmissions`; user credentials only; required trimmed `Idempotency-Key`; 201 then 200 `{changeId,roundNumber,status}`; not an F3.1.4 read model |
| G3 Dedicated permission / RBAC | **PASS** — `change-management.change.resubmit` action `update`; contributor + platform_admin + template_executor seed; **not** granted to `change_cab_recorder` |
| G4 Business authority | **PASS** — permission AND requester-or-immutable-owner; R4/R6/R7/R8 deny requester-without-permission, platform_admin-only, CAB-only, permission-only |
| G5 Owner membership / requester fast path | **PASS** — `assertLiveOwnerMembership` prefix-agnostic; requester exact-match returns before Catalog; Catalog fail-closed `membership_source_unavailable` / `owner_authority_unresolvable` / `actor_not_in_catalog` |
| G6 Identity freeze | **PASS** — `buildCorrectedChange` copies original identity; `replaceCurrent` and index projection refuse identity rewrite; body `requestedBy`/`ownerRef`/`systemRef`/`createdAt`/`changeId` rejected at parse |
| G7 Correctable fields | **PASS** — create-shaped non-identity fields only; `targetRef` optional and must canonicalize to original |
| G8 Execution-plan hidden retarget | **PASS** — omitted activity target allowed; present Component System must equal immutable `systemRef`; `execution_plan_identity_mismatch` |
| G9 Current-Round eligibility | **PASS** — ledger `evaluateAuthorization === REJECTED`; LEGACY / missing Round / PENDING / AUTHORIZED / stale Round → `ledger_required_only` or `round_not_terminal` |
| G10 Round-N policy/selector | **PASS** — `materializeLedgerRound(roundNumber=N+1)` binds current published policy/selectors; `requirementId = requirementRole`; no copied decisions |
| G11 Round-1 regression | **PASS** — create path still `materializeLedgerRound1`; audits use `round.roundNumber`; no generic new-round endpoint |
| G12 Idempotency | **PASS** — operation `change.resubmit`; hash `{changeId,request}`; completed replay before REJECTED check; payload mismatch; original `change.create` untouched |
| G13 Caller-owned transaction | **PASS** — lock, re-read, reserve, Round, audits + `resubmitted`, index, participants, `replaceCurrent`, complete; Catalog I/O is pre-transaction |
| G14 Index / participants | **PASS** — `projectCurrentNonIdentity` does not write identity/`authorization_mode`; participants rebuilt inside trx |
| G15 `replaceCurrent` | **PASS** — DevelopmentProvider only; not `create`/`createWithTransaction`; absent record fails closed; external provider `PROVIDER_UNAVAILABLE`; not on `IChangeManagementProvider` |
| G16 History immutability | **PASS** — R3/R4 assert Round-1 snapshot/requirements/decisions/audits unchanged; live Round-1 hash `d77eab65…` retained |
| G17 R1 PostgreSQL concurrency | **PASS** — deterministic `afterLock`/`beforeTransaction` barriers, not sleeps; one Round 2; loser `resubmission_conflict`; one resubmit reservation |
| G18 R2–R13 | **PASS** — independently re-run; R10 proven on **rejected** Change, not live post-Round-2 409 |
| G19 Failure injection | **PASS** — five hooks roll back Round/audits/index/provider/participants/reservation |
| G20 F3.1.3a D1–D6 | **PASS** — PostgreSQL 16.14 D1–D6 green; `CHG-2026-000003` still one AUTHORIZED Round 1 |
| G21 Live Round-2 evidence | **PASS** — read-only sqlite/`CHG-2026-000005`: Round 2, frozen identity, corrected title, zero Round-2 decisions, one `resubmitted`, create reservation intact |
| G22 Product / quality | **PASS** — CM 38/418 (+1 skipped live Catalog); GMUD 13/60; Catalog/Deployments 7/22; Delivery 26; lint/build PASS; tsc **UNCHANGED** 5 historical Knex errors |
| G23 Hard non-goals | **PASS** |

### R10 (rejected-round identity) — independent of live 409

Live retarget after Round 2 returned `CONFLICT` / `round_not_terminal`. That is eligibility, not identity proof.

Independent R10:

- Source: `ResubmissionService.assertFrozenIdentity` runs on a REJECTED current Round **before** the write transaction; mismatched `targetRef` → `VALIDATION_ERROR` / `change_identity_mismatch`.
- SQLite: `R10/R11 reject identity mutation before any durable write` against `createRejectedChange()`.
- PostgreSQL 16.14: `R10 different targetRef does not mutate locked state` against a freshly rejected Change; round rows unchanged.

---

## 5. Live durable facts inspected (read-only)

`CHG-2026-000005` (no new Round/Decision/Audit created by this review):

| Fact | Value |
|---|---|
| Identity | `requestedBy=user:default/diego.fernandes_outlook.com`, `createdAt=2026-09-21T01:24:28.023Z`, `targetRef=component:default/platform-engineering-core`, `ownerRef=group:default/cloud_azure_devops_platform_engineering`, `systemRef=system:default/platform-engineering` |
| Current projection | status `submitted`, title `F3.1.3b corrected disposable rejection proof` |
| Round 1 | snapshot sha `d77eab65…`, rejected primary `4cec1822-…`, `round_rejected` intact |
| Round 2 | policy `2026-09-19.1`, selector digest `6a0c2fb4…`, requirements `normal-primary-approval` + `cab-approval`, **zero** decisions, exactly one `change.authorization.resubmitted` |
| Idempotency | original `change.create` completed; one completed `change.resubmit` key `f313b-live-resubmit-000005` |
| `CHG-2026-000003` | still Round 1 only |
| Migrations | still the five historical files |

---

## 6. Independent quality re-run

| Gate | Result |
|---|---|
| Change Management | **PASS** — 38 suites passed, 1 skipped, 418 tests passed, 4 skipped |
| PostgreSQL 16.14 R1/R10/R12 | **PASS** |
| PostgreSQL 16.14 D1–D6 | **PASS** |
| GMUD frontend | **PASS** — 13/60 |
| Catalog + Deployments | **PASS** — 7/22 |
| Delivery | **PASS** — 26 |
| Lint | **PASS** |
| Build | **PASS** |
| TypeScript | **UNCHANGED** — exactly 5 historical dual-package Knex errors |

No `--forceExit`. No ADO source edit.

---

## 7. Next authorized activity

Author a constrained **F3.1.4** implementation prompt only after a separate explicit launch. Do **not** implement F3.1.4 or F3.2 from this review. Production cutover remains **NOT AUTHORIZED**.
