# GMUD F3.1 — Authorization Ledger Foundation implementation plan

- Status: F3.1.0 **IMPLEMENTED locally** — remote publication pending ADO access
- Date: 2026-09-01
- ADO implementation baseline: `platform-devops-developer-portal`, branch `feat/ado-repo-governance`, local HEAD `6e28611`
- F3.1.0 implementation commit: `be16ffb`
- Documentation baseline: `backstage-docs@c0ef3b4`
- Architecture authority: [ADR-009](../adr/ADR-009-change-authorization-model.md)

## Outcome

F3.1 is decomposed into five independently reviewed checkpoints. Only the first is
authorized by this handoff:

1. **F3.1.0 — Ledger contract and durable storage**: domain types, pure evaluators,
   four append-only ledger tables, repository contracts, legacy marking, and
   SQLite/Postgres tests.
2. **F3.1.1 — Published policy and selector resolution**.
3. **F3.1.2 — Fail-closed submission and first AuthorizationRound**.
4. **F3.1.3 — Server-authoritative decision command and new-round semantics**.
5. **F3.1.4 — Composed read representation and permission boundaries**.

F3.1.0 introduces no route, submission integration, permission grant, frontend
behavior, policy runtime, selector lookup, Teams integration, CAB Workbench, or
Azure DevOps enforcement.

## F3.1.0 durable contract

### Canonical facts

- `AuthorizationRound`, identified by `(changeId, roundNumber)`, preserves an
  immutable Change snapshot and hash, policy key/version/input/provenance,
  selector-bundle key/version/provenance, and server timestamp.
- `ApprovalRequirement` preserves kind, phase, mandatory/source facts, resolved
  principal snapshot, separation-of-duty key, and any post-execution SLA facts.
- `ApprovalDecision` is one append-only `approved` or `rejected` fact for exactly
  one requirement, with authenticated actor, authority/evidence context, server
  timestamp, idempotency key, and command hash. Rejection requires a reason.
- `AuthorizationAuditEvent` is append-only and receives a database-issued
  monotonic sequence for deterministic audit ordering.

The four tables are protected from UPDATE and DELETE by dialect-specific database
triggers. Repository contracts expose insert and read operations only. Composite
foreign keys keep requirements, decisions, and audit events bound to their round.

### Derived facts

- Current round is the greatest committed `roundNumber`; there is no `isCurrent`.
- `AuthorizationEvaluation` is derived only from mandatory pre-execution
  requirements: rejected wins, undecided remains pending, otherwise authorized.
- `GovernanceEvaluation` is derived only from mandatory post-execution
  requirements and snapshotted SLA inputs.
- `Change.status` is unchanged by F3.1.0.

### Legacy policy

`change_index.authorization_mode` explicitly distinguishes:

- existing finalized F2 changes: `LEGACY_PRE_F3`;
- existing unfinalized rows and all new rows: `LEDGER_REQUIRED`.

No historical round or decision is fabricated. A finalized `LEDGER_REQUIRED`
record without a round is a consistency failure, not an implicit legacy record.

## Persistence and compatibility

The migration adds:

- `change_authorization_rounds`;
- `change_authorization_requirements`;
- `change_authorization_decisions`;
- `change_authorization_audit_events`;
- `change_index.authorization_mode` and its bounded legacy backfill.

Canonical JSON is stored as text only when the complete artifact is audited as a
unit. Queryable integrity facts remain relational. SQLite and Postgres receive
equivalent foreign keys, checks, uniqueness constraints, indexes, and immutable
table triggers. Postgres bigint audit sequences are represented as strings in the
TypeScript boundary.

## Acceptance tests

F3.1.0 is accepted only when:

- all existing F2 Change Management tests remain green;
- SQLite migration up/down and legacy backfill pass;
- Postgres migration/repository behavior is validated in CI when a Postgres test
  URL is supplied;
- duplicate rounds/requirements/decisions and broken FKs fail;
- rejection without a reason fails;
- UPDATE/DELETE against ledger tables fail;
- audit ordering and transaction rollback are deterministic;
- snapshot/hash round-trips pass;
- authorization and governance evaluator matrices pass;
- architecture guards continue to prohibit provider/ADO/Teams canonical fields,
  workflow repositories, mutable current/evaluation state, and a policy DSL.

## Findings and prerequisites

The inspected ADO local HEAD is the documented `6e28611` baseline, but the remote
feature branch remains at `0b9cb38`. The worktree also contains unrelated changes;
F3.1.0 must be staged and reviewed independently.

The current F2 create service calls `buildChange()` twice, allowing server-generated
`activityId` and `createdAt` values to differ between index and provider snapshots.
That deviation must be corrected before F3.1.2 submission integration, but it does
not block isolated F3.1.0 storage work.

Committed app config also references RBAC CSV/conditional-policy files absent from
HEAD. That must be resolved before F3.1.4; F3.1.0 adds no permissions.

## Architecture invariants and non-goals

- The ledger is platform-owned and adjacent to the canonical index.
- Model C and `IChangeManagementProvider` remain active.
- CAB remains one collective authority; group membership is not snapshotted.
- Participant read grants no decision authority.
- No workflow/BPM engine, DAG, task engine, generic policy DSL, mutable decision,
  historical approval fabrication, Teams integration, CAB UI, execution endpoint,
  pipeline enforcement, scheduler, or real ITSM provider is introduced.

## Gate

**F3.1.0 implementation complete locally.** The commit is `be16ffb`; pushing to
the ADO branch was rejected because the configured remote credentials lack read
access. Reconcile remote publication before opening review.

**NO-GO for F3.1.1–F3.1.4** until F3.1.0 is reviewed and the preceding
checkpoint is explicitly accepted.
