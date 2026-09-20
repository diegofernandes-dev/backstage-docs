# F3.1.2b — Ledger Submission Architecture / Implementation Acceptance

- **Status:** CLOSED / ACCEPTED IMPLEMENTED BASELINE
- **Date:** 2026-09-20
- **Verdict:** `ACCEPT`
- **Implementation:** Azure DevOps `platform-devops-developer-portal@22495229502dabf2d99588599a156d862c5114fa`, branch `feat/ado-repo-governance`
- **Expected parent (verified):** product-convergence PASS `f48dc825ab5d1d16fafc3f70ef772d28613df1a0`
- **Documentation review baseline:** `backstage-docs@06c337aa7598a3de178ef84cbfcf061258a538ce` (`origin/main` at review start)
- **Authority:** F3.1.2 ACCEPTED IMPLEMENTATION CONTRACT + [`f3-1-2-final-architecture-rereview.md`](./f3-1-2-final-architecture-rereview.md) + [`f3-1-2b-implementation-evidence.md`](./f3-1-2b-implementation-evidence.md) + `prompts/f3-1-2b-ledger-submission-implementation.md` + ADR-009 (partially superseded by ADR-013) + ADR-013 + [`product-convergence-deployments-evidence.md`](./product-convergence-deployments-evidence.md) + this independent review

```text
F3.1.2b architecture/implementation acceptance: ACCEPT
F3.1.2b: CLOSED / ACCEPTED IMPLEMENTED BASELINE
Accepted ADO SHA: 22495229502dabf2d99588599a156d862c5114fa
F3.1.2: CLOSED / ACCEPTED IMPLEMENTED SUBMISSION BASELINE
Committed default newSubmissionAuthorizationMode: LEGACY_PRE_F3
Operational LEDGER_REQUIRED cutover: NOT YET AUTHORIZED
F3.1.3 planning/prompt authoring: GO
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
```

A separate **ledger cutover activation checkpoint** should precede F3.1.3 implementation. F3.1.3 decision-command product validation requires live ledger-governed Changes, and the committed default remains `LEGACY_PRE_F3`. This review did not perform that cutover.

Recommended sequence:

```text
1. Author narrow LEDGER_REQUIRED activation/cutover checkpoint
2. Execute + independently verify activation on the intended non-production product environment
3. Plan/author F3.1.3 decision-command slice
```

---

## 1. Status / verdict

F3.1.2b architecture/implementation acceptance: **ACCEPT**.

All mandatory gates **G1–G19 PASS**. The published slice at `2249522` is a faithful realization of the accepted F3.1.2b contract on top of the converged GMUD + Deployments product baseline. The committed runtime default remains `LEGACY_PRE_F3`. No operational `LEDGER_REQUIRED` cutover occurred.

ADO implementation was **not modified** by this review.

---

## 2. Reviewed baselines

### Docs baseline

| Item | Value |
|---|---|
| Repo | `diegofernandes-dev/backstage-docs` |
| Branch | `main` |
| `origin/main` SHA at review | `06c337aa7598a3de178ef84cbfcf061258a538ce` |
| Prompt | `prompts/f3-1-2b-architecture-implementation-acceptance.md` |

Canonical authority files read: F3.1.2 implementation plan, F3.1.2 final architecture re-review, F3.1.2b implementation evidence, F3.1.2b implementation prompt, ADR-009, ADR-013, product-convergence evidence, current-state, implementation-progress, prompts/README.

### Independent ADO source verification: **YES**

| Check | Result |
|---|---|
| Independent `az repos ref list` `objectId` | `22495229502dabf2d99588599a156d862c5114fa` (`refs/heads/feat/ado-repo-governance`) |
| Local inspected SHA | `22495229502dabf2d99588599a156d862c5114fa` |
| Parent of reviewed commit | **YES** — exact `f48dc825ab5d1d16fafc3f70ef772d28613df1a0` |
| Complete diff `f48dc82..2249522` inspected | **YES** — 20 files, `+2299 / −41` |
| Later tip drift beyond `2249522` | **NONE** |

SSH `git fetch` to Azure DevOps failed in-session (`remote: One or more errors occurred.`). Remote tip was confirmed independently via `az repos ref list`. Local branch already contained the exact tip SHA as a direct child of `f48dc82`.

---

## 3. Candidate lineage and scope

```text
f48dc82 (product convergence PASS)
  └─ 2249522 (F3.1.2b candidate — this review)
```

Diff `f48dc82..2249522`: **20 files**, `+2299 / −41`.

