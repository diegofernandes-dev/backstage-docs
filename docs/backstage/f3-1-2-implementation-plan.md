# F3.1.2 — Fail-Closed Submission + First AuthorizationRound — Revised Implementation Plan

- **Status:** REVISED PLANNING COMPLETE — READY FOR INDEPENDENT RE-REVIEW
- **Date:** 2026-09-19
- **Authority:** ADR-006, ADR-007 (Model C), ADR-008, ADR-009, ADR-012; F3.1 / F3.1.1 plans; F3.1.1a/b accepted baselines; F3.1.2 architecture review decisions
- **Revision prompt:** `prompts/f3-1-2-plan-revision.md`
- **Rejected first-plan review:** `docs/backstage/f3-1-2-plan-architecture-review.md` at `backstage-docs@0daa8fc0719bafd4d8b1e2f95c1ad0abeaa03422`
- **Docs revision baseline:** `diegofernandes-dev/backstage-docs@34d7257e0ed44d991ed9f7085d55125c47904ff7`
- **ADO accepted implementation baseline:** `platform-devops-developer-portal@188d8e9cc43423f3644b3cacfb9849257838a583` (`feat/ado-repo-governance`)
- **ADO source verification for this revision:** not independently repeated in this checkpoint; the immediately preceding architecture review independently verified exact SHA `188d8e9` and no F3.1.2-surface drift. A fresh re-review must re-verify the branch tip before ACCEPT.
- **F3.1.2 implementation:** **NO-GO**

```text
F3.1.2 revised planning: READY_FOR_REREVIEW
F3.1.2 implementation: NO-GO
F3.1.2a implementation prompt authoring: NO-GO pending re-review ACCEPT
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO
Migration required: NO
Planned implementation slices: 2
ADO implementation modified by this revision: NO
```


## 1. Status / authority / baselines

### Revision status

The first F3.1.2 plan was independently reviewed and rejected because it left three persisted-semantics decisions wrong or ambiguous and one rollback control optional. This revision incorporates the review decisions as normative architecture:

1. repository explicit authorization-mode mismatch remains fail-closed `CONFLICT`; stored-mode-wins is implemented by service orchestration;
2. the ledger finalize path must use one caller-owned Knex transaction for all platform writes (and the DevelopmentProvider write);
3. `requirementId = requirementRole` with fail-closed per-rule role uniqueness at policy registration/publication;
4. rollback to a pre-F3.1.2 binary while pending `LEDGER_REQUIRED` reservations exist is a mandatory operational correctness gate.

### What this revised plan is

A re-review candidate and implementation contract proposal for the first composition of:

1. existing `POST /changes` / Model C create path;
2. immutable idempotency `authorization_mode` (`LEGACY_PRE_F3` | `LEDGER_REQUIRED`);
3. F3.1.1a published policy + evaluator;
4. F3.1.1b active selector bundle + Catalog principal resolver;
5. F3.1.0 append-only authorization ledger (`AuthorizationRound` + requirements + audit).

### What this plan is not

- Not implementation authorization.
- Not an F3.1.2a/F3.1.2b implementation prompt.
- Not F3.1.3 decisions, F3.1.4 RBAC, Teams, CAB UI, Delivery, or execution eligibility transport.
- Not a claim of distributed atomicity across an external ITSM provider.
- Not a change to ADR-009 or ADR-012.

### Primary architecture answer

A genuinely new logical submission receives its authorization mode exactly once when its idempotency reservation is first created. Existing reservations always resume with the stored mode; the service must not re-request the deployment's current default for them. One canonical Change snapshot is built once and durably reused. For `LEDGER_REQUIRED`, policy/selector resolution happens only while Round 1 is absent; all required authorization facts are validated before persistence. On the DevelopmentProvider path, provider state + Round 1 + requirements + submission audit + index finalization + idempotency completion commit in one caller-owned transaction. For an external provider, the provider side effect remains outside the platform transaction and converges by idempotent create/orphan retry. No ledger-governed Change is discoverable until the platform transaction commits. Rollback to a binary that predates the ledger branch is forbidden while pending ledger reservations exist.

---

## 2. Current source reality at ADO baseline `188d8e9`

The independent architecture review inspected the exact accepted SHA `188d8e9cc43423f3644b3cacfb9849257838a583` and found no post-baseline drift on the F3.1.2 surface at review time. This revision does not claim a newer live ADO verification; the fresh re-review must repeat that check.

### P1 inventory — current responsibilities and verified capabilities

| Surface | Path / symbol | Current responsibility / capability at `188d8e9` |
|---|---|---|
| HTTP create | `changeManagementPlugin.ts` `POST /changes` | AuthZ permission, Idempotency-Key header, `service.createChange`, 201 JSON |
| Service create | `ChangeManagementService.createChange` | Parse → reserve (service currently omits explicit mode) → recover → `buildChange` **twice** → `insertPending` → `finalizeCreate` |
| `buildChange()` | private method | Server `activityId` UUIDs + `createdAt` ISO — divergent across the two current calls |
| Idempotency | `KnexIdempotencyRepository.reserve/claimChangeId/complete` | Actor-scoped key; immutable `authorization_mode`; payload mismatch `CONFLICT`; explicit requested-mode mismatch on an existing row `CONFLICT`; omitting mode returns the stored row without triggering the mode-mismatch check |
| Index | `KnexChangeIndexRepository` | `insertPending` (`is_finalized=false`), `finalize(..., trx?)`, finalized-only list/detail reads |
| Provider | `IChangeManagementProvider` / `DevelopmentProvider` | Model C operational record; create is idempotent by `changeId`; DevelopmentProvider supports caller-owned transaction via `createWithTransaction` |
| Current finalize | `finalizeCreate` | Dev: provider + finalize + complete in one Knex transaction. External: provider create outside platform transaction, then finalize + complete with orphan/retry behavior |
| Ledger | `KnexAuthorizationLedgerRepository.createRound(..., trx?)` | Inserts Round + requirements; caller-owned `trx` is already supported. If `trx` is omitted it opens/uses an independent transaction path |
| Ledger audit | `appendAuditEvent(..., trx?)` | Caller-owned `trx` already supported; F3.1.2b must pass the outer transaction explicitly |
| Policy | `evaluatePolicy` + `createPolicyRegistry` + published `default-change-authorization@2026-09-02.1` | Pure evaluator; current published rules use unique `requirementRole` values, but registry does not yet enforce that uniqueness |
| Selector | `bootstrapAuthorization` → `AuthorizationRuntime` | Startup-validated active policy + selector bundle + `CatalogPrincipalResolver`; runtime is not yet passed to submission service |
| Guards | `architecture.test.ts` | Still forbids `ChangeManagementService` from creating a Round |
| Errors | `ChangeManagementError` | Existing stable codes are sufficient for the planned F3.1.2 error contract |

