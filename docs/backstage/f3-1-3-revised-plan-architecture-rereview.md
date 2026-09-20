# F3.1.3 — Revised Plan Architecture Re-Review

## 1. Status / verdict

```text
F3.1.3 revised plan architecture re-review: ACCEPT
F3.1.3 plan: ACCEPTED IMPLEMENTATION CONTRACT
ADR-014: ACCEPTED / governing F3.1.3b authority + identity
Migration required: NO
F3.1.3a implementation-prompt authoring: GO
F3.1.3a implementation: still requires separate explicit authorization
F3.1.3b implementation-prompt authoring: NO-GO until F3.1.3a independent acceptance
F3.1.3b implementation: NO-GO
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
Production cutover: NOT AUTHORIZED
```

The narrowly revised F3.1.3 plan is now an implementation-ready contract.
ADR-014 closes the two F3.1.3b blockers from the historical 18/20 REJECT.
The 18 previously passing gates remain materially valid. F3.1.3a
decision-command contracts were not redesigned. `Migration required: NO`
remains source-correct at ADO `2249522`.

This review is documentation-only. No ADO code was modified. No
`ApprovalDecision` fact was created. No implementation prompt was authored.

---

## 2. Reviewed docs + ADO baselines

| Item | Value |
|---|---|
| Docs fetched (`main`) | `diegofernandes-dev/backstage-docs@338a25197f612d7519fdece95d7bfd9fb5e6f308` |
| Plan under review | `docs/backstage/f3-1-3-implementation-plan.md` (revision `f90f524`) |
| Historical REJECT | `docs/backstage/f3-1-3-plan-architecture-review.md` (preserved) |
| Governing ADR | `docs/adr/ADR-014-change-resubmission-authority-and-identity-boundary.md` |
| Also read | ADR-007, ADR-008, ADR-009, ADR-013, F3.1.2 non-prod LEDGER activation ACCEPT, current-state, implementation-progress, prompts/README, this prompt |
| Review prompt | `prompts/f3-1-3-revised-plan-architecture-rereview.md` |
| ADO expected SHA | `22495229502dabf2d99588599a156d862c5114fa` |
| ADO branch tip verified | `feat/ado-repo-governance` `objectId` = `22495229502dabf2d99588599a156d862c5114fa` |
| Independent source verification | **YES** |
| Source drift after `2249522` | **NONE** |

SSH `git fetch` to Azure DevOps failed in-session. Remote tip was confirmed
independently via `az repos ref list` (`diegolab` / `platform-devops` /
`platform-devops-developer-portal`, `refs/heads/feat/ado-repo-governance`).
Local HEAD already contained that exact SHA.

Working tree (ADO): untracked `.vscode/` only. Overlay and SQLite remain
gitignored. Committed `app-config.yaml` still
`newSubmissionAuthorizationMode: LEGACY_PRE_F3`.

Revision scope vs historical REJECT (`f9ffe98` → `f90f524`): F3.1.3b Q11
authority/identity, Q4 resubmit permission row, R4–R13, and necessary
cross-references/tests only. Nested decision POST, individual equality, live
CAB + dedicated `cab.record`, trx-aware ledger reads, milestones, rejection
projection, eligibility `ensureRound` safety, post-execution fail-closed,
D1–D6, and `Migration required: NO` were not rewritten.

---

## 3. Independent source-verification scope

Inspected at exact SHA `2249522`:

| Surface | Confirmation |
|---|---|
| HTTP | `changeManagementPlugin.ts` — create/list/get/eligibility only; no decision/resubmit route |
| Service | `createChange` / `listChanges` / `getChange`; architecture guard forbids `appendDecision` |
| Status | `ChangeStatus = 'submitted'` only (backend + frontend `STATUS_LABELS`) |
| Index | `finalize()` updates `is_finalized` / `external_*` only; no status/snapshot rewrite method; `authorization_mode` trigger-immutable; `status` string(32) with **no CHECK**; identity columns `target_ref` / `owner_ref` / `system_ref` / `requested_by` / `created_at` already present |
| Provider | `IChangeManagementProvider` = `create`/`get`; `DevelopmentProvider.createWithTransaction` upserts `record_json` when `change_id` exists — `replaceCurrent` is still required and must not reuse that upsert |
| Ledger writes | `createRound(trx?)`, `appendDecision(trx?)`, `appendAuditEvent(trx?)` |
| Ledger reads | `findRound` / `findCurrentRound` / `listRequirements` / `listAuditEvents` always use `this.knex` |
| Round insert | Postgres `FOR UPDATE` on `change_index`; expected `round_number = max+1`; PK `(change_id, round_number)` |
| Decision storage | unique `(change_id, round_number, requirement_id)`; unique `(actor_ref, idempotency_key)`; rejection CHECK; **no** unique on audit `(change_id, round_number, event_type)` |
| Permissions | only `.create` / `.read`; `createPermission` + `permissionsRegistry.addPermissions` + Casbin CSV already used; delivery already registers `update` actions |
| RBAC CSV | contributor + `platform_admin` get create/read; CAB group also has `platform_admin`; no decide/cab.record/resubmit yet |
| `template_executor` | seeded create/read only; plan may add resubmit as optional technical grant |
| Membership | `ownershipEntityRefs` = user ref + Entra TP groups prefixed `group:default/cloud_azure_devops_` only |
| `canReadChange` | requester / owner via `ownershipEntityRefs` / responsibleRef / `platform_admin` — **read only** |
| Selector guard | `architecture.test.ts` forbids `memberOf` / `getEntities` / `.relations` in selector sources; live membership must stay in new `decisionMembership.ts` |
| Execution plan | `executionPlanValidator.ts` checks activity `targetRef` kind `Component` + Catalog existence only; no System bind |
| Target context | `resolveSystemRef` / `resolveOwnerRef` from `spec.system` / `spec.owner`; omitted → `undefined` |
| Create HTTP body | `CreateChangeHttpRequest` user-editable fields include `targetRef`; `requestedBy` / `ownerRef` / `systemRef` / `createdAt` are server-owned |
| Idempotency | operation varchar(64); PK `(operation, requested_by, idempotency_key)`; `reserve(trx?)` exists; `change.create` only today |
| Errors | existing codes cover planned 3a/3b taxonomy via `details.reason` |
| Eligibility | `ensureRound` still fabricates sandbox Round 1 when no round exists |
| Round-1 audits | `ledgerSubmission.ts` hardcodes `roundNumber: 1` |
| GET detail | `provider.get()` with no index-status overlay |

Source drift after accepted F3.1.2: **NONE**. Decision/resubmission/index/provider/Catalog identity/RBAC semantics are unchanged from the historical review inventory.

---

## 4. Regression of the 18 previously passing gates

| Gate | Result | Re-review finding |
|---|---|---|
| G1 Source inventory | **PASS** | Plan P1 inventory still matches `2249522`. No invented API. Trx-aware read gap, create-only routes, upsert-shaped provider, and missing resubmit permission remain correctly identified. |
| G2 Slice decomposition | **PASS** | 3a decision command vs 3b same-`changeId` resubmission remain distinct. 3b still gated on 3a. No F3.1.4/F3.2 smuggling. |
| G3 Decision transport | **PASS** | Nested POST, required Idempotency-Key, server-owned actor/hash/evidence, 201/200 unchanged. |
| G4 Individual authority | **PASS** | Exact principal equality. No requester/owner/`platform_admin` override. |
| G5 CAB membership | **PASS** | Live Catalog `memberOf` against snapshotted Group; prefix-agnostic; fail-closed; dedicated `cab.record`; helper stays outside selector sources. |
| G6 Permission/RBAC | **PASS** | Distinct `decide` vs `cab.record`. `platform_admin` still does not receive `cab.record`. New `change.resubmit` is a 3b row and does not collapse 3a CAB/decision authority into admin. |
| G7 Decision idempotency | **PASS** | Requirement unique + actor-scoped key; unique-loser rollback then re-observe. Unchanged. |
| G8 Transaction / trx-aware read | **PASS** | Caller-owned txn + `change_index FOR UPDATE` + optional `trx` on named reads. No DDL. Catalog I/O outside the lock. |
| G9 Milestone audit | **PASS** | Exactly-once via parent lock + trx-aware exists-check. No new unique index. |
| G10 Rejection lifecycle | **PASS** | Index `status='rejected'` projection; GET status overlay only in 3a; no `authorized` lifecycle value. |
| G11 Eligibility safety | **PASS** | 3a still disables sandbox fabrication for `LEDGER_REQUIRED` (`NO_LEDGER_ROUND`). Decision commands still never call `ensureRound`. |
| G12 Post-execution timing | **PASS** | Generic command; `execution_completion_required` fail-closed; no completion surrogate invented. |
| G15 Provider replacement | **PASS** | `replaceCurrent(change, trx)` remains required. Revision adds the identity-copy invariant; it does not reopen the Model C composition. |
| G16 Resubmission idempotency | **PASS** | `change.resubmit` still fits varchar(64); actor-scoped PK; original `change.create` untouched; loser `resubmission_conflict`. Hash now explicitly normalized — compatible, not a 3a change. |
| G17 Round terminality | **PASS** | Resubmit only when `evaluateAuthorization() === 'REJECTED'`. Unchanged. |
| G18 Migration | **PASS** | See §8. Existing uniqueness, append-only triggers, round PK, unconstrained `change_index.status`, and idempotency PK already satisfy 3a + revised 3b. |
| G19 PostgreSQL proofs | **PASS** | D1–D6 and R1–R3 remain. R4–R13 are additive identity/authority proofs; R1 stays the concurrent PostgreSQL authority. |
| G20 Live product / scope | **PASS** | `CHG-2026-000003` happy path unchanged. Rejection/resubmission still uses a disposable Change. F3.1.4/F3.2/Teams/production remain excluded. |

