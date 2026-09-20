# F3.1.3 — Decision Command and New-Round Semantics Implementation Plan

- **Status:** PLANNING COMPLETE — READY FOR INDEPENDENT REVIEW
- **Date:** 2026-09-20
- **Authority:** ADR-006, ADR-007 (Model C), ADR-008, ADR-009 as partially superseded by ADR-013, ADR-013; F3.1 / F3.1.2 implementation plans; accepted F3.1.2b baseline; accepted non-production `LEDGER_REQUIRED` activation
- **Planning prompt:** `prompts/f3-1-3-planning.md`
- **Docs baseline reviewed:** `diegofernandes-dev/backstage-docs@733393fab5a55b358ee676ffc0b37a5646ec7939`
- **ADO branch tip verified:** `platform-devops-developer-portal@22495229502dabf2d99588599a156d862c5114fa` (`feat/ado-repo-governance`)
- **Accepted live LEDGER demo target:** operator-laptop overlay `LEDGER_REQUIRED`; committed default `LEGACY_PRE_F3`
- **F3.1.3 implementation:** **NO-GO**

```text
F3.1.3 planning: READY_FOR_REVIEW
F3.1.3 implementation: NO-GO
Migration required: NO
Recommended slices: F3.1.3a + F3.1.3b
ADO implementation modified by this checkpoint: NO
```

This checkpoint does not authorize implementation, does not create `ApprovalDecision` facts, and does not author an implementation prompt.

---

## 1. Status / authority / baselines

### What this plan is

An implementation contract proposal for server-authoritative approval/rejection commands on top of the accepted live-ledger F3.1.2 baseline, plus same-`changeId` resubmission after a terminal rejected round.

### What this plan is not

- Not implementation authorization.
- Not an F3.1.3a/F3.1.3b implementation prompt.
- Not F3.1.4 composed authorization/governance UI.
- Not CAB Workbench, CAB autonomy grants, Teams, break-glass, SLA jobs, execution start/completion, or production cutover.
- Not a claim that `AUTHORIZED` is a Change lifecycle value.

### Verified lineage

| Surface | SHA / fact | Result |
|---|---|---|
| Docs `origin/main` | `733393fab5a55b358ee676ffc0b37a5646ec7939` | Planning prompt present; F3.1.2 closed |
| Local ADO HEAD | `22495229502dabf2d99588599a156d862c5114fa` | Exact accepted F3.1.2b commit |
| Local `origin/feat/ado-repo-governance` | same SHA | No later accepted drift |
| Independent `az repos ref list` | `objectId` same SHA | Remote tip matches |
| Working tree (ADO) | untracked `.vscode/` only | Overlay and SQLite gitignored |
| Committed `app-config.yaml` | `newSubmissionAuthorizationMode: LEGACY_PRE_F3` | Unchanged |
| Laptop overlay | gitignored `app-config.local.yaml` `LEDGER_REQUIRED` | Accepted demo target |
| Laptop `CHG-2026-000003` | Round 1, primary + CAB, **zero decisions**, five canonical audits | Intact |

Source drift after accepted F3.1.2: **NONE**.

---

## 2. Current source reality at ADO `2249522`

### P1 inventory

| Surface | Path / symbol | Current responsibility at `2249522` |
|---|---|---|
| HTTP | `changeManagementPlugin.ts` | `POST /changes`, `GET /changes`, `GET /changes/:changeId`, `GET /changes/:changeId/execution-eligibility`. No decision/resubmit route |
| Service | `ChangeManagementService` | `createChange` / `listChanges` / `getChange` only. Architecture guard **forbids** `appendDecision` |
| Status | `ChangeStatus` | Literal `'submitted'` only (backend + frontend) |
| Index | `KnexChangeIndexRepository` | PK `change_id` — one row per Change. `finalize()` updates `is_finalized` / `external_*` only. **No status-update method.** `authorization_mode` is immutable by trigger; `status` is not |
| Provider | `DevelopmentProvider` | One `development_change_records` row per `changeId`; `create()` upserts `record_json`. `IChangeManagementProvider` has `create`/`get` only |
| Ledger repo | `KnexAuthorizationLedgerRepository` | Insert/read only. `createRound(trx?)`, `appendDecision(trx?)`, `appendAuditEvent(trx?)`, `findCurrentRound` = `MAX(round_number)`. Postgres `FOR UPDATE` on `change_index` before inserting a round |
| Decision storage | `change_authorization_decisions` | Unique `(change_id, round_number, requirement_id)`; unique `(actor_ref, idempotency_key)`; rejection CHECK requires non-empty `reason`; append-only triggers |
| Domain decision | `ApprovalDecision` | `actorRef`, optional `actingAuthorityRef` (**not** `authorityRef`), `authorizationEvidence`, required `idempotencyKey` + `commandHash` |
| Evaluators | `evaluateAuthorization` / `evaluateGovernance` | Pure. Not persisted |
| Eligibility | `EligibilityService` | Derives `PENDING_AUTHORIZATION` / `REJECTED` / `AUTHORIZED` + window. Still has leftover `ensureRound` sandbox fabrication if no round exists |
| Identity | `userInfo` + `resolveEntraOwnershipEntityRefs` | `ownershipEntityRefs` = user ref + Entra TP groups prefixed `group:default/cloud_azure_devops_` only. **Not** a general group-membership primitive |
| Selector | `CatalogPrincipalResolver` | Authority resolves to Group ref. Explicitly deferred decision-time membership |
| CAB binding | `app-config.yaml` | `cab-authority` → `group:default/cloud_azure_devops_platform_devops` |
| Primary binding | same | `normal-primary-approver` → `user:default/diego.fernandes_outlook.com` |
| Permissions | `permissions.ts` + `rbac-policy.csv` | Only `change-management.change.create` / `.read`. No decide/CAB permissions |
| Idempotency | `change_idempotency` | Operation **`change.create` only**. PK `(operation, requested_by, idempotency_key)` |
| Errors | `ChangeManagementErrorCode` | `VALIDATION_ERROR` `UNAUTHORIZED` `FORBIDDEN` `NOT_FOUND` `CONFLICT` `PROVIDER_UNAVAILABLE` `INTERNAL_ERROR` |
| Frontend | `ChangeManagementApi` / `GmudDetailPage` | create/list/get. No approve/reject control |
| Unique-error helper | `ledgerSubmission.ts` | Treats Postgres `23505` and SQLite unique/PK codes as contention |

