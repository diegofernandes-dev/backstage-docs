# F3.1.3b — Architecture / Implementation Acceptance Review

## Status

INDEPENDENT REVIEW ONLY — NO IMPLEMENTATION, DATA REPAIR, OR NEW ROUND CREATION AUTHORIZED.

Purpose: independently decide whether the published F3.1.3b implementation at ADO commit `5e70d8818f55072d8568b5eb15bae754abb943d1` faithfully implements the accepted rejected-Change resubmission / new-Round contract.

Canonical authority:
- docs/backstage/f3-1-3-implementation-plan.md
- docs/backstage/f3-1-3-revised-plan-architecture-rereview.md
- docs/backstage/f3-1-3a-architecture-implementation-acceptance.md
- docs/backstage/f3-1-3b-implementation-evidence.md
- prompts/f3-1-3b-resubmission-new-round-implementation.md
- docs/adr/ADR-007-change-record-authority.md
- docs/adr/ADR-008-multi-activity-change-execution-plan.md
- docs/adr/ADR-009-change-authorization-model.md
- docs/adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md
- docs/adr/ADR-014-change-resubmission-authority-and-identity-boundary.md
- docs/backstage/current-state.md

Accepted parent baseline:
`6bad066d945d49feaf642313ec37467e2658dc3f`

Candidate implementation:
`5e70d8818f55072d8568b5eb15bae754abb943d1`

## 1. Mandatory fresh baseline

Before reviewing:
1. Fetch latest `backstage-docs@main` and record exact SHA.
2. Read every authority file above, implementation-progress, prompts/README, and this prompt.
3. Independently verify the live ADO branch `feat/ado-repo-governance` tip.
4. Inspect candidate `5e70d88` directly and verify its parent is exactly accepted F3.1.3a `6bad066`.
5. Inspect the complete diff `6bad066..5e70d88`; do not rely on implementation evidence alone.
6. If the branch has advanced beyond `5e70d88`, classify later drift separately. Review the exact candidate commit unless later drift invalidates runtime/product evidence.
7. Independently inspect durable laptop runtime/database facts when still available, but do not create, delete, repair, approve, reject, or resubmit any Change merely to satisfy the review.

## 2. Review boundary

Do not:
- modify ADO source;
- repair implementation defects;
- create a new Round;
- alter existing Round/Decision/Audit facts;
- change laptop overlay/config;
- implement F3.1.4/F3.2;
- add UI actions;
- touch production;
- mutate Delivery/Kargo/Argo/GitOps state.

Return exactly `ACCEPT` or `REJECT`.

## 3. Mandatory gates

G1 — Lineage / scope
- candidate is direct child of accepted F3.1.3a `6bad066`; 
- changed paths are limited to F3.1.3b authority, validation, Round-N materialization, idempotency, index/provider/participant projection, route/RBAC, tests, and required architecture guards;
- no migration;
- no frontend resubmit/decision UI;
- no F3.1.4/F3.2 behavior;
- no production/GitOps mutation;
- committed default remains `LEGACY_PRE_F3`.

G2 — Resubmission HTTP contract
Verify exact route:
`POST /changes/:changeId/resubmissions`.

Verify:
- user credentials only;
- required trimmed non-empty `Idempotency-Key`; 
- create-shaped editable payload is bounded and cannot set server identity/governance/provider fields;
- first successful materialization returns 201;
- exact completed replay returns 200 with the same logical `changeId` / `roundNumber` / submitted result;
- response remains command-bounded and does not become an F3.1.4 read model.

G3 — Dedicated permission and RBAC
Verify literal permission:
`change-management.change.resubmit`, action `update`.

Verify:
- it is distinct from `change.create`, `change.read`, `decide`, and `cab.record`; 
- technical role grants match the accepted plan/current role model;
- `change_cab_recorder` does not receive resubmit merely because it is CAB;
- permission alone never authorizes the business operation.

G4 — Resubmission business authority
Verify service-side domain rule exactly:

```text
has change.resubmit
AND
(actor == original requestedBy
 OR actor is current member of immutable ownerRef)
```

Verify:
- original requester without permission is denied;
- owner member without permission is denied;
- platform_admin who is neither requester nor immutable-owner member is denied;
- CAB member/cab.record holder alone is denied;
- participant/responsible/read/create/decide alone are denied;
- no generic admin/governance override exists.