**Previously passing gates regression: PASS (18/18).**

G13 and G14 are re-reviewed below; they are no longer FAIL.

---

## 5. Blocker G13 re-review — resubmission actor authority

ADR-014 and revised Q11 now require **both**:

```text
actor has change-management.change.resubmit
AND
(actorRef == original requestedBy
 OR actor is a current member of immutable ownerRef)
```

### Source implementability

- Dedicated permission can be registered with the existing
  `createPermission({ name, attributes: { action: 'update' } })` +
  `permissionsRegistry.addPermissions` + Casbin CSV pattern. Delivery already
  uses `update`. `change.create` is not reused.
- Original requester still needs the explicit permission (R4 negative).
- Immutable-owner member still needs the explicit permission (R5).
- Owner proof is live Catalog `memberOf` (`relations.memberOf` and
  `spec.memberOf`), prefix-agnostic, **not** `ownershipEntityRefs`.
- Same new `decisionMembership.ts` as CAB; selector-source expansion remains
  forbidden.
- Membership source unavailable → `PROVIDER_UNAVAILABLE` before the write
  transaction.
- Missing/unresolvable immutable owner Group → `FORBIDDEN`
  `owner_authority_unresolvable`.
- No-owner Change: owner path cannot succeed; only original requester +
  permission.
- CSV may grant the literal capability to `contributor` / `platform_admin`
  (and optionally `template_executor`) **only because** Q11 still enforces
  requester-or-immutable-owner proof.
- `change_cab_recorder` is not granted resubmit as a CAB power.

### Explicit non-authorities (not standalone)

| Candidate | Result |
|---|---|
| `platform_admin` alone | Denied (R6). Technical CSV grant is not business authority. |
| CAB membership / `cab.record` alone | Denied (R7). |
| Participant read / `responsibleRef` / `change.read` | Denied. ADR-009 read-only. |
| `change.create` | Denied. Create authorizes a new identity. |
| `change.authorization.decide` | Denied. |
| Generic governance override | None exists. |

### Required challenges

| # | Challenge | Answer |
|---|---|---|
| 1 | Requester has no resubmit permission | `FORBIDDEN`; zero Round (R4). Permission layer is mandatory even for the original submitter. |
| 2 | `platform_admin` has permission but is neither requester nor immutable-owner member | `FORBIDDEN` `not_resubmission_actor` (R6). Laptop CAB/`platform_admin` group coincidence does not collapse this proof. |
| 3 | CAB member has permission but neither domain proof | `FORBIDDEN` (R7). `cab.record` is a different capability. |
| 4 | Original owner membership is removed before resubmit | Live membership fails; `FORBIDDEN`; zero Round (R5). Snapshot `ownerRef` remains historical identity, not a stale membership grant. |
| 5 | Catalog ownership drifts to a new owner | New Catalog owner alone cannot resubmit (R9). Frozen `ownerRef` remains Team A. |
| 6 | Immutable owner Group disappears from Catalog | Fail closed `owner_authority_unresolvable`; transaction never mutates. Requester + permission remains the only path if the Change still has that frozen ref and the actor is the original requester. |