### Confirmed schema capabilities (no new DDL required)

| Capability | Already present? | Mechanism |
|---|---|---|
| Multiple rounds per `changeId` | Yes | PK `(change_id, round_number)`; monotonic next-number check in `insertRound` |
| Per-round immutable Change snapshot | Yes | `change_authorization_rounds.change_snapshot_json` / sha256 |
| Current round without `isCurrent` | Yes | `ORDER BY round_number DESC LIMIT 1`; architecture tests forbid `isCurrent` |
| One terminal decision per requirement per round | Yes | `change_auth_decisions_requirement_uq` |
| Actor-scoped decision idempotency | Yes | `change_auth_decisions_idempotency_uq` |
| Append-only evidence | Yes | dialect UPDATE/DELETE triggers |
| Lifecycle status column | Yes | `change_index.status` string(32), written `'submitted'`, never updated |
| Second index row / snapshot version column | **No** | Index remains 1:1 with `changeId` |

### Laptop LEDGER facts independently re-read (read-only)

| changeId | mode | rounds | decisions | notes |
|---|---|---|---|---|
| `CHG-2026-000001` | `LEDGER_REQUIRED` | 1 sandbox | 1 historical | Untouched pre-existing sandbox |
| `CHG-2026-000002` | `LEGACY_PRE_F3` | 0 | 0 | Activation control |
| `CHG-2026-000003` | `LEDGER_REQUIRED` | 1 CAB-safe | **0** | Demo target |
| `CHG-2026-000004` | `LEGACY_PRE_F3` | 0 | 0 | Same-binary backout |

`CHG-2026-000003` Round 1:

- policy `default-change-authorization@2026-09-19.1`, matched `normal.low`
- `normal-primary-approval` → `user:default/diego.fernandes_outlook.com`
- `cab-approval` → `group:default/cloud_azure_devops_platform_devops`
- window `[2026-09-29T01:00:00.000Z, 2026-09-29T02:00:00.000Z)`
- Catalog `relations.memberOf` proves the signed-in user **is** a member of that CAB group
- This checkpoint created **zero** decisions

---

## 3. Objective and explicit non-goals

### Objective

Smallest correct decomposition such that, on a `LEDGER_REQUIRED` Change with a current round:

```text
authenticated actor
  -> permission + live decision-time authority
  -> one append-only ApprovalDecision per requirement per round
  -> exact replay returns the original logical result
  -> conflicting second decision is rejected
  -> derived AuthorizationEvaluation updates
  -> mandatory pre rejection materializes lifecycle rejected
  -> AUTHORIZED does not invent lifecycle authorized
  -> later correction keeps the same changeId and creates Round N+1
```

### Non-goals

- F3.1.4 permission-filtered authorization/governance detail representation
- CAB Workbench / autonomy grants / Teams / break-glass
- Additive requirements after submission
- Decision reversal, abstention, expiry
- Execution start/completion integration
- Production `LEDGER_REQUIRED` cutover
- Generic workflow/BPM
- Fabricating CAB membership or weakening actor checks for demo convenience

---

## 4. Primary architecture answer

F3.1.3 is **two micro-slices**. Decision recording and same-`changeId` resubmission share the ledger, but they are not the same persistence problem.

**F3.1.3a — server-authoritative decision command.** One nested HTTP command records one requirement decision. Individual authority is exact principal equality. CAB/authority authority is live Catalog membership against the snapshotted Group ref, plus a dedicated RBAC permission that is **not** implied by `platform_admin` or `change.read`. Persistence uses the existing append-only decision/audit tables. Concurrency serializes on `change_index` `FOR UPDATE` (PostgreSQL authoritative). Derived `AuthorizationEvaluation` is never stored. Mandatory pre-execution rejection materializes `ChangeStatus='rejected'` as a rebuildable index projection in the same transaction. Approval never changes lifecycle.

**F3.1.3b — rejection resubmission / new-round semantics.** Only after the current round is `REJECTED`. Same `changeId`, new monotonic round, new immutable snapshot on the round row, currently published policy/selectors, new requirements. Original `change.create` idempotency reservation is **not** reused. Index discovery columns become the rebuildable **current** projection; Round 1 keeps the birth snapshot. No DDL.

`Migration required: NO` for both slices.

---

## 5. Q1 — Exact decision command transport

**Resolved.**

### Route

```text
POST /api/change-management/changes/:changeId/rounds/:roundNumber/requirements/:requirementId/decisions
```

