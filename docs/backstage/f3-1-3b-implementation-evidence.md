# F3.1.3b — Rejected-Change Resubmission / New Round (implementation evidence)

- **Status:** IMPLEMENTATION PASS — not independently accepted; F3.1.3b is **not CLOSED**
- **Date:** 2026-09-21
- **Canonical docs baseline:** `backstage-docs@0c56a41675fb0a6b626059eb2b4257960e0b8bbd`
- **Authority:** F3.1.3 ACCEPTED IMPLEMENTATION CONTRACT + ADR-014 + `prompts/f3-1-3b-resubmission-new-round-implementation.md` (explicit launch)

This document records what was implemented and what was actually proven. It does
not close F3.1.3b, does not authorize F3.1.4 / F3.2, and does not flip the
committed runtime default to `LEDGER_REQUIRED`.

## 1. Implementation baseline

| Item | Value |
|---|---|
| Repository / branch | `platform-devops-developer-portal` / `feat/ado-repo-governance` |
| ADO baseline **before** (`az repos ref list`) | `6bad066d945d49feaf642313ec37467e2658dc3f` |
| ADO SHA **after** (`az repos ref list`) | `5e70d8818f55072d8568b5eb15bae754abb943d1` |
| Parent | Exact `6bad066…` — direct child |
| Publication | Plain fast-forward `6bad066..5e70d88` (HTTPS; no force) |
| Source drift vs accepted F3.1.3a SHA | **NONE** |
| Migrations | **NO** |
| Committed `newSubmissionAuthorizationMode` | `LEGACY_PRE_F3` |

**Publication transport note.** SSH `git fetch`/`push` to Azure DevOps failed
in-session (`remote: One or more errors occurred.`). Push used HTTPS with the
operator PAT. Remote tip was confirmed independently via `az repos ref list`
(`objectId` = `5e70d8818f55072d8568b5eb15bae754abb943d1`). Repository `origin`
SSH URL is unchanged.

## 2. Source verification

Fetched `diegofernandes-dev/backstage-docs@main` at `0c56a41` before editing.
Live ADO `feat/ado-repo-governance` tip was independently `6bad066` via
`az repos ref list`. Working tree parent matched that SHA. Committed
`app-config.yaml` still pins `LEGACY_PRE_F3`. Laptop gitignored overlay
`app-config.local.yaml` remains `LEDGER_REQUIRED`.

No later source drift touched resubmission, decision semantics, ledger Round
persistence, index/provider composition, participant index, permissions/RBAC,
Catalog membership, idempotency, or authorization policy/selector runtime.

No pre-existing `/resubmissions`, `change.resubmit`, Round-N writer, or
`DevelopmentProvider.replaceCurrent` existed on `6bad066`.

## 3. Changed paths

```
packages/backend/config/rbac/rbac-policy.csv
packages/backend/src/modules/changeManagement/DevelopmentProvider.test.ts
packages/backend/src/modules/changeManagement/DevelopmentProvider.ts
packages/backend/src/modules/changeManagement/architecture.test.ts
packages/backend/src/modules/changeManagement/authorization/EligibilityService.test.ts
packages/backend/src/modules/changeManagement/authorization/ResubmissionService.postgres.test.ts
packages/backend/src/modules/changeManagement/authorization/ResubmissionService.test.ts
packages/backend/src/modules/changeManagement/authorization/ResubmissionService.ts
packages/backend/src/modules/changeManagement/authorization/decisionMembership.test.ts
packages/backend/src/modules/changeManagement/authorization/decisionMembership.ts
packages/backend/src/modules/changeManagement/authorization/ledgerSubmission.ts
packages/backend/src/modules/changeManagement/executionPlanValidator.test.ts
packages/backend/src/modules/changeManagement/executionPlanValidator.ts
packages/backend/src/modules/changeManagement/idempotency.ts
packages/backend/src/modules/changeManagement/permissions.ts
packages/backend/src/modules/changeManagement/persistence/ChangeIndexRepository.ts
packages/backend/src/modules/changeManagement/persistence/IdempotencyRepository.ts
packages/backend/src/modules/changeManagement/persistence/KnexChangeIndexRepository.ts
packages/backend/src/modules/changeManagement/persistence/KnexIdempotencyRepository.ts
packages/backend/src/modules/changeManagement/targetContextResolver.ts
packages/backend/src/modules/changeManagement/testHelpers.ts
packages/backend/src/modules/changeManagement/types.ts
packages/backend/src/modules/changeManagement/validation.test.ts
packages/backend/src/modules/changeManagement/validation.ts
packages/backend/src/modules/templateExecutorRoleSeed.ts
packages/backend/src/plugins/changeManagementPlugin.ts
```

26 files, `+2447 / −27`. No migrations. No resubmit UI. Delivery routes were
not modified. `app-config.yaml` was not modified. Frontend plugin was not
modified.

## 4. Behavior implemented

HTTP contract:

```text
POST /api/change-management/changes/:changeId/resubmissions
Idempotency-Key: <required non-empty key>
```