```text
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

Matches the implementation-evidence change surface exactly. No migrations. No frontend product redesign. No Delivery/Kargo/Argo/GitOps mutation.

**Confirmed untouched:** Catalog Component Deployments tab, Delivery backend/plugin, `GET /changes/:changeId/execution-eligibility`, `api:catalog/delivery` (`name: 'delivery'` on the catalog plugin), Delivery RBAC/config, GMUD frontend plugin, Catalog behavior, historical policy `2026-09-02.1`, CAB-safe policy artifact `2026-09-19.1`, selector-bundle identity, ledger schema/migrations.

---

## 4. Mandatory gate matrix (G1–G19)

| Gate | Result | Concise basis |
|---|---|---|
| **G1** Lineage and scope | **PASS** | Exact child of `f48dc82`. 20-file F3.1.2b implementation/test/config surface only. No migration, frontend redesign, or Delivery domain rewrite. |
| **G2** Product-convergence preservation | **PASS** | Deployments/Delivery/eligibility/catalog-extension identity/GMUD/Catalog paths absent from the diff and still present at the reviewed SHA. Architecture guard keeps the eligibility route. |
| **G3** Committed cutover default remains safe | **PASS** | `app-config.yaml` pins `newSubmissionAuthorizationMode: LEGACY_PRE_F3` and forbids `LEDGER_REQUIRED` in that file. Only test wiring uses `LEDGER_REQUIRED`. Ordinary default create remains legacy with zero rounds. |
| **G4** Stored-mode-wins orchestration | **PASS** | Existing reservations omit requested mode; new reservations pass current config once; repository explicit mode mismatch remains `CONFLICT`; payload mismatch `CONFLICT` before policy/Catalog; concurrent first-insert loser retries with mode omitted and follows the DB winner; different actors remain independent. |
| **G5** Authorization runtime binding | **PASS** | Startup `bootstrapAuthorization()` runtime is injected, not rebuilt per request. Policy/bundle pins are process-immutable. `evaluatePolicy` runs once per uncommitted attempt; selectors resolve once per unique key in deterministic order via the resolver abstraction. Completed reservation / finalized index replay does not re-enter materialization. |
| **G6** CAB-safe policy semantics | **PASS** | Active pin remains `default-change-authorization@2026-09-19.1`. `normal.low` materializes `normal-primary-approval` + `cab-approval`. Medium/high remain primary + CAB. Emergency A/B + retrospective unchanged. No skipCab/grant/Workbench path. |
| **G7** Requirement identity and publication guard | **PASS** | `requirementId = requirementRole` with no hash fallback. Duplicate `requirementRole` fails closed in `createPolicyRegistry` at registration/startup before any Round can materialize. Historical identities are not in this diff. |
| **G8** Emergency separation of duty | **PASS** | Selector/type mapping and `assertSeparationOfDuty` run before the platform transaction. Same-person A/B is `CONFLICT` / `details.reason=separation_of_duty` with no Round/requirement/audit/finalize/complete. |
| **G9** Round 1 and immutable evidence | **PASS** | Round 1 stores pending-index Change snapshot + `sha256Canonical`, policy identity/digest/provenance/input/hashes/matched-rule provenance, selector-bundle identity/digest/provenance, deterministic requirements, and one shared `createdAt`. |
| **G10** Mandatory authorization audit | **PASS** | Same caller-owned transaction appends `round_created`, `policy_selected`, `selector_bundle_bound`, and one `requirement_materialized` per requirement. No decision/rejection/execution/eligibility audit types are introduced. |
| **G11** Caller-owned transaction atomicity | **PASS** | DevelopmentProvider path: `createWithTransaction` + `createRound` + every `appendAuditEvent` + `index.finalize` + `idempotency.complete` on one outer Knex `trx`. Failure injection after createRound/audit rolls back all durable Round/audit/provider/finalize/complete state. |
| **G12** External-provider boundary | **PASS** | External `create` remains outside the platform transaction and is idempotent by canonical `changeId`. Retry after injected platform failure converges without XA/2PC/outbox. Provider-specific identifiers do not enter Round artifacts. Executable fixture is `FakeChangeManagementProvider` (no live ITSM provider in-repo). |
| **G13** Visibility invariant | **PASS** | Readers use finalized index only. Pending index is invisible. Round 1 is not durable before commit. Finalized LEDGER Change without Round 1 fails closed. `findRound` is not used as authority to finalize. |
| **G14** Healthy concurrent Round-1 loser convergence | **PASS** | Same actor/key/payload/changeId: one winner; loser rolls back, re-reads outside the transaction, returns the same `{ changeId, status: submitted }` when coherent; never Round 2; never CONFLICT/INTERNAL_ERROR for healthy loss; loser does not mutate winner facts. |
| **G15** Transient vs invariant classification | **PASS** | Unique/serialization/monotonic-round contention uses one immediate coherence re-read. Transient not-yet-observable winner is retryable `PROVIDER_UNAVAILABLE`. Committed contradiction is `INTERNAL_ERROR`. No polling/sleeps/distributed locks/mutex/queue. |
| **G16** PostgreSQL authoritative concurrency | **PASS** | Independent re-run with disposable PostgreSQL 15: C1 both succeed with one reservation/index/Round/requirement set/audit set/DevelopmentProvider record; C2 barrier/hook race converges; C3 different payload is CONFLICT before a second Round. |
| **G17** Regression and quality | **PASS** | Independent re-run: Change Management + Delivery 34 suites / 374 tests PASS (4 skipped, pre-existing); ledgerSubmit SQLite+Postgres 30/30; Deployments frontend 5/20; GMUD frontend 13/58; lint PASS; `yarn build:all` PASS; TypeScript remains the historical five dual-package Knex errors in `changeManagementPlugin.ts` (line shift only; zero new errors). |
| **G18** Rollback correctness | **PASS** | Evidence and source establish the old-binary hazard: pre-F3.1.2 `createChange` has no `createRound` path and would finalize a pending `LEDGER_REQUIRED` reservation without Round 1. Mandatory drain query is recorded unchanged. |
| **G19** Hard non-goals | **PASS** | No F3.1.3 decision command, F3.1.4 authorization read/RBAC, F3.2 autonomy, Teams, additive user requirements, lifecycle transition, migration, production cutover, Kargo/Argo/GitOps mutation, or generic outbox/lock framework. |

**Gates: 19 / 19 PASS.**

---

## 5. Independent proof re-run

Executed against branch tip `2249522` (no ADO source edits):

| Gate | Result |
|---|---|
| Focused ledgerSubmit SQLite + PostgreSQL (`--no-cache`) | **PASS — 2 suites / 30 tests** |
| Change Management module + Delivery, disposable PostgreSQL 15 `127.0.0.1:55432` | **PASS — 34 suites / 374 tests**, 1 suite skipped, 4 tests skipped (pre-existing) |
| Deployments frontend `catalogEntityTabs` including tab-loader collision guard | **PASS — 5 suites / 20 tests** |
| GMUD frontend `@internal/plugin-change-management` | **PASS — 13 suites / 58 tests** |
| `yarn lint:all` | **PASS** |
| `yarn build:all` | **PASS** |
| TypeScript baseline | **SET_IDENTICAL** — exactly 5 historical dual-package Knex errors in `changeManagementPlugin.ts` at `(90,61)`, `(97,69)`, `(106,64)`, `(107,60)` `TS2345` and `(108,11)` `TS2322`. Line numbers shifted because this slice wired the ledger into the plugin; zero new errors and zero new files. |
| `--forceExit` | **Not used** |
| Operational `LEDGER_REQUIRED` cutover | **Not performed** |

PostgreSQL proof used a disposable local PostgreSQL 15 on `127.0.0.1:55432` (`CHANGE_MANAGEMENT_TEST_POSTGRES_URL=postgresql://postgres@127.0.0.1:55432/change_management_test`). The cluster was stopped after the review. No production database was used.

