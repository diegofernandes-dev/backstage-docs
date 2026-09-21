# F3.1.3b — Rejected-Change Resubmission / New-Round Implementation

## Status

IMPLEMENTATION PROMPT — DO NOT EXECUTE WITHOUT EXPLICIT USER LAUNCH.

Purpose: implement only **F3.1.3b**, the same-`changeId` rejected-Change correction/resubmission path that creates a new immutable authorization Round while preserving Change identity and all prior authorization history.

Canonical authority:
- docs/backstage/f3-1-3-implementation-plan.md
- docs/backstage/f3-1-3-revised-plan-architecture-rereview.md
- docs/backstage/f3-1-3a-architecture-implementation-acceptance.md
- docs/adr/ADR-007-change-record-authority.md
- docs/adr/ADR-008-multi-activity-change-execution-plan.md
- docs/adr/ADR-009-change-authorization-model.md
- docs/adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md
- docs/adr/ADR-014-change-resubmission-authority-and-identity-boundary.md
- docs/backstage/current-state.md

Accepted implementation baseline:
- ADO repo: `platform-devops-developer-portal`
- branch: `feat/ado-repo-governance`
- F3.1.3a accepted SHA: `6bad066d945d49feaf642313ec37467e2658dc3f`
- parent F3.1.2 accepted SHA: `22495229502dabf2d99588599a156d862c5114fa`
- accepted laptop demo target: isolated local runtime with gitignored `LEDGER_REQUIRED` overlay
- committed repository default remains `LEGACY_PRE_F3`

F3.1.3a is CLOSED / ACCEPTED. F3.1.3b implementation is authorized only when this prompt is explicitly launched.

## 1. Mandatory fresh baseline

Before editing:
1. Fetch latest `backstage-docs@main`; record exact SHA.
2. Read every authority file above, implementation-progress, prompts/README, and this prompt.
3. Fetch the live ADO branch tip and verify whether it is exactly `6bad066` or a later accepted descendant.
4. Inspect every commit/path after `6bad066` before editing.
5. If drift touches resubmission prerequisites, decision semantics, ledger Round persistence, index/provider composition, participant index, permissions/RBAC, Catalog membership, idempotency, or authorization policy/selector runtime in a conflicting way, STOP with `BLOCKED_BY_SOURCE_DRIFT`.
6. Verify committed `app-config.yaml` still defaults `newSubmissionAuthorizationMode: LEGACY_PRE_F3`.
7. Verify no existing source has already introduced `/resubmissions`, `change.resubmit`, Round-N writer logic, or provider replacement outside an independently accepted checkpoint.

Do not implement against stale source.

## 2. Exact objective

Implement this bounded transition only:

```text
current Change
  + current Round == REJECTED
  + authorized resubmission actor
  + corrected non-identity payload
        ↓
same changeId
same immutable identity
        ↓
new canonical corrected Change snapshot
        ↓
currently published policy + selectors
        ↓
new Round N+1 + new requirements
        ↓
submission-like authorization audits + resubmitted audit
        ↓
current discovery/provider projection replaced atomically
        ↓
status = submitted
```

All earlier Rounds, requirements, decisions, and audits remain immutable.

Do not implement generic edit/update of Change.

## 3. Transport

Implement:

```text
POST /api/change-management/changes/:changeId/resubmissions
Idempotency-Key: <required non-empty key>
```

User credentials only.

The request body may reuse the current create-shaped user-editable fields for ergonomics, but the server must enforce the identity freeze below. Do not accept client-controlled `requestedBy`, `ownerRef`, `systemRef`, identity `createdAt`, authorization facts, provider identifiers, or workflow metadata.

`targetRef` may be present only if it canonicalizes to the original immutable `targetRef`. A different target is not a correction.

Use existing public error codes and stable `details.reason` values from the accepted plan.

For response semantics, keep the API bounded to this command. Recommended minimal contract:

