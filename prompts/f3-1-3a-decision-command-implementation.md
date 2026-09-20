# F3.1.3a — Server-Authoritative Decision Command Implementation

## Status

IMPLEMENTATION PROMPT — DO NOT EXECUTE WITHOUT EXPLICIT USER LAUNCH.

Purpose: implement only **F3.1.3a**, the server-authoritative approval/rejection command, from the accepted F3.1.3 implementation contract.

This checkpoint may create real non-production `ApprovalDecision` facts only during the explicitly authorized implementation/product-proof execution. Prompt authoring itself creates none.

Canonical authority:
- docs/backstage/f3-1-3-implementation-plan.md
- docs/backstage/f3-1-3-revised-plan-architecture-rereview.md
- docs/backstage/f3-1-3-plan-architecture-review.md (historical REJECT; use only as evidence of already-closed concerns)
- docs/adr/ADR-009-change-authorization-model.md
- docs/adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md
- docs/adr/ADR-014-change-resubmission-authority-and-identity-boundary.md
- docs/backstage/f3-1-2b-architecture-implementation-acceptance.md
- docs/backstage/f3-1-2-ledger-required-nonprod-activation-acceptance.md
- docs/backstage/current-state.md

Accepted source/runtime baseline:
- ADO repo: `platform-devops-developer-portal`
- branch: `feat/ado-repo-governance`
- accepted source SHA: `22495229502dabf2d99588599a156d862c5114fa`
- accepted laptop demo target: non-production, local SQLite, effective gitignored overlay `LEDGER_REQUIRED`
- committed repository default: `LEGACY_PRE_F3`

F3.1.3 plan status:

```text
F3.1.3 plan: ACCEPTED IMPLEMENTATION CONTRACT
Migration required: NO
F3.1.3a implementation-prompt authoring: GO
F3.1.3a implementation: requires this explicit launch
F3.1.3b: NO-GO
F3.1.4: NO-GO
F3.2: NO-GO
Production cutover: NOT AUTHORIZED
```

## 1. Mandatory fresh baseline

Before editing:
1. Fetch latest `backstage-docs@main` and record exact SHA.
2. Read every authority file above, implementation-progress, prompts/README, and this prompt.
3. Fetch the actual ADO branch `feat/ado-repo-governance` and verify its live remote tip.
4. Expected starting source is `2249522`. Do not assume it if the branch advanced.
5. Inspect all drift after `2249522` before editing.
6. If later drift touches decision storage, ledger repository reads/writes, Change lifecycle/index composition, permission/RBAC wiring, Catalog membership, eligibility, or plugin routes in a way that contradicts the accepted plan, STOP with `BLOCKED_BY_SOURCE_DRIFT`.
7. Verify committed `app-config.yaml` still defaults `newSubmissionAuthorizationMode: LEGACY_PRE_F3`.
8. Verify the laptop demo target, if used for live proof, is still isolated non-production and its local overlay is intentionally `LEDGER_REQUIRED`.

Do not implement against a stale checkout.

## 2. Exact slice objective

Implement this flow and nothing beyond it:

```text
authenticated user
  -> POST one requirement decision
  -> server permission check
  -> server domain-authority proof
  -> append exactly one immutable ApprovalDecision
  -> append exactly one decision audit
  -> derive current AuthorizationEvaluation
  -> append at most one deterministic milestone audit
  -> on mandatory pre rejection, project Change.status = rejected
  -> return bounded decision result
```

F3.1.3a does **not** implement correction/resubmission or Round N+1.

## 3. Decision HTTP contract

Implement the accepted route exactly:

```text
POST /api/change-management/changes/:changeId/rounds/:roundNumber/requirements/:requirementId/decisions
```

User credentials only.

`Idempotency-Key` is required, trimmed, non-empty. Missing/blank -> existing `VALIDATION_ERROR` / 400.

Accepted body:

```ts
{
  outcome: 'approved' | 'rejected';
  reason?: string;
  comment?: string;
  cabMeetingRef?: string;
}
```

