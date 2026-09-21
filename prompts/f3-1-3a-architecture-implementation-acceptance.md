# F3.1.3a — Architecture / Implementation Acceptance Review

## Status

INDEPENDENT REVIEW ONLY — NO IMPLEMENTATION OR DATA REPAIR AUTHORIZED.

Purpose: independently decide whether the published F3.1.3a implementation at ADO commit `6bad066d945d49feaf642313ec37467e2658dc3f` faithfully implements the accepted server-authoritative decision-command contract.

Canonical authority:
- docs/backstage/f3-1-3-implementation-plan.md
- docs/backstage/f3-1-3-revised-plan-architecture-rereview.md
- docs/backstage/f3-1-3a-implementation-evidence.md
- prompts/f3-1-3a-decision-command-implementation.md
- docs/adr/ADR-009-change-authorization-model.md
- docs/adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md
- docs/adr/ADR-014-change-resubmission-authority-and-identity-boundary.md
- docs/backstage/f3-1-2b-architecture-implementation-acceptance.md
- docs/backstage/f3-1-2-ledger-required-nonprod-activation-acceptance.md
- docs/backstage/current-state.md

Accepted parent baseline:
`22495229502dabf2d99588599a156d862c5114fa`

Candidate implementation:
`6bad066d945d49feaf642313ec37467e2658dc3f`

## 1. Mandatory fresh baseline

Before reviewing:
1. Fetch latest `backstage-docs@main` and record exact SHA.
2. Read every authority file above, implementation-progress, prompts/README, and this prompt.
3. Independently verify live ADO `feat/ado-repo-governance` tip.
4. Inspect candidate `6bad066` directly and verify parent is exactly `2249522`.
5. Inspect the complete diff `2249522..6bad066`; do not rely on implementation evidence alone.
6. If the branch has advanced beyond `6bad066`, classify later drift separately. Review the exact candidate commit unless later drift invalidates runtime/product evidence.
7. Independently inspect the accepted laptop runtime/database if still available; do not mutate facts merely to make the review pass.

## 2. Review boundary

Do not:
- modify ADO source;
- repair implementation defects;
- create/delete/alter ApprovalDecision rows;
- alter laptop overlay/config to make a test pass;
- implement F3.1.3b;
- add `change.resubmit`; 
- implement F3.1.4/F3.2;
- alter production;
- mutate Delivery/Kargo/Argo/GitOps state.

Return exactly `ACCEPT` or `REJECT`.

## 3. Mandatory gates

G1 — Lineage / scope
- candidate is direct child of accepted `2249522`;
- diff is limited to F3.1.3a decision command, permissions/RBAC, trx-aware ledger reads, rejection lifecycle projection, eligibility safety, minimal frontend status/client support, and tests;
- no migration;
- no provider replacement;
- no Round-2/resubmission implementation;
- no approve/reject UI or CAB Workbench;
- committed runtime default remains `LEGACY_PRE_F3`.

G2 — HTTP command contract
Verify exact route:
`POST /changes/:changeId/rounds/:roundNumber/requirements/:requirementId/decisions`.

Verify:
- user credentials only;
- required trimmed `Idempotency-Key`;
- accepted body fields only;
- rejected outcome requires reason;
- `cabMeetingRef` restricted to CAB/authority requirements;
- client cannot supply actor/timestamp/hash/evidence authority fields;
- canonical command hash matches the accepted plan;
- first commit 201;
- exact replay 200;
- response is bounded decision/evaluation/lifecycle result, not an F3.1.4 read model.

G3 — Individual decision authority
Verify both permission and exact principal equality are enforced.

`platform_admin`, requester, owner, responsibleRef, or CAB membership must not override a mismatched individual principal.

Mismatch must fail with zero decision/audit.

G4 — CAB / authority decision proof
Verify:
- dedicated `...authorization.cab.record` permission;
- current live Catalog membership against the snapshotted authority Group;
- membership reads `relations.memberOf` + `spec.memberOf`, normalized/deduped and prefix-agnostic;
- does not use TP-filtered `ownershipEntityRefs` as authority;
- Group remains one collective decision, not N member decisions;
- `actingAuthorityRef` and `authorizationEvidence` are truthful/bounded;
- Catalog unavailable -> retryable provider failure with zero writes;
- actor absent -> fail closed;
- non-member -> fail closed.

G5 — Permission/RBAC separation
Verify:
- `...authorization.decide` and `...authorization.cab.record` are separate permissions;
- `platform_admin` does not automatically receive `cab.record`;
- configured CAB Group gets a dedicated CAB-recorder role;
- permission alone never bypasses individual/CAB domain proof;
- `change-management.change.resubmit` is absent.