If `ownerRef` is a User rather than a Group (source `resolveOwnerRef` can
return a fully-qualified non-Group owner), the owner-membership path fail-closes
as unresolvable Group membership. Only the original requester with permission
may resubmit. That is conservative and consistent with ADR-014's "ownerRef
group" wording, not a grant of extra authority.

Requester equality does not itself require Catalog I/O. Plan Q11 step 3 is
owner-membership Catalog I/O. An implementer must not treat Catalog downtime
as a requester-path business-authority change.

**G13 resubmission authority: PASS.** No actor-authority ambiguity remains.

---

## 6. Blocker G14 re-review — same-changeId identity boundary

ADR-014 freezes across rounds:

- `changeId`
- original `requestedBy`
- identity `createdAt`
- `targetRef`
- original snapshotted `ownerRef`
- original snapshotted `systemRef`

### Source-correct boundary

- Same-`changeId` correction cannot retarget. Body `targetRef` must equal the
  original via `parseEntityRef` / `stringifyEntityRef`. Mismatch →
  `VALIDATION_ERROR` `change_identity_mismatch` **before** Round / provider /
  index mutation (R10).
- Changed target / owner / System requires a new Change / new `changeId`.
- Server must **not** re-resolve `ownerRef` / `systemRef` from Catalog on
  resubmit. `CreateChangeHttpRequest` does not even accept those fields;
  R11 is the server identity freeze, not a client-writable owner field.
- Catalog ownership drift does not rewrite existing Change identity and does
  not transfer resubmission rights (R9 + ADR-007 snapshot rule).
- Allowed non-identity corrections match the current create contract: title,
  summary, classification, risk, window, rollback, evidence, execution-plan
  content with new `activityId`s (R12).
- New Round uses currently published policy/selectors/principal snapshots.
  Prior Round snapshots remain immutable ledger facts (append-only triggers).
- Index current projection updates non-identity columns only (`title`,
  `summary`, window, plan, status, classification/risk/rollback/evidence).
  Identity columns stay the original values. `authorization_mode` remains
  trigger-immutable.
- `DevelopmentProvider.replaceCurrent` must copy frozen identity from the
  original Change. Reusing `create()` / `createWithTransaction()` remains
  forbidden because those methods upsert arbitrary `record_json`.
- Detail/list remain coherent under Model C: Round history is immutable;
  index is rebuildable **current non-identity** discovery; provider is current
  operational detail with frozen identity. This is the documented narrowing of
  ADR-007's F2 "birth snapshot is the list row" already accepted as G15, now
  with an enforceable identity line.

### Execution-plan hidden retarget

Source today allows optional activity `targetRef` of kind `Component` and does
not bind it to `Change.systemRef`. ADR-014's guard is source-correct:

- omitted activity `targetRef` remains allowed (ADR-008);
- present activity Component resolves `systemRef` with the same `spec.system`
  rules as `targetContextResolver.ts`;
- resolved System must equal immutable Change `systemRef`, including both
  absent;
- cross-System, or introducing a System where the Change identity has none, is
  `VALIDATION_ERROR` `execution_plan_identity_mismatch` before mutation;
- activity `targetRef` never rewrites Change identity.

Create-path validation remains unconstrained; that is an accepted ADR-014 cost,
not a hidden 3b retarget.

### Required challenges

| # | Challenge | Answer |
|---|---|---|
| 1 | Changed Change `targetRef` with otherwise identical payload | `VALIDATION_ERROR` `change_identity_mismatch` before mutation (R10). Operator creates a new Change instead. |
| 2 | Catalog target owner changed from Team A to Team B | Identity/authority stay Team A. Team B membership alone cannot resubmit (R9). Index/provider identity columns stay Team A / original System. |
| 3 | Corrected execution activity points to Component in another System | `execution_plan_identity_mismatch` (R13). Zero Round/provider/index mutation. |
| 4 | Corrected activity Component has no System while Change has one | Mismatch (absent ≠ present). Denied before mutation. |
| 5 | Change has no System and corrected activity introduces one | Denied. Both-absent is the only matching empty pair. |
| 6 | Title/risk/window/rollback/plan-only correction | Creates Round N+1; frozen identity preserved on index, provider, and new round snapshot (R12). |