Rules:
- rejected requires non-empty trimmed `reason`;
- `cabMeetingRef` is accepted only for `cab` / `authority` requirements;
- reject any client-supplied actor/timestamp/decision/hash/evidence/provider/channel/correlation authority fields;
- no Teams/ADO/provider-specific canonical fields.

Canonical command hash:

```text
sha256Canonical({
  changeId,
  roundNumber,
  requirementId,
  outcome,
  reason: reason ?? null,
  comment: comment ?? null,
  cabMeetingRef: cabMeetingRef ?? null,
})
```

First commit -> HTTP 201.

Exact replay -> HTTP 200 with the original decision identity/timestamp/hash and no duplicate audit.

Bounded response:

```ts
{
  decision: ApprovalDecision;
  authorizationEvaluation: 'PENDING' | 'AUTHORIZED' | 'REJECTED';
  changeStatus: 'submitted' | 'rejected';
  roundNumber: number;
}
```

Do not create an F3.1.4 authorization read model.

## 4. Individual decision authority

For `requirement.kind === 'individual'`, require BOTH:
- server permission `change-management.change.authorization.decide`;
- exact authenticated actor equality:

```text
actor.userEntityRef == requirement.principalSnapshot.resolvedPrincipalRef
```

No requester/owner/platform-admin/governance override.

Mismatch -> `FORBIDDEN`, `details.reason=not_requirement_principal`, zero writes.

Persist:
- `actorRef` = authenticated user ref;
- omit `actingAuthorityRef`;
- authorization evidence only:

```ts
{
  kind: 'individual',
  matchedPrincipalRef: actor.userEntityRef,
  snapshotSelectorKey: requirement.principalSnapshot.selectorKey,
}
```

## 5. CAB / authority decision authority

For `requirement.kind === 'cab' | 'authority'`, require BOTH:
- server permission `change-management.change.authorization.cab.record`;
- live current Catalog membership of the authenticated user in the snapshotted authority Group.

One collective authority decision remains **one** ApprovalDecision. Never expand a CAB Group into member decisions.

Implement live membership in a dedicated helper such as:
`packages/backend/src/modules/changeManagement/authorization/decisionMembership.ts`

Membership contract:
1. load current User entity from Catalog using service credentials;
2. collect and normalize/dedupe `relations.memberOf` and `spec.memberOf` refs;
3. compare against `requirement.principalSnapshot.resolvedPrincipalRef`;
4. prefix-agnostic; do not use TP-prefix `ownershipEntityRefs` as authority;
5. do not place this membership logic under `authorization/selector/`;
6. do not change selector publication/resolution into group-member expansion.

Failure:
- Catalog/credentials unavailable -> `PROVIDER_UNAVAILABLE`, `membership_source_unavailable`, zero writes;
- User absent -> `FORBIDDEN`, `actor_not_in_catalog`, zero writes;
- not current member -> `FORBIDDEN`, `not_authority_member`, zero writes.

Persist:
- `actorRef` = authenticated user ref;
- `actingAuthorityRef` = snapshotted authority Group ref;
- bounded evidence:

```ts
{
  kind: 'authority_membership',
  authorityRef: actingAuthorityRef,
  actorRef,
  membershipSource: 'catalog',
  relation: 'memberOf',
  provenAt: decidedAt,
}
```

`cabMeetingRef`, when supplied, is supporting context only, never membership proof.

## 6. Permissions / RBAC

Add only the two F3.1.3a permissions:

```text
change-management.change.authorization.decide     action=update
change-management.change.authorization.cab.record action=update
```

Do **not** add `change-management.change.resubmit` in this slice. That belongs to F3.1.3b.

Minimum accepted RBAC shape:

```text
p, role:default/contributor, change-management.change.authorization.decide, update, allow
p, role:default/platform_admin, change-management.change.authorization.decide, update, allow

p, role:default/change_cab_recorder, change-management.change.authorization.cab.record, update, allow
g, <configured CAB group ref>, role:default/change_cab_recorder
```