G6 — Ledger read/write transaction contract
This is critical.

Verify one caller-owned transaction owns:
- parent `change_index` row lock (`FOR UPDATE` on PostgreSQL);
- authoritative current-round read;
- requirement read;
- existing-decision reads;
- decision insert;
- decision audit;
- in-transaction re-evaluation;
- milestone audit;
- rejection lifecycle projection when applicable.

Verify all ledger reads needed after an uncommitted write accept/use the same `trx`, including:
- `findRound`; 
- `findCurrentRound`; 
- `listRequirements`; 
- `listAuditEvents`; 
- decision lookup by requirement;
- decision lookup by actor/idempotency.

Reject if any correctness-critical post-insert read falls back to `this.knex` on another pooled PostgreSQL connection.

G7 — Decision idempotency / immutable terminal fact
Verify:
- one terminal decision per requirement/round;
- `(actorRef, idempotencyKey)` replay identity;
- exact replay returns original decision;
- same key changed payload conflicts;
- same actor/key reused on another requirement conflicts;
- different key after requirement already terminal conflicts;
- no UPDATE/DELETE/repair of immutable decision.

G8 — Concurrent unique-loser handling
Verify PostgreSQL unique violation handling:
- failed transaction is rolled back first;
- winner is re-observed outside the aborted transaction;
- duplicate approve can converge to replay;
- approve-vs-reject loser deterministically conflicts;
- no `ON CONFLICT UPDATE`; 
- no sleeps/polling/distributed lock/in-memory mutex/queue.

G9 — Exactly-once authorization milestones
Verify:
- every new decision emits exactly one `change.authorization.decision_recorded`; 
- first transition to all mandatory pre approvals emits exactly one `change.authorization.authorization_reached`; 
- first mandatory pre rejection emits exactly one `change.authorization.round_rejected`; 
- exact replay emits none of these again;
- no mutable AuthorizationEvaluation row exists.

Because there is no unique event-type DB constraint, verify every milestone writer is serialized by the same parent `change_index` lock and uses trx-aware audit existence reads.

G10 — Rejection lifecycle projection
Verify mandatory pre rejection atomically commits:
- rejecting ApprovalDecision;
- decision_recorded;
- round_rejected;
- `change_index.status='rejected'`.

Verify:
- canonical authority remains immutable decision/audit;
- list shows rejected from index projection;
- detail overlays **status only** from index onto provider detail;
- provider `record_json` is not rewritten;
- approval/AUTHORIZED never writes lifecycle `authorized`; 
- `ChangeStatus` frontend/backend supports `submitted | rejected`; 
- no executing/completed/cancelled behavior is invented.

G11 — Eligibility safety
Verify the implementation removes historical sandbox Round fabrication for ledger-governed missing-round cases.

For `LEDGER_REQUIRED` with no current Round:
- no Round is created by read;
- eligibility fails closed with `NO_LEDGER_ROUND`.

Also inspect LEGACY behavior explicitly: the implementation must not introduce an unreviewed semantic change to pre-existing `LEGACY_PRE_F3` eligibility handling merely as a side effect of deleting sandbox fabrication. Compare exact pre-change `2249522` behavior and candidate behavior. If changed, determine whether canonical architecture already authorized that behavior; otherwise REJECT.

When a current Round exists, eligibility uses current Round snapshot window.

G12 — Post-execution fail-closed
Verify post-execution requirements anchored on `execution_completion` cannot be decided before accepted completion evidence exists.

Expected:
`CONFLICT / execution_completion_required`, zero decision/audit.

No synthetic execution-completion fact, no lifecycle expansion.

G13 — PostgreSQL D1–D6 authoritative proof
Independently re-run on PostgreSQL 16 where feasible and inspect test construction.

Required:
- D1 exact replay;
- D2 concurrent duplicate approval;
- D3 concurrent approve vs reject;
- D4 two different requirements race to AUTHORIZED with exactly one milestone;
- D5 membership/Catalog fail-closed;
- D6 rejection atomicity/lifecycle.

Concurrency proof must use deterministic barriers/hooks/failure injection where races matter, not timing sleeps.

The implementation evidence records a PostgreSQL 16.14 re-verification. Confirm the actual candidate code and tests produce that result.

G14 — Rollback / failure injection
Verify at least:
- failure after decision insert before audit -> no partial durable decision;
- failure after decision audit before lifecycle projection -> no partial durable state;
- unique-loser aborted transaction does not continue querying before rollback.