### Confirmed source defects / gaps carried into implementation

1. **Double `buildChange()`** — must be corrected in F3.1.2a before any Round hash exists.
2. **Authorization runtime unwired** — deliberate F3.1.1b STOP; F3.1.2b wires it.
3. **Old-binary rollback hazard** — `188d8e9` accepts an existing `LEDGER_REQUIRED` reservation when mode is omitted and then follows the legacy finalize path with no Round.
4. **Policy role uniqueness not enforced** — F3.1.2b must add fail-closed registration/publication validation before using `requirementRole` as the canonical requirement ID.

### Drift rule for the next review

A fresh architecture re-review must verify the current ADO branch tip before ACCEPT. Any drift touching Change creation, idempotency, ledger transaction APIs, policy registry, selector runtime, or provider finalization must be reconciled before this revised plan can become the implementation contract.

---

## 3. Objective and explicit non-goals

### Objective

Smallest correct slice such that **genuinely new** ledger-governed submissions:

```text
POST /changes
  -> reserve / recover idempotency regime (stored mode wins forever)
  -> LEGACY_PRE_F3: exact current F2 path (after single-build fix)
  -> LEDGER_REQUIRED (new only):
       one canonical Change
       pin active published policy + active selector bundle (once)
       evaluate → resolve principals → fail-closed SoD
       create AuthorizationRound 1 + requirements + submission audit
       finalize index + complete idempotency consistently
  -> identical logical result on idempotent retry
```

### Non-goals (hard)

- Decision commands / authority membership (F3.1.3)
- Additive user-supplied mandatory requirements (deferred — §10)
- New governance RBAC roles (F3.1.4)
- Teams, CAB Workbench, eligibility transport
- Delivery/Kargo/Argo/GitOps fields or bindings
- Background migration of historical Changes to ledger governance
- Distributed XA across external providers
- Generalized feature-flag / workflow frameworks

---

## 4. Accepted invariants carried forward

1. **Reservation mode is authoritative** for a logical `(operation, requestedBy, idempotencyKey)` for its lifetime.
2. **Model C** — platform index + ledger adjacent; provider owns operational detail (`IChangeManagementProvider`).
3. **Unfinalized index rows are invisible** to `GET /changes` and `GET /changes/:id`.
4. **Provider create is idempotent by `changeId`** (DevelopmentProvider upsert; future providers must preserve this).
5. **Policy/selector publication integrity** remains F3.1.1a/b; F3.1.2 consumes runtime, does not invent a second one.
6. **ADR-009 orthogonality** — Round 1 does **not** change `Change.lifecycle`; remains `submitted`. `AUTHORIZED` is derived later from decisions.
7. **ADR-012** — no pipeline/Kargo/Argo/Git identity in Change authorization artifacts.

---

## 5. Proposed submission state machine

### Shared prefix and reservation-mode orchestration

```text
parse + payloadHash
→ lookup existing idempotency reservation
   ├─ existing:
   │    reserve/recover WITHOUT re-requesting deployment default
   │    repository validates payload; stored authorization_mode wins
   └─ absent:
        attempt first reserve with config newSubmissionAuthorizationMode
        └─ if concurrent unique race is lost:
             reserve/recover with mode omitted
             same payload => winner's stored mode wins
             different payload => CONFLICT
→ if completed → return cached {changeId,status}
→ validate target + executionPlan
→ ensure changeId claimed
→ ensure pending index exists with ONE canonical Change snapshot
→ branch only on reserved.authorizationMode
```

The service never changes a reservation's mode. The repository retains its explicit mode-mismatch fail-closed contract.

### `LEGACY_PRE_F3` branch

```text
canonicalChange = durable pending index snapshot
→ existing F2 Model C finalize path
→ provider + index.finalize + idempotency.complete
→ return {changeId, status: submitted}
```

No policy, selector resolution, Round, requirement, or authorization audit.

### `LEDGER_REQUIRED` branch

Healthy pending state for a new F3.1.2 submission has no committed Round 1 yet.

```text
canonicalChange = durable pending index snapshot
→ pin immutable AuthorizationRuntime references for this attempt
→ evaluate active published policy exactly once
→ resolve required selector principals in deterministic order
→ validate principal types
→ validate submission-time separation of duty
→ build Round 1 + requirements + submission audit events in memory
→ provider/platform finalization:
   DevelopmentProvider:
     BEGIN caller-owned trx
       provider.createWithTransaction(trx, canonicalChange)
       ledger.createRound(round1, trx)
       ledger.appendAuditEvent(event, trx) × N
       index.finalize(..., trx)
       idempotency.complete(..., trx)
     COMMIT
   External provider:
     provider.create(canonicalChange) outside platform trx (idempotent by changeId)
     BEGIN caller-owned platform trx
       ledger.createRound(round1, trx)
       ledger.appendAuditEvent(event, trx) × N
       index.finalize(..., trx)
       idempotency.complete(..., trx)
     COMMIT
→ return {changeId, status: submitted}
```

If a committed Round 1 is observed while the same reservation/index are still pending/unfinalized, that state is not a normal F3.1.2 crash window because Round/finalize/complete must commit together. F3.1.2 fails closed on that invariant breach; it does not create Round 2 and does not silently heal around an atomicity violation.

### Authoritative completion condition

For `LEDGER_REQUIRED`, completion is the successful platform-transaction commit that makes these durable together:

1. Round 1 and effective requirements;
2. required submission authorization audit events;
3. `change_index.is_finalized = true`;
4. `change_idempotency.state = completed`.

On the DevelopmentProvider path, the provider write joins that same transaction. On an external-provider path, provider existence is a prerequisite side effect outside the transaction and is reconciled by idempotent create on retry.

Readers remain index-finalization based; therefore a ledger-governed Change cannot become discoverable before Round 1 is committed.

---

## 6. Authorization-regime cutover decision

### Decision

One config pin under the existing authorization configuration:

```yaml
changeManagement:
  authorization:
    newSubmissionAuthorizationMode: LEGACY_PRE_F3 | LEDGER_REQUIRED
```

| Property | Rule |
|---|---|
| Owner | Platform/DevOps via reviewed Backstage app-config |
| Default when F3.1.2 capability first deploys | `LEGACY_PRE_F3` |
| Startup validation | Exact enum only; invalid value fails boot |
| Applies to | First creation of a genuinely new idempotency reservation |
| Never applies to | Existing reservations |
| Switch LEDGER → LEGACY | Stops creating new ledger reservations; existing ledger reservations continue ledger path |
| Generalized feature-flag framework | Not introduced |

### Repository contract remains unchanged

```text
payload mismatch on existing key                => CONFLICT
explicit requested authorizationMode mismatch   => CONFLICT
stored authorization_mode                       => immutable
```

F3.1.2 must **not** weaken this repository behavior.

### Service orchestration — stored mode wins without weakening the repository

Normative algorithm:

```text
desiredMode = config.newSubmissionAuthorizationMode
existing = idempotency.find(lookup)   # add/read-only lookup if not already exposed

if existing:
  # validate payload and recover existing reservation without re-requesting config mode
  reserved = idempotency.reserve({
    ...lookup,
    payloadHash,
    # authorizationMode intentionally omitted
  })
else:
  try:
    reserved = idempotency.reserve({
      ...lookup,
      payloadHash,
      authorizationMode: desiredMode,
    })
  catch CONFLICT:
    # handles a concurrent first-insert race without parsing an error message.
    # If the race winner used the same payload, recover its stored mode.
    # If the payload differs, this second reserve remains CONFLICT.
    reserved = idempotency.reserve({
      ...lookup,
      payloadHash,
      # authorizationMode intentionally omitted
    })

mode = reserved.authorizationMode
branch on mode only
```

The read-only lookup is orchestration support, not a semantic change to `reserve`. If the implementation baseline already exposes an equivalent lookup through the repository interface, use it; otherwise add the smallest read-only repository method in F3.1.2b.

### Required cases

| Case | Behavior |
|---|---|
| Existing `LEGACY_PRE_F3` + same payload after config flips to LEDGER | Stored legacy mode; legacy path; no policy work |
| Existing `LEGACY_PRE_F3` + different payload | `CONFLICT` before policy/Catalog |
| Existing `LEDGER_REQUIRED` + config flipped to LEGACY | Stored ledger mode; ledger path |
| New row + current config LEDGER | First winner stores `LEDGER_REQUIRED` |
| New row + current config LEGACY | First winner stores `LEGACY_PRE_F3` |
| Concurrent first attempts with different desired defaults | Idempotency unique key chooses one winner; loser recovers winner's row with mode omitted if payload matches |
| Explicit repository caller re-requests a different mode | Repository still returns `CONFLICT` |
| Same key under different actor | Independent reservation, unchanged |

Authorization regime is never inferred from app version, schema version, deployment time, wall clock, branch, or mere table presence.

---

## 7. Single canonical Change construction

### Decision (required #2)

**`buildChange()` correction is Micro-slice A (F3.1.2a)** — land and prove independently before enabling ledger submission wiring. It is in scope of the F3.1.2 *program* but ordered first because Round 1 hashes the Change snapshot.

### Exact correction

1. Call `buildChange(command, changeId)` **at most once** per logical create that still lacks a pending index row.
2. `insertPending({ change: thatExactObject, ... })`.
3. Thereafter **every** provider/ledger/hash path uses `indexRecord.snapshot` (re-read after insert), never a second builder call.
4. On recovery when pending index already exists: **do not rebuild**; use snapshot.
5. `changeSnapshotSha256 = sha256Canonical(change)` via existing `authorization/canonical.ts` (no second scheme).

### Why not only “fix inside the big PR”?

Independently reviewable, removes a known F2 correctness defect, shrinks ledger-hash review surface, and can ship while `newSubmissionAuthorizationMode` remains `LEGACY_PRE_F3`.

---

## 8. Policy / selector binding and principal resolution

### Decision (required #4 earliest binding)

Within one submission attempt that still lacks Round 1:

1. Read pins **once** from injected `AuthorizationRuntime` constructed at startup (`activePolicy`, `selectorBundle`, `principalResolver`) — the same objects `bootstrapAuthorization` already builds.
2. `evaluatePolicy(runtime.activePolicy, { classification: change.classification, risk: change.risk })` exactly once.
3. For each `RequirementDefinition`, `principalResolver.resolve(selectorKey)` once, in stable order (`requirementRole` ascending, then `selectorKey`).
4. Persist those identities on Round 1: policy key/version/artifactSha256/provenance/input/inputSha256/matchedRuleProvenance; selector bundle key/version/contentDigest/provenance.

**No double-read of “current active” config inside the attempt.** Runtime is process-immutable after startup deep-freeze.

### Earliest durable authorization-artifact binding (required #4)

| Phase | Policy/bundle/principals |
|---|---|
| Before Round 1 commit | Not durable; crash retry may re-resolve against current runtime pins |
| **At Round 1 + requirements commit** | **Fixed forever** for that logical submission |
| After Round 1 exists | Replay **must not** call `evaluatePolicy` or `resolve`; uses `findRound(changeId,1)` |

This matches ADR-009 Model B: ledger is self-contained evidence; runtime need not retain historical policy modules for replay.

---

## 9. Emergency separation-of-duty rule

### Decision (required #6)

Validation order (fail closed; nothing ledger-durable until all pass):