```ts
{
  changeId: string;
  roundNumber: number;
  status: 'submitted';
}
```

First successful materialization returns 201. Exact idempotent replay returns 200 with the same logical result. Do not expose ledger internals or create an F3.1.4 read model.

If current source conventions make a different equally bounded response necessary, document it explicitly in evidence and preserve first-commit/replay distinction; do not widen the response into provider/governance internals.

## 4. Resubmission permission and business authority

Register only the F3.1.3b permission:

```text
change-management.change.resubmit
action = update
```

An actor may resubmit iff BOTH hold:

```text
actor has change-management.change.resubmit
AND
(actorRef == original Change.requestedBy
 OR actor is a current member of immutable ownerRef Group)
```

Hard rules:
- original requester still needs the explicit permission;
- owner member still needs the explicit permission;
- `platform_admin` alone is not business authority;
- CAB membership / `cab.record` alone is not business authority;
- `change.create`, `change.read`, participant read, `responsibleRef`, or `decide` alone are not business authority;
- no generic governance override.

RBAC may grant the literal technical permission to `contributor`, `platform_admin`, and/or `template_executor` according to the accepted current role model, but service-side requester-or-owner proof is mandatory.

Do not grant `change.resubmit` to `change_cab_recorder` as a CAB power.

## 5. Owner-membership proof

Reuse the F3.1.3a prefix-agnostic live Catalog membership helper rather than TP-filtered `ownershipEntityRefs`.

For the owner path:
- compare against the **immutable original ownerRef**, not current Catalog ownership of the target;
- ownerRef must represent a resolvable Group for the membership path;
- current authenticated User membership is proven from normalized/deduped `relations.memberOf` + `spec.memberOf`; 
- Catalog/credentials unavailable -> `PROVIDER_UNAVAILABLE`, `membership_source_unavailable`, zero writes;
- User absent -> `FORBIDDEN`, `actor_not_in_catalog`, zero writes;
- immutable owner Group missing/unresolvable -> `FORBIDDEN`, `owner_authority_unresolvable`; 
- original Change with no ownerRef -> only original requester + permission can resubmit.

Important optimization/semantics:
- if actor is exactly original `requestedBy`, do not make Catalog owner-membership availability a prerequisite for that requester path;
- Catalog I/O is required only when proving the owner-member alternative.

All permission/domain/Catalog proof occurs before opening the write transaction.

## 6. Same-changeId immutable identity

These fields are frozen across all rounds:

```text
changeId
requestedBy
createdAt   # identity/birth time
targetRef
ownerRef
systemRef
```

Rules:
- never re-resolve ownerRef/systemRef from current Catalog during resubmission;
- Catalog ownership drift cannot transfer authority or rewrite identity;
- changed `targetRef` -> `VALIDATION_ERROR`, `change_identity_mismatch`, before transaction mutation;
- any source-level attempt to alter requestedBy/createdAt/ownerRef/systemRef -> same fail-closed result;
- retargeting means create a new Change/new changeId.

`DevelopmentProvider.replaceCurrent` and index projection code must preserve these exact original identity values.

## 7. Correctable fields

Only non-identity business/execution fields already supported by the create validation contract may change, including as source permits:
- title;
- summary/description;
- classification;
- risk;
- requested execution window;
- rollback information;
- evidence/references;
- execution plan.

Mint new activity IDs for the corrected execution plan according to the accepted canonical Change-building rules.

Do not invent new edit-only fields.

## 8. Execution-plan hidden-retarget guard

ADR-008 permits optional per-activity Component `targetRef`; F3.1.3b must prevent that from becoming a hidden cross-System retarget.

Rules:
- omitted activity targetRef remains allowed;
- present activity targetRef must remain kind `Component` and resolve via Catalog;
- resolve its System using the same canonical `spec.system` rules used by target-context resolution;
- resolved activity System must equal immutable Change `systemRef`, including both absent;
- mismatch -> `VALIDATION_ERROR`, `execution_plan_identity_mismatch`, before Round/provider/index mutation;
- activity target never rewrites Change targetRef/ownerRef/systemRef.

