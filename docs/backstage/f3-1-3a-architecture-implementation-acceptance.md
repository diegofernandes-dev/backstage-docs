# F3.1.3a — Architecture / Implementation Acceptance

- **Status:** CLOSED / ACCEPTED IMPLEMENTED BASELINE
- **Date:** 2026-09-21
- **Verdict:** `ACCEPT`
- **Implementation:** Azure DevOps `platform-devops-developer-portal@6bad066d945d49feaf642313ec37467e2658dc3f`, branch `feat/ado-repo-governance`
- **Expected parent (verified):** F3.1.2 accepted submission baseline `22495229502dabf2d99588599a156d862c5114fa`
- **Documentation review baseline:** `backstage-docs@cba9ec035201064928f2114051d8bb615ac4bdae` (`origin/main` at review start)
- **Authority:** F3.1.3 ACCEPTED IMPLEMENTATION CONTRACT + [`f3-1-3-revised-plan-architecture-rereview.md`](./f3-1-3-revised-plan-architecture-rereview.md) + [`f3-1-3a-implementation-evidence.md`](./f3-1-3a-implementation-evidence.md) + `prompts/f3-1-3a-decision-command-implementation.md` + `prompts/f3-1-3a-architecture-implementation-acceptance.md` + ADR-009 (partially superseded by ADR-013) + ADR-013 + ADR-014 + F3.1.2b / non-production LEDGER_REQUIRED acceptances + this independent review

```text
F3.1.3a architecture/implementation acceptance: ACCEPT
F3.1.3a: CLOSED / ACCEPTED IMPLEMENTED BASELINE
Accepted ADO SHA: 6bad066d945d49feaf642313ec37467e2658dc3f
F3.1.3b implementation-prompt authoring: GO
F3.1.3b implementation: still requires separate explicit authorization
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
Production cutover: NOT AUTHORIZED
```

This review did **not** author the F3.1.3b implementation prompt. ADO source, decision facts, laptop overlay, and production were not modified.

---

## 1. Status / verdict

F3.1.3a architecture/implementation acceptance: **ACCEPT**.

All mandatory gates **G1–G19 PASS**. Live happy-path and rejection evidence gates are **PASS** from independently inspected durable laptop facts plus product UI; this review did not create, delete, or repair `ApprovalDecision` rows.

The published slice at `6bad066` is a faithful realization of the accepted F3.1.3a server-authoritative decision-command contract. The committed runtime default remains `LEGACY_PRE_F3`.

ADO implementation was **not modified** by this review.

---

## 2. Reviewed baselines

### Docs baseline

| Item | Value |
|---|---|
| Repo | `diegofernandes-dev/backstage-docs` |
| Branch | `main` |
| `origin/main` SHA at review | `cba9ec035201064928f2114051d8bb615ac4bdae` |
| Prompt | `prompts/f3-1-3a-architecture-implementation-acceptance.md` |

Canonical authority files read: F3.1.3 implementation plan, F3.1.3 revised-plan re-review, F3.1.3a implementation evidence, F3.1.3a implementation prompt, this acceptance prompt, ADR-009, ADR-013, ADR-014, F3.1.2b acceptance, non-production LEDGER_REQUIRED acceptance, current-state, implementation-progress, prompts/README.

### Independent ADO source verification: **YES**

| Check | Result |
|---|---|
| Independent `az repos ref list` `objectId` | `6bad066d945d49feaf642313ec37467e2658dc3f` (`refs/heads/feat/ado-repo-governance`) |
| Local inspected SHA | `6bad066d945d49feaf642313ec37467e2658dc3f` |
| Parent of reviewed commit | **YES** — exact `22495229502dabf2d99588599a156d862c5114fa` |
| Complete diff `2249522..6bad066` inspected | **YES** — 35 files, `+2880 / −163` |
| Later tip drift beyond `6bad066` | **NONE** |

SSH `git fetch` to Azure DevOps failed in-session (`remote: One or more errors occurred.`). Remote tip was confirmed independently via `az repos ref list`. Local `feat/ado-repo-governance` already contained the exact tip SHA as a direct child of `2249522`. Working tree: untracked `.vscode/` only.

---

## 3. Candidate lineage and scope

```text
2249522 (F3.1.2 CLOSED / ACCEPTED IMPLEMENTED SUBMISSION BASELINE)
  └─ 6bad066 (F3.1.3a candidate — this review)
```

Diff `2249522..6bad066`: **35 files**, `+2880 / −163`.