1. Policy evaluation succeeds.
2. Every required `selectorKey` resolves.
3. `snapshot.principalType === definition.requiredPrincipalType`.
4. **Separation of duty:** group requirements that share a non-empty `separationOfDutyKey`; for each group, the set of `resolvedPrincipalRef` values among requirements with `principalType === 'user'` must have cardinality equal to the number of those requirements (all distinct). Same-person A/B → fail **before** Round 1 insert.

F3 MVP keeps emergency A/B as `user`-typed selectors (accepted F3.1.1 narrowing). This is **resolved-principal distinctness at submission**, not F3.1.3 decision-actor distinctness.

On SoD failure: no Round, no finalize, pending index may exist (invisible); client receives stable error (see §17); retry after config/Catalog correction may succeed with a *new* resolution — allowed because Round 1 never committed.

---

## 10. Effective requirement + Round 1 mapping

### Additive mandatory requirements

**DEFER** outside F3.1.2. F3.1.2 materializes policy-sourced requirements only.

### Canonical requirement identity

Normative rule:

```text
requirementId = requirementRole
```

There is no hash fallback and no runtime alternate scheme.

Hard policy invariant:

> Within the effective `requirements[]` of every published rule, `requirementRole` must be unique.

F3.1.2b must enforce that invariant fail-closed at policy registration/publication/startup (extend `createPolicyRegistry` or the existing equivalent validation boundary). A duplicate role prevents the policy from becoming usable; submission must never discover the collision while materializing a Round.

### Requirement field mapping

| `ApprovalRequirement` field | Source |
|---|---|
| `changeId` / `roundNumber` | Parent Round 1 |
| `requirementId` | Exactly `RequirementDefinition.requirementRole` |
| `kind` / `phase` / `mandatory` | `RequirementDefinition` |
| `source` | `'policy'` |
| `sourceRef` | `requirementRole` |
| `sourceProvenance` | Round policy provenance |
| `principalSnapshot` | Resolver output |
| `separationOfDutyKey` | Definition, if present |
| `slaPolicyKey` / `slaPolicyVersion` | Round policy identity when the definition has SLA metadata |
| `slaDurationSeconds` / `slaAnchor` | Definition SLA |
| `createdAt` | Same server timestamp as Round 1 creation |
| `addedByActorRef` / `additionReason` | Omitted |
| `decision` | Absent |

Requirements are immutable after insert.

### Round 1 record

| Field | Value |
|---|---|
| `roundNumber` | `1` |
| `changeSnapshot` / `changeSnapshotSha256` | Canonical pending-index snapshot + existing `sha256Canonical` |
| Policy identity fields | Pinned active published policy/evaluation result |
| `policyInput` | `{ classification, risk }` only |
| Selector-bundle identity fields | Pinned active selector bundle identity/digest/provenance |
| `createdAt` | One server timestamp |
| `requirements` | Deterministic mapping above |

### Minimum submission authorization audit

Append inside the same caller-owned platform transaction as Round 1:

| `eventType` | Payload |
|---|---|
| `change.authorization.round_created` | Round number, Change hash, policy identity, selector-bundle identity |
| `change.authorization.policy_selected` | Policy key/version/digest, matched-rule provenance, input hash |
| `change.authorization.selector_bundle_bound` | Bundle key/version/digest |
| `change.authorization.requirement_materialized` | One event per requirement with requirementId, selectorKey, principalType, resolved principal ref |

No decision, rejection, execution, or eligibility events are introduced by F3.1.2.

---

## 11. Transaction boundaries and crash-recovery matrix

### Transaction contract

Do not claim distributed atomicity.

| Store / action | Platform DB? | F3.1.2 transaction rule |
|---|---|---|
| Idempotency reservation / changeId claim | Yes | Durable before final platform transaction |
| Pending index snapshot | Yes | Durable and invisible before final platform transaction |
| `ledger.createRound(round1, trx)` | Yes | Caller-owned platform `trx` is mandatory |
| `ledger.appendAuditEvent(event, trx)` | Yes | Same caller-owned platform `trx` is mandatory |
| `index.finalize(..., trx)` | Yes | Same caller-owned platform `trx` |
| `idempotency.complete(..., trx)` | Yes | Same caller-owned platform `trx` |
| DevelopmentProvider create | Same Knex DB | Joins same caller-owned `trx` |
| Future external provider create | No / independent authority | Outside platform `trx`; idempotent by `changeId` |

Calling `createRound(round1)` or `appendAuditEvent(event)` without the outer `trx` from the LEDGER finalization path is an implementation defect.

### Corrected crash matrix

| # | Crash point | Durable facts | Retry behavior |
|---|---|---|---|
| C1 | After reservation, before changeId claim | Pending reservation with immutable mode | Claim/recover changeId; continue on stored mode |
| C2 | After changeId claim, before pending index | Reservation + changeId | Build Change once and insert pending snapshot |
| C3 | After pending index, before policy/provider | Reservation + canonical invisible snapshot | Reuse snapshot; evaluate/resolve because no Round is committed |
| C4 | External provider create succeeds, before platform transaction | External provider orphan + pending invisible platform state | Log/recover orphan; retry idempotent provider create by same `changeId`, then attempt platform transaction |
| C5 | Failure anywhere inside platform transaction before commit | No Round/requirement/audit/finalize/complete writes from that transaction are durable; DevelopmentProvider write also rolls back | Retry from pending snapshot (external provider may already exist and is idempotently reconciled) |
| C6 | Platform transaction commits, response not yet delivered | Round 1 + requirements + audit + finalized index + completed idempotency are all durable together; Dev provider durable too | Retry observes completed reservation and returns same logical result |
| C7 | Response lost after commit | Same fully durable state as C6 | Idempotent completed return |

There is **no normal F3.1.2 state** where Round 1/finalized index are committed but idempotency completion from the same path is still pending.

A committed Round 1 alongside pending/unfinalized platform state is an invariant breach, not a normal recovery milestone; fail closed rather than inventing Round 2 or silently normalizing the inconsistency.

### External-provider convergence

External provider create remains outside the platform transaction. Correctness is retry convergence:

1. provider create must be idempotent by canonical `changeId`;
2. provider success + platform transaction failure creates an orphan/retry situation;
3. retry repeats idempotent provider create and then retries the complete platform transaction;
4. no XA/2PC or outbox framework is introduced in F3.1.2.

---

## 12. Visibility / finalization invariant

| Reader | Sees Change when |
|---|---|
| `GET /changes` | `change_index.is_finalized=true` plus participant predicate |
| `GET /changes/:id` | finalized index route |
| Future authorization reads | Out of scope for F3.1.2 |

For a new `LEDGER_REQUIRED` submission, **visibility authority is the platform transaction itself**:

```text
create Round 1 + requirements
append required submission audit
finalize index
complete idempotency
COMMIT
```

These writes use the same caller-owned `trx`. Do not use a non-transactional `findRound()` check as the authority that permits index finalization.

Therefore:

- before commit: index remains pending/invisible and no Round from that attempt is durable;
- after commit: Round 1 and finalized index are durable together;
- a finalized LEDGER Change without Round 1 is a correctness violation.

Legacy finalized Changes and legacy create semantics remain unchanged.

---

## 13. Idempotent replay and concurrency

### Replay invariants

| Invariant | Mechanism |
|---|---|
| Authorization regime never changes | Existing reservation's stored `authorization_mode` is the only branch authority |
| Repository explicit mode reinterpretation remains fail-closed | Existing `reserve` mismatch-CONFLICT contract retained |
| No duplicate provider record | Provider create idempotent by `changeId` |
| No duplicate index | `change_id` uniqueness + pending snapshot reuse |
| No Round 2 from initial-submission retry | Round 1 is the only submission round; DB uniqueness/monotonic rules remain |
| No duplicate requirements/audit from a committed submission | They commit once with Round 1 in the platform transaction |
| No principal/policy drift after commit | Completed reservation short-circuits; Round 1 is durable evidence |
| Conflicting payload | Repository `CONFLICT` before policy/Catalog work |
| Canonical Change stability | Pending index snapshot reused; no second `buildChange()` on recovery |

### Reservation concurrency

Two processes may race to create the first reservation while running different deployment defaults.

Correctness sequence:

1. both pre-read no row;
2. both attempt `reserve(... authorizationMode=desiredMode)`;
3. DB unique key chooses one winner;
4. loser handles the conflict by re-reserving/recovering with mode omitted;
5. same payload → winner's stored mode is returned;
6. different payload → remains `CONFLICT`.

The service never overwrites the winner's mode.

### Other concurrency

| Scenario | Arbiter |
|---|---|
| Duplicate pending-index insert | `change_id` uniqueness; loser re-reads canonical snapshot |
| Concurrent provider create | Provider idempotency by `changeId` |
| Concurrent Round 1 platform transactions | DB uniqueness + one transaction wins; loser re-reads completed/finalized state or fails closed on invariant mismatch |
| Two different idempotency keys | Two independent Changes |
| Retry racing with commit | Before commit sees pending; after commit sees completed; no partial Round/finalize visibility |

No in-memory mutex is correctness authority.

---

## 14. Model C / provider boundary

F3.1.2 adds platform ledger writes beside the index. It does **not**:

- move authorization into the provider;
- add ADO/Teams/Kargo fields to `Change` or Round;
- require provider transaction participation.

External provider create remains outside the platform transaction; correctness is **retry convergence**, not 2PC. ADR-007 Model C preserved; ADR-012 Delivery boundary preserved (authorization only).

---

## 15. Wiring / dependency changes

### Decision (required #9)

Smallest DI change:

1. Extend `ChangeManagementServiceOptions` with:
   - `authorizationRuntime: AuthorizationRuntime`
   - `authorizationLedger: AuthorizationLedgerRepository`
   - `newSubmissionAuthorizationMode: AuthorizationMode` (from config)
2. Plugin: pass `bootstrapAuthorization(...)` result + `new KnexAuthorizationLedgerRepository(knex)` + config enum into the service (today runtime is intentionally discarded after logging).
3. Service uses `runtime.activePolicy` / `runtime.selectorBundle` / `runtime.principalResolver` only on `LEDGER_REQUIRED` path when Round 1 is absent.
4. No global mutable singleton; no ad-hoc config re-read; no frontend authority; no direct Catalog calls in the service for principals (resolver abstraction already exists).

Update `architecture.test.ts`: replace “service must not contain `createRound`” with “service may call ledger only when `authorizationMode === LEDGER_REQUIRED`” / forbid decision APIs / forbid Delivery identifiers — keep fail-closed intent.

---

## 16. Migration decision

### Decision (required #8)

**NO migration.** F3.1.0 schema already has:

- `change_idempotency.authorization_mode`
- `change_index.authorization_mode`
- rounds / requirements / decisions / audit tables
- immutability triggers and uniqueness

F3.1.2 is behavior + wiring + tests only. No future-proof columns.

---

## 17. Error / HTTP contract

Map to existing codes; add **at most one** narrow detail key, not a new public workflow status.

| Failure | Code | HTTP | Retryable? |
|---|---|---|---|
| Bad body / window / plan | `VALIDATION_ERROR` | 400 | No (fix request) |
| Idempotency payload mismatch | `CONFLICT` | 409 | No |
| Catalog principal missing | `NOT_FOUND` | 404 | No until Catalog fixed |
| Catalog / creds unavailable | `PROVIDER_UNAVAILABLE` | 503 | Yes |
| Selector missing / kind mismatch / config invariant | `INTERNAL_ERROR` | 500 | No (ops fix + restart) |
| SoD same-person A/B | `CONFLICT` | 409 | No until bindings distinct (`details.reason=separation_of_duty`) |
| Unsupported policy model / eval throw | `INTERNAL_ERROR` | 500 | No |
| Storage failure | `INTERNAL_ERROR` / `PROVIDER_UNAVAILABLE` | 500/503 | Yes if 503 |
| Provider failure / orphan | `PROVIDER_UNAVAILABLE` | 503 | Yes |
| Index/reservation mode disagree | `INTERNAL_ERROR` | 500 | No (fail closed) |

