# F3.1.2 — Fail-Closed Submission + First AuthorizationRound — Implementation Plan

- **Status:** PLANNING COMPLETE — READY FOR INDEPENDENT ARCHITECTURE REVIEW
- **Date:** 2026-09-19
- **Authority:** ADR-006, ADR-007 (Model C), ADR-008, ADR-009, ADR-012; F3.1 / F3.1.1 plans; F3.1.1a/b accepted baselines
- **Prompt:** `prompts/f3-1-2-planning.md`
- **Docs baseline reviewed:** `diegofernandes-dev/backstage-docs@d65bf5e1446e682576557583b841ddde7e5a890c`
- **ADO baseline inspected:** `platform-devops-developer-portal@188d8e9cc43423f3644b3cacfb9849257838a583` (`feat/ado-repo-governance`)
- **ADO remote tip at planning time:** `188d8e9` (HTTPS `git ls-remote`; SSH fetch unavailable — tip equal to accepted baseline; **no post-baseline drift to reconcile**)
- **F3.1.2 implementation:** **NO-GO** (this document is planning only)

```text
F3.1.2 planning: READY_FOR_REVIEW
F3.1.2 implementation: NO-GO
Planning gates resolved: 20/20
Challenge scenarios answered: 15/15
Migration required: NO
Planned implementation slices: 2
ADO implementation modified by this checkpoint: NO
```

---

## 1. Status / authority / baselines

### What this plan is

A reviewable, implementation-ready design for the first composition of:

1. existing `POST /changes` / Model C create path;
2. immutable idempotency `authorization_mode` (`LEGACY_PRE_F3` | `LEDGER_REQUIRED`);
3. F3.1.1a published policy + evaluator;
4. F3.1.1b active selector bundle + Catalog principal resolver;
5. F3.1.0 append-only authorization ledger (`AuthorizationRound` + requirements + audit).

### What this plan is not

- Not implementation authorization.
- Not an F3.1.2 implementation prompt.
- Not F3.1.3 decisions, F3.1.4 RBAC, Teams, CAB UI, Delivery, or execution eligibility transport.
- Not a claim of distributed atomicity across an external ITSM provider.

### Primary architecture question (answered)

> How can F3.1.2 introduce first-round authorization for genuinely new submissions without changing the authorization regime of any pre-existing logical submission, without creating divergent Change snapshots, without weakening Model C/provider isolation, and without leaving ambiguous or unrecoverable partial state across crash/retry boundaries?

**Answer in one paragraph:** A config-owned cutover selects `LEDGER_REQUIRED` only for *new* reservations; an existing reservation’s stored mode wins forever (including across deployment). One canonical `Change` is built once and reused from the pending index snapshot thereafter. Ledger path evaluates the pinned active policy + active selector bundle once, fail-closes on principal/SoD failures *before* Round 1 commit, then persists Round 1 + requirements + submission audit and finalizes the index in the same platform DB transaction that completes idempotency (DevelopmentProvider may join that transaction; a future external provider create remains outside it and reuses F2 orphan/retry reconciliation). Discoverability requires finalized index; for `LEDGER_REQUIRED`, finalize is forbidden unless Round 1 exists. Replay after Round 1 is bound to the durable round artifacts and never re-evaluates a newer policy/bundle. Rollback past a binary that understands `LEDGER_REQUIRED` is forbidden while any pending `LEDGER_REQUIRED` reservation remains.

ADR-009 and ADR-012 are **not** modified; no contradiction was found.

---

## 2. Current source reality at ADO baseline `188d8e9`

Inspected in isolated worktree at exact SHA `188d8e9cc43423f3644b3cacfb9849257838a583` (parent `d3c0751`). Remote tip confirmed equal via HTTPS `git ls-remote`.

### P1 inventory — current responsibilities