Mounted on the existing `change-management` plugin. User credentials only (not service).

This is the minimum transport needed to exercise commands on the accepted laptop product target. It is not a Teams/ADO contract and introduces no provider fields.

### Headers

| Header | Rule |
|---|---|
| `Idempotency-Key` | **Required.** Trimmed non-empty string. Missing/blank → `VALIDATION_ERROR` |
| Auth | Existing Backstage user session |

Unlike `POST /changes`, the key is not optional. Decision replay is a first-class contract.

### Body

```ts
{
  outcome: 'approved' | 'rejected';
  reason?: string;          // required non-empty trimmed when outcome=rejected
  comment?: string;         // optional
  cabMeetingRef?: string;   // optional; accepted only for kind cab|authority
}
```

Rejected fields: any Teams/ADO/provider identifier, `actorRef`, `actingAuthorityRef`, `decidedAt`, `decisionId`, `commandHash`, `authorizationEvidence`, `channel`, `correlationRef`. Server fills those.

### Command hash

```text
commandHash = sha256Canonical({
  changeId,
  roundNumber,
  requirementId,
  outcome,
  reason: reason ?? null,
  comment: comment ?? null,
  cabMeetingRef: cabMeetingRef ?? null,
})
```

Uses existing `sha256Canonical`. Actor and timestamps are **not** hashed; they are server-controlled and already bound by `(actor_ref, idempotency_key)`.

### Response

First commit: **201**

Exact replay: **200**

```ts
{
  decision: ApprovalDecision; // public: no internal table names
  authorizationEvaluation: 'PENDING' | 'AUTHORIZED' | 'REJECTED';
  changeStatus: 'submitted' | 'rejected';
  roundNumber: number;
}
```

`authorizationEvaluation` is derived at response time from current-round requirements after the transaction, not read from a stored evaluation row.

Exact replay returns the original `decisionId` / `decidedAt` / `commandHash` and does not append a second audit event.

---

## 6. Q2 — Individual decision authority

**Resolved.**

For `kind === 'individual'`:

```text
actor.userEntityRef == requirement.principalSnapshot.resolvedPrincipalRef
```

No governance/admin override exists in ADR-009 or ADR-013, and none is invented here. `platform_admin`, requester, `ownerRef`, and `responsibleRef` do **not** authorize an individual decision.

If the actor does not match: `FORBIDDEN` with `details.reason=not_requirement_principal`.

`actingAuthorityRef` is omitted. `authorizationEvidence` stores only:

```ts
{
  kind: 'individual',
  matchedPrincipalRef: actor.userEntityRef,
  snapshotSelectorKey: requirement.principalSnapshot.selectorKey,
}
```

---

## 7. Q3 — CAB / authority decision authority

**Resolved.**

For `kind === 'cab' | 'authority'`: one collective decision. Do **not** expand the Group into N member decisions.

### Authoritative membership source

Live Catalog membership of the **current authenticated User**, proven at decision time:

1. Load `catalog.getEntityByRef(actor.userEntityRef)` with **service** credentials (same pattern as `CatalogPrincipalResolver`).
2. Collect `relations[memberOf]` **and** `spec.memberOf` entity refs.
3. Membership holds iff that set includes `requirement.principalSnapshot.resolvedPrincipalRef`.

This is **not** `ownershipEntityRefs.includes(...)`. That helper filters to `group:default/cloud_azure_devops_*` and would silently fail closed for a future non-TP CAB group, or coincidentally pass on the laptop because the current CAB binding uses a TP group. Decision-time membership must be prefix-agnostic.

The historical principal snapshot names the authority identity. Current membership is live and is **not** taken from `resolvedAt`. Later Catalog changes do not rewrite the snapshot; they only affect who may act now.

### Failure behavior

| Condition | Result | Decision rows |
|---|---|---|
| Catalog / credentials error | `PROVIDER_UNAVAILABLE` | zero |
| User entity missing | `FORBIDDEN` `details.reason=actor_not_in_catalog` | zero |
| Not a current member of snapshotted authority | `FORBIDDEN` `details.reason=not_authority_member` | zero |

No retry-as-success. No cached membership.

### Persisted identity

- `actorRef` = authenticated `userEntityRef`
- `actingAuthorityRef` = `principalSnapshot.resolvedPrincipalRef`
- `authorizationEvidence`:

```ts
{
  kind: 'authority_membership',
  authorityRef: actingAuthorityRef,
  actorRef,
  membershipSource: 'catalog',
  relation: 'memberOf',
  provenAt: decidedAt, // server UTC
}
```

Optional `cabMeetingRef` from the command is stored on the decision when provided. It is supporting context, not membership proof.

Laptop proof does **not** fabricate membership: Catalog already has

`user:default/diego.fernandes_outlook.com memberOf group:default/cloud_azure_devops_platform_devops`.

That user may record the CAB decision only because they are a current member of the snapshotted authority **and** they hold the CAB-record permission below — not because they are `platform_admin`.

---

## 8. Q4 — Permission boundary

**Resolved.**

Literal minimum permissions (plugin prefix matches existing create/read):

| Permission name | Action | Authorizes |
|---|---|---|
| `change-management.change.authorization.decide` | `update` | Individual requirement decision command |
| `change-management.change.authorization.cab.record` | `update` | `cab` / `authority` collective decision command |

Both endpoints also require the matching permission **and** the Q2/Q3 actor proof.

### What is not granted