Matches the implementation-evidence change surface exactly. No migrations. No `app-config.yaml` change. No Delivery/Kargo/Argo/GitOps mutation. No approve/reject UI or CAB Workbench. No `change.resubmit`. No `DevelopmentProvider.replaceCurrent`.

**Confirmed untouched:** Catalog Component Deployments tab identity (`api:catalog/delivery` / `name: 'delivery'`), Delivery backend module, committed `newSubmissionAuthorizationMode: LEGACY_PRE_F3`, historical policies, selector-bundle identity.

---

## 4. LEGACY_PRE_F3 eligibility comparison (`2249522` vs `6bad066`)

Parent `EligibilityService.evaluate` for `authorizationMode !== 'LEDGER_REQUIRED'` already returned **`DENY` / `NO_LEDGER_ROUND`** before any round load. Candidate keeps that exact branch.

Sandbox `ensureRound` fabrication existed only on the **`LEDGER_REQUIRED` missing-round** path. Removing it is the accepted F3.1.3a safety contract (`NO_LEDGER_ROUND`, no Round created by read). It is not a LEGACY semantic change.

Window source moved from index birth snapshot to **current Round snapshot** only after a current Round exists — authorized by the plan. LEGACY Changes never reach that branch.

Independent laptop fact: `CHG-2026-000002` remains `LEGACY_PRE_F3`, zero rounds, derived eligibility **`DENY` / `NO_LEDGER_ROUND`**.

**LEGACY eligibility regression check: PASS.** No unreviewed semantic regression from the sandbox-round cleanup.

---

## 5. Mandatory gate matrix (G1–G19)