Use the actual current CAB group config/source; do not hardcode a different governance authority merely to make tests pass.

Critical invariant:
`platform_admin` does **not** automatically receive `cab.record`.

Permission alone is never domain authority. Individual equality / live CAB membership still apply.

## 7. Decision persistence and trx-aware reads

Use existing append-only ledger tables and existing unique constraints. No migration.

Extend ledger read APIs minimally so all reads participating in the decision transaction can use the same caller-owned `trx`:
- `findRound(..., trx?)`;
- `findCurrentRound(..., trx?)`;
- `listRequirements(..., trx?)`;
- `listAuditEvents(..., trx?)`;
- `findDecisionByRequirement(..., trx?)`;
- `findDecisionByIdempotency(..., trx?)`.

Do not weaken existing append-only triggers or unique constraints.

PostgreSQL correctness depends on these reads using the caller-owned transaction; `this.knex` from another pooled connection cannot be used for the post-insert evaluation that must observe the uncommitted decision.

## 8. Exact decision transaction

Catalog membership I/O and permission/domain proof occur before opening the write transaction.

Then one caller-owned Knex transaction must:
1. lock parent `change_index` row (`FOR UPDATE` on PostgreSQL);
2. re-load authoritative current Round and target requirement through the same `trx`;
3. verify `LEDGER_REQUIRED`, current round, requirement existence, phase/terminal rules;
4. read existing decision by requirement and by `(actorRef, Idempotency-Key)` through the same `trx`;
5. apply exact replay/conflict rules;
6. append new `ApprovalDecision` on the same `trx`;
7. append `change.authorization.decision_recorded` on the same `trx`;
8. re-read current requirements through the same `trx` and derive authorization;
9. append at most one milestone event on the same `trx`;
10. if mandatory pre rejection, update the lifecycle projection to `rejected` on the same `trx`;
11. commit.

No nested transaction. No partial decision/audit/projection commit.

## 9. Decision idempotency / concurrency

Existing DB uniqueness remains authority:
- terminal decision unique by `(changeId, roundNumber, requirementId)`;
- replay unique by `(actorRef, idempotencyKey)`.

Rules:
- same actor + same key + same hash/requirement/outcome -> exact replay;
- same key + changed command -> `CONFLICT`, `idempotency_payload_mismatch`;
- different key + already-decided requirement -> `CONFLICT`, `conflicting_decision`;
- same actor reusing the same key on another requirement -> conflict;
- concurrent duplicate approval -> one insert winner, loser rollback then re-observe as replay;
- concurrent approve vs reject -> one terminal decision, loser deterministic `conflicting_decision`;
- post-commit network retry with same exact command -> replay.

If an insert loses a PostgreSQL uniqueness race:
1. rollback the failed transaction;
2. re-read committed winner state in a new transaction/context;
3. return replay or deterministic conflict from immutable winner facts.

Do not continue inside an aborted PostgreSQL transaction.
Do not use `ON CONFLICT UPDATE`.
Do not use polling, sleeps, distributed locks, queues, or in-memory mutexes.

## 10. Authorization transition audits

New decision -> exactly one:
`change.authorization.decision_recorded`

After in-transaction re-evaluation:
- first transition to all mandatory pre approvals -> exactly one `change.authorization.authorization_reached`;
- first mandatory pre rejection -> exactly one `change.authorization.round_rejected`;
- still pending -> no milestone.

Milestone identity is system-owned as accepted by the plan.

There is no new DB uniqueness for milestone event type. Exactly-once is guaranteed by:
- one `change_index FOR UPDATE` serialization point;
- trx-aware audit existence read;
- all F3.1.3a milestone writers using that same lock.

Do not persist `AuthorizationEvaluation` as mutable state.

## 11. Rejection lifecycle projection

Mandatory pre rejection must atomically produce:
- rejecting immutable `ApprovalDecision`;
- `decision_recorded` audit;
- `round_rejected` audit;
- `change_index.status = 'rejected'` projection.

Canonical authority remains the decision + audit. Index status is rebuildable projection.

