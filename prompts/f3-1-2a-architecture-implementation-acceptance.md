# F3.1.2a — Canonical Change Architecture / Implementation Acceptance Review

## Status

REVIEW ONLY — NO IMPLEMENTATION AUTHORIZED.

Purpose: independently decide whether the published F3.1.2a implementation at ADO commit ccee1e1676a2763e68880e5383ce1e5e48742843 satisfies the accepted F3.1.2a contract without scope leakage.

Canonical authority:
- docs/backstage/f3-1-2-implementation-plan.md
- docs/backstage/f3-1-2-final-architecture-rereview.md
- docs/backstage/f3-1-2a-implementation-evidence.md
- prompts/f3-1-2a-canonical-change-implementation.md

## Mandatory fresh baseline

1. Fetch latest main from diegofernandes-dev/backstage-docs and record exact SHA.
2. Read every canonical authority file above plus current-state, implementation-progress, and prompts/README.
3. Independently fetch ADO repo platform-devops-developer-portal, branch feat/ado-repo-governance.
4. Verify remote branch tip and inspect commit ccee1e1676a2763e68880e5383ce1e5e48742843 directly.
5. Verify parent is exactly 188d8e9cc43423f3644b3cacfb9849257838a583.
6. Inspect the complete diff 188d8e9..ccee1e1; do not rely only on implementation evidence.
7. If branch tip has advanced beyond ccee1e1, classify later drift separately. Review F3.1.2a at its exact commit unless later drift invalidates the acceptance claim.

## Review scope

Expected changed files only:
- packages/backend/src/modules/changeManagement/ChangeManagementService.ts
- packages/backend/src/modules/changeManagement/ChangeManagementService.test.ts
- packages/backend/src/modules/changeManagement/ChangeManagementService.recovery.test.ts

Reject unexpected production scope expansion unless it is demonstrably required by the accepted contract.

## Mandatory gates

G1 — Single canonical construction
- For a new logical submission, buildChange() runs at most once before the pending snapshot exists.
- No alternate second construction path produces an independent Change object with new activity IDs/timestamps.

G2 — Durable snapshot as recovery authority
- After insertPending, finalization uses the durable index snapshot, not a separately rebuilt Change.
- Existing pending-index recovery does not call buildChange().
- Recovery preserves original Change.createdAt and activityId values.

G3 — Provider/index structural identity
- Provider receives the same canonical snapshot represented by the durable index.
- Confirm tests use deep structural equality over the whole Change or an equivalently complete assertion, not just selected fields.

G4 — F2 behavior preserved
- POST /changes remains LEGACY_PRE_F3.
- No AuthorizationRound is created.
- Existing idempotency/reservation semantics are unchanged.
- Provider/index finalization semantics remain Model C compatible.
- List/detail participant behavior is unaffected.

G5 — Hard non-goals respected
Confirm there is no:
- AuthorizationRuntime wiring
- AuthorizationLedgerRepository wiring
- createRound/ApprovalRequirement behavior
- authorization_mode semantic change
- newSubmissionAuthorizationMode
- policy/selector publication change
- F3.1.1c behavior
- CAB behavior/autonomy
- RBAC/route/frontend/Delivery change
- migration

G6 — Failure/recovery semantics
- A failure after pending snapshot creation can retry using that exact snapshot.
- The new code does not create a state where provider and index can diverge due to snapshot construction.
- No new silent error swallowing or recovery loop was introduced.

G7 — Tests and proof quality
Independently re-run or inspect enough evidence to verify:
- A1 single-build test
- A2 provider/index equality test
- A3 pending recovery stability test
- existing create/recovery/integration/list/detail regressions
- SQLite path
- disposable PostgreSQL path where supported
- lint
- build
- repository-wide TypeScript baseline

Do not accept only because the implementation evidence says PASS.

G8 — Diff minimality
- Production change is the smallest reasonable fix.
- Test additions may be larger than production change, but should target the accepted invariants rather than introducing new framework code.

## Decision

Return exactly one:
ACCEPT
or
REJECT

ACCEPT only if the exact published implementation satisfies every F3.1.2a invariant and no material scope leakage or source contradiction remains.

Do not use CONDITIONAL_ACCEPT.

## If ACCEPT

Record:
F3.1.2a architecture/implementation acceptance: ACCEPT
F3.1.2a: CLOSED / ACCEPTED IMPLEMENTED BASELINE
Accepted ADO SHA: ccee1e1676a2763e68880e5383ce1e5e48742843
F3.1.1c implementation: still requires separate explicit authorization
F3.1.2b implementation: NO-GO until F3.1.1c is implemented and independently accepted

Set the next authorized activity to F3.1.1c CAB-safe policy implementation if its prompt is already authored and the user explicitly launches it.

## If REJECT

Identify only concrete defects in the published slice. Keep F3.1.2b NO-GO. Do not repair code from inside the review.

## Canonical documentation

Create:
- docs/backstage/f3-1-2a-architecture-implementation-acceptance.md

Update factually:
- docs/backstage/current-state.md
- docs/backstage/implementation-progress.md
- prompts/README.md

Commit documentation only and STOP.

## Final report

Return:
Docs baseline reviewed: <sha>
ADO commit reviewed: ccee1e1676a2763e68880e5383ce1e5e48742843
Parent verified: YES | NO
Independent ADO source verification: YES | NO
Single canonical build gate: PASS | FAIL
Durable recovery snapshot gate: PASS | FAIL
Provider/index identity gate: PASS | FAIL
F2 regression gate: PASS | FAIL
Hard non-goals gate: PASS | FAIL
Test/proof gate: PASS | FAIL
Diff minimality gate: PASS | FAIL
F3.1.2a architecture/implementation acceptance: ACCEPT | REJECT
ADO implementation modified by review: NO
Final docs SHA: <sha>

## STOP

STOP after the review. Do not implement F3.1.1c, F3.1.2b, F3.1.3, F3.1.4, or F3.2.