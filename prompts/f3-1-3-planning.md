# F3.1.3 — Decision Command and New-Round Semantics Planning

## Status

PLANNING / SOURCE-VERIFICATION ONLY — NO IMPLEMENTATION AUTHORIZED.

Purpose: produce the implementation-ready plan for F3.1.3 on top of the accepted live-ledger F3.1.2 baseline.

F3.1.2 is closed and accepted. Non-production `LEDGER_REQUIRED` activation is independently accepted. The operator-laptop runtime is an accepted F3.1.3 demonstration target.

Architecture authority:
- docs/adr/ADR-009-change-authorization-model.md
- docs/adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md
- docs/backstage/f3-1-implementation-plan.md
- docs/backstage/f3-1-2-implementation-plan.md
- docs/backstage/f3-1-2b-architecture-implementation-acceptance.md
- docs/backstage/f3-1-2-ledger-required-nonprod-activation-acceptance.md
- docs/backstage/current-state.md

Accepted implementation/runtime baseline:
- ADO repo: `platform-devops-developer-portal`
- branch: `feat/ado-repo-governance`
- accepted F3.1.2 implementation SHA: `22495229502dabf2d99588599a156d862c5114fa`
- accepted non-production effective mode: `LEDGER_REQUIRED` via gitignored laptop overlay
- committed repository default remains `LEGACY_PRE_F3`

## 1. Mandatory fresh baseline

Before planning:
1. Fetch latest `backstage-docs@main` and record exact SHA.
2. Read all architecture authority files above, implementation-progress, prompts/README, and this prompt.
3. Fetch the actual ADO branch tip and record exact SHA.
4. Verify whether the tip still equals `2249522` or has later accepted drift.
5. Inspect actual source for:
   - AuthorizationLedgerRepository + Knex implementation;
   - ApprovalDecision type/storage/idempotency constraints;
   - AuthorizationAuditEvent support;
   - AuthorizationEvaluation / GovernanceEvaluation;
   - ChangeManagementService and current lifecycle/status handling;
   - ChangeIndexRepository / DevelopmentProvider status semantics;
   - HTTP router/plugin composition;
   - identity ownership/membership resolution;
   - Backstage permission/RBAC definitions and policies;
   - execution-eligibility service/route;
   - current GMUD frontend detail/read model;
   - tests and migrations relevant to decisions and rounds.
6. Independently inspect the accepted laptop ledger facts when useful, especially `CHG-2026-000003` with Round 1 primary + CAB and zero decisions.

If source reality contradicts ADR-009 or the accepted F3.1 decomposition, do not invent a workaround. Record the contradiction and return BLOCKED.

## 2. Planning boundary

F3.1.3 owns:
- server-authoritative approval/rejection command semantics;
- decision-time actor authorization;
- exact idempotent replay / conflicting-decision behavior;
- append-only decision + audit persistence;
- derived authorization/governance transitions caused by decisions;
- rejected-round terminal semantics;
- resubmission/new-round semantics under the same `changeId` when correction is allowed;
- the minimum backend transport needed to exercise those commands on the accepted non-production product target.

F3.1.3 does NOT automatically own:
- F3.1.4 composed read representation / general permission-filtered authorization UI;
- CAB Workbench;
- CAB autonomy grants;
- Teams;
- additive requirements after submission;
- decision reversal / abstention / expiry;
- break-glass;
- SLA jobs/escalation automation;
- execution start/completion integration;
- production cutover;
- generic workflow/BPM.

Do not implement code in this checkpoint.

## 3. Required architecture/source questions

The plan must answer every question below from actual source + accepted architecture.

### Q1 — Exact decision command transport
Define the minimum HTTP/service command shape for one requirement decision.

At minimum decide:
- route shape;
- command body;
- required Idempotency-Key handling;
- approved vs rejected representation;
- rejection reason/comment requirements;
- optional authority/CAB evidence fields;
- response shape;
- stable public errors.

Do not expose provider-specific or Teams-specific fields as canonical command contract.

### Q2 — Individual decision authority
For an `individual` requirement, define exactly when the authenticated actor may record the decision.

Expected invariant to validate against source:
`actorRef == requirement.principalSnapshot.principalRef`

Determine whether any governance/admin override exists in accepted architecture. If none is already authorized, do not invent one.

### Q3 — CAB / authority decision authority
For `authority` / `cab` requirements, define the exact server-side proof that the current authenticated actor may act for the snapshotted authority ref.

Inspect current Entra/Catalog identity and group-membership primitives.

Plan must define:
- authoritative membership source;
- failure behavior if membership cannot be proven;
- how current membership is distinguished from the historical principal snapshot;
- what `ApprovalDecision.authorizationEvidence` stores;
- what actorRef and authorityRef are persisted.

Do not expand the CAB requirement into N member decisions.

### Q4 — Permission boundary
Choose the literal minimum backend permissions required by F3.1.3.

At minimum evaluate separation between:
- recording an individual requirement decision;
- recording an authority/CAB collective decision.

Do not grant decision capability merely because the actor can read the Change.

