# F3.1.2b — Ledger-Governed Submission Integration Implementation

## Status

IMPLEMENTATION PROMPT — DO NOT EXECUTE WITHOUT EXPLICIT USER LAUNCH.

Purpose: implement only F3.1.2b from the already accepted F3.1.2 implementation contract, on top of the converged active product branch.

Architecture authority:
- docs/backstage/f3-1-2-implementation-plan.md
- docs/backstage/f3-1-2-final-architecture-rereview.md
- docs/backstage/f3-1-2a-architecture-implementation-acceptance.md
- docs/backstage/f3-1-1c-architecture-implementation-acceptance.md
- docs/adr/ADR-009-change-authorization-model.md
- docs/adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md
- docs/backstage/product-convergence-deployments-evidence.md

Accepted prerequisites:
- F3.1.2a CLOSED / ACCEPTED at ADO ccee1e1676a2763e68880e5383ce1e5e48742843;
- F3.1.1c CLOSED / ACCEPTED at ADO 3b302ab5c9caab38f96491b389b7ea9fe0b66c2f;
- product convergence PASS at ADO f48dc825ab5d1d16fafc3f70ef772d28613df1a0.

F3.1.2b implementation is a new, explicit checkpoint. Do not infer authorization from older prompts.

## 1. Mandatory fresh baseline

Before editing:
1. Fetch latest backstage-docs main and record exact SHA.
2. Read all authority files above, current-state, implementation-progress, prompts/README, and this prompt.
3. Fetch the actual Azure DevOps repository platform-devops-developer-portal, branch feat/ado-repo-governance.
4. Verify the live branch tip.
5. Expected starting product tip is f48dc825ab5d1d16fafc3f70ef772d28613df1a0. Do not assume it if the live remote differs.
6. Inspect the complete drift from the last accepted F3.1.1c tip 3b302ab..current-tip before editing.
7. The known accepted post-F3.1.1c drift is the Deployments product-convergence commit f48dc82. It intentionally touched:
   - changeManagementPlugin.ts to add the existing execution-eligibility read route while preserving authorization bootstrap;
   - app-config/RBAC/package surfaces for Delivery;
   - Delivery frontend/backend/runtime;
   - additive change-management exports;
   and explicitly did NOT change ChangeManagementService ledger submission behavior or enable LEDGER_REQUIRED.
8. Reconcile that known drift rather than treating it as an automatic blocker.
9. If later drift touches ChangeManagementService create flow, idempotency semantics, ledger transaction APIs, authorization policy/selector runtime, provider finalization, or the exact config contract in a way that contradicts the accepted plan, STOP with BLOCKED_BY_SOURCE_DRIFT and document the conflict.

Do not implement against a stale local checkout.

## 2. Objective

Implement the smallest correct composition such that genuinely new submissions can be assigned one immutable authorization regime and ledger-governed submissions create AuthorizationRound 1 atomically with platform finalization.

Target behavior:

```text
POST /changes
  -> parse + payload hash
  -> recover existing reservation OR create new reservation with current default mode
  -> existing reservation: stored authorization_mode always wins
  -> LEGACY_PRE_F3:
       exact current F2/F3.1.2a finalize behavior
       no policy/selector/Round/audit
  -> LEDGER_REQUIRED:
       reuse canonical pending Change snapshot
       bind active immutable authorization runtime
       evaluate CAB-safe policy exactly once
       resolve principals deterministically
       fail closed on selector/type/SoD errors
       materialize Round 1 + policy requirements + submission audit
       finalize platform state in one caller-owned transaction
       return { changeId, status: submitted }
```

Lifecycle remains submitted. F3.1.2b does not authorize the Change; it materializes authorization requirements only.

## 3. Cutover / reservation-mode contract

Introduce/finish the existing config pin:

```yaml
changeManagement:
  authorization:
    newSubmissionAuthorizationMode: LEGACY_PRE_F3 | LEDGER_REQUIRED
```