Implement the minimum projection API, e.g. `projectLifecycleStatus`, on the index repository.

Expand backend and frontend `ChangeStatus` to:

```text
submitted | rejected
```

Add the existing UI label convention for rejected (`Rejeitada`).

`GET /changes` should naturally show the index projection.

`GET /changes/:changeId` must overlay **status only** from the index projection onto provider detail. Do not rewrite provider `record_json` in F3.1.3a and do not overlay unrelated index fields.

Approval / `AUTHORIZED` never writes lifecycle `authorized`; successful approval leaves `Change.status = submitted`.

## 12. Eligibility safety

Remove/disable the historical sandbox behavior that fabricates a Round 1 for a `LEDGER_REQUIRED` Change with no Round.

For ledger-governed Change with missing Round:
fail closed with the accepted `NO_LEDGER_ROUND` behavior.

Do not delete historical sandbox facts. Existing `CHG-2026-000001` already has its Round and must remain readable.

When a current Round exists, eligibility must use the current Round snapshot's requested window, not fabricate/rebind governance.

## 13. Post-execution requirements

Keep the decision command generic, but current source has no accepted execution-completion fact.

For `phase === post_execution` where the requirement needs the `execution_completion` anchor and no accepted completion evidence exists:

```text
CONFLICT
details.reason = execution_completion_required
zero decision
zero decision audit
```

The requirement row itself is not completion evidence.

Do not invent execution start/completion events or lifecycle in F3.1.3a.

## 14. Error contract

Use existing public codes. At minimum preserve:
- malformed body / missing key / invalid CAB field / rejection without reason -> `VALIDATION_ERROR` / 400;
- missing permission -> `FORBIDDEN` / 403;
- individual mismatch -> `FORBIDDEN` `not_requirement_principal`;
- CAB non-member -> `FORBIDDEN` `not_authority_member`;
- actor missing Catalog -> `FORBIDDEN` `actor_not_in_catalog`;
- Catalog unavailable -> `PROVIDER_UNAVAILABLE` `membership_source_unavailable`;
- Change / round / requirement missing -> existing `NOT_FOUND` semantics;
- exact replay -> 200 original result;
- command-hash mismatch -> `CONFLICT` `idempotency_payload_mismatch`;
- conflicting terminal decision -> `CONFLICT` `conflicting_decision`;
- LEGACY/no valid ledger command target -> `CONFLICT` `ledger_required_only`;
- stale/non-current round -> `CONFLICT` `stale_round`;
- terminal round -> `CONFLICT` `terminal_round`;
- post-execution without completion -> `CONFLICT` `execution_completion_required`;
- invariant mismatch/corruption -> `INTERNAL_ERROR` fail closed.

Do not add HTTP codes merely to distinguish internal cases; use stable `details.reason`.

## 15. Mandatory PostgreSQL proofs D1-D6

SQLite may provide functional coverage. PostgreSQL 16 is authoritative for concurrency.

Implement deterministic tests:
- **D1** exact decision replay: same actor/key/hash -> original decisionId, one decision, one decision audit;
- **D2** concurrent duplicate approval: one insert winner; loser exact replay;
- **D3** concurrent approve vs reject same requirement: exactly one terminal decision; loser deterministic `conflicting_decision`;
- **D4** two different requirements race toward AUTHORIZED: exactly one `authorization_reached`; derived evaluation AUTHORIZED; lifecycle still submitted;
- **D5** authority membership failure or Catalog unavailable: FORBIDDEN/PROVIDER_UNAVAILABLE; zero decision/audit;
- **D6** mandatory pre rejection: one decision + `round_rejected` + index status rejected atomically; eligibility REJECTED.

Use deterministic barriers/hooks/failure injection for concurrency where needed. Do not prove correctness with sleeps.

Also prove:
- transaction rollback after decision insert but before audit leaves zero partial state;
- rollback after decision audit but before lifecycle projection leaves zero partial state;
- replay emits no duplicate milestone;
- append-only UPDATE/DELETE protections remain green;
- no Round 2 is created by decision command.