Do not change create-path semantics in this slice unless required solely to share a pure validation helper without behavior change.

## 9. When resubmission is allowed

Resubmission is allowed only when the **current max Round** derives:

```text
AuthorizationEvaluation = REJECTED
```

Equivalent projected lifecycle should be `rejected`, but the immutable ledger evaluation is authoritative for eligibility to resubmit.

Reject:
- PENDING current Round;
- AUTHORIZED current Round;
- missing Round;
- LEGACY_PRE_F3 Change;
- stale/non-current historical rejected Round when a newer Round exists.

Use `CONFLICT`, `details.reason=round_not_terminal` where contracted, and fail closed on ledger/index invariant mismatch.

## 10. New canonical snapshot / Round N

Build one new canonical corrected Change snapshot with:
- same changeId;
- frozen identity fields;
- corrected allowed fields;
- status submitted;
- new server materialization time only where the canonical model already distinguishes such a field; do not overwrite identity createdAt;
- new canonical execution-plan activity IDs;
- no provider-specific fields.

Bind the **currently published** immutable authorization policy and selector bundle at resubmission time.

Re-evaluate current policy from the corrected classification/risk.
Resolve new principal snapshots deterministically using existing selector runtime.
Apply the same fail-closed principal type and separation-of-duty validation used by Round 1 submission.

Create exactly the next monotonic Round:

```text
roundNumber = currentRound.roundNumber + 1
```

`requirementId = requirementRole` remains exact.
Prior Round snapshots/requirements/decisions/audits remain untouched.

## 11. Generalize ledger submission safely

Current Round-1 submission helpers may contain Round-1 assumptions. Generalize only the reusable materialization needed for F3.1.3b.

Mandatory:
- submission-like audit payloads use the actual `round.roundNumber`, never hardcoded 1;
- F3.1.2 create path must still create **only Round 1**;
- F3.1.3b resubmission is the only new path that may create Round N>1;
- do not turn ledgerSubmission into a generic workflow engine;
- preserve F3.1.2 idempotency/cutover behavior unchanged.

New Round audits inside the resubmission transaction:
- `change.authorization.round_created`; 
- `change.authorization.policy_selected`; 
- `change.authorization.selector_bundle_bound`; 
- one `change.authorization.requirement_materialized` per requirement;
- exactly one `change.authorization.resubmitted` for the successful new Round.

Do not copy prior decisions into the new Round.

## 12. Idempotency contract

Use the existing idempotency repository with new operation:

```text
change.resubmit
```

No migration.

Contract:
- `requested_by` on this operation = authenticated resubmitting actor;
- changeId remains the existing Change ID;
- required new Idempotency-Key;
- payload hash covers `changeId` + normalized corrected request body;
- original `change.create` reservation is never reused or mutated;
- existing authorization_mode facts are never rewritten;
- exact same actor/key/payload replay returns the same logical Round result without another Round/audit/provider/index rewrite;
- same actor/key different payload -> existing payload-mismatch conflict semantics.

Reservation + completion must live inside the serialized resubmission transaction so a losing Round race cannot leave a dangling pending resubmit reservation.

Do not implement a second idempotency store.

## 13. Current projection / Model C composition

After successful Round N materialization, atomically rebuild the current product projection for the same Change.

### Index
Update only current **non-identity** discovery columns from corrected snapshot, including source-supported:
- title;
- summary;
- requested window;
- classification;
- risk;
- rollback/evidence;
- executionPlan;
- status -> `submitted`.

Leave index identity columns untouched:
- target_ref;
- owner_ref;
- system_ref;
- requested_by;
- created_at;
- authorization_mode.