Hard rules:
- exact enum validation at startup;
- config applies only to first creation of a genuinely new idempotency reservation;
- existing reservations never re-request the current config mode;
- stored reservation authorization_mode is immutable and authoritative;
- KnexIdempotencyRepository.reserve explicit requested-mode mismatch remains CONFLICT;
- same key + different payload remains CONFLICT before policy/Catalog work;
- same key under different actor remains independent;
- authorization regime is never inferred from app version, deploy time, schema version, branch, or table presence.

Normative orchestration:

```text
desiredMode = config.newSubmissionAuthorizationMode
existing = idempotency.find(lookup)  # use existing equivalent or add smallest read-only lookup

if existing:
  reserved = reserve(lookup + payloadHash, authorizationMode omitted)
else:
  try:
    reserved = reserve(lookup + payloadHash + desiredMode)
  catch CONFLICT:
    reserved = reserve(lookup + payloadHash, authorizationMode omitted)

branch ONLY on reserved.authorizationMode
```

The catch/recovery path must not parse driver error strings to distinguish the healthy first-insert race from payload conflict.

## 4. Deployment safety default

The F3.1.2b-capable binary must land with the committed default still:

```text
newSubmissionAuthorizationMode: LEGACY_PRE_F3
```

Do NOT flip the normal committed runtime default to LEDGER_REQUIRED inside this implementation checkpoint.

Tests must exercise LEDGER_REQUIRED through controlled config/test wiring.

Actual operational cutover to LEDGER_REQUIRED is a later explicit activation step after F3.1.2b implementation is independently accepted.

## 5. Authorization runtime binding

Use the existing startup-built AuthorizationRuntime from F3.1.1b/F3.1.1c.

Extend ChangeManagementServiceOptions minimally with:
- authorizationRuntime: AuthorizationRuntime;
- authorizationLedger: AuthorizationLedgerRepository;
- newSubmissionAuthorizationMode: AuthorizationMode.

Plugin wiring must:
- preserve the product-convergence execution-eligibility route;
- preserve Delivery registration and api:catalog/delivery collision fix;
- pass the startup AuthorizationRuntime into ChangeManagementService instead of discarding it;
- construct/use KnexAuthorizationLedgerRepository with the existing database;
- pass the validated new-submission mode config;
- preserve active policy default-change-authorization@2026-09-19.1;
- preserve the existing active selector bundle and cab-authority selector;
- not create a mutable global singleton;
- not add direct Catalog principal lookup logic into ChangeManagementService.

Within a ledger submission attempt that has no committed Round 1:
1. pin runtime references once;
2. evaluate active policy exactly once using only { classification, risk };
3. resolve requirements in deterministic order: requirementRole ascending, then selectorKey;
4. resolve each selector exactly once;
5. validate required principal type;
6. validate separation of duty before starting the final platform transaction;
7. build Round 1, requirements, and required audit events in memory.

After a committed Round 1 exists, replay must not re-evaluate policy or re-resolve principals.

## 6. CAB-safe policy contract

F3.1.2b must consume the already accepted policy default-change-authorization@2026-09-19.1.

Required matrix:
- normal.low -> normal-primary approval + CAB approval;
- normal.medium -> primary + CAB;
- normal.high -> primary + CAB;
- emergency -> existing A/B pre-execution + post-execution CAB retrospective, unchanged.

F3.1.2b MUST NOT implement:
- skipCab;
- CabAutonomyGrant;
- waiver/exception engine;
- autonomy RBAC;
- CAB Workbench;
- dynamic low-risk delegation.

Low-risk autonomy remains F3.2.

## 7. Requirement identity and policy validation

Canonical requirement identity is exactly:

```text
requirementId = requirementRole
```

No hash fallback. No alternate scheme.

Before a policy can be usable, each rule's effective requirements must have unique requirementRole values.

Implement fail-closed uniqueness validation at the accepted policy registration/publication/startup boundary.

A duplicate requirementRole must prevent that policy from being registered/activated; submission must never discover the collision while persisting Round 1.

Preserve historical policy publication integrity and do not rewrite existing policy identities.

## 8. Emergency separation of duty