User credentials only. Missing/blank `Idempotency-Key` → `VALIDATION_ERROR` /
400. First materialization HTTP **201**; exact same actor/key/payload replay
HTTP **200** with `{ changeId, roundNumber, status: 'submitted' }` and no
second Round/audit/provider/index rewrite.

Authority (both required):

```text
change-management.change.resubmit
AND
(actorRef == original requestedBy
 OR current Catalog member of immutable ownerRef)
```

- original requester still needs the permission;
- owner member still needs the permission;
- `platform_admin` alone is not business authority;
- CAB / `cab.record` alone is not business authority;
- `change.create`, `change.read`, participant read, `responsibleRef`, or
  `decide` alone are not business authority;
- `change_cab_recorder` is **not** granted `change.resubmit`.

Owner-membership reuses F3.1.3a prefix-agnostic live Catalog `memberOf`
(`assertLiveOwnerMembership`). Requester path does not require Catalog I/O.
Catalog/credentials unavailable → `PROVIDER_UNAVAILABLE` /
`membership_source_unavailable`. All permission/domain/Catalog/identity/
execution-plan proof occurs **before** the write transaction.

Identity freeze across rounds:

```text
changeId, requestedBy, createdAt, targetRef, ownerRef, systemRef
```

Changed `targetRef` → `VALIDATION_ERROR` / `change_identity_mismatch`. Hidden
cross-System activity `targetRef` → `VALIDATION_ERROR` /
`execution_plan_identity_mismatch`. OwnerRef/systemRef are never re-resolved
from current Catalog.

Round N+1 binds currently published policy + selector bundle, new requirements
(`requirementId = requirementRole`), submission-like audits with the actual
`round.roundNumber`, and exactly one `change.authorization.resubmitted`.
Prior Round snapshots/requirements/decisions/audits remain untouched.

Transaction (caller-owned Knex):

1. lock `change_index` (`FOR UPDATE` on PostgreSQL);
2. re-read authoritative index/current Round;
3. verify LEDGER_REQUIRED + current Round REJECTED;
4. re-verify frozen identity;
5. resolve next Round number;
6. reserve `change.resubmit` idempotency inside the transaction;
7. materialize Round N + requirements;
8. append Round/policy/selector/requirement audits + `resubmitted`;
9. project index **non-identity** fields + status `submitted`;
10. rebuild participants;
11. `DevelopmentProvider.replaceCurrent(change, trx)` — does **not** call
    `create` / `createWithTransaction`;
12. complete resubmit idempotency;
13. commit.

Completed exact-replay is observed **before** the REJECTED eligibility check
so a successful Round-2 reservation can return HTTP 200 after the Change is
already `submitted`/`PENDING`.

External/non-development provider resubmission fails closed
(`PROVIDER_UNAVAILABLE`). No XA/2PC/outbox.

## 5. Permission / RBAC bindings

| Permission | Action | Bound to |
|---|---|---|
| `change-management.change.resubmit` | update | `role:default/contributor`, `role:default/platform_admin`, `template_executor` seed |
| `change-management.change.authorization.cab.record` | update | `role:default/change_cab_recorder` only (unchanged) |

`change.resubmit` is **not** granted to `change_cab_recorder`. Technical
permission on `platform_admin` / `contributor` still requires requester-or-owner
proof in `ResubmissionService`.

## 6. Automated proofs

| Gate | Result |
|---|---|
| Change Management SQLite (`--testPathPatterns changeManagement`) | **PASS** — 34 suites passed, 5 skipped (postgres without URL + live Catalog), 400 tests passed, 22 skipped |
| PostgreSQL 16.14 R1 + R10 + R12 | **PASS** — `ResubmissionService.postgres.test.ts` |
| PostgreSQL 16.14 D1–D6 | **PASS** — `DecisionCommandService.postgres.test.ts` |
| F3.1.2 ledger-submit postgres C1–C3/T1 | **PASS** |
| Combined postgres suites | **PASS** — 3 suites / 13 tests on disposable Docker `postgres:16-alpine` (`change-management-pg16-f313b`, `127.0.0.1:55432`) |
| R1 concurrent resubmissions | **PASS** — exactly one Round N+1; loser `resubmission_conflict`; no Round N+2 |
| R2 PENDING/AUTHORIZED | **PASS** |
| R3 history immutability | **PASS** (asserted inside R4: Round-1 snapshot/requirements/decisions/audits unchanged) |
| R4 requester ± permission | **PASS** |
| R5 immutable-owner member | **PASS** |
| R6/R7/R8 platform_admin / CAB / permission-only | **PASS** |
| R9 Catalog ownership drift | **PASS** |
| R10/R11 identity freeze | **PASS** |
| R12 allowed correction | **PASS** (PostgreSQL) |
| R13 execution-plan hidden retarget | **PASS** |
| Failure injection (5 hooks) | **PASS** — afterCreateRound, afterAuditEvents, afterIndexProjection, afterParticipantRebuild, afterProviderReplace; zero durable Round/audit/index/provider/participant/resubmit reservation |
| GMUD frontend | **PASS** — 13 suites / 60 tests |
| Catalog + Deployments tab (`catalogEntityTabs\|App.test\|Sidebar`) | **PASS** — 7 suites / 22 tests |
| Delivery backend | **PASS** — 1 suite / 26 tests |
| Lint (`yarn lint:all`) | **PASS** |
| Build (`yarn build:all`) | **PASS** |
| TypeScript baseline | **UNCHANGED** — exactly 5 historical dual-package Knex errors in `changeManagementPlugin.ts`; zero new errors |