### Participants
Rebuild `change_index_activity_participants` from the corrected execution plan inside the same transaction. It is derived/non-authoritative discovery state.

### DevelopmentProvider
Add explicit:

```text
replaceCurrent(change, trx)
```

for resubmission only.

Do not call/reinterpret `create()` or `createWithTransaction()` for corrected snapshots.

`replaceCurrent` must:
- update only the existing provider record for the same changeId;
- preserve frozen identity exactly;
- write current corrected non-identity operational detail;
- result status `submitted`; 
- participate in the caller-owned transaction;
- fail if the expected current record is absent/incoherent rather than silently creating a new Change.

External/non-development provider resubmission is not supported in this slice. Fail closed with existing `PROVIDER_UNAVAILABLE` semantics; no XA/2PC/outbox.

GET detail after successful resubmission should show the corrected current provider record with original identity and submitted status. List should show corrected current non-identity index projection.

## 14. Caller-owned transaction

After permission/domain/Catalog/identity/execution-plan validation succeeds, one caller-owned Knex transaction must:
1. lock the `change_index` parent row (`FOR UPDATE` on PostgreSQL);
2. re-read authoritative index/current Round through the same transaction;
3. verify LEDGER_REQUIRED + current Round is REJECTED;
4. re-verify frozen identity against locked authoritative state;
5. resolve the next expected Round number from locked current state;
6. reserve `change.resubmit` idempotency inside this transaction;
7. materialize new Round N + requirements;
8. append Round/policy/selector/requirement audits + `resubmitted`; 
9. project index non-identity fields + status submitted;
10. rebuild participants;
11. `DevelopmentProvider.replaceCurrent(change, trx)`;
12. complete resubmit idempotency;
13. commit.

All durable product state changes are atomic in DevelopmentProvider mode.

No Catalog network I/O while holding the write lock. Policy/selector evaluation and principal resolution should occur before the final transaction where possible, then locked state is revalidated before commit.

## 15. Concurrency contract

PostgreSQL is authoritative.

Two concurrent resubmissions after the same rejected Round:
- serialize on parent `change_index FOR UPDATE`; 
- at most one Round N+1 is created;
- winner atomically updates projection/provider/participants/idempotency;
- loser must not create Round N+2 as an accidental retry;
- loser returns `CONFLICT`, `details.reason=resubmission_conflict`; 
- loser leaves no pending idempotency reservation or partial projection.

Exact same actor/key/payload replay after winner commit is not the competing-resubmission conflict; it returns the original logical result.

Do not use polling, sleeps, distributed locks, in-memory mutexes, queues, or background workers.

## 16. Error contract

Use existing error codes. Required cases:
- missing/blank Idempotency-Key / malformed body -> `VALIDATION_ERROR` / 400;
- no resubmit permission -> `FORBIDDEN` / 403;
- permission but not requester/immutable-owner member -> `FORBIDDEN`, `not_resubmission_actor`; 
- owner membership source unavailable -> `PROVIDER_UNAVAILABLE`, `membership_source_unavailable`; 
- actor absent -> `FORBIDDEN`, `actor_not_in_catalog`; 
- immutable owner Group missing/unresolvable -> `FORBIDDEN`, `owner_authority_unresolvable`; 
- Change missing -> `NOT_FOUND`; 
- LEGACY_PRE_F3 -> `CONFLICT`, `ledger_required_only`; 
- current Round not REJECTED -> `CONFLICT`, `round_not_terminal`; 
- identity mismatch -> `VALIDATION_ERROR`, `change_identity_mismatch`; 
- hidden execution-plan System mismatch -> `VALIDATION_ERROR`, `execution_plan_identity_mismatch`; 
- same key changed body -> `CONFLICT`, `idempotency_payload_mismatch`; 
- competing concurrent resubmission loser -> `CONFLICT`, `resubmission_conflict`; 
- external provider unsupported -> `PROVIDER_UNAVAILABLE`; 
- invariant contradiction -> `INTERNAL_ERROR` fail closed.