G15 — Regression / quality
Independently verify/re-run enough to establish:
- full Change Management SQLite suite;
- F3.1.2 submission/idempotency/concurrency regressions;
- F3.1.1 policy/selector/publication regressions;
- participant list/detail;
- GMUD frontend;
- Catalog;
- Deployments tab;
- Delivery backend reads;
- architecture guards;
- lint;
- full build;
- repository-wide TypeScript baseline.

Historical five duplicate-Knex TypeScript errors are acceptable only if exact set/location is unchanged from parent. Zero new errors.

G16 — Product-convergence preservation
Verify F3.1.3a did not regress:
- `/gmud`;
- Catalog;
- Catalog Component Deployments tab;
- Delivery reads;
- `api:catalog/delivery` distinct extension identity;
- no new extension/config collision.

G17 — Live happy-path evidence
If the accepted non-production laptop evidence remains available, independently confirm durable facts for `CHG-2026-000003` or the exact recorded equivalent:
- Round 1 only;
- primary + CAB requirements;
- primary approval immutable;
- CAB approval by a real current CAB member with `cab.record`; 
- derived AUTHORIZED;
- lifecycle still submitted;
- exactly one authorization_reached;
- replay preserves same decision ids/timestamps;
- eligibility reflects real window/context rather than returning ALLOW merely because authorization is complete.

Do not create replacement facts merely to satisfy review.

G18 — Live rejection evidence
Independently confirm `CHG-2026-000005` or exact recorded disposable equivalent:
- one mandatory-pre rejection;
- one decision_recorded;
- one round_rejected;
- lifecycle/list/detail rejected;
- eligibility DENY/REJECTED;
- no authorization_reached;
- one Round only;
- resubmit/round creation routes still absent.

G19 — Hard non-goals
Confirm candidate adds none of:
- F3.1.3b resubmission/new-round behavior;
- `change.resubmit` permission;
- `DevelopmentProvider.replaceCurrent`; 
- F3.1.4 composed authorization/governance UI;
- approve/reject buttons/CAB Workbench;
- F3.2 autonomy;
- Teams decisions;
- execution lifecycle;
- migration;
- production cutover;
- Kargo/Argo/GitOps mutation.

## 4. Decision

Return exactly one:
`ACCEPT` or `REJECT`.

Do not use CONDITIONAL_ACCEPT.

ACCEPT only if the exact published candidate is a faithful implementation of the accepted F3.1.3a contract and every material gate passes.

## 5. If ACCEPT

Record exactly:

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

Do not author the F3.1.3b implementation prompt from inside this review.

## 6. If REJECT

Identify only concrete source defects or proof gaps.

Do not repair code inside the review.

Keep F3.1.3b prompt authoring/implementation NO-GO.

## 7. Canonical documentation

Create:
- `docs/backstage/f3-1-3a-architecture-implementation-acceptance.md`

Update factually:
- `docs/backstage/current-state.md`;
- `docs/backstage/implementation-progress.md`;
- `prompts/README.md`.

Preserve implementation evidence and all prior plan/review history.

Commit documentation only and STOP.

## 8. Required final report

Return:

```text
Docs baseline reviewed: <sha>
ADO candidate reviewed: 6bad066d945d49feaf642313ec37467e2658dc3f
Parent verified: YES | NO
Independent ADO source verification: YES | NO
Lineage/scope gate: PASS | FAIL
Decision transport gate: PASS | FAIL
Individual authority gate: PASS | FAIL
CAB membership gate: PASS | FAIL
Permission/RBAC separation gate: PASS | FAIL
Transaction/trx-aware-read gate: PASS | FAIL
Decision idempotency gate: PASS | FAIL
Concurrent loser gate: PASS | FAIL
Milestone-audit gate: PASS | FAIL
Rejection lifecycle gate: PASS | FAIL
Eligibility safety gate: PASS | FAIL
LEGACY eligibility regression check: PASS | FAIL
Post-execution fail-closed gate: PASS | FAIL
PostgreSQL D1-D6 gate: PASS | FAIL
Rollback/failure-injection gate: PASS | FAIL
Regression/quality gate: PASS | FAIL
Product-convergence preservation gate: PASS | FAIL
Live happy-path evidence gate: PASS | FAIL | PARTIAL_EXPLAINED
Live rejection evidence gate: PASS | FAIL | PARTIAL_EXPLAINED
Hard non-goals gate: PASS | FAIL
F3.1.3a architecture/implementation acceptance: ACCEPT | REJECT
ADO implementation modified by review: NO
F3.1.3b implementation performed: NO
Final docs SHA: <sha>
```

## 9. STOP

STOP after the review.

Do not modify ADO implementation, create/repair decision facts, author the F3.1.3b implementation prompt, or implement F3.1.3b/F3.1.4/F3.2.