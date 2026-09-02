# GMUD F3.1 — Authorization Ledger Foundation implementation plan

- Status: F3.1.0-V **READY FOR ARCHITECTURE ACCEPTANCE AND PUBLICATION REVIEW**
- Date: 2026-09-01
- Accepted ADO implementation baseline: `platform-devops-developer-portal@6e28611` (F2.2.1)
- Unreviewed F3.1.0 candidate: `be16ffb02b59d791f0086d8e5086e4428c82b90d`
- Corrective local candidate (F3.1.0-C): `7a9347e318b3a3a0b5bf4d457f206ce033a6f833`, branch `review/f3-1-0-cutover-safety`
- Closure local candidate (F3.1.0-V): `06ec9cf`, branch `review/f3-1-0-cutover-safety`
- Documentation baseline for this closure: `backstage-docs@ef1e97253e6e0fb11e1a9d69368f3332ad5ffa74`
- Architecture authority: [ADR-009](../adr/ADR-009-change-authorization-model.md)

## Outcome

F3.1 is decomposed into five independently reviewed checkpoints. The first has a
corrected local candidate ready for architecture review; this handoff does not
authorize merge, ADO publication, or a later slice:

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

`change_index.authorization_mode` establishes a safe incremental-deployment
boundary:

- every existing finalized F2 change: `LEGACY_PRE_F3`;
- every existing unfinalized/pending F2 row: `LEGACY_PRE_F3`;
- every new Change created by the still-F2 submission path: `LEGACY_PRE_F3`.

The database default and the current F2 repository write are both
`LEGACY_PRE_F3`, making either application/schema rollout order safe. The marker
is constrained to the two declared values and immutable after insertion.

No historical round, requirement, decision, or audit event is fabricated. A
future F3.1.2 path must explicitly insert `LEDGER_REQUIRED` only as part of the
submission flow that creates AuthorizationRound 1 and its effective requirements.
The global default remains fail-safe; the mere presence of ledger tables never
opts a Change into ledger governance.

## Cross-cutover idempotency regime (F3.1.0-V)

The idempotency reservation (`change_idempotency`, F2 initial migration) is
written before `change_index` necessarily exists — `ChangeManagementService
.createChange` reserves it first, then resolves and inserts the index row. It is
therefore the earliest durable boundary for a logical submission, and F3.1.0-V
adds `change_idempotency.authorization_mode` (same two values, same default,
same immutability-by-trigger shape as `change_index.authorization_mode`) so that
boundary durably captures the regime.

**Invariant:** one logical idempotent submission (`operation`, `requestedBy`,
`idempotencyKey`) never changes authorization regime across a retry, including a
retry that crosses a deployment boundary.

- First reservation for a key persists the caller's requested mode, or
  `LEGACY_PRE_F3` when none is given — which is what the still-F2 submission path
  always does today, so current behavior is unchanged.
- A retry (same key, same payload) resolves to the stored regime regardless of
  what the current application version would otherwise default to; the column is
  never updated after insert.
- A retry with a payload mismatch keeps the existing `CONFLICT` behavior; a retry
  with a payload match but a different requested regime is also `CONFLICT` —
  fail closed rather than silently reinterpreting the submission.
- Different keys are independent (a new key may reserve a different mode); a
  different actor with the same key is independent (existing actor scoping is
  unchanged).
- Existing idempotency rows are backfilled to `LEGACY_PRE_F3` on migration — they
  originated under F2 behavior, and no ledger regime is fabricated for them.

`change_index.authorization_mode` is no longer independently hardcoded — the
index row inherits the value carried on the reservation at insert time via
`PendingIndexInput.authorizationMode`, and the service asserts the two agree
whenever both are read together, throwing (not repairing) on disagreement. This
makes the two columns one business decision recorded twice, not two.

**F3.1.2 boundary (not implemented here):** the future ledger-aware submission
path reserves with `LEDGER_REQUIRED`, preserves that mode across its own retries,
builds one canonical Change, creates `AuthorizationRound` 1 with its effective
requirements, and finalizes the index with `authorization_mode = LEDGER_REQUIRED`
— fail-closed throughout. If F3.1.2 code finds an existing reservation whose mode
is `LEGACY_PRE_F3`, it must resume F2 legacy submission semantics for that
logical request, never reinterpret it as a new ledger submission.

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

Corrective-candidate evidence at `7a9347e`:

- Change Management: 14 suites / 98 tests PASS, including SQLite and a disposable
  PostgreSQL 16 instance;
- PostgreSQL migration up/down, parent-row serialization, same-round concurrency,
  audit bigint mapping, FKs, checks, and immutability PASS;
- backend lint PASS;
- backend build PASS;
- the test command still retained the previously observed open handle after Jest
  reported success and was explicitly stopped; this is recorded, not represented
  as a clean process exit.

Closure-candidate evidence at `06ec9cf` (F3.1.0-V):

- Change Management: 15 suites / 109 tests PASS (SQLite, `CI=1 --runInBand`),
  including a new focused suite proving the cross-cutover idempotency-regime
  semantics above and extended SQLite/Postgres migration coverage for the new
  `change_idempotency.authorization_mode` column;
- Same suite against disposable PostgreSQL 16 with `--detectOpenHandles`: 15
  suites / 109 tests PASS, 0 open handles, **process exit code 0**;
- Baseline `6e28611` reproduced with the identical command: 9 suites / 68 tests
  PASS, 0 open handles, exit code 0;
- Baseline `6e28611` and candidate `06ec9cf` both reproduced entering Jest watch
  mode (hang, manual termination required) when `CI` is unset — the previously
  reported non-exit is this pre-existing `@backstage/cli` behavior
  (`repo test`/`package test` auto-append `--watch`/`--watchAll` without `CI`),
  not a resource leak, and not introduced by F3;
- backend lint PASS (`CI=1`); backend build PASS (`CI=1`);
- `tsc` repo-wide: 6 pre-existing errors, identical before and after this
  checkpoint's changes, none in the Change Management test path;
- no `--forceExit` added.

## Findings and prerequisites

The accepted ADO baseline is `6e28611`, while the remote feature branch remains at
`0b9cb38`. `be16ffb` was created after a planning-only checkpoint and is therefore
an unreviewed candidate, not an accepted implementation baseline. The corrective
commit `7a9347e` is a direct child of `be16ffb`, was produced in an isolated clean
worktree, and has not been pushed. Unrelated changes in the primary worktree were
not touched.

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

**GO for architecture acceptance and publication review of the F3.1.0-V
candidate.** Review the lineage `be16ffb` → `7a9347e` → `06ec9cf` as historical
candidate, cutover-safety correction, and closure. None of the three commits is
accepted, merged, or published to ADO by this handoff. The accepted
implementation baseline remains F2.2.1 at `6e28611`.

**NO-GO for F3.1.1–F3.1.4** until F3.1.0 is explicitly accepted.