Before any Round commit:
1. policy evaluation succeeds;
2. every selector resolves;
3. principal type matches requiredPrincipalType;
4. for each non-empty separationOfDutyKey group, all user principal refs must be distinct.

Same-person emergency A/B:
- return CONFLICT;
- details.reason = separation_of_duty;
- no Round;
- no requirement;
- no authorization audit;
- no finalized index;
- no idempotency completion from the failed attempt.

The pending canonical index may remain invisible and recoverable.

## 9. Round 1 and requirement mapping

Round 1:
- roundNumber = 1;
- changeSnapshot = canonical durable pending-index snapshot;
- changeSnapshotSha256 = existing sha256Canonical(change);
- one createdAt server timestamp shared by Round and requirements;
- persist active policy identity/digest/provenance/input/input hash/matched rule provenance;
- persist selector bundle key/version/contentDigest/provenance;
- policyInput is exactly { classification, risk };
- requirements are deterministic policy-sourced requirements only.

ApprovalRequirement mapping must follow the accepted plan exactly, including:
- requirementId = requirementRole;
- source = policy;
- sourceRef = requirementRole;
- sourceProvenance = Round policy provenance;
- principalSnapshot = resolver output;
- separationOfDutyKey if present;
- SLA fields from the definition/policy identity where applicable;
- no decision field;
- no addedByActorRef/additionReason.

Additive user-supplied requirements remain deferred.

## 10. Mandatory submission audit

Append inside the same final platform transaction:
- change.authorization.round_created;
- change.authorization.policy_selected;
- change.authorization.selector_bundle_bound;
- one change.authorization.requirement_materialized per materialized requirement.

Audit payloads must use the accepted minimal canonical identities.

Do not add decision/rejection/execution/eligibility events in this slice.

## 11. Caller-owned transaction — mandatory

For DevelopmentProvider, one caller-owned Knex transaction MUST contain:

```text
provider.createWithTransaction(trx, canonicalChange)
ledger.createRound(round1, trx)
ledger.appendAuditEvent(event, trx) x N
index.finalize(..., trx)
idempotency.complete(..., trx)
COMMIT
```

Passing undefined/no outer trx to createRound or appendAuditEvent from the LEDGER finalization path is an implementation defect.

Do not redesign KnexAuthorizationLedgerRepository transaction support; it already exists unless fresh source proves otherwise.

For an external provider:
1. provider.create(canonicalChange) remains outside the platform transaction;
2. create must remain idempotent by canonical changeId;
3. then one caller-owned platform transaction commits Round + requirements + audit + index.finalize + idempotency.complete;
4. external provider orphan/retry converges by same changeId;
5. no XA/2PC/outbox framework.

## 12. Visibility and invariant contract

For LEDGER_REQUIRED, there is no valid normal state where the Change is finalized/discoverable without committed Round 1.

Before final transaction commit:
- pending index remains invisible;
- no Round from the attempt is durable.

After commit:
- Round 1 + requirements + required audit + finalized index + completed idempotency are durable together;
- DevelopmentProvider operational record is also durable in that transaction.

A committed Round 1 with pending/unfinalized platform state is an invariant breach, not a recovery milestone.

Do not use non-transactional findRound() as authority to permit finalization.

## 13. Healthy concurrent Round-1 loser — exact contract

For same actor + same Idempotency-Key + same payload + same changeId + LEDGER_REQUIRED:

Both workers may evaluate/resolve before finalization.

If one worker loses on the expected Round-1 uniqueness / serialization / locking boundary:
1. rollback the losing local platform transaction;
2. re-read committed state OUTSIDE the rolled-back transaction;
3. validate all of:
   - reservation exists;
   - reservation.changeId equals expected changeId;
   - reservation.authorizationMode = LEDGER_REQUIRED;
   - reservation.state = completed;
   - finalized index exists for same changeId;
   - index authorization mode = LEDGER_REQUIRED;
   - Round 1 exists for same changeId / roundNumber 1;
   - accepted immutable routing/snapshot invariants are coherent;
