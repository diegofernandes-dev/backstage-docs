# F3.1.2 — Final Focused Architecture Re-Review After Concurrency + ADR-013 Alignment

## Status

REVIEW ONLY — NO IMPLEMENTATION AUTHORIZED.

The F3.1.2 plan has now been revised twice for implementation-contract correctness:

1. first architecture review REJECT closed repository mode semantics, caller-owned transaction participation, deterministic requirement identity, and rollback correctness;
2. second re-review REJECT closed deterministic healthy Round-1 loser convergence;
3. ADR-013 subsequently changed the target normal-low governance baseline to CAB-required by default and introduced a separate future F3.2 autonomy workstream.

The current plan must now be independently reviewed as one coherent implementation contract.

## Mandatory fresh baseline

Before reviewing:

1. Fetch latest main from diegofernandes-dev/backstage-docs and record the exact SHA.
2. Read in full:
   - docs/adr/ADR-009-change-authorization-model.md
   - docs/adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md
   - docs/backstage/f3-1-implementation-plan.md
   - docs/backstage/f3-1-2-implementation-plan.md
   - docs/backstage/f3-1-2-plan-architecture-review.md
   - docs/backstage/f3-1-2-revised-plan-architecture-rereview.md
   - docs/backstage/current-state.md
   - docs/backstage/implementation-progress.md
   - prompts/README.md
3. Independently inspect the actual current ADO implementation branch:
   - repository: platform-devops-developer-portal
   - branch: feat/ado-repo-governance
   - last accepted baseline: 188d8e9cc43423f3644b3cacfb9849257838a583
4. Record the actual branch tip.
5. If the branch has drifted on Change creation, idempotency, authorization policy/registry, selector runtime, ledger transaction APIs, or provider finalization, inspect and reconcile that drift before deciding.
6. Do not rely only on documentation claims for source facts that can be checked directly.

## Review boundary

This is a factual architecture/source re-review only.

Do not:
- modify ADO code;
- repair the plan during the review;
- publish a policy version;
- implement F3.1.2a/F3.1.1c/F3.1.2b;
- author an implementation prompt from inside this review;
- implement CAB autonomy;
- start F3.1.3/F3.1.4/F3.2;
- change ADR-009/012/013.

Write a fresh review document. Preserve both historical REJECT documents.

## Mandatory regression gates

### G1 — Repository mode contract

Verify:
- explicit requested authorization-mode mismatch remains repository CONFLICT;
- stored-mode-wins is service orchestration for existing reservations;
- newSubmissionAuthorizationMode applies only to genuinely new reservations;
- concurrent first-insert race converges on the DB winner's stored mode.

### G2 — Caller-owned transaction

Verify the plan still mandates one caller-owned platform transaction for:
- DevelopmentProvider createWithTransaction;
- Round 1 + requirements;
- every required authorization audit append;
- index.finalize;
- idempotency.complete.

External provider create remains outside the platform transaction and converges through idempotent retry/orphan recovery.

### G3 — Requirement identity

Verify:
- requirementId = requirementRole exactly;
- no hash fallback;
- per-rule requirementRole uniqueness is fail-closed at policy registration/publication/startup.

### G4 — Rollback correctness

Verify rollback to a pre-F3.1.2 binary is forbidden while pending LEDGER_REQUIRED reservations exist and the runbook query remains a correctness gate, not optional hardening.

### G5 — Concurrent Round-1 convergence

Verify the corrected plan has one deterministic healthy-race contract:

same actor + same Idempotency-Key + same payload:
- losing Round-1 transaction rolls back;
- loser re-reads winner's committed reservation/index/Round 1;
- coherent winner facts => same logical success;
- transient DB lock/serialization not yet observable => existing retryable semantics;
- genuine committed contradiction => INTERNAL_ERROR fail-closed;
- never Round 2;
- never CONFLICT merely for losing the healthy same-payload race.

Verify C1–C4 test contract is sufficient, with disposable PostgreSQL as the authoritative real-concurrency proof.

### G6 — ADR-013 target alignment

Verify current target is now:

- normal.low => primary + CAB by default;
- normal.medium => primary + CAB;
- normal.high => primary + CAB;
- emergency unchanged;
- bounded low-risk autonomy is a future CAB governance capability, not present in F3.1.2.

Verify the old accepted F3.1.1a policy identity is treated as immutable historical evidence and is not edited/reused.

### G7 — F3.1.1c prerequisite

Verify the plan correctly introduces a separate narrow prerequisite:

F3.1.1c — CAB-safe policy baseline publication