`platform_admin` must not automatically become CAB business authority unless current accepted RBAC explicitly says so.

F3.1.4 may own broader authorization/governance read permissions later, but F3.1.3 decision endpoints themselves must be server-protected now.

### Q5 — Decision idempotency
Define the exact idempotency key and command hash contract.

ADR-009 requires:
- one terminal decision per requirement per round;
- exact replay returns the original logical result;
- conflicting second decision is rejected;
- no destructive repair/reversal.

Plan the database/service concurrency behavior for:
- same decision replay;
- same key + changed command;
- different key + same terminal decision;
- concurrent approve vs reject;
- concurrent duplicate approve;
- retry after network failure post-commit.

Use database uniqueness/transaction semantics; no polling or distributed lock framework.

### Q6 — Decision transaction boundary
Define one caller-owned transaction boundary for:
- `ApprovalDecision` insert;
- required decision audit event;
- rejection/authorization-reached audit events if generated;
- any legitimate lifecycle/projection update required by accepted source architecture.

Do not create partially committed decision/audit state.

### Q7 — Authorization transition audit
Determine the exact audit events when a pre-execution decision changes derived authorization.

At minimum plan whether/how to append:
- decision recorded;
- authorization reached when the last mandatory pre requirement is approved;
- rejection when any mandatory pre requirement is rejected.

Events must be deterministic under concurrent decisions and idempotent replay.

Do not persist mutable `AuthorizationEvaluation` as a competing authority.

### Q8 — Change lifecycle on rejection
ADR-009 states:
`submitted --mandatory pre rejection--> rejected`.

Inspect how current Change lifecycle/status is persisted under Model C.

Plan must decide exactly how F3.1.3 realizes `rejected` without violating:
- append-only authorization evidence;
- provider/index authority boundaries;
- canonical Change snapshot immutability;
- existing list/detail behavior.

If the current persistence model cannot represent the lifecycle transition safely without a new accepted mechanism, call this out as a blocker or isolate it as an explicit micro-slice. Do not silently leave lifecycle `submitted` while claiming ADR compliance.

### Q9 — Approval does NOT mutate lifecycle
Confirm that reaching `AUTHORIZED` does not change Change lifecycle to `authorized`.

Expected result:
- Change may remain `submitted`;
- AuthorizationEvaluation becomes `AUTHORIZED` by derivation;
- execution eligibility may still be DENY for window/context reasons.

### Q10 — Post-execution requirements
Decide what F3.1.3 does with `post_execution` requirements such as emergency CAB retrospective.

Because accepted execution-start/completion integration is not yet implemented, plan must explicitly choose and justify one of:
- support recording post-execution decisions only when an accepted completion anchor already exists in current source;
- keep command generic but fail closed when required execution-completion evidence is unavailable;
- defer post-execution decision command behavior to the execution-lifecycle slice.

Do not allow a retrospective decision before its governance anchor merely because the requirement row exists.

### Q11 — New round / resubmission semantics
ADR-009 requires:
- a rejected round is terminal;
- correction/resubmission retains the same `changeId`;
- new immutable Change snapshot;
- new current published policy evaluation;
- new selector/principal snapshots;
- new immutable requirements;
- monotonically increasing `roundNumber`;
- all prior rounds/decisions/evidence preserved.

Inspect current Change/index/provider persistence carefully and define exactly how a corrected snapshot can coexist under the same `changeId`.

The plan must answer:
- command/route for resubmission;
- which Change fields may be corrected;
- whether original idempotency reservation is reused or a new operation/key is required;
- how provider/index snapshot versioning works;
- how participant/read discovery behaves;
- how current-round selection works;
- whether any schema change is actually required;
- transaction boundary for new snapshot + Round N + requirements + audit;
- concurrency if two resubmissions race;
- behavior if prior round is PENDING/AUTHORIZED vs REJECTED.

A new round may be created only after the prior round is terminal.

If current source cannot safely support same-changeId snapshot revision without schema work, do not compress this into the decision command. Recommend a separate F3.1.3b micro-slice with explicit migration/review if needed.

### Q12 — Relationship to live demo target
Use the accepted laptop `LEDGER_REQUIRED` target as a real validation surface.

Plan at least one demonstrable happy path for `CHG-2026-000003` or a fresh equivalent:
- primary actor approves;
- CAB actor records collective approval;
- derived authorization becomes AUTHORIZED;
- no Change lifecycle `authorized` state is invented;
- execution eligibility remains governed by window/context.

Do not fabricate CAB membership. If the current signed-in user cannot validly act for CAB, the plan must require a real authorized CAB test actor/group membership or use a fresh non-prod selector configuration only through an independently governed setup step — not by weakening authorization logic.

Also plan one rejection/resubmission demonstration on a disposable non-production Change.

## 4. Required source-level concurrency matrix