G5 — Owner-membership proof / requester fast path
Verify owner path reuses prefix-agnostic live Catalog membership against the **immutable original ownerRef**.

Required:
- normalized/deduped `relations.memberOf` + `spec.memberOf`; 
- no TP-prefixed ownership helper as authority;
- Catalog unavailable -> `PROVIDER_UNAVAILABLE` / `membership_source_unavailable`, zero writes;
- missing User -> fail closed;
- missing/unresolvable immutable owner Group -> `owner_authority_unresolvable`; 
- no ownerRef -> owner-member path impossible;
- exact original-requester path does **not** unnecessarily depend on Catalog owner-membership availability.

G6 — Same-change identity freeze
Verify these are immutable across Round 1 -> Round N:
- changeId;
- requestedBy;
- identity createdAt;
- targetRef;
- ownerRef;
- systemRef.

Verify:
- owner/system are not re-resolved from current Catalog;
- current Catalog ownership drift cannot transfer identity or authority;
- changed targetRef fails before durable mutation;
- no provider/index helper can accidentally rewrite identity fields;
- retargeting requires a new Change/new changeId.

G7 — Correctable-field boundary
Verify only accepted non-identity business/execution fields may change.

Ensure no hidden input path can mutate frozen identity through DTO spreading, provider record replacement, index projection, or derived Catalog context.

G8 — Execution-plan hidden-retarget guard
Verify:
- omitted activity targetRef remains valid;
- present target is a Component and is Catalog-resolved;
- activity System resolution uses canonical source rules;
- activity System must equal immutable Change systemRef, including both absent;
- cross-System or System-introduction mismatch fails `execution_plan_identity_mismatch` before mutation;
- activity target never rewrites Change target/owner/System.

G9 — Resubmission eligibility / current Round
Verify only a `LEDGER_REQUIRED` Change whose **current max Round** derives `REJECTED` can materialize a new Round.

Verify fail-closed behavior for:
- LEGACY_PRE_F3;
- no Round;
- current PENDING;
- current AUTHORIZED;
- a historical rejected Round when a newer current Round exists.

Ledger facts, not merely projected status, must be authoritative.

G10 — New canonical snapshot and policy rebinding
Verify Round N snapshot contains:
- same immutable identity;
- corrected allowed fields;
- submitted lifecycle projection;
- new activity IDs where required;
- canonical snapshot/hash;
- current published policy identity;
- current selector bundle identity/digest;
- newly resolved principal snapshots;
- `requirementId = requirementRole`; 
- no copied prior decisions.

Verify policy/selector/principal resolution is pinned deterministically once for the successful attempt and persisted coherently.

G11 — Round-N generalization without Round-1 regression
Inspect `ledgerSubmission` or equivalent carefully.

Verify:
- audit payloads use actual `round.roundNumber`, not hardcoded 1;
- normal F3.1.2/F3.1.3a create path still creates only Round 1;
- only F3.1.3b may create Round >1;
- no generic workflow/new-round endpoint was introduced;
- Round-1 submission semantics and CAB-safe policy behavior remain unchanged.

G12 — Resubmission idempotency
Verify new operation namespace `change.resubmit` uses the existing idempotency store with no DDL.

Required:
- operation actor/requested_by is authenticated resubmitting actor;
- key is new/required;
- payload hash includes changeId + normalized correction body;
- original `change.create` reservation remains untouched;
- exact replay after Round 2 exists returns original result before current-Round terminal check blocks it;
- same key / changed body conflicts;
- reuse of an already-bound actor/key from another logical operation cannot silently become a resubmission replay;
- failed/lost competing attempt leaves no dangling pending resubmit reservation.

G13 — Caller-owned atomic transaction
Verify one transaction owns all durable F3.1.3b state:
- parent `change_index FOR UPDATE`; 
- authoritative re-read/current Round check;
- identity revalidation;
- resubmit reservation;
- Round N + requirements;
- round/policy/selector/requirement audits;
- exactly one `change.authorization.resubmitted`; 
- index current non-identity projection + status submitted;
- participant rebuild;
- DevelopmentProvider.replaceCurrent;
- idempotency completion.

No partial state may become visible.