- `change-management.change.read` / `.create` never imply decide/CAB record.
- `role:default/platform_admin` does **not** receive `cab.record`.
- Participant read, `ownerRef`, `responsibleRef` remain read-only.

### RBAC CSV (minimum)

```text
p, role:default/contributor, change-management.change.authorization.decide, update, allow
p, role:default/platform_admin, change-management.change.authorization.decide, update, allow

p, role:default/change_cab_recorder, change-management.change.authorization.cab.record, update, allow
g, group:default/cloud_azure_devops_platform_devops, role:default/change_cab_recorder
```

The CAB permission is bound to a **new role** assigned to the configured CAB group, not folded into `platform_admin`. On the laptop those happen to be the same Entra group; that is selector configuration, not an RBAC collapse.

F3.1.4 may add audit-read / governance-read-all later. F3.1.3 decision endpoints are server-protected now.

Decision responses return the recorded decision and derived evaluation. They do not become the F3.1.4 composed authorization read model.

---

## 9. Q5 — Decision idempotency and concurrency

**Resolved.**

Database uniqueness is the authority. No polling, no distributed lock framework.

### Keys

- Terminal uniqueness: `(changeId, roundNumber, requirementId)`
- Replay uniqueness: `(actorRef, idempotencyKey)`

### Service algorithm (one caller-owned transaction)

1. `SELECT change_index WHERE change_id=? FOR UPDATE` (PostgreSQL; SQLite uses the same transaction without `FOR UPDATE`).
2. Load current round = max `round_number`. Fail if missing / not `LEDGER_REQUIRED` / round param ≠ current / requirement missing / round already terminal (except exact replay).
3. Authorize permission + Q2/Q3 **before** insert.
4. `SELECT` existing decision by requirement identity **and** by `(actorRef, idempotencyKey)`.
5. If either exists → compare `commandHash` + requirement identity + outcome; match → replay (200); else `CONFLICT`.
6. If neither exists → `appendDecision` + decision audit + derived milestone audit + optional lifecycle projection.
7. Commit.

Concurrent first-writers that both passed step 4: unique violation (`23505` / SQLite unique). The loser **rolls back**, re-reads the committed winner under a new transaction, and follows the replay/conflict rule. Do not `ON CONFLICT UPDATE`. Do not continue after a failed INSERT inside the aborted Postgres transaction.

### Matrix

| Case | Result |
|---|---|
| Same actor + same key + same hash | Exact replay; original decision; no second row; no second audit |
| Same key + changed command | `CONFLICT` `details.reason=idempotency_payload_mismatch` |
| Different key + same requirement | `CONFLICT` `details.reason=conflicting_decision` |
| Concurrent duplicate approve | One winner insert; loser observes replay |
| Concurrent approve vs reject | One terminal decision; loser `CONFLICT` `conflicting_decision` |
| Retry after network failure post-commit | Exact replay |
| Same actor + same key + different requirement | `CONFLICT` (idempotency unique is global per actor, not per requirement) |

Callers mint a fresh `Idempotency-Key` per requirement command.

---

## 10. Q6 — Decision transaction boundary

**Resolved.**

One caller-owned Knex transaction owns:

1. parent `change_index` lock;
2. `ApprovalDecision` insert;
3. `change.authorization.decision_recorded` audit;
4. at most one milestone audit (`authorization_reached` **or** `round_rejected`);
5. lifecycle projection `change_index.status='rejected'` when Q8 applies.

`appendDecision` / `appendAuditEvent` already accept `trx` and must be passed the outer transaction. They must not open a nested transaction.

Partial commit is forbidden. Unique-violation losers roll back the whole unit and re-observe.

No `AuthorizationEvaluation` row is written.

---

## 11. Q7 — Authorization transition audit

**Resolved.**

Event types follow the F3.1.2b namespace (`change.authorization.*`), not the pre-F3 sandbox `authorization.decision.recorded` string left on `CHG-2026-000001`.

| Event | When | Identity |
|---|---|---|
| `change.authorization.decision_recorded` | Every newly committed decision (not exact replay) | `actorRef` |
| `change.authorization.authorization_reached` | First time mandatory pre-execution requirements of **this round** are all approved | `systemRef=system:change-management` |
| `change.authorization.round_rejected` | First time any mandatory pre-execution requirement of **this round** is rejected | `systemRef=system:change-management` |

After the decision insert, still holding `FOR UPDATE`, re-read requirements (which already join decisions) and `evaluateAuthorization`.

- `AUTHORIZED` and no existing `authorization_reached` for this round → append exactly one.
- `REJECTED` and no existing `round_rejected` for this round → append exactly one + Q8 projection.
- `PENDING` → no milestone.
- Exact replay → no new events.

Because all decisions on a Change serialize on `change_index`, two requirements racing toward `AUTHORIZED` cannot both emit `authorization_reached`. PostgreSQL is the authoritative proof (D4).

Do not persist mutable evaluation state. Eligibility continues to derive on read.

### Eligibility safety (in F3.1.3a)

`EligibilityService.ensureRound` must **not** fabricate a sandbox Round 1 for a `LEDGER_REQUIRED` Change that lacks a round. Missing round → `NO_LEDGER_ROUND`. Decision commands never call `ensureRound`; they only load the existing current round.

---

## 12. Q8 — Change lifecycle on rejection

**Resolved.**

ADR-009: `submitted --mandatory pre rejection--> rejected`.