**G14 same-change identity: PASS.** 'Same business Change' now has an
enforceable source-level boundary.

---

## 7. ADR-014 quality gate

| Requirement | Result |
|---|---|
| Explicitly fills the ADR-009 gap rather than silently reinterpreting old text | **PASS** — ADR-009 named the transition in the passive voice and Q19's new-`changeId` rule; it did not name actors or freeze identity fields. ADR-014 records that gap. |
| Consistent with ADR-007 snapshot authority and ADR-008 execution-plan semantics | **PASS** — owner/system remain snapshots; Catalog drift does not rewrite history; omitted activity `targetRef` stays allowed; present Component targets stay kind `Component`. |
| Does not grant CAB autonomy or reopen ADR-013 | **PASS** — CAB/`cab.record` explicitly non-authority; ADR-013 not superseded. |
| Does not authorize implementation by itself | **PASS** — §8 and Gate keep F3.1.3 implementation NO-GO until this plan ACCEPT + later prompt. |
| Defines positive authority and explicit non-authorities | **PASS** |
| Defines Catalog ownership drift semantics | **PASS** |
| Defines retargeting as new Change / new `changeId` | **PASS** |
| Avoids provider-specific identities | **PASS** |

**ADR-014 coherence: PASS.**

---

## 8. Resubmission transaction / idempotency / migration

Revised 3b composes with the already-passing mechanics:

1. Permission + identity match + owner-membership Catalog I/O **before** the
   write transaction.
2. Identity / execution-plan System mismatch fails before Round / provider /
   index mutation.
3. One caller-owned transaction then locks `change_index`.
4. Current round must be `REJECTED`.
5. `change.resubmit` idempotency reservation uses authenticated actor + new
   key; payload hash includes `changeId` + normalized corrected body.
6. Round N + requirements + audits + index **non-identity** projection +
   participant rebuild + `replaceCurrent` commit atomically.
7. Identity columns remain untouched. `authorization_mode` cannot change.
8. Concurrent resubmission loser maps to `resubmission_conflict`.
9. Original `change.create` reservation is untouched.

No new DDL is required:

- `change_idempotency.operation` is already varchar(64);
- round PK and monotonic insert already exist;
- `change_index.status` is unconstrained string(32);
- identity and non-identity index columns already exist;
- participant table already exists for rebuild;
- ledger append-only triggers already protect Round history;
- Casbin CSV / permission registry need no schema.

**Migration required: NO.**

---

## 9. Revised proof matrix R4–R13

| ID | Sufficiency |
|---|---|
| R4 | Requester + permission success; requester without permission denied |
| R5 | Immutable-owner member + permission success; removed membership denied |
| R6 | `platform_admin` alone denied |
| R7 | CAB-only actor denied |
| R8 | Permission without requester/owner proof denied |
| R9 | New Catalog owner cannot take over existing Change |
| R10 | Changed `targetRef` denied before mutation |
| R11 | Immutable owner/system mismatch denied before mutation |
| R12 | Allowed correction creates Round N+1 with frozen identity |
| R13 | Hidden cross-System execution-plan retarget denied |

R1 remains the PostgreSQL concurrent authority. R4–R13 may be functional
SQLite/Postgres tests. Internally consistent with ADR-014 and Q11.

**R4–R13 proof matrix: PASS.**

---

## 10. Decision

```text
F3.1.3 revised plan architecture re-review: ACCEPT
F3.1.3 plan: ACCEPTED IMPLEMENTATION CONTRACT
ADR-014: ACCEPTED / governing F3.1.3b authority + identity
Migration required: NO
F3.1.3a implementation-prompt authoring: GO
F3.1.3a implementation: still requires separate explicit authorization
F3.1.3b implementation-prompt authoring: NO-GO until F3.1.3a independent acceptance
F3.1.3b implementation: NO-GO
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
Production cutover: NOT AUTHORIZED
ADO implementation modified: NO
```

Do not author the F3.1.3a implementation prompt from inside this review.
Do not implement F3.1.3/F3.1.4/F3.2.

Historical first plan review REJECT remains canonical evidence. ADR-014 and
the revised-plan history are preserved.