No Catalog network I/O may be held under the database write lock.

G14 — Index projection / participant rebuild
Verify index update changes only accepted current **non-identity** discovery fields and status.

Identity columns including authorization_mode remain untouched.

Verify participant rows are transactionally replaced from the corrected execution plan and are derived/non-authoritative.

Check list/discovery behavior shows current corrected non-identity data without rewriting historical Round snapshots.

G15 — DevelopmentProvider.replaceCurrent
Verify explicit replacement exists only for resubmission and does not reinterpret `create()` / `createWithTransaction()`.

Required:
- same existing changeId only;
- caller-owned transaction;
- frozen identity copied/preserved exactly;
- corrected non-identity current record;
- status submitted;
- missing/incoherent provider record fails closed instead of creating a new Change;
- external/non-dev provider fails closed; no 2PC/outbox invention.

G16 — History immutability
After Round 2, independently verify/source-prove Round 1 remains immutable:
- snapshot/hash;
- requirements;
- decisions;
- audits;
- rejection evidence.

Append-only UPDATE/DELETE protections remain in force.

G17 — Concurrency R1
PostgreSQL 16 is authoritative.

Two concurrent resubmissions after one rejected Round must yield:
- exactly one Round N+1;
- no Round N+2;
- one winner projection/provider/participants/audit set;
- loser `CONFLICT / resubmission_conflict`; 
- no dangling loser reservation.

Exact same actor/key/payload retry of the committed winner must still replay rather than become `resubmission_conflict`.

Inspect test construction: deterministic synchronization/failure hooks, not sleep-based correctness.

G18 — Authority / identity matrix R2-R13
Independently inspect/re-run enough to verify all accepted proofs:
- R2 PENDING/AUTHORIZED blocked;
- R3 prior history immutable;
- R4 requester + permission / requester without permission;
- R5 immutable-owner member + live membership / membership removed;
- R6 platform_admin alone denied;
- R7 CAB alone denied;
- R8 permission-only denied;
- R9 Catalog ownership drift cannot transfer resubmit authority;
- R10 targetRef mismatch fails before mutation;
- R11 owner/System/requestedBy/createdAt identity cannot change;
- R12 allowed correction creates one next Round and preserves identity;
- R13 cross-System activity target fails closed.

G19 — Failure injection / rollback
Verify injected failures after each material point roll back everything:
- after Round insert;
- after authorization audits;
- after index projection;
- after participant rebuild;
- after provider replacement before idempotency completion/commit.

After each failure there must be:
- no durable new Round;
- no new resubmission audits;
- old rejected index projection intact;
- old provider current record intact;
- old participants intact;
- no dangling/completed failed resubmit reservation.

G20 — F3.1.3a regression
Re-prove accepted decision semantics remain unchanged:
- decision route/transport;
- individual authority;
- CAB membership + cab.record;
- decision replay/conflicts;
- PostgreSQL D1-D6;
- rejection lifecycle;
- eligibility missing-Round safety;
- post-execution fail-closed;
- accepted AUTHORIZED/submitted happy-path Change receives no accidental Round 2.

G21 — Live Round-2 evidence
Without creating new facts, independently inspect the recorded disposable Change (`CHG-2026-000005`) if still available.

Verify:
- before history: Round 1 rejected with its original decision/audits;
- exactly one Round 2 exists;
- same changeId;
- frozen requestedBy/createdAt/targetRef/ownerRef/systemRef;
- corrected current title/non-identity projection;
- Round 2 current policy/selector + primary/CAB requirements;
- Round 2 zero decisions;
- exactly one resubmitted audit;
- exact replay did not duplicate Round/audits/projection/idempotency;
- current eligibility is based on Round 2 and is PENDING_AUTHORIZATION (or another factually justified current gate, but not Round-1 REJECTED);
- list/detail returned to submitted.

The live retarget attempt in implementation evidence occurred after Round 2 already existed and therefore returned `round_not_terminal`; do **not** treat that live 409 as the R10 identity proof. R10 must be established independently from source/tests on a rejected-round path.