Do not create new HTTP status classes merely for internal distinctions.

## 17. Mandatory proof matrix

### R1 — authoritative PostgreSQL concurrency
Two concurrent resubmissions after one rejected Round:
- exactly one Round N+1;
- loser `resubmission_conflict`; 
- no Round N+2;
- one provider replacement;
- one index current-projection rewrite;
- one participant rebuild set;
- one canonical resubmission audit set;
- no dangling resubmit reservation.

Use deterministic barriers/hooks, not sleeps.

### R2 — non-rejected current Round
PENDING and AUTHORIZED current rounds both reject resubmission with zero new Round.

### R3 — history immutability
After Round 2:
- Round 1 snapshot unchanged;
- Round 1 requirements unchanged;
- Round 1 decisions unchanged;
- Round 1 audits unchanged;
- append-only UPDATE/DELETE guards still pass.

### R4 — original requester
- requester + resubmit permission succeeds;
- same requester without permission is forbidden, zero Round.

### R5 — immutable-owner member
- current member of frozen owner Group + permission succeeds;
- remove live membership -> forbidden, zero Round.

### R6 — platform_admin not authority
`platform_admin` with technical permission but neither requester nor owner member -> forbidden.

### R7 — CAB not authority
CAB member / cab.record holder but neither requester nor immutable-owner member -> forbidden.

### R8 — permission alone
Actor with resubmit permission but no requester/owner proof -> `not_resubmission_actor`.

### R9 — Catalog ownership drift
If current target Catalog owner differs from frozen ownerRef, new owner membership alone cannot resubmit the old Change.

### R10 — target identity
Different targetRef under same changeId fails before any Round/provider/index/participant/idempotency completion mutation.

### R11 — owner/System identity
Any path that attempts to alter frozen ownerRef/systemRef/requestedBy/createdAt fails closed before mutation.

### R12 — allowed correction
Correct allowed fields -> exactly Round N+1, new current policy/principal/requirements, status submitted, current non-identity projection updated, frozen identity preserved everywhere.

### R13 — execution-plan hidden retarget
Activity Component resolving to different System from immutable Change systemRef -> `execution_plan_identity_mismatch`, zero durable mutation.

Functional R2-R13 should run on SQLite and/or PostgreSQL as appropriate. R1 concurrency must run on disposable PostgreSQL 16.

## 18. Failure injection

Prove atomic rollback for at least:
- after Round N insert before audits;
- after audits before index projection;
- after index projection before participant rebuild;
- after participants before provider replacement;
- after provider replacement before idempotency completion/commit.

Every injected failure must leave:
- no new durable Round;
- no new resubmission audits;
- old rejected current projection intact;
- old provider record intact;
- old participant rows intact;
- no completed/dangling resubmit reservation from the failed attempt.

## 19. F3.1.3a regression contract

F3.1.3b must not change accepted F3.1.3a behavior.

Re-prove at least:
- decision route still works;
- individual authority unchanged;
- CAB membership / `cab.record` unchanged;
- decision idempotency and D1-D6 remain green;
- rejected lifecycle remains canonical decision/audit + index projection;
- exact replay still stable;
- eligibility missing-Round safety unchanged;
- post-execution fail-closed unchanged;
- `CHG-2026-000003` or durable equivalent remains AUTHORIZED/submitted and gains **no** Round 2 from resubmission work.

Do not repurpose decision permissions as resubmit authority.

## 20. Live non-production product proof

Use only the accepted isolated laptop `LEDGER_REQUIRED` target if still available and safe.

Preferred resubmission target is existing disposable rejected `CHG-2026-000005` **only if** it still has:
- exactly one rejected Round;
- immutable rejection decision/audits from F3.1.3a;
- no existing Round 2;
- actor/identity facts compatible with ADR-014.