## 16. Functional / regression test contract

Run at minimum:
- full Change Management SQLite suite;
- disposable PostgreSQL decision/concurrency suite D1-D6;
- F3.1.2 submission/idempotency/concurrency regressions;
- F3.1.1 policy/selector/publication regressions;
- participant list/detail/read tests;
- permission/RBAC tests for decide vs cab.record separation;
- membership normalization/fail-closed tests;
- eligibility missing-round negative test;
- rejection list/detail status tests;
- GMUD frontend regression including rejected label;
- Catalog regression;
- Deployments tab + Delivery read regression;
- architecture guards;
- backend/frontend lint;
- full accepted build;
- repository-wide TypeScript baseline comparison.

Current historical TypeScript debt is acceptable only if the exact pre-change target baseline is unchanged. Zero new errors.

No `--forceExit`. Do not hide failed tests.

## 17. Live non-production product proof

After code/tests pass, use only the already accepted isolated laptop `LEDGER_REQUIRED` product target if it is still available and safe.

First re-verify its runtime/database facts. Never fabricate a principal/CAB membership.

### Happy path

Preferred existing target is `CHG-2026-000003` **only if it still has zero decisions and the accepted Round 1 primary+CAB requirements**.

If those immutable facts have changed, do not overwrite/delete them; create a fresh disposable normal-low LEDGER_REQUIRED Change with equivalent governed selectors and use that instead.

Prove through the real HTTP/product path:
1. actual primary principal records `approved` on the primary requirement;
2. actual current CAB member with `cab.record` records collective `approved` on CAB requirement;
3. derived AuthorizationEvaluation becomes `AUTHORIZED`;
4. Change lifecycle remains `submitted`;
5. exactly one `authorization_reached` exists;
6. execution eligibility remains governed by the actual window/context (for the recorded 000003 window, expected outside-window DENY until its window; record the real current result rather than fabricate it).

Exact replay one command and prove no duplicate fact/audit.

### Rejection path

Create/use a separate disposable non-production LEDGER_REQUIRED Change. Do **not** reject the accepted happy-path Change.

Reject one mandatory pre requirement with a real non-empty reason and prove:
- one decision;
- one `round_rejected`;
- public lifecycle/list/detail show `rejected`;
- execution eligibility denies as rejected;
- no lifecycle `authorized` exists;
- no resubmission/new Round route exists.

Browser/product proof should confirm GMUD, Catalog, and Deployments remain reachable and no new extension/config collision appears.

Do not implement approve/reject buttons merely for the proof. API/product tooling is sufficient for F3.1.3a; F3.1.4 owns composed authorization UI.

## 18. Expected source surface

Use actual current paths after source verification. Expected F3.1.3a surface includes:
- `packages/backend/src/modules/changeManagement/types.ts`;
- authorization decision DTO/types if needed;
- `AuthorizationLedgerRepository.ts`;
- `KnexAuthorizationLedgerRepository.ts`;
- `ChangeManagementService.ts` or a narrowly dedicated decision command service;
- `ChangeIndexRepository.ts`;
- `KnexChangeIndexRepository.ts`;
- `changeIndexMapper.ts` if applicable;
- new `authorization/decisionMembership.ts`;
- `authorization/EligibilityService.ts`;
- `packages/backend/src/plugins/changeManagementPlugin.ts`;
- Change Management `permissions.ts`;
- `packages/backend/config/rbac/rbac-policy.csv`;
- frontend `ChangeStatus`/status labels only;
- focused unit/integration/PostgreSQL tests;
- architecture guard tests.

No migration files.

No resubmission route/service/provider replacement/index snapshot rewrite.

## 19. Forbidden scope