| Gate | Result | Concise independent basis |
|---|---|---|
| **G1** Lineage / scope | **PASS** | Exact child of `2249522`. 35-file F3.1.3a surface only. No migration, provider replacement, Round-2/resubmit, approve/reject UI, or committed `LEDGER_REQUIRED`. |
| **G2** HTTP command contract | **PASS** | Nested `POST /changes/:changeId/rounds/:roundNumber/requirements/:requirementId/decisions`; `allow: ['user']`; required trimmed `Idempotency-Key`; strict body (`outcome`/`reason`/`comment`/`cabMeetingRef`); rejection requires reason; `cabMeetingRef` only for cab/authority; client cannot supply actor/timestamp/hash/evidence; canonical `sha256Canonical({changeId, roundNumber, requirementId, outcome, reason, comment, cabMeetingRef})`; 201 first / 200 replay; bounded `{decision, authorizationEvaluation, changeStatus, roundNumber}`. |
| **G3** Individual decision authority | **PASS** | `decide` permission **and** `actor.userEntityRef == principalSnapshot.resolvedPrincipalRef`. Mismatch `FORBIDDEN` / `not_requirement_principal` with zero writes. `platform_admin` / requester / owner / CAB membership cannot substitute. |
| **G4** CAB / authority membership | **PASS** | Dedicated `cab.record`. Live Catalog `relations.memberOf` + `spec.memberOf`, normalized/deduped, prefix-agnostic via `decisionMembership.ts` (not selector/). Not TP `ownershipEntityRefs`. One collective decision; `actingAuthorityRef` + `authorizationEvidence.kind=authority_membership` / `membershipSource=catalog`. Catalog down → `PROVIDER_UNAVAILABLE` / `membership_source_unavailable` zero writes; actor absent / non-member fail closed. |
| **G5** Permission/RBAC separation | **PASS** | Separate `decide` and `cab.record`. `platform_admin` has `decide` only. CSV `role:default/change_cab_recorder` + `g, group:default/cloud_azure_devops_platform_devops, role:default/change_cab_recorder`. Permission alone does not bypass domain proof. `change.resubmit` absent from permissions and CSV. |
| **G6** Caller-owned txn / trx-aware reads | **PASS** | Catalog/permission/domain proof **before** writer txn. One `knex.transaction`. `change_index FOR UPDATE` (PostgreSQL). `findCurrentRound` / `listRequirements` / `listAuditEvents` / `findDecisionByRequirement` / `findDecisionByIdempotency` all `db = trx ?? this.knex`. Post-insert re-eval uses the same `trx`. Dedicated test proves uncommitted decision is visible on the same connection and rolled back after injected failure. |
| **G7** Decision idempotency | **PASS** | One terminal decision per requirement/round; `(actorRef, idempotencyKey)` replay identity; exact replay returns original; same key changed payload `idempotency_payload_mismatch`; different key after terminal `conflicting_decision`; same actor/key on another requirement fails `isExactReplay` (requirementId in hash/identity). Insert-only `appendDecision`; no UPDATE/DELETE/ON CONFLICT. |
| **G8** Concurrent unique-loser | **PASS** | Unique violation caught **after** `knex.transaction` rejects (auto-rollback), then `observeCommittedWinner` under a new lock. Duplicate approve converges to replay; approve-vs-reject `conflicting_decision`. SQLite unique-loser uses injected `23505` + hidden uncommitted reads. No sleeps/polling/distributed lock/mutex/queue. |
| **G9** Exactly-once milestones | **PASS** | New decision → one `decision_recorded`. First AUTHORIZED → one `authorization_reached`. First mandatory-pre rejection → one `round_rejected`. Replay emits none. No mutable evaluation row. Milestone writers serialized by parent `FOR UPDATE` + trx-aware `listAuditEvents`. Event-type uniqueness is **not** a DB constraint (accepted: no DDL). |
| **G10** Rejection lifecycle projection | **PASS** | Same txn: rejecting decision + `decision_recorded` + `round_rejected` + `projectLifecycleStatus('rejected')`. List uses index snapshot; detail overlays **status only** (`{...toPublicChange(record.change), status: indexRecord.snapshot.status}`). Live `CHG-2026-000005` provider `record_json.status` remains `submitted` while index/list/detail are `rejected`. AUTHORIZED leaves lifecycle `submitted`. Frontend/backend `ChangeStatus` is `submitted \| rejected` only. |
| **G11** Eligibility safety | **PASS** | `ensureRound` / sandbox-policy removed. `LEDGER_REQUIRED` with no current Round → `NO_LEDGER_ROUND`, `createRound` not called. Current Round uses `round.changeSnapshot.requestedWindow`. |
| **G11b** LEGACY eligibility regression | **PASS** | See §4. Pre/post `2249522` LEGACY path identical: `DENY` / `NO_LEDGER_ROUND`. |
| **G12** Post-execution fail-closed | **PASS** | `requirement.phase === 'post_execution'` → `CONFLICT` / `execution_completion_required` before insert. Emergency post-execution fixture test: zero decisions. No synthetic completion fact / lifecycle expansion. |
| **G13** PostgreSQL D1–D6 | **PASS** | Independent re-run on disposable Docker `postgres:16-alpine` **16.14** (`127.0.0.1:55432`). `DecisionCommandService.postgres.test.ts` PASS (D1–D6). Construction: real `Promise.all` races serialized by `FOR UPDATE` (no timing sleeps); unique-loser also proven by hook/failure injection on SQLite. |
| **G14** Rollback / failure injection | **PASS** | After-insert-before-audit hook → zero durable decision/audit. After-decision-audit-before-lifecycle hook on rejection → zero decision, index remains `submitted`. Unique-loser does not query the aborted transaction. |
| **G15** Regression / quality | **PASS** | Independent re-run at `6bad066`: Change Management + Delivery `changeManagement\|delivery` **39 passed / 1 skipped**, **428 passed / 4 skipped**; GMUD frontend **13/60**; Catalog/Deployments/App/Sidebar **7/22**; `yarn lint:all` PASS; `yarn build:all` PASS; TypeScript exactly the historical five dual-package Knex errors in `changeManagementPlugin.ts` at `(95,61)`, `(102,69)`, `(111,64)`, `(112,60)` `TS2345` and `(113,11)` `TS2322` (line shift from decision-route wiring; zero new errors). No `--forceExit`. |
| **G16** Product-convergence preservation | **PASS** | `/gmud` list/detail live; Catalog `/` owned components live; Deployments tab on `idp-showcase-api` live (Synced/Healthy, GMUD context); Delivery reads served by the tab. `deliveryApiExtension.name === 'delivery'`. No new extension/config collision in this diff. |
| **G17** Live happy-path evidence | **PASS** | Durable SQLite + signed-in UI for `CHG-2026-000003` independently inspected; no facts created. Round 1 only; `normal-primary-approval` + `cab-approval`; primary `5b64d801-cdef-4288-9523-07420c838033` at `2026-09-21T01:23:53.158Z`; CAB `b465a891-9db3-4e5e-a2e5-f083ffe035fc` with `actingAuthorityRef=group:default/cloud_azure_devops_platform_devops`, evidence `kind=authority_membership` / `membershipSource=catalog`; two `decision_recorded`; **exactly one** `authorization_reached`; lifecycle/index/UI **Submetida**; derived eligibility **`DENY` / `OUTSIDE_WINDOW`** against Round snapshot window `[2026-09-29T01:00:00.000Z, 2026-09-29T02:00:00.000Z)`. Detail has no approve/reject buttons. |
| **G18** Live rejection evidence | **PASS** | Durable `CHG-2026-000005`: one primary rejection (`4cec1822-…`, reason present); one `decision_recorded`; one `round_rejected`; **no** `authorization_reached`; one Round; index/list/detail **Rejeitada**; derived eligibility **`DENY` / `REJECTED`**; provider `record_json` still `submitted`. Source has no `/resubmit` or generic `/rounds` writer. This review did not POST new decisions. |
| **G19** Hard non-goals | **PASS** | No F3.1.3b resubmission/new-round, `change.resubmit`, `replaceCurrent`, F3.1.4 authorization UI, approve/reject buttons, CAB Workbench, F3.2 autonomy, Teams, execution lifecycle expansion, migration, production cutover, or Kargo/Argo/GitOps mutation. |

