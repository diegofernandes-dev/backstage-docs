# F3.1.2b — Ledger Submission Architecture / Implementation Acceptance Review

## Status

REVIEW ONLY — NO IMPLEMENTATION OR CUTOVER AUTHORIZED.

Purpose: independently decide whether the published F3.1.2b implementation at ADO commit 22495229502dabf2d99588599a156d862c5114fa satisfies the accepted F3.1.2 implementation contract on top of the converged GMUD + Deployments product baseline.

Canonical authority:
- docs/backstage/f3-1-2-implementation-plan.md
- docs/backstage/f3-1-2-final-architecture-rereview.md
- docs/backstage/f3-1-2b-implementation-evidence.md
- prompts/f3-1-2b-ledger-submission-implementation.md
- docs/adr/ADR-009-change-authorization-model.md
- docs/adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md
- docs/backstage/product-convergence-deployments-evidence.md

Accepted prerequisite baselines:
- F3.1.2a accepted at ccee1e1676a2763e68880e5383ce1e5e48742843;
- F3.1.1c accepted at 3b302ab5c9caab38f96491b389b7ea9fe0b66c2f;
- product convergence PASS at f48dc825ab5d1d16fafc3f70ef772d28613df1a0.

## 1. Mandatory fresh baseline

Before reviewing:
1. Fetch latest backstage-docs main and record exact SHA.
2. Read every canonical authority file above plus current-state, implementation-progress, and prompts/README.
3. Independently fetch ADO repo platform-devops-developer-portal, branch feat/ado-repo-governance.
4. Verify remote branch tip and inspect commit 22495229502dabf2d99588599a156d862c5114fa directly.
5. Verify parent is exactly f48dc825ab5d1d16fafc3f70ef772d28613df1a0.
6. Inspect complete diff f48dc82..2249522; do not rely only on implementation evidence.
7. If branch tip advanced beyond 2249522, classify later drift separately. Review this slice at the exact implementation commit unless later drift invalidates acceptance evidence.

## 2. Review boundary

This is an independent architecture/source acceptance review only.

Do not:
- modify ADO code;
- flip runtime default to LEDGER_REQUIRED;
- implement operational cutover;
- implement F3.1.3 decisions;
- implement F3.1.4 read/RBAC;
- implement F3.2 CAB autonomy;
- repair defects discovered during the review;
- change Delivery/Kargo/Argo/GitOps state.

Return exactly ACCEPT or REJECT.

## 3. Mandatory gates

G1 — Lineage and scope
- Reviewed commit is exact child of f48dc82.
- Diff is limited to F3.1.2b implementation/test/config surfaces.
- No migration.
- No frontend product redesign.
- No unrelated Delivery domain change.

G2 — Product-convergence preservation
Verify the implementation preserves:
- Catalog Component Deployments tab;
- Delivery backend/plugin runtime;
- GET /changes/:changeId/execution-eligibility route;
- api:catalog/delivery distinct extension identity;
- Delivery RBAC/config;
- GMUD create/list/detail behavior;
- Catalog behavior.

Reject if authorization wiring regressed the recently converged product line.

G3 — Committed cutover default remains safe
- app-config committed default is exactly LEGACY_PRE_F3.
- No production/runtime environment is silently switched to LEDGER_REQUIRED by this commit.
- LEDGER_REQUIRED is exercised only through controlled test/config wiring in this slice.

G4 — Stored-mode-wins orchestration
Verify exact behavior:
- current config mode is applied only to genuinely new reservations;
- existing reservations recover with authorizationMode omitted;
- stored reservation mode is authoritative forever for that logical submission;
- repository explicit requested-mode mismatch remains CONFLICT;
- payload mismatch remains CONFLICT before policy/Catalog;
- same key/different actor remains independent;
- concurrent first-insert race converges on DB winner's stored mode without parsing driver messages.

G5 — Authorization runtime binding
- startup-built AuthorizationRuntime is injected, not recreated ad hoc;
- active policy/bundle references are pinned once per uncommitted Round-1 attempt;
- policy evaluates exactly once per attempt;
- principals resolve in deterministic order;
- each selector resolves once;
- service does not bypass resolver abstraction with direct Catalog principal calls;
- replay after committed Round 1 does not re-evaluate policy or re-resolve principals.