Do not leak provider payloads or policy rule ASTs in responses.

**Minimal taxonomy addition:** none of the enum; only optional `details.reason` / `details.selectorKey` already patterned by resolver.

---

## 18. Observability

Low-cardinality structured logs (extend existing `change.create.*` style):

| Event | Fields |
|---|---|
| `change.create.authorization_mode` | `changeId?`, `mode`, `source: reserved\|config` |
| `change.authorization.round_create.started\|completed\|failed` | `changeId`, `roundNumber`, `reason?` |
| `change.authorization.policy_selected` | `policyKey`, `policyVersion` |
| `change.authorization.selector_resolution.failed` | `selectorKey`, `errorCode` |
| `change.create.recover` | existing + `mode` |

Never log emails, tokens, raw evidence/comments, or full provider payloads.

---

## 19. Deployment and rollback sequencing

### Verified old-binary threat model

At `188d8e9`:

1. service calls `reserve` without an explicit mode;
2. an existing `LEDGER_REQUIRED` reservation is returned rather than rejected for mode mismatch;
3. service copies the stored mode but has no ledger branch;
4. it follows the legacy provider/finalize/complete path and can make the Change discoverable with no Round 1.

Therefore rollback to a pre-F3.1.2 binary while a ledger reservation is pending is a **correctness hazard**.

### Safe deployment sequence

1. Deploy F3.1.2-capable binary with:
   `newSubmissionAuthorizationMode: LEGACY_PRE_F3`.
2. Prove legacy create/recovery behavior and F3.1.2a invariants.
3. Intentionally enable `LEDGER_REQUIRED` for new reservations.
4. Existing reservations always continue using their stored mode.
5. Switching config back to `LEGACY_PRE_F3` is allowed on the F3.1.2-capable binary; it affects new reservations only.

### Mandatory rollback correctness gate

Before rollback to **any binary that predates F3.1.2 ledger submission support**, operators must run:

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

If the count is nonzero, binary rollback is **FORBIDDEN** until those logical submissions are completed/drained through supported F3.1.2 recovery.

This is a mandatory **RUNBOOK CORRECTNESS GATE** for F3.1.2. It is not an optional code guard. Deployment automation may enforce the same query in the future, but such automation is outside this slice unless separately authorized.

Completed ledger submissions remain durable facts and do not block rollback solely because they exist; the hazard is pending ledger reservations that an old binary could finalize without Round 1.

---

## 20. Test matrix

### F3.1.2a — canonical Change

| ID | Case |
|---|---|
| A1 | One logical create calls `buildChange()` at most once before pending snapshot exists |
| A2 | Provider input and pending index snapshot are structurally identical |
| A3 | Recovery with existing pending index never rebuilds activity IDs/timestamps |
| A4 | Existing F2 create/recovery regressions remain green |

### Idempotency / cutover

| ID | Case |
|---|---|
| I1 | Repository explicit authorization-mode mismatch remains `CONFLICT` (keep F3.1.0-V contract) |
| I2 | Existing LEGACY reservation + config LEDGER resumes legacy with no policy work |
| I3 | Existing LEDGER reservation + config LEGACY resumes ledger |
| I4 | New reservation stores the current config mode |
| I5 | Same key / same payload replay returns same logical result |
| I6 | Same key / different payload is `CONFLICT` before policy/Catalog |
| I7 | Concurrent first insert across processes with different desired defaults converges on DB winner's stored mode |
| I8 | Same key under different actor remains independent |

### Policy / requirement identity

| ID | Case |
|---|---|
| Q1 | Duplicate `requirementRole` inside one published rule fails registry/publication/startup validation |
| Q2 | Round materialization always sets `requirementId === requirementRole` |
| Q3 | Retry-before-commit uses the same single identity scheme; no hash fallback |
| Q4 | Active published policy/bundle used and exact identities persisted |

### Principal resolution / SoD

| ID | Case |
|---|---|
| S1 | Catalog principal missing → fail closed; no platform final transaction |
| S2 | Catalog unavailable → retryable provider-unavailable semantics; no Round durable |
| S3 | Required principal-type mismatch fails closed |
| S4 | Emergency A/B resolving to same User ref → `CONFLICT` with `details.reason=separation_of_duty`; no Round durable |
| S5 | Different User refs materialize deterministic requirements |

### Caller-owned transaction — SQLite + disposable PostgreSQL

| ID | Case | SQLite | Postgres |
|---|---|---|---|
| T1 | DevelopmentProvider + Round + requirements + audit + index.finalize + idempotency.complete use one caller-owned `trx` | ✓ | ✓ |
| T2 | Throw after `createRound(..., trx)` before commit → no provider/Round/audit/finalize/complete mutation durable | ✓ | ✓ |
| T3 | Throw after audit append before finalize → entire platform transaction rolls back | ✓ | ✓ |
| T4 | Successful commit exposes Round/requirements/audit/finalized index/completed idempotency together | ✓ | ✓ |
| T5 | Service/unit transaction spy proves non-undefined outer `trx` is passed to `createRound` and every `appendAuditEvent` call | ✓ | ✓ |
| T6 | Concurrent Round 1 create is constrained by DB uniqueness; no Round 2 from submission retry | ✓ | ✓ |
| T7 | Immutability triggers remain green | ✓ | ✓ |

### External Model C provider

| ID | Case |
|---|---|
| M1 | External provider failure before side effect → platform state remains pending/invisible |
| M2 | External provider success + platform transaction failure → orphan log / retry converges by same `changeId` |
| M3 | Retry never duplicates provider operational record |
| M4 | No provider-specific authorization fields enter Round/requirements |

### Rollback compatibility

| ID | Case |
|---|---|
| B1 | Source/evidence proves pre-F3.1.2 binary can process pending `LEDGER_REQUIRED` through legacy finalize |
| B2 | Implementation evidence contains the mandatory pending-ledger rollback query and zero-count rule |
| B3 | Switching config LEDGER → LEGACY on F3.1.2 binary leaves existing ledger reservations on ledger path |

### Regression / quality