It must:
- publish a new immutable policy version;
- change only normal.low from primary-only to primary + CAB;
- retain medium/high/emergency behavior;
- use existing publication-integrity mechanisms;
- reuse cab-authority selector;
- contain no autonomy/grant/waiver engine;
- be accepted before F3.1.2b can enable LEDGER_REQUIRED.

F3.1.1c must not be mislabeled as a third F3.1.2 slice.

### G8 — F3.1.2a isolation

Verify F3.1.2a remains only:
- single canonical Change construction;
- pending-snapshot recovery reuse.

It must remain independent of CAB policy/autonomy and safe to implement after plan acceptance.

### G9 — F3.1.2b scope

Verify F3.1.2b:
- consumes the CAB-safe active policy;
- materializes primary + CAB for normal-low;
- has no requester skipCab;
- has no CabAutonomyGrant;
- has no waiver/exception engine;
- has no CAB Workbench;
- has no autonomy RBAC;
- preserves exactly the accepted submission/ledger scope.

### G10 — Future F3.2 separation

Verify ADR-013's F3.2 workstream is truly deferred:
- append-only autonomy grant storage;
- grant/revoke/renew commands;
- autonomy RBAC + CAB authority membership;
- Round applicability integration;
- multi-activity all-covered rule;
- grant/revoke race;
- CAB Workbench.

None of those should leak into F3.1.2 implementation scope.

### G11 — No migration claim

Verify F3.1.2 itself still needs no migration.

Do not make a claim about future F3.2 migration/storage until that workstream is planned.

### G12 — Source/test completeness

Verify expected source paths and test matrix are implementable against the actual ADO tip.

Specifically ensure:
- the new F3.1.1c dependency is reflected without requiring F3.1.2b to mutate the old policy artifact;
- normal-low integration proof exists;
- concurrency proof remains authoritative PostgreSQL;
- TypeScript baseline, lint/build, architecture guards, SQLite/Postgres regression expectations remain coherent.

## Decision

Return exactly one:

ACCEPT
or
REJECT

ACCEPT only if:
- all prior blockers remain closed;
- the concurrency correction is exact;
- ADR-013 alignment is internally consistent;
- F3.1.1c is a narrow prerequisite;
- F3.1.2a remains isolated;
- F3.1.2b remains autonomy-free and implementation-ready;
- actual source inspection does not reveal contradictory drift.

REJECT if any implementation-semantic choice remains open.

Do not use CONDITIONAL_ACCEPT.

## If ACCEPT

Record:

```text
F3.1.2 plan architecture re-review: ACCEPT
F3.1.2 plan: ACCEPTED IMPLEMENTATION CONTRACT
F3.1.2a implementation-prompt authoring: GO
F3.1.1c implementation planning/prompt authoring: GO
F3.1.2a implementation: still requires separate explicit authorization
F3.1.1c implementation: still requires separate explicit authorization
F3.1.2b implementation: NO-GO until F3.1.2a + F3.1.1c accepted
F3.2 implementation: NO-GO
```

The recommended execution order after separate authorization is:

```text
1. F3.1.2a canonical Change
2. F3.1.1c CAB-safe policy publication
3. F3.1.2b ledger submission integration
4. F3.1.3 decisions
5. F3.1.4 read/RBAC
6. F3.2 CAB Governance & Delegated Autonomy
```

F3.1.2a and F3.1.1c may be independently planned because their code surfaces are orthogonal, but F3.1.2b must not enable LEDGER_REQUIRED until both accepted prerequisites exist.

## If REJECT

Identify only concrete blocking contracts. Do not redesign already-accepted decisions unless source reality proves them impossible.

Keep all implementation NO-GO and set the next activity to the narrowest required plan correction.

## Canonical documentation updates

Create:
- docs/backstage/f3-1-2-final-architecture-rereview.md

Update:
- docs/backstage/current-state.md
- docs/backstage/implementation-progress.md
- prompts/README.md

Preserve:
- docs/backstage/f3-1-2-plan-architecture-review.md
- docs/backstage/f3-1-2-revised-plan-architecture-rereview.md

Commit documentation only and STOP.

## Final report

Return:

```text
Docs baseline reviewed: <sha>
ADO baseline expected: 188d8e9cc43423f3644b3cacfb9849257838a583
ADO branch tip verified: <sha>
Independent ADO source verification: YES | NO
Prior critical blockers closed: <N>/4
Concurrency convergence gate: PASS | FAIL
ADR-013 alignment gate: PASS | FAIL
F3.1.1c prerequisite gate: PASS | FAIL
F3.1.2a isolation gate: PASS | FAIL
F3.1.2b autonomy-free scope gate: PASS | FAIL
F3.1.2 plan architecture re-review: ACCEPT | REJECT
ADO implementation modified: NO
Final docs SHA: <sha>
```

STOP.