The plan must specify authoritative PostgreSQL proofs for at least:
- D1 exact decision replay;
- D2 concurrent duplicate approval;
- D3 concurrent approve vs reject for the same requirement — exactly one terminal decision wins, loser gets deterministic conflict;
- D4 decisions on two different requirements in the same round racing toward AUTHORIZED — exactly one authorization-reached audit event if such an event is part of the accepted plan;
- D5 authority membership failure/unavailable source fails closed with zero decision;
- D6 rejection produces terminal round/lifecycle behavior atomically;
- R1 two concurrent resubmissions after rejection — at most one next round number/snapshot wins;
- R2 resubmission while prior round is non-terminal fails closed;
- R3 prior decisions remain immutable after new round.

SQLite may provide functional coverage, but PostgreSQL is authoritative for real concurrency.

## 5. Error taxonomy

Produce a concrete error table using existing public codes wherever possible.

Must cover:
- Change not found;
- round not found;
- requirement not found;
- actor not authorized for individual requirement;
- actor not authorized for CAB/authority;
- membership/authority provider unavailable;
- rejection without reason;
- exact idempotent replay;
- conflicting second decision;
- terminal requirement;
- stale/non-current round decision attempt;
- resubmission before terminal rejection;
- concurrent resubmission conflict;
- invariant corruption.

Do not invent new HTTP codes when an existing stable error category is sufficient.

## 6. Slice decomposition decision

Based on actual source, choose the smallest safe decomposition.

Preferred candidates:

```text
F3.1.3a — server-authoritative decision command
F3.1.3b — rejection resubmission / new-round semantics
```

Use one F3.1.3 slice only if source reality proves both concerns can be implemented and reviewed without widening the blast radius.

Do not create micro-slices merely for ceremony; do split if lifecycle/snapshot/versioning makes resubmission materially different from decision recording.

## 7. UI boundary decision

State clearly what, if anything, F3.1.3 changes in the Backstage frontend.

Default expectation:
- F3.1.3 establishes backend command authority and can be proven through API/product test tooling;
- F3.1.4 owns the composed authorization/governance representation and permission-filtered detail experience;
- CAB Workbench remains outside F3.1.

If a tiny existing GMUD detail action is required for product proof, justify it explicitly and ensure it does not become the future CAB Workbench or bypass F3.1.4 permission/read design.

## 8. Migration decision

Return exactly:
`Migration required: YES` or `Migration required: NO`.

If YES:
- identify exact durable-state gap;
- define tables/columns/indexes/triggers;
- SQLite/PostgreSQL parity;
- backfill behavior;
- rollback implications;
- why existing ledger schema cannot satisfy the contract.

No migration 'for future proofing'.

## 9. Expected source paths

After inspecting actual ADO source, list exact files expected to change for each proposed micro-slice.

Separate:
- domain/types;
- ledger repository;
- service;
- router/plugin;
- permission/RBAC;
- provider/index/lifecycle persistence if genuinely required;
- frontend if genuinely required;
- tests;
- migrations if required.

## 10. Acceptance gates

Produce explicit implementation acceptance criteria covering:
- source lineage;
- decision authorization;
- CAB authority membership;
- server-side permissions;
- idempotency/conflicting decision semantics;
- decision transaction atomicity;
- immutable audit;
- authorization evaluation transitions;
- rejection lifecycle;
- new-round monotonicity/history preservation;
- post-execution decision timing;
- PostgreSQL concurrency;
- live laptop product proof;
- GMUD/Catalog/Deployments regressions;
- lint/build/TypeScript baseline;
- no F3.1.4/F3.2 leakage.

## 11. Output

Create:
- `docs/backstage/f3-1-3-implementation-plan.md`

Update factually:
- `docs/backstage/current-state.md`
- `docs/backstage/implementation-progress.md`
- `prompts/README.md`

Planning result must be exactly one of:

```text
F3.1.3 planning: READY_FOR_REVIEW
F3.1.3 implementation: NO-GO
```

or:

```text
F3.1.3 planning: BLOCKED
F3.1.3 implementation: NO-GO
Blocking contract: <precise>
```

Do not author the implementation prompt from inside this planning checkpoint.

Commit documentation only and STOP.

## 12. Required final report

Return:

```text
Docs baseline reviewed: <sha>
ADO branch tip verified: <sha>
Accepted live LEDGER demo target verified: YES | NO
Decision command transport: RESOLVED | BLOCKED
Individual authority contract: RESOLVED | BLOCKED
CAB/authority membership contract: RESOLVED | BLOCKED
Permission boundary: RESOLVED | BLOCKED
Decision idempotency/concurrency: RESOLVED | BLOCKED
Decision transaction/audit: RESOLVED | BLOCKED
Rejection lifecycle: RESOLVED | BLOCKED
Post-execution requirement behavior: RESOLVED | BLOCKED
Resubmission/new-round contract: RESOLVED | BLOCKED
Migration required: YES | NO | BLOCKED
Recommended slices: <one slice | F3.1.3a + F3.1.3b | blocked>
F3.1.3 planning: READY_FOR_REVIEW | BLOCKED
F3.1.3 implementation: NO-GO
ADO implementation modified: NO
Final docs SHA: <sha>
```

## 13. STOP

STOP after planning.

Do not modify ADO implementation, do not create ApprovalDecision facts, and do not implement F3.1.3/F3.1.4/F3.2.