- all F2 / F3.1.0 / F3.1.1 suites;
- participant list/detail visibility;
- architecture guards for provider/Delivery/Teams authority leakage;
- lint/build;
- repository-wide TypeScript error set set-identical to the accepted `188d8e9` baseline;
- live Catalog remains opt-in evidence, not a merge dependency when fail-closed resolver fakes cover the contract.

Failure injection must target actual transaction boundaries, not impossible “after finalize but before complete commit” states.

---

## 21. Exact source paths expected to change

### F3.1.2a — canonical Change

Expected:

```text
packages/backend/src/modules/changeManagement/ChangeManagementService.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.recovery.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.integration.test.ts
```

No ledger wiring and no authorization-mode config change in 2a.

### F3.1.2b — ledger submission

Expected implementation surface after source re-verification:

```text
packages/backend/src/modules/changeManagement/ChangeManagementService.ts
packages/backend/src/plugins/changeManagementPlugin.ts

# idempotency orchestration / read-only lookup if not already exposed:
packages/backend/src/modules/changeManagement/persistence/IdempotencyRepository.ts
packages/backend/src/modules/changeManagement/persistence/KnexIdempotencyRepository.ts
packages/backend/src/modules/changeManagement/persistence/idempotencyAuthorizationMode.test.ts

# authorization config / runtime:
packages/backend/src/modules/changeManagement/authorization/selector/config.ts
packages/backend/src/modules/changeManagement/authorization/selector/types.ts
app-config.yaml

# policy requirementRole uniqueness:
packages/backend/src/modules/changeManagement/authorization/policy/registry.ts
packages/backend/src/modules/changeManagement/authorization/policy/registry.test.ts  # or the existing registry/publication test file

# guards / focused submission tests:
packages/backend/src/modules/changeManagement/architecture.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.ledgerSubmit*.test.ts
```

Important repository rule:

- `KnexIdempotencyRepository.reserve` **must keep** explicit mode-mismatch `CONFLICT`.
- A `KnexIdempotencyRepository.ts` change is justified only to add/expose the smallest read-only lookup needed by service orchestration or equivalent race-safe support; not to weaken `reserve`.
- `KnexAuthorizationLedgerRepository` transaction capability already exists and should be **used**, not redesigned, unless fresh source drift proves otherwise.

Small pure helpers under the existing authorization module are allowed only when they reduce service complexity without creating another framework.

Must not change: frontend, Delivery, decision APIs, migrations unless the fresh re-review discovers a real schema contradiction, or production selector data.

---

## 22. Slice decomposition / implementation order

Exactly two ordered micro-slices remain sufficient:

| Order | Slice | Content | Enables new LEDGER submissions? |
|---|---|---|---|
| 1 | **F3.1.2a** | Single canonical Change construction + recovery reuse; no authorization integration | No |
| 2 | **F3.1.2b** | New-reservation mode orchestration, runtime DI, policy-role uniqueness validation, selector/SoD, Round 1 + requirements + audit, mandatory caller-owned transaction, rollback runbook evidence | Yes, only after config intentionally switches to `LEDGER_REQUIRED` |

No third transaction-capability slice: the ledger APIs already accept caller-owned `trx`.

F3.1.2b does not modify repository mismatch semantics; it may add the smallest read-only reservation lookup required by service orchestration.

---

## 23. Risks and rejected alternatives

| Rejected alternative | Why |
|---|---|
| Infer authorization mode from deploy time / schema / app version | Reinterprets logical submissions across deploys |
| Remove repository explicit mode-mismatch `CONFLICT` | Weakens an accepted fail-closed invariant unnecessarily |
| Always pass current config mode on retry | Produces false conflicts after cutover and violates stored-mode authority |
| Runtime fallback from `requirementRole` to hash | Creates two persisted identity schemes |
| Non-transactional `findRound` check as finalize authority | Cannot guarantee Round + visibility atomicity |
| Omit `trx` when calling ledger writes | Allows independent ledger commit and breaks the platform-transaction contract |
| Second `buildChange()` | Allows provider/index/Round snapshot divergence |
| Distributed XA / 2PC / generic outbox framework | Out of scope; Model C recovery is idempotent convergence |
| Additive user requirements in F3.1.2 | Permission/contract not ready |
| Migration “for future proofing” | No durable-state gap identified |
| Re-evaluate policy/selector after committed Round | Violates immutable authorization evidence |
| Roll back freely to pre-F3.1.2 with pending ledger reservations | Old binary can finalize without Round 1 |
| Optional rollback hard guard | Correctness control cannot be optional; use mandatory runbook gate |
| Put authorization into provider | Violates ADR-007/009 |
| Put Delivery correlation/provider IDs into Round | Violates ADR-012 |

### Residual risks accepted by this plan

- Before Round commit, Catalog/policy may change between retries. This is acceptable because the pending Change remains invisible and no canonical authorization artifact has been committed.
- External provider orphan is possible by Model C design; idempotent create/retry is the recovery contract.
- Production selector-bundle readiness remains a separate production-adoption concern and does not change F3.1.2 architecture.

---

## 24. Answers to all 15 challenge scenarios

1. **Legacy reserved yesterday, config is LEDGER today:** service finds/recover-reserves the existing row without re-requesting today's default; stored `LEGACY_PRE_F3` wins; no policy/Round.
2. **LEDGER reserved, crash before provider:** reservation and possibly canonical pending snapshot survive; retry uses stored ledger mode and reuses snapshot.
3. **External provider create succeeds, crash before platform commit:** provider orphan may exist; retry calls idempotent create with same `changeId`, then retries the complete platform transaction.
4. **Platform transaction commits, client receives network error:** idempotency is already completed with Round/finalize; retry returns same logical result and does not re-evaluate policy.
5. **Emergency A/B different selector keys resolve to same User:** fail before platform transaction; no Round/requirements/audit/finalize/complete from that attempt.
6. **Catalog unavailable during resolution:** no platform final transaction starts; pending index remains invisible; retry later.
7. **Active bundle changes between two separate requests:** allowed. Between retries before Round commit: current immutable runtime for the retry may bind. After commit: replay uses durable completed/ledger evidence, no re-resolution.
8. **Two workers same key concurrently:** unique idempotency key selects the winner; loser recovers winner's stored mode when payload matches.
9. **Rollback to pre-F3.1.2 with pending LEDGER reservation:** forbidden by mandatory zero-pending runbook correctness gate.
10. **Future external ITSM provider:** provider create remains outside platform transaction; correctness is idempotent create + orphan/retry, not XA.
11. **Mismatching payload with existing key:** repository returns `CONFLICT` before policy/Catalog authorization work.
12. **Successful F3.1.2 submission means authorized?** No. Lifecycle remains `submitted`; Round 1 only materializes requirements. Decision authorization comes later.
13. **Does Round 1 change lifecycle?** No.
14. **Inactive historical policy on replay:** after successful commit, the durable Round contains policy/bundle/principal evidence; submission replay does not require executing historical policy code.
15. **When is a new ledger Change discoverable?** Only after the platform transaction commits Round 1/requirements/audit and sets the index finalized while completing idempotency.

