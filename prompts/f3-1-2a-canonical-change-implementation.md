# F3.1.2a — Canonical Change Construction Implementation

## Status

IMPLEMENTED / PUBLISHED at ADO `ccee1e1676a2763e68880e5383ce1e5e48742843` after
explicit user launch. Canonical evidence:
[`docs/backstage/f3-1-2a-implementation-evidence.md`](../docs/backstage/f3-1-2a-implementation-evidence.md).
**Next gate:** independent F3.1.2a architecture/implementation acceptance review.
Do not re-execute this prompt without a new authorization.

Architecture authority:
- docs/backstage/f3-1-2-implementation-plan.md
- docs/backstage/f3-1-2-final-architecture-rereview.md
- ADR-006, ADR-007, ADR-008, ADR-009, ADR-012, ADR-013

Final review status: F3.1.2 plan ACCEPTED IMPLEMENTATION CONTRACT. This prompt was
executed under explicit user launch; F3.1.1c and F3.1.2b remain unauthorized by it.

## Objective

Implement only F3.1.2a: one logical create must produce one canonical Change snapshot, and recovery must reuse the durable pending snapshot instead of rebuilding server-generated fields.

At accepted ADO baseline 188d8e9, ChangeManagementService still calls buildChange() twice. Server-generated createdAt/activityId values can therefore diverge between provider and index snapshots.

## Mandatory fresh baseline

1. Fetch latest backstage-docs main and record the exact SHA.
2. Read the canonical F3.1.2 plan, final architecture re-review, current-state, implementation-progress, and this prompt.
3. Fetch the actual ADO repo platform-devops-developer-portal, branch feat/ado-repo-governance.
4. Record the exact tip. Expected accepted baseline is 188d8e9cc43423f3644b3cacfb9849257838a583.
5. Inspect ChangeManagementService and focused create/recovery tests before editing.
6. If post-baseline drift touches canonical Change construction, pending-index recovery, provider finalize, or idempotency recovery, stop with BLOCKED_BY_SOURCE_DRIFT.

## Required production behavior

New logical submission:
parse/reserve/validate/claim changeId -> build canonical Change exactly once -> persist pending index snapshot -> pass that same snapshot into provider/finalize -> complete existing F2 flow.

Recovery with an existing pending index:
read the durable pending Change snapshot -> reuse it -> do not call buildChange() -> do not regenerate activityId or createdAt -> finalize using that recovered snapshot.

## Expected source surface

- packages/backend/src/modules/changeManagement/ChangeManagementService.ts
- existing focused ChangeManagementService create/recovery/integration tests

Small local refactoring is allowed only to make the single-build/recovery invariant explicit. Do not create a new framework.

## Hard non-goals

Do not:
- wire AuthorizationRuntime
- instantiate or call AuthorizationLedgerRepository
- create AuthorizationRound or ApprovalRequirement
- change authorization_mode behavior
- add newSubmissionAuthorizationMode
- change idempotency repository semantics
- change policy/selector files
- publish F3.1.1c
- add CAB behavior or autonomy
- change RBAC/routes/frontend/Delivery
- add migrations
- start F3.1.2b, F3.1.3, F3.1.4, or F3.2

Resulting POST /changes must remain legacy/F2 behavior except for canonical snapshot consistency. It must still create no AuthorizationRound.

## Mandatory proof

A1 — single build: one new logical submission constructs Change at most once before the pending snapshot exists.

A2 — provider/index equality: provider input and durable index snapshot are structurally identical, including changeId, Change createdAt, activityIds, activity order, execution plan, requested window, and other canonical creation fields.

A3 — recovery stability: arrange an existing pending snapshot, retry, prove buildChange() is not called and original server-generated fields are preserved.

A4 — regressions: existing F2 create/idempotency/provider-recovery/list/detail behavior stays green.

Use deep structural equality where practical. Do not prove only two selected fields.

## Validation

Run:
- focused F3.1.2a tests
- existing Change Management regression suites
- SQLite path
- disposable PostgreSQL path where already supported
- backend lint
- backend build
- repository-wide TypeScript baseline comparison

TypeScript errors must remain set-identical to the accepted/current baseline. Do not fix unrelated debt. Do not add --forceExit.

## Publication

After gates pass, commit only F3.1.2a changes and publish by normal fast-forward to feat/ado-repo-governance. No force push/history rewrite. Record parent SHA, final SHA, and verify remote tip.

If publication is unavailable, report the candidate accurately and do not pretend it is published.

## Documentation after real implementation

Create docs/backstage/f3-1-2a-implementation-evidence.md and update current-state, implementation-progress, and prompts/README factually.

Next gate after implementation is an independent F3.1.2a architecture/implementation acceptance review. F3.1.2b remains NO-GO.

## Final report

Return:
Docs baseline reviewed: <sha>
ADO parent/tip before implementation: <sha>
Source drift: NONE | NON_BLOCKING | BLOCKED
F3.1.2a implementation: PASS | FAIL | BLOCKED_BY_SOURCE_DRIFT
Single canonical Change build: PASS | FAIL
Pending recovery reuses canonical snapshot: PASS | FAIL
Provider/index snapshot equality: PASS | FAIL
Authorization/ledger behavior added: NO
Migrations added: NO
Focused tests: <result>
Change Management regressions: <result>
SQLite: <result>
PostgreSQL: <result>
Lint: <result>
Build: <result>
TypeScript baseline: <result>
Implementation commit: <sha | NOT_COMMITTED>
Remote ADO tip after publication: <sha | NOT_PUBLISHED>
Canonical evidence: docs/backstage/f3-1-2a-implementation-evidence.md
Next gate: F3.1.2a independent acceptance review

## STOP

Stop after F3.1.2a. Do not implement F3.1.1c or F3.1.2b and do not continue into another slice.