G6 — CAB-safe policy semantics
- active policy is default-change-authorization@2026-09-19.1;
- normal.low materializes exactly primary + CAB;
- normal.medium/high remain primary + CAB;
- emergency semantics remain unchanged;
- no skipCab/CabAutonomyGrant/waiver/Workbench/autonomy RBAC path exists.

G7 — Requirement identity and publication guard
- requirementId equals requirementRole exactly;
- no hash fallback;
- duplicate requirementRole is rejected fail-closed at registration/publication/startup before submission can materialize a Round;
- historical policy identities remain immutable.

G8 — Emergency separation of duty
- selector/type validation precedes final platform transaction;
- same-person A/B returns CONFLICT with details.reason=separation_of_duty;
- failure leaves no committed Round/requirement/audit/finalize/idempotency-complete state.

G9 — Round 1 and immutable evidence
Verify Round 1 persists:
- roundNumber=1;
- canonical pending-index Change snapshot and sha256Canonical hash;
- exact policy identity/digest/provenance/input/input hash/matched-rule provenance;
- exact selector-bundle identity/digest/provenance;
- deterministic policy requirements;
- one shared server timestamp as contracted where applicable.

G10 — Mandatory authorization audit
Inside the same final platform transaction verify:
- change.authorization.round_created;
- change.authorization.policy_selected;
- change.authorization.selector_bundle_bound;
- one change.authorization.requirement_materialized per requirement.

No decision/rejection/execution/eligibility audit events are introduced by F3.1.2b.

G11 — Caller-owned transaction atomicity
DevelopmentProvider path must use one caller-owned Knex transaction containing:
- provider.createWithTransaction;
- ledger.createRound;
- every ledger.appendAuditEvent;
- index.finalize;
- idempotency.complete.

All ledger calls from this path receive the outer trx explicitly.

Verify failure injection proves no partial durable Round/audit/finalize/complete/provider state on rollback.

G12 — External provider boundary
Verify:
- external provider create remains outside platform transaction;
- create remains idempotent by canonical changeId;
- platform transaction contains Round/audit/finalize/complete;
- retry converges without XA/2PC/outbox framework;
- provider-specific identifiers do not enter canonical authorization artifacts.

If the current repository has no executable external provider fixture, evidence may be contract/test based, but must be explicit rather than silently marked PASS.

G13 — Visibility invariant
For LEDGER_REQUIRED:
- pending index is invisible before final transaction commit;
- no Round from the local attempt is durable before commit;
- after commit, Round/requirements/audit/finalized index/completed reservation appear coherently;
- finalized ledger Change without Round 1 is treated as invariant failure;
- non-transactional findRound is not used as authority to finalize.

G14 — Healthy concurrent Round-1 loser convergence
For same actor + same Idempotency-Key + same payload + same changeId + LEDGER_REQUIRED:
- one transaction wins;
- loser rolls back first;
- loser re-reads committed state outside rolled-back trx;
- coherent winner facts return the same logical submitted success;
- never Round 2;
- never CONFLICT merely for losing;
- never INTERNAL_ERROR merely for healthy loss;
- no loser mutation of winner's Round/requirements/audit.

G15 — Transient vs invariant classification
- transient lock/serialization with winner not yet observable uses bounded immediate re-read and existing retryable semantics;
- no polling/sleeps/distributed locks/in-memory mutex/worker queue/new durable state;
- true committed contradiction fails closed as INTERNAL_ERROR, not false success and not idempotency CONFLICT.

G16 — PostgreSQL authoritative concurrency proof
Independently inspect/re-run enough to verify:
- C1 identical concurrent create calls -> both same logical success and exactly one reservation/index/Round/requirement set/audit set/DevelopmentProvider record;
- C2 deterministic finalization race uses barriers/hooks/failure injection, not timing sleeps, and proves loser rollback before re-read;
- C3 same key/different payload -> one wins, other CONFLICT before authorization; no second Change/Round/provider;
- C4 controlled committed invariant corruption -> fail closed.

PostgreSQL is authoritative for C1/C2. SQLite alone is insufficient.

G17 — Regression and quality proof
Independently verify/re-run:
- full Change Management suite;
- F3.1.2a canonical snapshot/recovery;
- F3.1.1a/b/c policy/selector/publication;
- participant list/detail;
- Deployments frontend smoke/tab tests;
- Delivery backend tests;
- GMUD frontend tests;
- Catalog API-extension/tab-loader collision guard;
- lint;
- build;
- repository-wide TypeScript baseline.