---

## 6. Decision

```text
F3.1.2b architecture/implementation acceptance: ACCEPT
F3.1.2b: CLOSED / ACCEPTED IMPLEMENTED BASELINE
Accepted ADO SHA: 22495229502dabf2d99588599a156d862c5114fa
F3.1.2: CLOSED / ACCEPTED IMPLEMENTED SUBMISSION BASELINE
Committed default newSubmissionAuthorizationMode: LEGACY_PRE_F3
Operational LEDGER_REQUIRED cutover: NOT YET AUTHORIZED
F3.1.3 planning/prompt authoring: GO
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
```

No material implementation defect or proof gap remains against the accepted F3.1.2b contract. F3.1.3 implementation remains gated on a later explicit prompt; a narrow non-production LEDGER_REQUIRED activation checkpoint should be authored first so decision-command work can be demonstrated against live ledger-governed Changes.

---

## 7. Final report

```text
Docs baseline reviewed: 06c337aa7598a3de178ef84cbfcf061258a538ce
ADO commit reviewed: 22495229502dabf2d99588599a156d862c5114fa
Parent verified: YES
Independent ADO source verification: YES
Scope/lineage gate: PASS
Product-convergence preservation gate: PASS
Committed LEGACY default gate: PASS
Stored-mode-wins gate: PASS
Authorization-runtime gate: PASS
CAB-safe policy gate: PASS
Requirement identity/uniqueness gate: PASS
Emergency SoD gate: PASS
Round/evidence gate: PASS
Audit gate: PASS
Caller-owned transaction gate: PASS
External-provider boundary gate: PASS
Visibility invariant gate: PASS
Healthy concurrency gate: PASS
PostgreSQL C1/C2 gate: PASS
C3/C4 negative gates: PASS
Regression/quality gate: PASS
Rollback gate: PASS
Hard non-goals gate: PASS
F3.1.2b architecture/implementation acceptance: ACCEPT
ADO implementation modified by review: NO
Operational LEDGER_REQUIRED cutover performed: NO
Recommended next gate: cutover activation
Final docs SHA: recorded by the documentation commit that publishes this review
```

---

## 8. STOP

```text
STOP
ADO implementation modified: NO
Operational LEDGER_REQUIRED cutover performed: NO
F3.1.2b: CLOSED / ACCEPTED IMPLEMENTED BASELINE
F3.1.2: CLOSED / ACCEPTED IMPLEMENTED SUBMISSION BASELINE
Recommended next gate: author narrow LEDGER_REQUIRED activation/cutover checkpoint
F3.1.3 planning/prompt authoring: GO
F3.1.3 implementation: still requires separate explicit authorization
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
```