Source today cannot represent that value: `ChangeStatus='submitted'` only, and `change_index.status` is never updated. Leaving list/detail at `submitted` while claiming ADR-009 lifecycle compliance is forbidden.

F3.1.3a realizes `rejected` as a **rebuildable projection**, not a second authority:

- Canonical fact: the rejecting `ApprovalDecision` + `round_rejected` audit.
- Projection: `UPDATE change_index SET status='rejected'` in the same transaction.
- `ChangeStatus` TypeScript union becomes `'submitted' | 'rejected'`.
- `GET /changes` already reads index snapshot status → list shows rejected.
- `GET /changes/:changeId` currently returns `provider.get()` which would stay `submitted`. F3.1.3a **overlays** `status` from the index projection onto the public Change. Provider `record_json` is not rewritten (no operational mutation, no new provider method in 3a).
- `executing` / `completed` / `cancelled` remain unimplemented.

This does not violate append-only authorization evidence. Index `status` is already a mutable non-ledger column (`finalize()` already updates other index columns; only `authorization_mode` is trigger-immutable).

Approval / `AUTHORIZED` does **not** write lifecycle `authorized` and does not change `status`.

---

## 13. Q9 — Approval does not mutate lifecycle

**Resolved.**

Confirmed against ADR-009 and current eligibility:

- After both `CHG-2026-000003` pre requirements are approved, `evaluateAuthorization` → `AUTHORIZED`.
- `changeStatus` remains `submitted`.
- Eligibility uses current-round evaluation then the index/round window. The demo window is `[2026-09-29T01:00Z, 2026-09-29T02:00Z)`, so after authorization the live check remains `DENY` / `OUTSIDE_WINDOW` until that instant.
- No `authorized` lifecycle value is added to `ChangeStatus`.

---

## 14. Q10 — Post-execution requirements

**Resolved.**

Keep the decision command generic. Fail closed when the required execution-completion anchor does not exist.

`slaAnchor` in the published emergency policy is `execution_completion`. No accepted execution-start or execution-completion command, audit, or lifecycle fact exists in current source. `GovernanceEvaluation` already requires `executionCompletedAt` as an input that nothing produces.

Rule:

```text
if requirement.phase === 'post_execution'
AND no accepted execution-completion evidence exists for this changeId
-> CONFLICT details.reason=execution_completion_required
-> zero decision
```

Do not treat the requirement row itself as the governance anchor. Do not invent a completion fact. Do not defer to a second command shape. The later execution-lifecycle slice supplies the missing evidence; the same decision route then becomes usable without transport change.

Normal-low demo (`CHG-2026-000003`) has only pre-execution requirements, so this rule is not on the happy path.

---

## 15. Q11 — New round / resubmission semantics

**Resolved, isolated as F3.1.3b.**

Current source can store Round N without DDL (`insertRound` already demands `expectedRoundNumber = max+1` under `FOR UPDATE`). Current source **cannot** store a second index row or a versioned index snapshot. `change.create` idempotency is 1:1 with the original submission and must not be reused. `IChangeManagementProvider` has no replace API. `ChangeIndexRepository` cannot rebuild activity participants after a corrected plan.

Those are application/composition gaps, not missing ledger tables. Compressing them into the decision command would mix two review surfaces. They do **not** require a migration.

### Command (F3.1.3b only)

```text
POST /api/change-management/changes/:changeId/resubmissions
Idempotency-Key: <required, new key>
```

Body: same user-editable `CreateChangeHttpRequest` shape as create.

Permission: `change-management.change.create` **and** actor is requester **or** current `ownerRef` member **or** `platform_admin`. No CAB-only resubmit. No invented extra role.

### When allowed

Current round `evaluateAuthorization() === 'REJECTED'` (equivalently projected `changeStatus='rejected'`). Otherwise `CONFLICT` `details.reason=round_not_terminal`.

A new round may be created only after the prior round is terminal. `PENDING` / `AUTHORIZED` fail closed (R2).

### Identity / fields

| Field | Resubmission behavior |
|---|---|
| `changeId` | Unchanged |
| `requestedBy` | Unchanged (original requester) |
| `createdAt` | Unchanged (Change identity time) |
| User-editable create fields | May be corrected |
| `ownerRef` / `systemRef` | Re-resolved from new `targetRef` |
| `activityId`s | Newly minted |
| Round snapshot | New canonical JSON + sha256 on Round N |
| Policy / selectors | Currently published versions, new principal snapshots |
| Requirements | New immutable rows; `requirementId = requirementRole` |
| Prior rounds / decisions / audits | Untouched |

### Idempotency

New operation `change.resubmit`. New key. Payload hash covers `changeId` + corrected body. Original `change.create` reservation is never reused or mutated (`authorization_mode` remains immutable).

### Snapshot / discovery composition

- History authority: each round's `change_snapshot_*`.
- Current-round selection: max `round_number` (unchanged).
- Index discovery columns (title, summary, window, plan, owner/system, **status**) become the rebuildable **current** projection for `LEDGER_REQUIRED` Changes, updated in the resubmission transaction. This is a documented narrowing of ADR-007's F2 "birth snapshot is the list row" for ledger-governed **resubmission only**: list remains discovery, not live provider workflow, but discovery follows the current business snapshot of the same `changeId`.
- Round 1 snapshot remains the birth evidence.
- `change_index_activity_participants` is rebuilt from the new plan (derived, non-authoritative).
- `DevelopmentProvider`: add an explicit `replaceCurrent(change, trx)` used only by resubmission. Do not overload `create()` semantics. External/non-dev providers fail closed `PROVIDER_UNAVAILABLE` until a later provider slice.
- `GET` detail after resubmission returns the replaced operational record (dev) with `status='submitted'`.