4. coherent winner facts -> return the same logical success { changeId, status: submitted };
5. never create Round 2;
6. never mutate winner Round/requirements/audit;
7. never return CONFLICT merely because this worker lost the healthy same-payload race;
8. never return INTERNAL_ERROR merely because this worker lost the healthy race.

Transient winner not yet observable:
- rollback;
- bounded immediate coherence re-read only;
- if coherent -> success;
- otherwise existing retryable storage / PROVIDER_UNAVAILABLE semantics;
- no sleeps/polling loops/distributed locks/in-memory mutexes/queues/background workers/new durable state.

True committed contradiction -> INTERNAL_ERROR fail-closed, not CONFLICT.

PostgreSQL is the authoritative concurrency proof dialect.

## 14. Error contract

Preserve existing public codes.

Required mappings:
- body/window/plan invalid -> VALIDATION_ERROR / 400;
- payload mismatch -> CONFLICT / 409;
- principal missing -> NOT_FOUND / 404;
- Catalog unavailable -> PROVIDER_UNAVAILABLE / 503;
- selector/config/principal-kind invariant -> INTERNAL_ERROR / 500;
- SoD same-person -> CONFLICT / 409 + details.reason=separation_of_duty;
- unsupported policy/evaluation invariant -> INTERNAL_ERROR / 500;
- provider/orphan retryable -> PROVIDER_UNAVAILABLE / 503;
- reservation/index mode disagreement -> INTERNAL_ERROR / 500;
- healthy Round loser with coherent winner -> normal 201 success;
- transient lock/serialization without visible winner -> existing retryable 503 semantics;
- committed invariant contradiction -> INTERNAL_ERROR / 500.

Do not add a new workflow status or leak provider payloads/policy ASTs.

## 15. Known product-convergence drift that must remain intact

Current target includes accepted Deployments convergence at f48dc82.

F3.1.2b MUST preserve:
- Catalog Component Deployments tab;
- Delivery backend plugin/module;
- execution-eligibility GET route already added to changeManagementPlugin;
- Delivery RBAC/config;
- api:catalog/delivery distinct extension identity;
- additive change-management type/label exports;
- Catalog and GMUD frontend behavior.

Do NOT:
- modify Delivery domain or routes as part of authorization wiring;
- remove the execution-eligibility route;
- merge feat/delivery-mvp-slice;
- import Delivery production/sandbox credentials or GitOps resources;
- add Delivery/Kargo/Argo identities to Change/Round/requirements.

Run product-convergence regressions after F3.1.2b.

## 16. Expected source surface

After fresh source verification, expected F3.1.2b changes may include:

```text
packages/backend/src/modules/changeManagement/ChangeManagementService.ts
packages/backend/src/plugins/changeManagementPlugin.ts

packages/backend/src/modules/changeManagement/persistence/IdempotencyRepository.ts
packages/backend/src/modules/changeManagement/persistence/KnexIdempotencyRepository.ts
packages/backend/src/modules/changeManagement/persistence/idempotencyAuthorizationMode.test.ts

packages/backend/src/modules/changeManagement/authorization/selector/config.ts
packages/backend/src/modules/changeManagement/authorization/selector/types.ts

packages/backend/src/modules/changeManagement/authorization/policy/registry.ts
packages/backend/src/modules/changeManagement/authorization/policy/registry.test.ts

packages/backend/src/modules/changeManagement/architecture.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.ledgerSubmit*.test.ts

app-config.yaml
```

Use actual current paths if repository layout has evolved.

A KnexIdempotencyRepository production change is justified only for the smallest read-only lookup/equivalent race-safe support needed by service orchestration. Do not weaken reserve().

No migration is expected or authorized. If source reality appears to require one, STOP with BLOCKED_BY_SCHEMA_CONTRADICTION rather than improvising.

Frontend changes are out of scope except if required solely to repair a regression caused by this slice; such a need should normally fail the checkpoint for review rather than expand scope.

## 17. Mandatory test contract