| Surface | Path / symbol | Current responsibility at `188d8e9` |
|---|---|---|
| HTTP create | `changeManagementPlugin.ts` `POST /changes` | AuthZ permission, Idempotency-Key header, `service.createChange`, 201 JSON |
| Service create | `ChangeManagementService.createChange` | Parse → reserve (`LEGACY_PRE_F3` only) → recover → buildChange **twice** → `insertPending` → `finalizeCreate` |
| `buildChange()` | private method | Server `activityId` UUIDs + `createdAt` ISO — **non-deterministic across calls** |
| Idempotency | `KnexIdempotencyRepository.reserve/claimChangeId/complete` | Actor-scoped key; immutable `authorization_mode`; payload-hash CONFLICT; **mode-mismatch CONFLICT if caller re-requests a different mode** |
| Index | `KnexChangeIndexRepository` | `insertPending` (`is_finalized=false`), `finalize`, `findFinalizedByChangeId` / `listReadable` (**finalized only**) |
| Provider | `ProviderRegistry` / `IChangeManagementProvider` / `DevelopmentProvider` | Model C operational record; Dev provider supports `createWithTransaction` + idempotent upsert by `changeId` |
| Finalize txn | `finalizeCreate` | Dev: single knex txn (provider + finalize + complete). External: provider.create then platform txn (finalize + complete) with `change.create.orphan` on platform failure |
| Ledger | `KnexAuthorizationLedgerRepository.createRound` | Requires parent `change_index` row; monotonic `roundNumber`; inserts round + requirements in one txn; append-only |
| Policy | `evaluatePolicy` + `createPolicyRegistry` + published `default-change-authorization@2026-09-02.1` | Pure; input `{classification,risk}` only; not called from submission |
| Selector | `bootstrapAuthorization` → `AuthorizationRuntime` | Startup-validated active policy + bundle + `CatalogPrincipalResolver`; **held in plugin local, not passed to service** |
| Guards | `architecture.test.ts` | Asserts `ChangeManagementService` source does **not** contain `createRound(` |
| Errors | `ChangeManagementError` codes | `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `PROVIDER_UNAVAILABLE`, `INTERNAL_ERROR` |

### Confirmed deviations carried into this plan

1. **Double `buildChange()`** — index pending snapshot and provider/finalize input can diverge (`activityId`, `createdAt`). **Must fix before Round 1 hashing.**
2. **Authorization runtime unwired** — deliberate F3.1.1b STOP; F3.1.2 wires it.
3. **Pre-F3.1.2 binary does not refuse `LEDGER_REQUIRED`** — if an old binary recovered a `LEDGER_REQUIRED` reservation it would finalize without Round 1. Cutover/rollback sequencing must make this unreachable (see §6, §19).

### Drift reconciliation

| Check | Result |
|---|---|
| Accepted planning baseline | `188d8e9` |
| Local `origin/feat/ado-repo-governance` | `188d8e9` |
| HTTPS remote tip | `188d8e9` |
| Post-baseline commits on F3.1.2 surface | **None** |
| Action | Plan against `188d8e9` directly |

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

### Shared prefix (all modes)

```text
parse + payloadHash
→ reserve(idempotency)     [durable regime boundary]
→ if completed → return cached {changeId,status}
→ if changeId+finalized index → heal complete → return
→ catalog target + executionPlan validation
→ ensure changeId claimed
→ ensure pending index exists with ONE canonical Change snapshot
→ branch on authorizationMode
```

### `LEGACY_PRE_F3` branch (unchanged semantics)

```text
canonicalChange = index.snapshot
→ finalizeCreate (provider + finalize + complete)   [existing Model C]
→ return {changeId, status: submitted}
```

No policy, no selector resolution, no Round.

### `LEDGER_REQUIRED` branch

```text
canonicalChange = index.snapshot
→ if Round 1 already exists:
     skip policy/selector; jump to finalizeCreateLedgerRecovery