ADR-007 owner/system "GET does not re-resolve" remains true for historical rounds; the new round snapshots the newly resolved refs.

### Transaction (F3.1.3b)

One caller-owned transaction:

1. lock `change_index`;
2. prove current round `REJECTED`;
3. reserve/complete `change.resubmit` idempotency;
4. materialize Round N + requirements + submission-like audits (`round_created`, `policy_selected`, `selector_bundle_bound`, `requirement_materialized`) plus `change.authorization.resubmitted`;
5. project index current snapshot + `status='submitted'`;
6. rebuild participants;
7. `DevelopmentProvider.replaceCurrent`.

`ledgerSubmission.ts` currently hardcodes `roundNumber: 1` in audit payloads. F3.1.3b must generalize to `round.roundNumber`. Do not create Round 2 from the F3.1.2b create path.

### Concurrency

| Case | Result |
|---|---|
| Two concurrent resubmissions after rejection | One next `round_number` wins (`FOR UPDATE` + PK + expected number); loser `CONFLICT` `details.reason=resubmission_conflict` (R1) |
| Resubmit while PENDING/AUTHORIZED | `CONFLICT` `round_not_terminal` (R2) |
| Prior decisions after Round 2 exists | Immutable; append-only triggers (R3) |

---

## 16. Q12 — Relationship to live demo target

**Resolved.**

Preserve the accepted laptop `LEDGER_REQUIRED` overlay. Do not flip committed `app-config.yaml`. Do not use production. PostgreSQL remains the concurrency proof target; the laptop SQLite runtime remains the product demonstration target.

### Happy path — `CHG-2026-000003` (do not reject this Change)

1. Signed-in `user:default/diego.fernandes_outlook.com` records `approved` on `normal-primary-approval` (Q2 match).
2. Same user records collective `approved` on `cab-approval` because Catalog `memberOf` the snapshotted CAB group **and** `change_cab_recorder` (Q3+Q4). No fabricated membership.
3. Derived evaluation `AUTHORIZED`.
4. `changeStatus` remains `submitted`.
5. `GET .../execution-eligibility` remains `DENY` / `OUTSIDE_WINDOW` (window 2026-09-29).
6. Zero extra rounds. Historical `CHG-2026-000001` decision untouched.

If a future selector rebinding makes this user unable to act for CAB, stop and use a governed non-prod selector republication — never weaken Q3.

### Rejection / resubmission path — disposable new non-prod Change

Create a fresh laptop `LEDGER_REQUIRED` Change (not 000003). Reject a mandatory pre requirement with a reason. Prove: one decision, `round_rejected`, `changeStatus=rejected`, eligibility `DENY/REJECTED`, list/detail overlay rejected, no `authorized` lifecycle. F3.1.3b then resubmits that disposable Change to Round 2 and proves history of Round 1 is intact.

This planning checkpoint created neither demo decision.

---

## 17. Required PostgreSQL concurrency proofs

SQLite may provide functional coverage. PostgreSQL is authoritative.

| ID | Proof |
|---|---|
| **D1** | Exact decision replay: same actor/key/hash → original `decisionId`, still one row, still one `decision_recorded` |
| **D2** | Concurrent duplicate approval: one insert winner; loser replay |
| **D3** | Concurrent approve vs reject on the same requirement: exactly one terminal decision; loser deterministic `CONFLICT` `conflicting_decision` |
| **D4** | Two different requirements racing to `AUTHORIZED`: exactly one `authorization_reached`; evaluation `AUTHORIZED`; lifecycle still `submitted` |
| **D5** | Authority membership failure **or** Catalog unavailable: `FORBIDDEN` / `PROVIDER_UNAVAILABLE`; zero decision; zero audit |
| **D6** | Mandatory pre rejection: one decision + `round_rejected` + `change_index.status='rejected'` atomically; eligibility `REJECTED` |
| **R1** | Two concurrent resubmissions after rejection: at most one Round N+1 |
| **R2** | Resubmission while current round non-terminal: zero new round |
| **R3** | After Round 2, Round 1 decisions/requirements/audits unchanged (UPDATE/DELETE still blocked) |

Reuse the existing disposable Postgres 16 harness (`CHANGE_MANAGEMENT_TEST_POSTGRES_URL` / `authorization/postgres.test.ts` pattern).

---

## 18. Error taxonomy

No new HTTP codes. Discriminate with `details.reason` / existing `details` keys (`changeId`, `idempotencyKey`, plus `roundNumber`, `requirementId` as needed).