### Idempotency / cutover
- I1 explicit repository requested-mode mismatch remains CONFLICT;
- I2 existing LEGACY + config LEDGER resumes legacy with zero policy work;
- I3 existing LEDGER + config LEGACY resumes ledger;
- I4 new reservation stores configured mode;
- I5 same-key/same-payload replay same result;
- I6 same-key/different payload CONFLICT before policy/Catalog;
- I7 concurrent first inserts with different desired defaults converge on DB winner stored mode;
- I8 same key different actor independent.

### Policy / identity
- Q1 duplicate requirementRole fails before use;
- Q2 requirementId === requirementRole;
- Q3 no hash fallback;
- Q4 exact policy/bundle identities persisted;
- Q5 normal.low materializes exactly primary + CAB;
- Q6 no autonomy/bypass path.

### Principal resolution / SoD
- S1 missing principal fail closed;
- S2 Catalog unavailable retryable/no Round;
- S3 principal type mismatch fail closed;
- S4 emergency A/B same user -> CONFLICT separation_of_duty/no Round;
- S5 distinct users -> deterministic requirements.

### Transaction proof — SQLite + disposable PostgreSQL
- T1 Dev provider + Round + requirements + audit + finalize + complete share one outer trx;
- T2 throw after createRound before commit -> no partial durable state;
- T3 throw after audit before finalize -> full rollback;
- T4 successful commit exposes all durable state together;
- T5 transaction spies prove non-undefined outer trx passed to createRound and every appendAuditEvent;
- T6 Round 1 uniqueness remains enforced;
- T7 immutability triggers remain green.

### Concurrent Round-1 convergence
- C1 authoritative PostgreSQL: two identical concurrent createChange calls -> both same success; exactly one reservation/index/Round 1/effective requirement set/canonical audit set/DevelopmentProvider record; no Round 2/duplicates;
- C2 authoritative PostgreSQL deterministic finalization race using barriers/hooks/failure injection, not timing sleeps; prove loser rollback occurs before outside-trx coherence re-read and leaves no partial state;
- C3 same key/different payload concurrent -> one wins, other CONFLICT before authorization; no second Change/Round/provider;
- C4 controlled committed invariant corruption -> INTERNAL_ERROR fail closed, never false success.

### External provider
- M1 failure before side effect -> pending/invisible;
- M2 provider success + platform transaction failure -> orphan/retry convergence;
- M3 retry no duplicate provider record;
- M4 no provider-specific authorization fields enter Round/requirements.

### Rollback safety
- B1 prove a pre-F3.1.2 binary could legacy-finalize a pending LEDGER reservation;
- B2 implementation evidence includes mandatory rollback query and zero-count rule;
- B3 config LEDGER -> LEGACY on F3.1.2-capable binary leaves existing LEDGER reservations on ledger path.

### Regressions
- full Change Management module;
- F3.1.2a canonical snapshot proofs;
- F3.1.1a/b/c policy + selector + publication suites;
- participant list/detail;
- architecture guards;
- Deployments frontend smoke/tab tests;
- Delivery backend tests needed to prove convergence remains intact;
- GMUD frontend tests;
- Catalog tab-loader/API-extension collision guard;
- lint;
- build;
- repository-wide TypeScript baseline comparison.

Do not use sleeps as concurrency correctness proof. Do not use --forceExit.

## 18. Rollback correctness gate

Implementation evidence MUST record this exact operational gate before rollback to any binary predating F3.1.2 ledger submission support:

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

If nonzero, rollback to the older binary is FORBIDDEN until those logical submissions are completed/drained through supported F3.1.2 recovery.

This is mandatory runbook correctness evidence, not optional hardening.

## 19. Validation

At minimum run:
- focused F3.1.2b suites;
- full Change Management SQLite suite;
- disposable PostgreSQL suite including authoritative concurrency C1/C2;
- F3.1.1 publication/selector regressions;
- F3.1.2a regressions;
- product-convergence Deployments/Catalog/GMUD regression suites;
- backend lint;
- frontend/app lint where relevant;
- build:all or the repository's accepted full build;
- repository-wide TypeScript baseline comparison.