### Additional revision-specific challenge answers

- **Repository mode mismatch:** retained as explicit fail-closed `CONFLICT`; service orchestration prevents deployment-default changes from becoming explicit reinterpretation requests.
- **Requirement identity:** exactly `requirementRole`; duplicate roles make the policy unusable before submission.
- **Impossible partial platform commit:** Round/finalize/complete are one transaction; any plan/test assuming a normal split commit is invalid.

---

## 25. Implementation acceptance criteria

A future implementation checkpoint may claim PASS only when all relevant criteria are proven.

### F3.1.2a

- [ ] one canonical Change snapshot per logical create;
- [ ] no second `buildChange()` after pending snapshot exists;
- [ ] provider/index equality and recovery stability proven;
- [ ] F2 regressions green;
- [ ] no authorization integration introduced.

### F3.1.2b

- [ ] repository explicit authorization-mode mismatch remains `CONFLICT`;
- [ ] service applies config mode only to genuinely new reservations;
- [ ] concurrent first-insert race converges on DB winner's stored mode;
- [ ] legacy reservation retries never enter policy path;
- [ ] `requirementId = requirementRole` only;
- [ ] duplicate `requirementRole` fails policy registration/publication/startup;
- [ ] same-person emergency SoD fails before any Round commit;
- [ ] DevelopmentProvider path passes one caller-owned `trx` to provider + Round + every audit append + index.finalize + idempotency.complete;
- [ ] transaction failure leaves no partial Round/finalize/complete state;
- [ ] external-provider orphan/retry converges without 2PC;
- [ ] no migration, decision API, Delivery field, Teams/CAB UI, or F3.1.3 behavior;
- [ ] SQLite + disposable Postgres failure-injection matrix passes;
- [ ] architecture guards and authorization regressions pass;
- [ ] lint/build pass;
- [ ] repository-wide TypeScript error set is set-identical to accepted baseline;
- [ ] config default remains `LEGACY_PRE_F3` until explicit enablement;
- [ ] implementation evidence contains the mandatory pre-F3.1.2 rollback query and zero-pending rule.

---

## 26. GO / NO-GO recommendation for a separate implementation checkpoint

```text
F3.1.2 revised planning: READY_FOR_REREVIEW
F3.1.2 implementation: NO-GO
F3.1.2a implementation prompt authoring: NO-GO pending re-review ACCEPT
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO
```

**Next gate:** fresh independent architecture re-review of this revised plan against the actual current ADO branch tip.

Only a re-review `ACCEPT` may authorize authoring the constrained F3.1.2a implementation prompt. This revision does not create that prompt.

---

## 27. STOP

STOP. Do not implement F3.1.2, fix `buildChange()` in ADO, add migrations/routes, wire `POST /changes`, create AuthorizationRounds, create an implementation prompt, or start F3.1.3/F3.1.4.

---

## Appendix A — Explicit decisions checklist

| # | Decision | Revised resolution |
|---|---|---|
| 1 | Cutover mechanism | `newSubmissionAuthorizationMode` applies only to first reservation creation; existing stored mode wins through service orchestration |
| 2 | Repository mode mismatch | Preserve explicit requested-mode mismatch `CONFLICT`; no semantic weakening |
| 3 | Race-safe first insert | DB unique key selects winner; loser recovers winner with mode omitted if same payload |
| 4 | `buildChange` fix | F3.1.2a prerequisite micro-slice |
| 5 | Transaction boundary | Caller-owned `trx` mandatory for ledger/audit/finalize/complete; Dev provider joins |
| 6 | Visibility authority | Same platform transaction commit, not non-transactional `findRound` |
| 7 | Earliest durable authorization binding | Round 1 platform transaction commit |
| 8 | Requirement identity | Exactly `requirementId = requirementRole` |
| 9 | Requirement-role uniqueness | Fail closed at policy registration/publication/startup |
| 10 | A/B distinctness | Resolved User refs distinct per `separationOfDutyKey` before commit |
| 11 | Additive requirements | Deferred |
| 12 | Migration | NO |
| 13 | DI shape | Inject immutable authorization runtime + ledger + new-submission mode into service |
| 14 | Rollback | Mandatory RUNBOOK_CORRECTNESS_GATE: zero pending ledger reservations before pre-F3.1.2 binary rollback |
| 15 | Error taxonomy | Existing codes; SoD uses `CONFLICT` + stable reason |
| 16 | Slice decomposition | Exactly two: 2a canonical Change, 2b ledger submission |

## Appendix B — Planning gate coverage (P1–P20)

| Gate | Section |
|---|---|
| P1 | §2 |
| P2 | §7 |
| P3 | §6 |
| P4 | §8 |
| P5 | §9 |
| P6 | §10 |
| P7 | §10 |
| P8 | §11 |
| P9 | §12 |
| P10 | §13 |
| P11 | §13 |
| P12 | §17 |
| P13 | §4, §6, challenge 1 |
| P14 | §3, §14 |
| P15 | §3 |
| P16 | §16 |
| P17 | §15 |
| P18 | §20 |
| P19 | §18 |
| P20 | §19 |