Five historical Knex type errors are acceptable only if the exact pre-change baseline set is unchanged. Zero new errors.

G18 — Rollback correctness gate
Verify implementation evidence and source reasoning establish the old-binary hazard and record the exact mandatory query:

```sql
SELECT COUNT(*)
FROM change_idempotency
WHERE authorization_mode = 'LEDGER_REQUIRED'
  AND state = 'pending';
```

Required result before rollback to any pre-F3.1.2 binary: 0.

Nonzero means rollback is forbidden until drained by supported recovery.

G19 — Hard non-goals
Confirm no:
- F3.1.3 decision command;
- F3.1.4 authorization read/RBAC;
- F3.2 CAB autonomy;
- Teams approval;
- additive user requirements;
- lifecycle transition on submission;
- migration;
- production rollout/cutover;
- Kargo/Argo/GitOps mutation;
- generic workflow/outbox/distributed lock framework.

## 4. Decision

Return exactly one:
ACCEPT
or
REJECT

Do not use CONDITIONAL_ACCEPT.

ACCEPT only if all material gates pass and the exact published implementation is a faithful realization of the accepted F3.1.2b contract.

## 5. If ACCEPT

Record exactly:

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

Also state whether a separate **ledger cutover activation checkpoint** should precede F3.1.3 implementation. Evaluate this factually from the product state:
- if real/manual product validation of F3.1.3 would require live ledger-governed Changes, recommend authoring a narrow cutover/activation prompt before implementing F3.1.3;
- do not perform that cutover inside this review.

Recommended sequence after ACCEPT unless source evidence contradicts it:

```text
1. Author narrow LEDGER_REQUIRED activation/cutover checkpoint
2. Execute + independently verify activation on the intended non-production product environment
3. Plan/author F3.1.3 decision-command slice
```

This keeps the product demonstrable before adding decision APIs.

## 6. If REJECT

Identify only concrete implementation defects or proof gaps.
Do not repair code from inside the review.
Keep operational cutover, F3.1.3, F3.1.4, and F3.2 NO-GO.

## 7. Canonical documentation updates

Create:
- docs/backstage/f3-1-2b-architecture-implementation-acceptance.md

Update factually:
- docs/backstage/current-state.md
- docs/backstage/implementation-progress.md
- prompts/README.md

Preserve implementation evidence and historical F3.1.2 plan/review documents.

Commit documentation only and STOP.

## 8. Required final report

Return:

```text
Docs baseline reviewed: <sha>
ADO commit reviewed: 22495229502dabf2d99588599a156d862c5114fa
Parent verified: YES | NO
Independent ADO source verification: YES | NO
Scope/lineage gate: PASS | FAIL
Product-convergence preservation gate: PASS | FAIL
Committed LEGACY default gate: PASS | FAIL
Stored-mode-wins gate: PASS | FAIL
Authorization-runtime gate: PASS | FAIL
CAB-safe policy gate: PASS | FAIL
Requirement identity/uniqueness gate: PASS | FAIL
Emergency SoD gate: PASS | FAIL
Round/evidence gate: PASS | FAIL
Audit gate: PASS | FAIL
Caller-owned transaction gate: PASS | FAIL
External-provider boundary gate: PASS | FAIL | NOT_APPLICABLE_EXPLAINED
Visibility invariant gate: PASS | FAIL
Healthy concurrency gate: PASS | FAIL
PostgreSQL C1/C2 gate: PASS | FAIL
C3/C4 negative gates: PASS | FAIL
Regression/quality gate: PASS | FAIL
Rollback gate: PASS | FAIL
Hard non-goals gate: PASS | FAIL
F3.1.2b architecture/implementation acceptance: ACCEPT | REJECT
ADO implementation modified by review: NO
Operational LEDGER_REQUIRED cutover performed: NO
Recommended next gate: <cutover activation | F3.1.3 planning | blocker>
Final docs SHA: <sha>
```

## 9. STOP

STOP after the review.

Do not:
- modify ADO implementation;
- flip LEDGER_REQUIRED;
- implement F3.1.3/F3.1.4/F3.2;
- alter Delivery/Kargo/Argo/GitOps state.