If those facts changed, do not rewrite/delete history. Create a fresh disposable LEDGER_REQUIRED Change, reject one mandatory pre requirement through the accepted decision API, then use that Change.

Live proof:
1. capture Round-1/index/provider/participant facts before resubmit;
2. resubmit as a genuinely authorized actor with a new Idempotency-Key and a harmless allowed correction;
3. prove same changeId and Round 2;
4. prove Round 1 decisions/audits untouched;
5. prove Round 2 uses current policy/selectors/new requirements and zero decisions;
6. prove list/detail show corrected current fields and `submitted`; 
7. prove frozen target/owner/system/requester/createdAt unchanged;
8. prove execution eligibility now evaluates Round 2 and is pending authorization (DENY / PENDING_AUTHORIZATION unless another accepted gating reason has priority; record actual response);
9. exact replay returns same Round 2 with no duplicate audit/projection;
10. prove a forbidden identity-change attempt does not mutate state.

Do not approve Round 2 in this checkpoint merely to continue the story. F3.1.3b proof ends with the new Round materialized and awaiting new decisions.

Browser/product checks:
- GMUD list/detail reachable;
- rejected -> submitted projection visible after resubmit;
- Catalog reachable;
- Deployments tab reachable;
- Delivery reads healthy;
- no new extension/config collision.

F3.1.3b is API-only. Do not add a resubmit button.

## 21. Expected source surface

Use actual current paths after source verification. Expected changes may include:
- Change Management types/request validation for resubmit;
- `permissions.ts` + RBAC CSV (+ template executor seed only if justified by current role policy);
- `authorization/decisionMembership.ts` reuse/generalization for owner membership;
- `ChangeManagementService` resubmit orchestration;
- `authorization/ledgerSubmission.ts` safe Round-N generalization;
- plugin route `POST /changes/:changeId/resubmissions`; 
- `ChangeIndexRepository` / Knex implementation for current non-identity projection;
- participant-index repository support for transactional rebuild;
- `IChangeManagementProvider` / `DevelopmentProvider.replaceCurrent`; 
- execution-plan validation helper for same-System resubmit guard;
- idempotency operation `change.resubmit` using existing repository;
- focused SQLite/PostgreSQL tests including R1-R13 + failure injection;
- architecture guards.

No migration files.
No F3.1.4 frontend authorization UI.

## 22. Full validation

Run at minimum:
- focused F3.1.3b unit/integration tests;
- PostgreSQL 16 authoritative R1;
- R2-R13 negatives/positive proofs;
- F3.1.3a PostgreSQL D1-D6 regression;
- full Change Management SQLite suite;
- F3.1.2 create/idempotency/concurrency regressions;
- F3.1.1 policy/selector/publication regressions;
- GMUD frontend regression;
- Catalog + Deployments regression;
- Delivery backend regression;
- permission/RBAC tests;
- participant read/list/detail tests;
- architecture guards;
- lint;
- full build;
- repository-wide TypeScript baseline comparison.

Accepted historical TypeScript debt is allowed only if the exact pre-change candidate baseline set is unchanged. Zero new errors.

No `--forceExit`.

## 23. Forbidden scope

MUST NOT:
- change F3.1.3a accepted decision semantics;
- add approve/reject/resubmit UI buttons;
- implement F3.1.4 composed authorization/governance read panel;
- implement CAB Workbench;
- implement F3.2 autonomy;
- add Teams approval;
- implement decision reversal/expiry;
- implement execution start/completion lifecycle;
- allow retarget under same changeId;
- add ownership-transfer administration;
- support external-provider resubmission by inventing 2PC/outbox;
- add migrations;
- change committed default to LEDGER_REQUIRED;
- perform production cutover;
- mutate Kargo/Argo/GitOps desired state;
- introduce workflow engine/queue/distributed lock.

## 24. Publication