→ else:
     bind pins from AuthorizationRuntime (activePolicy, selectorBundle)
     evaluatePolicy(activePolicy, {classification, risk})
     resolve each requirement.selectorKey via principalResolver
     validate requiredPrincipalType vs snapshot.principalType
     fail-closed separation-of-duty on resolved User refs
     build AuthorizationRound(roundNumber=1, ...) + audit events (in memory)
→ finalizeCreateLedger:
     [Dev provider] ONE platform txn:
        provider.createWithTransaction(canonicalChange)
        ledger.createRound(round)          # inserts requirements
        ledger.appendAuditEvent(...) × N
        index.finalize(...)
        idempotency.complete(...)
     [External provider] provider.create OUTSIDE txn, then platform txn:
        createRound + audits + finalize + complete
        on platform failure → change.create.orphan + retryable PROVIDER_UNAVAILABLE
→ return {changeId, status: submitted}
```

### Authoritative completion condition (LEDGER)

A logical `LEDGER_REQUIRED` submission is **complete** iff all hold:

1. `change_idempotency.state = completed`;
2. `change_index.is_finalized = true` with matching `authorization_mode`;
3. `change_authorization_rounds` contains `(changeId, roundNumber=1)` with mandatory requirements rows;
4. provider record exists for `changeId` (reconciled on retry if orphaned).

Readers (`GET` list/detail) observe the Change only when (2) holds; (3) is enforced as a precondition of (2).

---

## 6. Authorization-regime cutover decision

### Decision (required #1)

**Cutover mechanism:** a single required config pin under existing authorization config:

```yaml
changeManagement:
  authorization:
    newSubmissionAuthorizationMode: LEGACY_PRE_F3 | LEDGER_REQUIRED