| Situation | Code | HTTP | `details.reason` |
|---|---|---|---|
| Malformed body / missing Idempotency-Key / rejection without reason / `cabMeetingRef` on individual | `VALIDATION_ERROR` | 400 | field / `rejection_reason_required` |
| Unauthenticated | `UNAUTHORIZED` | 401 | (existing; unused if plugin keeps credential gate) |
| Missing decide/CAB permission | `FORBIDDEN` | 403 | (NotAllowedError mapping) |
| Individual actor ≠ snapshot | `FORBIDDEN` | 403 | `not_requirement_principal` |
| CAB/authority not current member | `FORBIDDEN` | 403 | `not_authority_member` |
| Actor user missing from Catalog | `FORBIDDEN` | 403 | `actor_not_in_catalog` |
| Change missing | `NOT_FOUND` | 404 | `changeId` |
| Round missing | `NOT_FOUND` | 404 | `round_not_found` |
| Requirement missing | `NOT_FOUND` | 404 | `requirement_not_found` |
| Catalog/membership source unavailable | `PROVIDER_UNAVAILABLE` | 503 | `membership_source_unavailable` |
| Exact replay | 200 + original body | — | — |
| Conflicting second decision | `CONFLICT` | 409 | `conflicting_decision` |
| Same key, different hash | `CONFLICT` | 409 | `idempotency_payload_mismatch` |
| LEGACY_PRE_F3 / no ledger round | `CONFLICT` | 409 | `ledger_required_only` |
| Decision on non-current round | `CONFLICT` | 409 | `stale_round` |
| New decision after round `REJECTED`/`AUTHORIZED` terminal for new facts | `CONFLICT` | 409 | `terminal_round` |
| Post-execution without completion evidence | `CONFLICT` | 409 | `execution_completion_required` |
| Resubmission before terminal rejection | `CONFLICT` | 409 | `round_not_terminal` |
| Concurrent resubmission lost | `CONFLICT` | 409 | `resubmission_conflict` |
| Ledger/index invariant mismatch | `INTERNAL_ERROR` | 500 | (no repair) |

`terminal_round` applies to **new** decisions on a round whose evaluation is already `REJECTED` (or, for pre-execution, already `AUTHORIZED` only if the requirement is already decided — undecided post-execution remains Q10). Remaining undecided pre requirements on a rejected round stay undecided; later approval cannot repair the round.

---

## 19. Slice decomposition

```text
F3.1.3a — server-authoritative decision command
F3.1.3b — rejection resubmission / new-round semantics
```

One combined slice is rejected: lifecycle/index/provider current-snapshot composition and a new idempotency operation are materially different from append-only decision recording, and they would force reviewers to accept GET/list semantics in the same gate as CAB membership.

F3.1.3a **does** include rejection lifecycle projection (Q8/D6). It does **not** include resubmission.

Implementation of either slice remains **NO-GO** until this plan is independently accepted and a separate implementation prompt is authorized. Prefer implementing 3a first against the laptop demo; 3b stays gated on 3a ACCEPT.

---

## 20. UI boundary

Default holds.

| Slice | Frontend |
|---|---|
| F3.1.3a | Expand `ChangeStatus` + `STATUS_LABELS.rejected = 'Rejeitada'` so list/detail do not blank-chip after D6. **No** approve/reject button, no CAB Workbench, no F3.1.4 requirement/decision panel. Optional `ChangeManagementClient.recordDecision` for HTTP product tests only |
| F3.1.3b | Same labels; resubmission is API-only |

F3.1.4 owns composed authorization/governance representation and permission-filtered detail.

---

## 21. Migration decision

```text
Migration required: NO
```

Existing ledger uniqueness, append-only triggers, round PK, and `change_index.status` already satisfy the contract. F3.1.3b updates index projection columns in place and inserts Round N; it does not add snapshot-version columns, `isCurrent`, or a second index row.

No migration "for future proofing". No CHECK expansion on `change_index.status` in this slice.

---

## 22. Expected source paths

### F3.1.3a

| Area | Paths |
|---|---|
| Domain/types | `packages/backend/src/modules/changeManagement/types.ts`; `authorization/types.ts` (command DTO only if needed) |
| Ledger repository | `AuthorizationLedgerRepository.ts`; `KnexAuthorizationLedgerRepository.ts` — add read helpers `findDecisionByRequirement` / `findDecisionByIdempotency`; no insert-API change |
| Service | `ChangeManagementService.ts` (or a dedicated `DecisionCommandService` constructed by the plugin and used by the service). Architecture guard must allow `appendDecision` **only** on the decision path |
| Index projection | `ChangeIndexRepository.ts`; `KnexChangeIndexRepository.ts`; `changeIndexMapper.ts` — `projectLifecycleStatus` |
| Membership | new `authorization/decisionMembership.ts` (Catalog live memberOf); do **not** reuse TP-prefix `entraOwnership` as the CAB proof |
| Router/plugin | `packages/backend/src/plugins/changeManagementPlugin.ts` |
| Permissions/RBAC | `permissions.ts`; `packages/backend/config/rbac/rbac-policy.csv` |
| Eligibility safety | `authorization/EligibilityService.ts` — no sandbox round fabrication for `LEDGER_REQUIRED` |
| Frontend | `plugins/change-management/src/model/types.ts`; optional API client method |
| Tests | new decision command tests; extend `KnexAuthorizationLedgerRepository.test.ts`; **PostgreSQL** D1–D6 in `authorization/postgres.test.ts` or sibling; RBAC/membership fail-closed tests; architecture guard update |
| Migrations | **none** |

### F3.1.3b

| Area | Paths |
|---|---|
| Domain/types | `types.ts` already has `rejected` from 3a |
| Ledger / policy | `authorization/ledgerSubmission.ts` generalized to Round N; `ChangeManagementService` resubmit |
| Router/plugin | `POST /changes/:changeId/resubmissions` |
| Provider/index | `DevelopmentProvider.replaceCurrent`; participant rebuild; index current-projection update |
| Idempotency | `change.resubmit` operation via existing `IdempotencyRepository` |
| Frontend | none required beyond 3a labels |
| Tests | R1–R3 PostgreSQL; history preservation; non-terminal fail-closed |
| Migrations | **none** |