MUST NOT:
- implement F3.1.3b resubmission;
- add `change-management.change.resubmit` permission;
- create Round 2/new-round logic;
- add `DevelopmentProvider.replaceCurrent`;
- rewrite current provider snapshot;
- implement F3.1.4 authorization/governance panel;
- add approve/reject UI buttons or CAB Workbench;
- implement CAB autonomy/F3.2;
- add Teams approval;
- add decision reversal/abstention/expiry;
- add execution-start/completion lifecycle;
- add additive requirements;
- add migrations;
- change committed default to LEDGER_REQUIRED;
- perform production cutover;
- mutate Kargo/Argo/GitOps desired state;
- add workflow/outbox/distributed-lock frameworks.

## 20. Publication

If all implementation gates pass:
1. commit only F3.1.3a-scoped code/tests/config;
2. parent must be the verified live accepted branch tip unless later accepted drift was explicitly reconciled;
3. normal fast-forward push to `feat/ado-repo-governance`;
4. no force push/history rewrite;
5. verify remote ADO tip independently;
6. record exact parent and implementation SHA.

Do not change/delete the Delivery branch.

## 21. Canonical evidence

After a real implementation attempt create:
- `docs/backstage/f3-1-3a-implementation-evidence.md`

Update factually:
- `docs/backstage/current-state.md`;
- `docs/backstage/implementation-progress.md`;
- `prompts/README.md`.

Evidence must record:
- docs baseline;
- exact ADO parent/final SHA;
- complete changed-path inventory;
- permission/RBAC bindings;
- transaction boundary;
- trx-aware read changes;
- D1-D6 results;
- SQLite/full Change Management results;
- product-convergence regressions;
- lint/build/TypeScript baseline;
- live non-prod happy/rejection proof and exact Change IDs used;
- committed default still LEGACY_PRE_F3;
- no migration;
- no 3b/3.1.4/F3.2 leakage.

Do not mark F3.1.3a CLOSED solely because implementation PASSed.

## 22. Next gate

After implementation PASS, next gate is an independent **F3.1.3a architecture/implementation acceptance review**.

F3.1.3b implementation-prompt authoring remains NO-GO until that independent review returns ACCEPT.

## 23. Required final report

Return:

```text
Docs baseline reviewed: <sha>
ADO parent/tip before implementation: <sha>
Source drift: NONE | ACCEPTED_ONLY | BLOCKED
F3.1.3a implementation: PASS | FAIL | BLOCKED_BY_SOURCE_DRIFT
Decision route contract: PASS | FAIL
Individual authority: PASS | FAIL
CAB live-membership authority: PASS | FAIL
Permission/RBAC separation: PASS | FAIL
trx-aware ledger reads: PASS | FAIL
Caller-owned transaction: PASS | FAIL
Decision idempotency: PASS | FAIL
D1 exact replay: PASS | FAIL
D2 duplicate approval concurrency: PASS | FAIL
D3 approve-vs-reject concurrency: PASS | FAIL
D4 authorization milestone concurrency: PASS | FAIL
D5 membership fail-closed: PASS | FAIL
D6 rejection atomicity/lifecycle: PASS | FAIL
Post-execution fail-closed: PASS | FAIL
Eligibility missing-round safety: PASS | FAIL
Change Management tests: <result>
PostgreSQL tests: <result>
GMUD/Catalog/Deployments regressions: <result>
Lint: <result>
Build: <result>
TypeScript baseline: <result>
Live non-prod happy-path proof: PASS | PARTIAL | NOT_AVAILABLE_EXPLAINED
Live non-prod rejection proof: PASS | PARTIAL | NOT_AVAILABLE_EXPLAINED
Committed default remains LEGACY_PRE_F3: YES | NO
Migrations added: NO
F3.1.3b behavior added: NO
F3.1.4/F3.2 behavior added: NO
Implementation commit: <sha | NOT_COMMITTED>
Remote ADO tip after publication: <sha | NOT_PUBLISHED>
Canonical evidence: docs/backstage/f3-1-3a-implementation-evidence.md
Next gate: F3.1.3a independent architecture/implementation acceptance review
```

## 24. STOP

STOP after F3.1.3a implementation/evidence.

Do not author or implement F3.1.3b, F3.1.4, or F3.2 from inside this checkpoint.