No `--forceExit`.

## 7. Live non-production product proof

Isolated operator-laptop `yarn start`. Loaded
`app-config.yaml, app-config.local.yaml`. Runtime:

```text
newSubmissionAuthorizationMode LEDGER_REQUIRED
policy default-change-authorization@2026-09-19.1
selector bundle selector-bundle-dev@2026-09-07.1
contentDigest 6a0c2fb4f30037603848cb72700e77d60f68cf35a25e6db055c3690780d28b94
```

Authenticated product identity: `user:default/diego.fernandes_outlook.com`.

Preferred disposable rejected Change `CHG-2026-000005` was still valid:

- LEDGER_REQUIRED, exactly one rejected Round, primary rejected, `round_rejected`;
- `requestedBy=user:default/diego.fernandes_outlook.com`;
- `ownerRef=group:default/cloud_azure_devops_platform_engineering`;
- `createdAt=2026-09-21T01:24:28.023Z`;
- no Round 2 before this checkpoint.

`CHG-2026-000003` was left AUTHORIZED/submitted Round 1 (not used for resubmit).

| Step | Result |
|---|---|
| Before eligibility | `DENY` / `REJECTED` / Round 1 |
| Before list/detail | `rejected` / `Rejeitada`; title `F3.1.3a disposable rejection proof` |
| POST resubmit (`Idempotency-Key: f313b-live-resubmit-000005`) | HTTP **201**, `{ changeId: CHG-2026-000005, roundNumber: 2, status: submitted }` |
| Exact replay same key/payload | HTTP **200**, same Round 2 |
| After eligibility | `DENY` / `PENDING_AUTHORIZATION` / Round 2 |
| After list/detail | `submitted` / `Submetida`; corrected title `F3.1.3b corrected disposable rejection proof` |
| Frozen identity | `changeId` / `requestedBy` / `createdAt=2026-09-21T01:24:28.023Z` / `targetRef=component:default/platform-engineering-core` / `ownerRef=group:default/cloud_azure_devops_platform_engineering` / `systemRef=system:default/platform-engineering` |
| Round 1 | snapshot/requirements/decision `4cec1822-…` / audits including `decision_recorded` + `round_rejected` **untouched** |
| Round 2 | policy `default-change-authorization@2026-09-19.1`, selector digest `6a0c2fb4…`, requirements `normal-primary-approval` + `cab-approval`, **zero** decisions |
| Round 2 audits | `round_created`, `policy_selected`, `selector_bundle_bound`, two `requirement_materialized`, exactly one `change.authorization.resubmitted` |
| Replay audit/idempotency | still one Round 2, still one `resubmitted` audit, one completed `change.resubmit` reservation; original `change.create` reservation untouched |
| New activity IDs | minted (`95242386-a812-4013-b3f4-2891d95a341a`); Round-1 snapshot title remains the original |
| Post-Round-2 retarget attempt | HTTP **409** `CONFLICT` / `round_not_terminal`; no extra Round; no extra idempotency row; frozen identity intact. Rejected-round identity mismatch is covered by automated R10 (`VALIDATION_ERROR` / `change_identity_mismatch`) |
| `CHG-2026-000003` | still one Round 1, two decisions, status `submitted` |

### Product surfaces

| Surface | Result |
|---|---|
| `/gmud` | reachable; `CHG-2026-000005` **Submetida** with corrected title; `CHG-2026-000003` Submetida |
| `/gmud/CHG-2026-000005` | reachable; **Submetida**; corrected title; no resubmit/approve/reject buttons |
| Catalog `/` | reachable (owned components) |
| Deployments tab | `/catalog/default/component/idp-showcase-api/deployments` visible; “Dados atualizados” |
| Delivery reads | HTTP 200: 1 release-candidate, 3 deployments, 10 events |

No F3.1.4 authorization UI. No CAB Workbench. No resubmit button.

## 8. Explicit non-goals proven absent

- No approve/reject/resubmit UI buttons
- No F3.1.4 composed authorization/governance read panel
- No CAB Workbench / F3.2 autonomy
- No migrations (`knex_migrations` still the five historical files)
- No committed `LEDGER_REQUIRED` default
- No Teams / execution lifecycle / production cutover
- No Kargo/Argo/GitOps mutation
- No external-provider resubmission protocol

## 9. Next gate

Independent **F3.1.3b architecture/implementation acceptance review**.

```text
F3.1.3b implementation: PASS
F3.1.3b CLOSED: NO
F3.1.4/F3.2: NO-GO
Production cutover: NOT AUTHORIZED
```