Provider/ADO/Teams canonical fields remain forbidden.

---

## 23. Acceptance gates

Implementation of a slice is acceptable only when all of the following hold for that slice's scope:

1. **Source lineage** — child of accepted `2249522` on `feat/ado-repo-governance`; committed default still `LEGACY_PRE_F3`.
2. **Decision authorization** — individual equality only; no admin override.
3. **CAB membership** — live Catalog memberOf against snapshotted Group; unavailable source fails closed with zero decision; no member expansion.
4. **Server-side permissions** — decide vs cab.record vs read vs `platform_admin` separated as specified.
5. **Idempotency / conflict** — D1–D3.
6. **Transaction atomicity** — decision + required audits + rejection projection commit together or not at all.
7. **Immutable audit** — UPDATE/DELETE still blocked; replay does not duplicate events.
8. **Evaluation transitions** — derived only; D4 single `authorization_reached`.
9. **Rejection lifecycle** — D6; list/detail show `rejected`; no `authorized` lifecycle value.
10. **New-round monotonicity / history** — F3.1.3b: R1–R3; same `changeId`; prior evidence intact.
11. **Post-execution timing** — fail closed without completion evidence.
12. **PostgreSQL concurrency** — D1–D6 (3a) and R1–R3 (3b) pass on disposable Postgres 16.
13. **Live laptop product proof** — overlay remains `LEDGER_REQUIRED`; `CHG-2026-000003` happy path as in Q12; disposable rejection Change for D6; no fabricated CAB membership.
14. **GMUD / Catalog / Deployments regressions** — create/list/detail, Catalog, Deployments tab, Delivery read remain intact.
15. **Lint / build / TypeScript baseline** — no new repo-wide `tsc` debt beyond the accepted baseline set.
16. **No F3.1.4 / F3.2 leakage** — no Workbench, no autonomy grant, no composed authorization UI, no Teams, no production cutover.

---

## 24. Risks and rejected alternatives

| Alternative | Decision |
|---|---|
| One F3.1.3 slice for decisions + resubmission | Rejected — different blast radius (index/provider current snapshot vs append-only decision) |
| Leave `ChangeStatus='submitted'` after rejection | Rejected — contradicts ADR-009 Q8; isolated as 3a projection, not ignored |
| Migration adding snapshot_version / `isCurrent` | Rejected — rounds already version snapshots; current = max round |
| Reuse `change.create` key for resubmission | Rejected — reservation is the original logical create and its immutable mode |
| `ownershipEntityRefs.includes(cabGroup)` as CAB proof | Rejected as the authoritative check — TP prefix is accidental laptop fit |
| Grant `cab.record` to `role:default/platform_admin` | Rejected — ADR-013 |
| Persist `AuthorizationEvaluation` | Rejected — competing authority |
| Allow post-execution decision because the requirement row exists | Rejected — Q10 |
| Approve/reject UI / CAB inbox in F3.1.3 | Rejected — F3.1.4 / Workbench |
| Weaken CAB check so the laptop user can demo | Rejected — Catalog already proves membership |
| `ON CONFLICT UPDATE` for replay | Rejected — append-only |

---

## 25. Explicit decisions checklist

| # | Decision | Resolution |
|---|---|---|
| 1 | Transport | Nested POST decisions route; required Idempotency-Key |
| 2 | Individual authority | Exact `actorRef == resolvedPrincipalRef`; no override |
| 3 | CAB authority | Live Catalog memberOf + dedicated permission |
| 4 | Permissions | `...authorization.decide` and `...authorization.cab.record`; new `change_cab_recorder` role |
| 5 | Idempotency | Existing uniques; select-then-insert; unique-loser re-observe |
| 6 | Transactions | Caller-owned trx + `change_index` FOR UPDATE |
| 7 | Audit | `decision_recorded` / `authorization_reached` / `round_rejected`; no stored evaluation |
| 8 | Rejection lifecycle | Index status projection + GET overlay in 3a |
| 9 | AUTHORIZED lifecycle | Unchanged `submitted` |
| 10 | Post-execution | Generic command; fail closed without completion evidence |
| 11 | Resubmission | F3.1.3b; same changeId; new operation/key; no DDL |
| 12 | Migration | NO |
| 13 | Slices | F3.1.3a + F3.1.3b |
| 14 | UI | Labels only; no decision controls |
| 15 | Demo | Laptop LEDGER_REQUIRED; `CHG-2026-000003` happy path; disposable rejection Change |
| 16 | Concurrency authority | PostgreSQL |

---

## 26. GO / NO-GO

```text
F3.1.3 planning: READY_FOR_REVIEW
F3.1.3 implementation: NO-GO
F3.1.3a implementation prompt authoring: NO-GO pending independent ACCEPT
F3.1.3a implementation: NO-GO
F3.1.3b implementation: NO-GO
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
Production cutover: NOT AUTHORIZED
```

**Next gate:** independent architecture review of this plan against ADR-009 / ADR-013 and ADO `2249522`. Only an `ACCEPT` may authorize authoring the constrained F3.1.3a implementation prompt.

---

## 27. STOP

STOP. Do not modify ADO implementation, do not create `ApprovalDecision` facts, do not implement F3.1.3/F3.1.4/F3.2, and do not author an implementation prompt from this checkpoint.