If every implementation gate passes:
1. commit only F3.1.3b-scoped source/tests/config;
2. parent must be the verified accepted live ADO tip, expected `6bad066` unless later accepted drift is explicitly reconciled;
3. normal fast-forward push to `feat/ado-repo-governance`; 
4. no force push/history rewrite;
5. independently verify remote ADO tip;
6. record exact parent and final implementation SHA.

## 25. Canonical evidence

After a real implementation attempt create:
- `docs/backstage/f3-1-3b-implementation-evidence.md`

Update factually:
- `docs/backstage/current-state.md`; 
- `docs/backstage/implementation-progress.md`; 
- `prompts/README.md`.

Evidence must record:
- docs baseline;
- ADO parent/final SHA;
- complete changed-path inventory;
- resubmit permission/RBAC;
- requester/owner authority proof behavior;
- identity-freeze enforcement;
- Catalog ownership-drift proof;
- execution-plan System guard;
- Round-N policy/selector/requirements evidence;
- idempotency behavior;
- provider/index/participant transaction composition;
- R1-R13 results;
- failure-injection results;
- F3.1.3a D1-D6 regression;
- GMUD/Catalog/Deployments regressions;
- lint/build/TypeScript baseline;
- live non-prod Round-2 proof and exact Change ID;
- committed default remains LEGACY_PRE_F3;
- migration count zero;
- no F3.1.4/F3.2 leakage.

Do not mark F3.1.3b CLOSED merely because implementation tests pass.

## 26. Next gate

After implementation PASS, next gate is an independent **F3.1.3b architecture/implementation acceptance review**.

Only after that review returns ACCEPT may F3.1.3 be considered fully implemented/closed and the next F3.1.x checkpoint be selected.

F3.1.4 and F3.2 remain NO-GO.

## 27. Required final report

Return:

```text
Docs baseline reviewed: <sha>
ADO parent/tip before implementation: <sha>
Source drift: NONE | ACCEPTED_ONLY | BLOCKED
F3.1.3b implementation: PASS | FAIL | BLOCKED_BY_SOURCE_DRIFT
Resubmission route: PASS | FAIL
Dedicated resubmit permission: PASS | FAIL
Requester authority: PASS | FAIL
Immutable-owner membership authority: PASS | FAIL
platform_admin standalone denied: PASS | FAIL
CAB standalone denied: PASS | FAIL
Same-change identity freeze: PASS | FAIL
Catalog ownership drift protection: PASS | FAIL
Execution-plan hidden retarget guard: PASS | FAIL
Round-N materialization: PASS | FAIL
Current policy/selector rebinding: PASS | FAIL
requirementId=requirementRole: PASS | FAIL
Resubmission idempotency: PASS | FAIL
Caller-owned transaction: PASS | FAIL
Provider replaceCurrent: PASS | FAIL
Index non-identity projection: PASS | FAIL
Participant rebuild: PASS | FAIL
R1 PostgreSQL concurrency: PASS | FAIL
R2-R13: PASS | FAIL
Failure injection: PASS | FAIL
F3.1.3a D1-D6 regressions: PASS | FAIL
Change Management tests: <result>
PostgreSQL tests: <result>
GMUD/Catalog/Deployments regressions: <result>
Lint: <result>
Build: <result>
TypeScript baseline: <result>
Live non-prod Round-2 proof: PASS | PARTIAL | NOT_AVAILABLE_EXPLAINED
Committed default remains LEGACY_PRE_F3: YES | NO
Migrations added: NO
F3.1.4/F3.2 behavior added: NO
Implementation commit: <sha | NOT_COMMITTED>
Remote ADO tip after publication: <sha | NOT_PUBLISHED>
Canonical evidence: docs/backstage/f3-1-3b-implementation-evidence.md
Next gate: F3.1.3b independent architecture/implementation acceptance review
```

## 28. STOP

STOP after F3.1.3b implementation/evidence.

Do not implement F3.1.4, F3.2, production cutover, or another delivery/change slice from inside this checkpoint.