G22 — Product convergence / quality
Independently verify/re-run enough to establish:
- full Change Management suite;
- PostgreSQL R1 plus D1-D6 regression;
- F3.1.2 submission/concurrency regressions;
- F3.1.1 policy/selector regressions;
- GMUD frontend;
- Catalog;
- Deployments tab;
- Delivery backend reads;
- architecture guards;
- permissions/RBAC;
- lint;
- full build;
- repository-wide TypeScript baseline.

Historical TypeScript debt is acceptable only if the exact pre-candidate baseline set is unchanged. Zero new errors.

G23 — Hard non-goals
Confirm candidate adds none of:
- resubmit UI/button;
- approve/reject UI expansion;
- F3.1.4 composed authorization/governance panel;
- CAB Workbench;
- F3.2 autonomy;
- Teams approval;
- execution lifecycle;
- decision reversal/expiry;
- ownership-transfer administration;
- external-provider resubmission protocol;
- migrations;
- production cutover;
- Kargo/Argo/GitOps mutation;
- generic workflow/queue/distributed-lock framework.

## 4. Decision

Return exactly one:
`ACCEPT` or `REJECT`.

Do not use CONDITIONAL_ACCEPT.

ACCEPT only if the exact published candidate is a faithful implementation of the accepted F3.1.3b + ADR-014 contract and no material proof gap remains.

## 5. If ACCEPT

Record exactly:

```text
F3.1.3b architecture/implementation acceptance: ACCEPT
F3.1.3b: CLOSED / ACCEPTED IMPLEMENTED BASELINE
Accepted ADO SHA: 5e70d8818f55072d8568b5eb15bae754abb943d1
F3.1.3: CLOSED / ACCEPTED IMPLEMENTED DECISION + RESUBMISSION BASELINE
F3.1.4 planning/prompt authoring: GO
F3.1.4 implementation: still requires separate explicit authorization
F3.2 implementation: NO-GO
Production cutover: NOT AUTHORIZED
```

Do not author the F3.1.4 prompt from inside this review.

## 6. If REJECT

Identify only concrete source defects or proof gaps.
Do not repair code inside the review.
Keep F3.1.4 implementation NO-GO and set the next activity to the narrowest correction.

## 7. Canonical documentation

Create:
- `docs/backstage/f3-1-3b-architecture-implementation-acceptance.md`

Update factually:
- `docs/backstage/current-state.md`; 
- `docs/backstage/implementation-progress.md`; 
- `prompts/README.md`.

Preserve implementation evidence, F3.1.3a acceptance, ADR-014, and all prior plan/review history.

Commit documentation only and STOP.

## 8. Required final report

Return:

```text
Docs baseline reviewed: <sha>
ADO candidate reviewed: 5e70d8818f55072d8568b5eb15bae754abb943d1
Parent verified: YES | NO
Independent ADO source verification: YES | NO
Lineage/scope gate: PASS | FAIL
Resubmission transport gate: PASS | FAIL
Permission/RBAC gate: PASS | FAIL
Business-authority gate: PASS | FAIL
Owner-membership/requester-path gate: PASS | FAIL
Identity-freeze gate: PASS | FAIL
Correctable-fields gate: PASS | FAIL
Execution-plan retarget gate: PASS | FAIL
Current-Round eligibility gate: PASS | FAIL
Round-N policy/selector gate: PASS | FAIL
Round-1 regression gate: PASS | FAIL
Resubmission idempotency gate: PASS | FAIL
Atomic transaction gate: PASS | FAIL
Index/participant projection gate: PASS | FAIL
Provider replaceCurrent gate: PASS | FAIL
History immutability gate: PASS | FAIL
R1 PostgreSQL concurrency gate: PASS | FAIL
R2-R13 gate: PASS | FAIL
Failure-injection gate: PASS | FAIL
F3.1.3a D1-D6 regression gate: PASS | FAIL
Live Round-2 evidence gate: PASS | FAIL | PARTIAL_EXPLAINED
Product/quality gate: PASS | FAIL
Hard non-goals gate: PASS | FAIL
Migration required: NO | CONTRADICTED
F3.1.3b architecture/implementation acceptance: ACCEPT | REJECT
ADO implementation modified by review: NO
F3.1.4/F3.2 implemented by review: NO
Final docs SHA: <sha>
```

## 9. STOP

STOP after the independent review.

Do not modify ADO implementation/data, author the F3.1.4 prompt, or implement F3.1.4/F3.2 from inside this checkpoint.