The current accepted baseline still has five historical duplicate-Knex TypeScript errors. Compare exact locations/set against the real pre-change target tip; introduce zero new errors.

If convergence changed that baseline, use the actual f48dc82/current-tip pre-change error set as the authoritative comparison and document it.

## 20. Publication

If all gates pass:
1. commit only F3.1.2b-scoped changes;
2. normal fast-forward push to feat/ado-repo-governance;
3. no force push/history rewrite;
4. record exact parent SHA and final implementation SHA;
5. verify remote ADO tip independently.

Do not change or delete feat/delivery-mvp-slice.

## 21. Canonical evidence after real implementation

Create:
- docs/backstage/f3-1-2b-implementation-evidence.md

Update factually:
- docs/backstage/current-state.md
- docs/backstage/implementation-progress.md
- prompts/README.md

Do not mark F3.1.2b CLOSED solely because implementation tests pass.

Next gate after implementation is an independent F3.1.2b architecture/implementation acceptance review.

Operational LEDGER_REQUIRED cutover remains separate unless that later acceptance/cutover is explicitly authorized.

## 22. Hard non-goals

MUST NOT:
- implement F3.1.3 decision commands;
- implement F3.1.4 read/RBAC;
- implement F3.2 CAB autonomy;
- add CAB Workbench;
- add Teams approval;
- add additive user-supplied requirements;
- redesign Delivery;
- modify Kargo/Argo/GitOps desired state;
- perform production rollout;
- add workflow/outbox/distributed-lock frameworks;
- add migrations;
- change Change lifecycle on submission;
- expose new frontend authorization authority.

## 23. Required final report

Return:

```text
Docs baseline reviewed: <sha>
ADO parent/tip before implementation: <sha>
Known product-convergence drift reconciled: YES | NO
Source drift: NONE | EXPECTED_CONVERGENCE_ONLY | BLOCKED
F3.1.2b implementation: PASS | FAIL | BLOCKED_BY_SOURCE_DRIFT | BLOCKED_BY_SCHEMA_CONTRADICTION
Committed default newSubmissionAuthorizationMode: LEGACY_PRE_F3
Stored-mode-wins orchestration: PASS | FAIL
Repository explicit mode mismatch remains CONFLICT: PASS | FAIL
CAB-safe normal.low primary + CAB: PASS | FAIL
requirementId=requirementRole: PASS | FAIL
Duplicate requirementRole startup/publication guard: PASS | FAIL
Emergency SoD fail-closed: PASS | FAIL
Caller-owned transaction gate: PASS | FAIL
DevelopmentProvider atomic path: PASS | FAIL
Healthy concurrent loser convergence: PASS | FAIL
PostgreSQL C1/C2 authoritative concurrency proof: PASS | FAIL
Different-payload C3: PASS | FAIL
Invariant-corruption C4: PASS | FAIL
External provider convergence: PASS | FAIL | NOT_APPLICABLE_EXPLAINED
Rollback zero-pending rule documented: PASS | FAIL
F3.1.2a regressions: <result>
F3.1.1 policy/selector regressions: <result>
Change Management tests: <result>
Deployments/Catalog/GMUD convergence regressions: <result>
SQLite: <result>
PostgreSQL: <result>
Lint: <result>
Build: <result>
TypeScript baseline: <result>
Migrations added: NO
F3.1.3/F3.1.4/F3.2 behavior added: NO
Implementation commit: <sha | NOT_COMMITTED>
Remote ADO tip after publication: <sha | NOT_PUBLISHED>
Canonical evidence: docs/backstage/f3-1-2b-implementation-evidence.md
Next gate: F3.1.2b independent architecture/implementation acceptance review
```

## 24. STOP

STOP after F3.1.2b implementation/evidence.

Do not:
- flip the committed default to LEDGER_REQUIRED;
- implement the operational cutover;
- implement F3.1.3 decisions;
- implement F3.1.4 RBAC/read model;
- implement CAB autonomy;
- continue into another slice.