**Gates: 19 / 19 PASS** (G11b recorded as the required LEGACY regression check).

---

## 6. Independent proof re-run

Executed against branch tip `6bad066` (no ADO source edits):

| Gate | Result |
|---|---|
| Change Management + Delivery (`--testPathPatterns 'changeManagement\|delivery'`, `--no-cache`, Postgres URL set) | **PASS — 39 suites / 428 tests**, 1 suite skipped, 4 tests skipped (pre-existing live Catalog skip) |
| PostgreSQL 16.14 D1–D6 (`DecisionCommandService.postgres.test.ts`) | **PASS** |
| F3.1.2 ledgerSubmit SQLite + Postgres | **PASS** (included in the same backend run) |
| GMUD frontend `@internal/plugin-change-management` | **PASS — 13 suites / 60 tests** |
| Catalog + Deployments tab + App + Sidebar | **PASS — 7 suites / 22 tests** |
| `yarn lint:all` | **PASS** |
| `yarn build:all` | **PASS** |
| TypeScript baseline | **SET_IDENTICAL** — historical five Knex dual-package errors only |
| `--forceExit` | **Not used** |
| Decision facts created/repaired | **NONE** |

PostgreSQL proof used disposable Docker `postgres:16-alpine` 16.14 on `127.0.0.1:55432` (`CHANGE_MANAGEMENT_TEST_POSTGRES_URL=postgresql://postgres@127.0.0.1:55432/change_management_test`). The container was removed after the review. No production database was used.

Laptop runtime was independently restarted with the existing gitignored overlay only (`Loaded config from app-config.yaml, app-config.local.yaml`; startup `newSubmissionAuthorizationMode LEDGER_REQUIRED`; policy `default-change-authorization@2026-09-19.1`; selector bundle digest `6a0c2fb4f30037603848cb72700e77d60f68cf35a25e6db055c3690780d28b94`). Overlay and committed default were not edited.

---

## 7. Decision

```text
F3.1.3a architecture/implementation acceptance: ACCEPT
F3.1.3a: CLOSED / ACCEPTED IMPLEMENTED BASELINE
Accepted ADO SHA: 6bad066d945d49feaf642313ec37467e2658dc3f
F3.1.3b implementation-prompt authoring: GO
F3.1.3b implementation: still requires separate explicit authorization
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
Production cutover: NOT AUTHORIZED
```

No material implementation defect or proof gap remains against the accepted F3.1.3a contract. Next authorized activity is authoring a constrained F3.1.3b implementation prompt. Do **not** implement F3.1.3b from inside prompt authoring.

---

## 8. Final report

```text
Docs baseline reviewed: cba9ec035201064928f2114051d8bb615ac4bdae
ADO candidate reviewed: 6bad066d945d49feaf642313ec37467e2658dc3f
Parent verified: YES
Independent ADO source verification: YES
Lineage/scope gate: PASS
Decision transport gate: PASS
Individual authority gate: PASS
CAB membership gate: PASS
Permission/RBAC separation gate: PASS
Transaction/trx-aware-read gate: PASS
Decision idempotency gate: PASS
Concurrent loser gate: PASS
Milestone-audit gate: PASS
Rejection lifecycle gate: PASS
Eligibility safety gate: PASS
LEGACY eligibility regression check: PASS
Post-execution fail-closed gate: PASS
PostgreSQL D1-D6 gate: PASS
Rollback/failure-injection gate: PASS
Regression/quality gate: PASS
Product-convergence preservation gate: PASS
Live happy-path evidence gate: PASS
Live rejection evidence gate: PASS
Hard non-goals gate: PASS
F3.1.3a architecture/implementation acceptance: ACCEPT
ADO implementation modified by review: NO
F3.1.3b implementation performed: NO
Final docs SHA: recorded by the documentation commit that publishes this review
```