```

| Property | Rule |
|---|---|
| Owner | Platform/DevOps via reviewed app-config (same ownership as active policy pin) |
| Default at F3.1.2 land | `LEGACY_PRE_F3` (binary ships capable; ledger path dark until enabled) |
| Startup validation | Value must be exactly one of the two enums; invalid → fail boot |
| Applies to | **First insert only** of a new idempotency reservation |
| Does not apply to | Existing reservations — stored `authorization_mode` always wins |
| Rollback of switch | Setting back to `LEGACY_PRE_F3` stops *new* ledger reservations; does not reinterpret existing `LEDGER_REQUIRED` rows |

No generalized feature-flag framework.

### Reserve semantics refinement (required for cutover safety)

**Today (`188d8e9`):** retry that *re-requests* a different `authorizationMode` → `CONFLICT`.

**F3.1.2 required behavior:** on an existing row, **stored mode always wins**; a differing requested mode is **not** a CONFLICT (optional warn log only). Payload mismatch remains CONFLICT.

Rationale: F3.1.0-P / planning contract forbid a misleading conflict merely because the deployed binary now defaults to `LEDGER_REQUIRED`. The F3.1.0-V test asserting mode-mismatch CONFLICT must be updated to assert “stored wins”.

Service algorithm:

```text
desiredMode = config.newSubmissionAuthorizationMode
reserved = idempotency.reserve({ ..., authorizationMode: desiredMode })
# repository: insert with desiredMode OR return existing (ignore desiredMode mismatch)
mode = reserved.authorizationMode   # never the desiredMode if existing
if mode == LEGACY → legacy path
if mode == LEDGER_REQUIRED → ledger path (even if config later flipped off)
```

### Required cases

| Case | Behavior |
|---|---|
| Existing `LEGACY_PRE_F3` + same payload after F3.1.2 | Resume legacy path; no policy |
| Existing `LEGACY_PRE_F3` + mismatching payload | `CONFLICT` before Catalog/policy |
| Existing `LEDGER_REQUIRED` + crash retry | Resume ledger recovery; no second Round |
| No reservation yet + switch `LEDGER_REQUIRED` | Insert `LEDGER_REQUIRED`; ledger path |
| No reservation yet + switch `LEGACY` | Insert `LEGACY_PRE_F3`; legacy path |
| Two concurrent first attempts same actor/key | Unique constraint on idempotency PK arbitrates; loser observes winner’s row |
| Same key different actor | Independent reservations (unchanged) |

Regime is **never** inferred from app version, schema version, wall clock, branch, or mere presence of ledger tables.

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

### Additive mandatory requirements — Decision (required #7)

**DEFER** outside F3.1.2. Request contract and dedicated permission (`change.authorization.requirement.add` conceptual) are not ready. F3.1.2 persists `source: 'policy'` only.

### Requirement field mapping

| `ApprovalRequirement` field | Source |
|---|---|
| `changeId` / `roundNumber` | Parent round (`1`) |
| `requirementId` | Server: stable round-local id = `requirementRole` (unique per published rule set; deterministic, no UUID drift on retry-before-commit). If collision ever appears across roles, use `sha256Canonical({role,selectorKey}).slice(0,32)` — pick one scheme and test it; **prefer `requirementRole` as id** given current published artifact uniqueness |
| `kind` / `phase` / `mandatory` | `RequirementDefinition` |
| `source` | `'policy'` |
| `sourceRef` | `requirementRole` |
| `sourceProvenance` | Round `policyProvenance` (policy identity provenance string) |
| `principalSnapshot` | Resolver output |
| `separationOfDutyKey` | Definition (optional) |
| `slaPolicyKey` / `slaPolicyVersion` | Round’s `policyKey` / `policyVersion` when definition has `sla`; else omitted |
| `slaDurationSeconds` / `slaAnchor` | Definition `sla` |
| `createdAt` | Server timestamp shared with round `createdAt` for all requirements in the round |
| `addedByActorRef` / `additionReason` | Omitted (no additive path) |
| `decision` | Absent |

Requirements are immutable after insert (existing triggers).

### Round 1 record

| Field | Value |
|---|---|
| `roundNumber` | `1` |
| `changeSnapshot` / `changeSnapshotSha256` | Canonical index snapshot + `sha256Canonical` |
| Policy identity fields | From evaluation result / active policy |
| `policyInput` | `{ classification, risk }` only |
| Selector bundle fields | From `selectorBundle` (`contentDigest` → `selectorBundleSha256`) |
| `createdAt` | One server ISO timestamp for the round |
| `requirements` | Mapped as above |

### Minimum audit events (ADR-009 submission subset)

Append in the same platform transaction as Round 1 (system identity `systemRef: 'change-management'`):

| `eventType` | Payload (low cardinality) |
|---|---|
| `change.authorization.round_created` | `{ roundNumber: 1, changeSnapshotSha256, policyKey, policyVersion, selectorBundleKey, selectorBundleVersion }` |
| `change.authorization.policy_selected` | `{ policyKey, policyVersion, policyArtifactSha256, matchedRuleProvenance, policyInputSha256 }` |
| `change.authorization.selector_bundle_bound` | `{ selectorBundleKey, selectorBundleVersion, selectorBundleSha256 }` |
| `change.authorization.requirement_materialized` | one per requirement: `{ requirementId, selectorKey, principalType, resolvedPrincipalRef }` (refs are Catalog entity refs already used as principals — not emails) |

No decision, rejection, execution, or eligibility events.

---

## 11. Transaction boundaries and crash-recovery matrix

### Decision (required #3)

**Do not claim distributed atomicity.** Stores and participation:

| Store | Same DB as platform knex? | In create txn today? |
|---|---|---|
| `change_idempotency` | Yes | Reserve **before** create txn; complete **inside** finalize txn |
| `change_index` | Yes | Pending outside/before; finalize inside |
| Authorization ledger | Yes | **New:** createRound + audits inside finalize txn for LEDGER |
| `DevelopmentProvider` | Yes (same knex) | Inside finalize txn via `createWithTransaction` |
| Future external provider | **No** | `create` outside; platform txn after |

### Crash matrix

| # | Crash point | Durable facts | Retry behavior |
|---|---|---|---|
| C1 | After reserve, before changeId claim | Idempotency pending, mode set, no changeId | Claim new/existing changeId; continue |
| C2 | After changeId, before pending index | Reservation has changeId; no index | `buildChange` once → `insertPending` |
| C3 | After pending index, before provider | Pending invisible index; no round | Reuse snapshot; LEDGER: evaluate if no round |
| C4 | After provider create, before platform txn (external) | Provider orphan possible | F2 orphan log; retry `provider.create` by changeId (idempotent); then platform txn |
| C5 | During platform txn before commit | Nothing new durable | Retry from C3/C4 |
| C6 | After Round 1+finalize committed, before idempotency complete | Finalized + Round 1 | `healCompletedIdempotency` / complete; return success |
| C7 | After complete, response lost | Fully complete | `reserve` sees `completed` → return same `{changeId,status}` |

### LEDGER extension to F2 orphan semantics

Reuse `change.create.orphan` when external provider succeeded and platform txn failed. On retry, before finalize: if Round 1 missing, create it in the platform txn; if Round 1 present, only finalize+complete as needed. **Never** call `createRound` with `roundNumber=2` from submission retry — monotonic guard + `findRound(1)` short-circuit.

### DevelopmentProvider path

Single knex transaction includes provider upsert + `createRound` + audits + finalize + complete. Stronger atomicity than external providers; still not a general distributed claim.

---

## 12. Visibility / finalization invariant

### Decision

| Reader | Sees Change when |
|---|---|
| `GET /changes` | `is_finalized=true` (+ participant predicate) |
| `GET /changes/:id` | `findFinalizedByChangeId` |
| Future authorization reads | Round exists (F3.1.3+); out of scope to expose now |

**Invariant:** never finalize a `LEDGER_REQUIRED` index row unless `findRound(changeId,1)` would succeed inside the same transaction (check-then-finalize). Conversely, unfinalized pending + optional absent round remains invisible.

**Authoritative completion:** §5 completion condition.

---

## 13. Idempotent replay and concurrency

### Replay invariants

| Invariant | Mechanism |
|---|---|
| No duplicate provider record | Provider upsert / create-by-changeId |
| No duplicate index | PK on `change_id`; insert race → re-read |
| No Round 2 from submission retry | `findRound(1)` short-circuit; `createRound` monotonicity |
| No duplicate Round 1 | Unique `(change_id, round_number)` |
| No duplicate requirements | Inserted only with Round 1 |
| No principal drift after Round 1 | Skip resolve when round exists |
| No policy/bundle drift after Round 1 | Skip evaluate when round exists |
| Conflicting payload | `CONFLICT` at reserve |

### Concurrency

| Scenario | Arbiter |
|---|---|
| Two workers same actor/key | Idempotency PK unique insert |
| Duplicate index insert | `change_id` PK; loser re-reads |
| Concurrent provider create | Provider idempotency by `changeId` |
| Two different keys | Two Changes |
| Retry vs in-flight complete | Finalized/completed short-circuits; DB constraints on double finalize/complete are benign (complete updates `state=pending` only) |

**No in-memory mutex as correctness authority.**

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

### Decision (required #10)

**Question:** What happens to a logical submission reserved as `LEDGER_REQUIRED` if the binary rolls back before completion?

**Answer:** A pre-F3.1.2 binary must **never** process that reservation. Therefore:

### Safe sequence

1. **Deploy F3.1.2 binary** with `newSubmissionAuthorizationMode: LEGACY_PRE_F3` (ledger path present but unused for *new* keys).
2. Verify legacy + recovery tests in target env.
3. **Enable** `LEDGER_REQUIRED` for new submissions.
4. **Rollback rules:**
   - **Switch rollback** (LEDGER → LEGACY): allowed anytime; pending `LEDGER_REQUIRED` still handled by F3.1.2 binary ledger path; new keys become legacy.
   - **Binary rollback to pre-F3.1.2:** **FORBIDDEN** while any `change_idempotency` row exists with `authorization_mode='LEDGER_REQUIRED'` AND `state='pending'`. Ops must drain (complete or abandon via supported recovery) first.
   - Optional hard guard in F3.1.2a/b: if somehow an older build is unavoidable, do not ship cutover enablement until monitoring proves zero pending ledger reservations.

5. Document runbook query:

```sql
SELECT COUNT(*) FROM change_idempotency
WHERE authorization_mode = 'LEDGER_REQUIRED' AND state = 'pending';
```

Must be `0` before binary rollback past F3.1.2.

This is a **compatibility gate + config sequencing**, not a feature-flag product.

---

## 20. Test matrix

### Pure / domain (unit)

| ID | Case | Dialect |
|---|---|---|
| U1 | Single `buildChange` → identical snapshot/hash for index & round | n/a |
| U2 | Policy input derivation `{classification,risk}` only | n/a |
| U3 | Requirement mapping from definition + snapshot | n/a |
| U4 | Emergency A/B same `User` ref rejected | n/a |
| U5 | Deterministic Round 1 construction (stable requirementIds) | n/a |

### Idempotency / cutover (unit + integration)

| ID | Case |
|---|---|
| I1 | Legacy reservation retry after F3.1.2 binary + switch LEDGER → legacy path |
| I2 | New LEDGER happy path |
| I3 | Same key/same payload replay after complete |
| I4 | Same key/different payload CONFLICT (no policy work) |
| I5 | Stored mode wins when desired mode differs (replace old CONFLICT test) |
| I6 | Crash C1–C7 recovery (failure injection) |

### Persistence

| ID | Case | SQLite | Postgres |
|---|---|---|---|
| P1 | Round+requirements+audit commit | ✓ | ✓ |
| P2 | Platform txn rollback leaves no round/finalize | ✓ | ✓ |
| P3 | Uniqueness / concurrent createRound(1) | ✓ | ✓ |
| P4 | Finalize rejected without Round 1 (LEDGER) | ✓ | ✓ |
| P5 | Immutability triggers still pass | ✓ | ✓ |

### Provider / Model C

| ID | Case |
|---|---|
| M1 | Dev provider success inside shared txn |
| M2 | External fake provider fail before create |
| M3 | External success + platform txn fail → orphan log + retry converges |
| M4 | No provider-specific auth fields in round JSON |

### Policy / selector

| ID | Case |
|---|---|
| S1 | Active published policy/bundle used |
| S2 | Catalog missing → NOT_FOUND; no round |
| S3 | Catalog unavailable → 503; no round |
| S4 | Same-person emergency |
| S5 | After Round 1, flip runtime pins in test double → replay unchanged |

### Regression

| ID | Case |
|---|---|
| R1 | All F2 / F3.1.0 / F3.1.1 suites green |
| R2 | List/detail participant visibility unchanged |
| R3 | Architecture guards updated, still forbid Delivery/Teams authority leakage |
| R4 | Lint / build |
| R5 | Repo-wide `tsc` error set set-identical to `188d8e9` baseline (file/line/col/code) |

### Failure-injection harness

Reuse patterns in `ChangeManagementService.recovery.test.ts` / fake provider: throw after reserve, after pending insert, after provider create, after round insert (txn abort), after finalize before complete.

**Live Catalog:** opt-in only (existing `CatalogPrincipalResolver.live.test.ts` style); not required for merge if unit resolver fakes cover fail-closed paths.

---

## 21. Exact source paths expected to change

### F3.1.2a — canonical Change (micro-slice)

```
packages/backend/src/modules/changeManagement/ChangeManagementService.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.recovery.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.integration.test.ts
```

### F3.1.2b — ledger submission

```
packages/backend/src/modules/changeManagement/ChangeManagementService.ts
packages/backend/src/plugins/changeManagementPlugin.ts
packages/backend/src/modules/changeManagement/persistence/KnexIdempotencyRepository.ts
packages/backend/src/modules/changeManagement/persistence/idempotencyAuthorizationMode.test.ts
packages/backend/src/modules/changeManagement/authorization/selector/config.ts  # read newSubmissionAuthorizationMode
packages/backend/src/modules/changeManagement/authorization/selector/types.ts
app-config.yaml  # default LEGACY_PRE_F3 + comment
packages/backend/src/modules/changeManagement/architecture.test.ts
# new focused tests:
packages/backend/src/modules/changeManagement/ChangeManagementService.ledgerSubmit*.test.ts
```

Possibly small pure helpers (same module tree), e.g. `authorization/buildRoundFromPolicy.ts` — only if it keeps the service readable; not a new package.

**Must not change:** frontend, Delivery, migrations (unless review finds a true schema gap — none found), ADR-009/012, decision APIs, production selector fabrication.

---

## 22. Slice decomposition / implementation order

### Decision (required #12)

**Two explicitly ordered micro-slices:**

| Order | Slice | Content | Enables LEDGER submissions? |
|---|---|---|---|
| 1 | **F3.1.2a** | Single canonical `buildChange` + snapshot reuse on recovery | No (`LEGACY` only still) |
| 2 | **F3.1.2b** | Mode-wins reserve semantics, DI wiring, LEDGER path, Round 1, tests, config switch default OFF then enablement runbook | Yes, when switch set |

Split justified by correctness risk isolation (hash/snapshot) and safer rollback of review comments — not aesthetics.

No third slice for tests/docs.

---

## 23. Risks and rejected alternatives

| Rejected | Why |
|---|---|
| Infer mode from deploy time / schema | Violates cross-cutover invariant |
| Keep mode-mismatch CONFLICT | Breaks legacy retry after cutover binary |
| Second `buildChange` “usually ok” | Breaks Round hash / provider equality |
| Distributed XA / outbox framework | Out of scope; F2 orphan/retry suffices |
| Additive requirements in F3.1.2 | Permission/contract incomplete |
| Migration “just in case” | No missing invariant |
| Re-evaluate policy on every retry | Violates immutable round evidence |
| Finalize before Round 1 | Violates visibility invariant |
| Roll back binary freely after enablement | Can finalize LEDGER without Round on old binary |
| Put authorization inside provider | Violates ADR-007/009 |
| Delivery correlation fields on Round | Violates ADR-012 |

### Residual risks (accepted, mitigated)

- Window between pending index and Round 1 where Catalog/policy can still change on retry — mitigated by fail-closed + invisible pending; binding at Round commit.
- External provider orphan — mitigated by existing reconciliation.
- Production still inherits dev selector bundle until prod publication — **unchanged F3.1.1b deviation**; production rollout separately gated.

---

## 24. Answers to all 15 challenge scenarios

1. **Legacy reserved yesterday, deploy today, retry:** Stored `LEGACY_PRE_F3` wins; legacy finalize path; no policy/Round.
2. **LEDGER reserved, crash before provider:** Reservation (+ maybe changeId/pending index) survives; retry reuses snapshot; evaluates only if Round 1 absent; continues.
3. **Provider create OK, crash before Round/index finalize:** Orphan log; retry idempotent provider create; platform txn creates Round 1 + finalize + complete; no second provider row.
4. **Round 1 durable, client retries:** `findRound(1)` / finalized/completed short-circuit; no re-evaluation; pins frozen in ledger.
5. **Emergency A/B same User:** Fail closed before Round insert; **nothing** authorization-durable; pending index may exist but is invisible; no requirements persisted.
6. **Catalog unavailable mid-resolution:** No Round; no finalize; 503; pending invisible; retry later.
7. **Active bundle changes between two separate requests:** Allowed (different logical submissions). **Between retries of same logical request before Round 1:** re-bind to current runtime possible. **After Round 1:** forbidden / skipped.
8. **Two workers same key:** Idempotency table primary key arbitrates.
9. **Rollback to pre-F3.1.2 with outstanding LEDGER pending:** Prevented by runbook drain + policy that binary rollback is forbidden until pending LEDGER count is 0; switch-off alone is insufficient for binary rollback.
10. **Future external ITSM provider:** Platform txn does not include provider create; correctness via idempotent create + orphan retry; no 2PC claim.
11. **Mismatching payload:** `CONFLICT` at reserve **before** policy/Catalog authorization work (Catalog target validation occurs only after successful reserve — implementers must keep authorization evaluation after the idempotency gate; payload CONFLICT returns immediately in `reserve`).
12. **Successful F3.1.2 submission ≠ authorized:** Lifecycle `submitted`; `AuthorizationEvaluation` remains `PENDING` until mandatory pre-execution decisions (F3.1.3). Round 1 only materializes requirements.
13. **Does Round 1 change lifecycle?** **No.** Stays `submitted`. ADR-009: `authorized` is not a lifecycle state.
14. **Inactive historical policy on replay?** After Round 1, replay uses **ledger** artifacts, not registry. Runtime historical modules are unnecessary for submission replay (Model B).
15. **Discoverability condition:** `change_index.is_finalized = true` (and for LEDGER, finalize only after Round 1 exists). List/detail both require finalized.

---

## 25. Implementation acceptance criteria

A future implementation checkpoint may claim PASS only when:

- [ ] F3.1.2a merged/proven: single canonical snapshot on create/recovery; F2 suites green
- [ ] F3.1.2b: `LEDGER_REQUIRED` path creates exactly Round 1 + mapped requirements + audit subset
- [ ] Stored mode wins; legacy retries never enter policy path
- [ ] Emergency same-person fail-closed with zero durable round
- [ ] Crash matrix C1–C7 covered by tests (SQLite + disposable Postgres)
- [ ] No migration; no decision APIs; no Delivery fields
- [ ] Architecture guards updated and green
- [ ] `tsc` baseline set-identical; lint/build green
- [ ] Default config remains `LEGACY_PRE_F3` until explicit enablement
- [ ] Rollback runbook documented in implementation evidence

---

## 26. GO / NO-GO recommendation for a separate implementation checkpoint

```text
F3.1.2 planning: READY_FOR_REVIEW
F3.1.2 implementation: NO-GO
```

**Next gate:** independent architecture review of **this plan**. Only after ACCEPT may a separate, constrained implementation prompt authorize F3.1.2a (then F3.1.2b).

This checkpoint does **not** create that implementation prompt.

---

## 27. STOP

STOP. Do not implement F3.1.2, fix `buildChange()` in ADO, add migrations/routes, wire `POST /changes`, create AuthorizationRounds, create an implementation prompt, or start F3.1.3/F3.1.4.

---

## Appendix A — Explicit decisions checklist (prompt §7)

| # | Decision | Resolution |
|---|---|---|
| 1 | Cutover mechanism | Config `newSubmissionAuthorizationMode`; default LEGACY; stored mode wins |
| 2 | `buildChange` fix | F3.1.2a prerequisite micro-slice |
| 3 | Transaction / recovery | §5 + §11 state machine; no distributed atomicity |
| 4 | Earliest durable auth binding | Round 1 commit |
| 5 | Round 1 + requirements + audit boundary | Same platform DB transaction as finalize+complete (Dev: includes provider) |
| 6 | A/B distinctness | Resolved User refs distinct per `separationOfDutyKey` before commit |
| 7 | Additive requirements | Deferred |
| 8 | Migration | NO |
| 9 | DI shape | Inject `AuthorizationRuntime` + ledger repo + mode enum into service |
| 10 | Rollback rule | No pre-F3.1.2 binary while pending LEDGER reservations exist |
| 11 | Error taxonomy | Existing codes + `details.reason=separation_of_duty` |
| 12 | Slice decomposition | Two micro-slices: 2a canonical build, 2b ledger submit |

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
