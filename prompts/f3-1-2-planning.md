# F3.1.2 — Fail-Closed Submission + First AuthorizationRound Planning

## Status

**PLANNING / ARCHITECTURE ONLY — NO IMPLEMENTATION AUTHORIZED.**

F3.1.1a and F3.1.1b are closed, accepted implemented baselines. This checkpoint prepares the implementation plan for the first point where the existing Change submission path, published authorization policy, selector resolution, and authorization ledger are composed.

Accepted implementation baseline:

- Azure DevOps repo: `platform-devops-developer-portal`
- branch: `feat/ado-repo-governance`
- accepted HEAD for this planning baseline: `188d8e9cc43423f3644b3cacfb9849257838a583`
- F3.1.1a parent baseline: `d3c0751a15b908cec8f5595c97e52f41226344ed`
- canonical docs: `diegofernandes-dev/backstage-docs@main`

This checkpoint must produce a reviewable, implementation-ready plan. It must **not** modify implementation source, migrations, configuration, runtime state, Azure DevOps, Delivery, or production infrastructure.

---

## 1. Objective

Design the smallest correct F3.1.2 slice that makes **new ledger-governed Change submissions** create their first immutable authorization round safely and deterministically, while preserving all pre-existing F2/F3.1.0 idempotency and Model C behavior.

The planned target behavior is conceptually:

```text
POST /changes
  -> reserve / recover idempotency regime
  -> preserve LEGACY_PRE_F3 retries exactly
  -> for genuinely new LEDGER_REQUIRED submissions:
       build one canonical Change snapshot
       select one pinned published policy
       evaluate policy
       use one pinned active selector bundle
       resolve required principals
       fail closed on selector / principal / separation-of-duty failures
       create AuthorizationRound 1 + effective requirements + audit evidence
       persist/finalize Change/index/provider state consistently
  -> return the same logical result on idempotent retry
```

The plan must explain the exact sequencing, transaction boundaries, crash windows, replay behavior, and error semantics. Do not hide hard parts behind phrases such as “atomically create everything” when the existing Model C provider boundary may not share the same transaction.

---

## 2. Mandatory fresh-baseline procedure

Before planning:

1. `git fetch origin main` in `diegofernandes-dev/backstage-docs`.
2. Record the exact fetched docs SHA.
3. Read at minimum:
   - `docs/adr/ADR-006-change-management-backend-contract.md`
   - `docs/adr/ADR-007-change-record-authority.md`
   - `docs/adr/ADR-008-multi-activity-change-execution-plan.md`
   - `docs/adr/ADR-009-change-authorization-model.md`
   - `docs/adr/ADR-012-delivery-management-gitops-promotion.md` if present under the canonical ADR path; otherwise locate the canonical ADR-012 file
   - `docs/backstage/f3-1-implementation-plan.md`
   - `docs/backstage/f3-1-1-implementation-plan.md`
   - `docs/backstage/f3-1-1a-architecture-acceptance.md`
   - `docs/backstage/f3-1-1b-implementation-evidence.md`
   - `docs/backstage/f3-1-1b-architecture-acceptance.md`
   - `docs/backstage/current-state.md`
   - `docs/backstage/implementation-progress.md`
   - this prompt.
4. Inspect the actual ADO implementation at exact baseline `188d8e9` before designing changes.
5. If the branch head has advanced, reconcile the difference before planning. Record the actual head and classify any drift that touches the F3.1.2 surface.
6. Do not plan from remembered method signatures. Read the actual service/repository/plugin/policy/selector implementations.

If ADO source cannot be accessed independently, stop with a planning blocker rather than inventing implementation details that depend on unseen code.

---

## 3. Strict planning-only boundary

You MAY:

- inspect source, migrations, tests, config and git history;
- run existing non-mutating tests;
- use disposable SQLite/Postgres environments;
- inspect a running development Backstage read-only;
- write/update canonical planning documentation only.

You MUST NOT:

- edit `platform-devops-developer-portal`;
- add migrations or routes;
- fix `buildChange()`;
- wire `POST /changes`;
- create an AuthorizationRound in any persistent shared environment;
- change authorization config / selector bundles / publication manifest;
- create F3.1.3 decision commands;
- implement permissions or F3.1.4;
- touch Teams/CAB UI;
- modify Delivery/Kargo/Argo/GitOps;
- create a production selector bundle;
- create an F3.1.2 implementation prompt as part of this checkpoint.

If a source defect is discovered, describe the required implementation change in the plan; do not apply it.

---

## 4. Planning question

The plan must answer:

> How can F3.1.2 introduce first-round authorization for genuinely new submissions without changing the authorization regime of any pre-existing logical submission, without creating divergent Change snapshots, without weakening Model C/provider isolation, and without leaving ambiguous or unrecoverable partial state across crash/retry boundaries?

This is the primary architecture question. Every section below exists to make the answer implementable.

---

## 5. Mandatory design gates

Resolve every gate below. A plan is not ready for implementation review if any gate is hand-waved.

### P1 — Exact baseline and F3.1.2 change surface

Inventory the current implementation, including exact relevant classes/functions/interfaces and their current responsibilities:

- `ChangeManagementService.createChange`;
- the current `buildChange()` call sites and generated fields;
- `IdempotencyRepository` reservation/recovery model;
- `ChangeIndexRepository`;
- `ProviderRegistry` / `IChangeManagementProvider` / `DevelopmentProvider`;
- `AuthorizationLedgerRepository`;
- policy registry/evaluator;
- authorization selector bootstrap/resolver;
- plugin dependency wiring;
- canonical hash/timestamp helpers;
- HTTP request/response validation relevant to submission.

Identify the minimal files that F3.1.2 is expected to modify. No speculative framework extraction.

### P2 — One canonical Change snapshot

The existing double-`buildChange()` behavior is a mandatory blocker.

Design the exact correction so that:

- server-generated `changeId`, `createdAt`, activity IDs and all snapshot fields are generated once;
- index, provider record, policy input, AuthorizationRound snapshot/hash and audit evidence derive from the same canonical Change value;
- no second builder invocation can create a semantically different snapshot;
- retry does not regenerate a competing canonical snapshot after a durable reservation has already progressed.

State whether this correction belongs inside the F3.1.2 implementation slice or must be a tiny prerequisite commit/checkpoint. Prefer the smallest reviewable approach and justify it.

### P3 — Authorization-regime selection and cutover

The plan must define **exactly** how a genuinely new submission becomes `LEDGER_REQUIRED`.

It must preserve the accepted invariant:

> Existing reservation mode wins forever for that logical idempotent submission.

Required cases:

- existing `LEGACY_PRE_F3` reservation + same payload after F3.1.2 deployment;
- existing `LEGACY_PRE_F3` reservation + mismatching payload;
- existing `LEDGER_REQUIRED` reservation + retry during/after a crash;
- no reservation yet;
- two concurrent first attempts using the same actor + idempotency key;
- same key under a different actor.

Do not infer the regime from application version, schema version, wall-clock date, deployment date, branch, or presence of ledger tables.

If a runtime/config rollout switch is required, define its owner, default, startup validation, rollback semantics and why it cannot reinterpret an existing reservation. Avoid introducing a generalized feature-flag framework for one cutover.

### P4 — Policy and selector consistency at submission

Design how one submission binds to exactly:

- one published policy key/version/model/digest/provenance;
- one active selector-bundle key/version/digest/provenance;
- one immutable policy input derived from the canonical Change;
- one set of effective requirements;
- one set of resolved principal snapshots.

The plan must prove there is no “read active policy twice and get two versions” or mutable-current-policy reinterpretation inside one submission.

Use the existing F3.1.1a/b runtime rather than creating a second policy or selector service.

### P5 — Fail-closed principal resolution and emergency A/B effective-person distinctness

F3.1.2 is the first layer where resolved principals become part of a real round.

Plan the exact validation order:

1. policy evaluation;
2. required selector lookup/resolution;
3. resolved-principal type validation;
4. separation-of-duty validation.

For emergency A/B, the plan must fail the submission **before Round 1 becomes committed** if required distinct pre-execution selectors resolve to the same effective `User` ref.

Do not conflate this with F3.1.3 decision-time actor distinctness. F3.1.2 proves the snapshotted resolved principals are distinct; F3.1.3 later proves actual decision actors/authority evidence as required.

Define stable error classification and retryability for:

- selector missing/config invalid;
- Catalog missing entity;
- Catalog unavailable;
- same-person A/B resolution;
- unsupported/invalid policy/bundle state.

### P6 — Effective requirement construction

Specify the deterministic mapping from F3.1.1 policy output + selector resolution into persisted `ApprovalRequirement` records.

For every field, state its source:

- round-local requirement ID;
- `kind`;
- `phase`;
- `mandatory`;
- source / sourceRef / sourceProvenance;
- principal snapshot;
- separation-of-duty key;
- SLA key/version/duration/anchor;
- server-controlled creation timestamp.

Requirements must be immutable after round creation.

Do not silently implement “additional mandatory requirements” unless the current request contract and dedicated permission model are already ready. The planning document must explicitly decide whether additive user-supplied requirements are:
- in F3.1.2 scope, with a complete safe contract and permission boundary; or
- deferred to a later bounded slice.

Default to deferral unless concrete current architecture/source evidence makes their inclusion necessary for the first vertical round-creation slice.

### P7 — Round 1 construction and canonical audit facts

Define the exact Round 1 record:

- `roundNumber = 1`;
- immutable Change snapshot + hash;
- policy identity/digest/provenance/input + input hash;
- matched rule provenance;
- selector-bundle identity/digest/provenance;
- server timestamp;
- effective requirements;
- initial audit events.

Specify the minimum audit events that F3.1.2 must append to satisfy ADR-009 for submission / round creation / policy selection / selector resolution, without implementing F3.1.3 decision events or execution lifecycle events.

Avoid duplicating mutable derived evaluations as canonical facts.

### P8 — Transaction boundary vs. Model C provider boundary

This is a critical gate.

The plan must inspect whether the following stores currently share one DB transaction:

- idempotency reservation;
- canonical index;
- authorization ledger;
- `DevelopmentProvider`.

Then plan for the general Model C rule that a future real provider may be external and **cannot** participate in the platform DB transaction.

Do not claim distributed atomicity that does not exist.

Provide an explicit state machine / sequence showing which facts become durable in which order and how crash-safe retry converges.

The plan must answer at least these crash points:

- after reservation, before canonical Change generation;
- after canonical snapshot generation, before provider create;
- after provider create, before index/ledger transaction;
- after Round 1/requirements insert, before index finalize;
- after index/ledger commit, before idempotency completion;
- after idempotency completion response is lost.

If existing F2 orphan/reconciliation semantics are reused, show exactly how. If they are insufficient for `LEDGER_REQUIRED`, define the smallest required extension.

### P9 — Visibility invariant for partially completed submissions

Define when a new ledger-governed Change becomes visible to:

- `GET /changes`;
- `GET /changes/:id`;
- future authorization reads.

No reader may observe a Change labeled `LEDGER_REQUIRED` as finalized/usable if its Round 1 or mandatory requirements are absent.

Conversely, retries must be able to recover durable partial work without fabricating a second round.

State the authoritative completion condition.

### P10 — Idempotent replay contract

Define the result of retrying the exact same request after each crash window.

Required invariants:

- no duplicate provider record;
- no duplicate index record;
- no Round 2 from a retry of initial submission;
- no duplicate Round 1;
- no duplicate requirements;
- no changed principal snapshots on replay after Round 1 is already committed;
- no re-evaluation under a newer active policy/bundle once the logical submission has durably bound to its original authorization artifacts;
- conflicting payload remains `CONFLICT`.

The plan must identify the earliest durable point at which the selected policy/bundle/principal results are fixed for replay.

### P11 — Concurrency

Plan deterministic tests/behavior for:

- two concurrent attempts for same actor/idempotency key;
- duplicate DB insert races;
- concurrent provider create recovery;
- two different idempotency keys creating two different Changes;
- retry racing with completion of the first request.

Use existing DB constraints/locking where possible. Do not create an in-memory mutex as correctness authority.

### P12 — Error taxonomy and HTTP contract

Map failure domains to existing canonical errors/statuses where possible.

At minimum classify:

- bad client input;
- idempotency conflict;
- Catalog principal not found;
- Catalog/provider temporarily unavailable;
- policy/selector configuration/internal invariant violation;
- separation-of-duty violation;
- storage failure;
- provider failure;
- recovery-in-progress vs retryable failure if such distinction exists.

Avoid leaking provider-specific details or turning policy internals into public workflow status.

If a new stable error code is required, specify it narrowly.

### P13 — Legacy compatibility

Prove that F3.1.2 changes do not rewrite old Changes.

Explicitly preserve:

- all existing F2 Changes;
- all existing `LEGACY_PRE_F3` reservations;
- all pending/unfinalized legacy recovery paths;
- provider routing through stored `providerKey`;
- existing list/detail participant visibility;
- no fabricated authorization rounds for history.

No background migration to ledger governance.

### P14 — ADR-012 / Delivery boundary

F3.1.2 authorizes a business Change; it does not execute deployments.

The plan must ensure no canonical F3.1.2 field or logic requires:

- Azure DevOps pipeline/build/environment IDs;
- Kargo objects;
- Argo Applications;
- Git branches as environment identity;
- ReleaseCandidate / DeploymentRequest / ChangeBinding persistence inside Change authorization.

Delivery may later bind a governed deployment request to `changeId + activityId`; F3.1.2 must remain provider-neutral and execution-agnostic.

### P15 — Scope boundary with F3.1.3 / F3.1.4

F3.1.2 must not implement:

- approval/decision commands;
- authority-membership decision checks;
- CAB recorder command;
- rejection/resubmission/new-round command;
- cancellation semantics beyond any already-existing behavior required for submission;
- composed authorization detail UI;
- new RBAC governance roles;
- Teams;
- CAB Workbench;
- execution eligibility transport.

The plan may identify interfaces needed by later slices, but must not prebuild them without a concrete F3.1.2 need.

### P16 — Schema/migration decision

Inspect the accepted F3.1.0 schema and determine whether F3.1.2 actually needs a migration.

Prefer **no migration** if the existing ledger and idempotency schema already carries the required facts.

If a migration is required, justify the exact missing invariant and provide SQLite/Postgres behavior, constraints, rollback, and compatibility.

Do not add “future-proof” columns.

### P17 — Wiring strategy

Specify how accepted F3.1.1a/b runtime dependencies reach the submission service.

The current plugin constructs authorization bootstrap objects without submission use.

Plan the smallest explicit dependency injection change needed to let `ChangeManagementService` consume a policy/selector submission capability without:

- global mutable singletons;
- re-reading config ad hoc;
- making the frontend authoritative;
- coupling service code directly to Backstage Catalog APIs if the resolver abstraction already exists;
- introducing a generic workflow engine.

### P18 — Testing strategy

Produce a concrete test matrix, not “add unit tests”.

At minimum include:

**Pure/domain**
- one canonical snapshot/hash;
- policy input derivation;
- requirement mapping;
- A/B same-person rejection;
- deterministic Round 1 construction.

**Idempotency/cutover**
- legacy reservation retry after F3.1.2;
- new ledger reservation happy path;
- same key/same payload replay;
- same key/different payload conflict;
- existing mode mismatch behavior;
- crash/recovery cases.

**Persistence**
- SQLite and disposable PostgreSQL;
- Round 1 + requirements + audit transaction rollback;
- uniqueness/concurrency;
- no partial finalized index without round;
- immutability triggers still pass.

**Provider/Model C**
- provider create success/failure/retry;
- orphan/reconciliation behavior;
- no provider-specific authorization fields.

**Policy/selector**
- active published policy/bundle;
- Catalog missing/unavailable;
- same-person emergency resolution;
- policy/bundle cannot change the logical submission on retry.

**Regression**
- existing F2/F3.1.0/F3.1.1 tests;
- list/detail behavior;
- architecture guards;
- lint/build;
- repository-wide TypeScript error baseline comparison.

Describe which tests are unit, integration, SQLite, PostgreSQL, live-Catalog opt-in, or failure-injection.

### P19 — Observability

Define a minimal low-cardinality event/log model for F3.1.2.

Useful events may include:

- authorization submission mode selected/recovered;
- round creation started/completed/failed;
- policy selected;
- selector resolution failed;
- submission recovery resumed.

Never log emails, secrets, access tokens, raw comments/evidence, or high-cardinality provider payloads.

Do not create a generalized observability framework.

### P20 — Rollback / deployment sequencing

Plan safe deployment and rollback semantics.

The key question is not “can we deploy the binary?” but:

> What happens to a logical submission reserved as `LEDGER_REQUIRED` if the binary rolls back before that submission completes?

The plan must answer this explicitly.

A rollback must never cause a `LEDGER_REQUIRED` reservation to be processed by an old binary that only understands legacy semantics.

If that means rollout requires a compatibility gate, config switch sequencing, or minimum-version constraint, specify it. Do not hand-wave this cross-version case.

---

## 6. Required challenge scenarios

The planner must answer all of these explicitly:

1. A legacy request was reserved yesterday, deployment happens, retry arrives today. What path executes?
2. A new F3.1.2 request reserves `LEDGER_REQUIRED`, then the process crashes before provider create. What survives and what does retry do?
3. Provider create succeeds but the process crashes before Round 1/index finalize. How is the orphan recovered without a duplicate provider record?
4. Round 1 is durable but the client receives a network error and retries. Why does policy/selector evaluation not drift to a new version?
5. Emergency A/B selectors are different keys but both resolve to the same `User`. What exactly is persisted?
6. Catalog becomes unavailable halfway through resolving requirements. Is any round visible/durable?
7. The active selector bundle changes between two separate requests. Is that allowed? Between two retries of the same logical request?
8. Two workers handle the same idempotency key concurrently. Which DB fact arbitrates?
9. A rollback deploys a binary that predates F3.1.2 while `LEDGER_REQUIRED` reservations exist. How is unsafe processing prevented?
10. A future real ITSM provider is external and its create call cannot share the platform transaction. Why does the plan remain correct?
11. A client attempts a mismatching payload with an existing key. Does any policy/Catalog work happen before conflict is returned?
12. Does a successful F3.1.2 submission mean the Change is authorized? Why not?
13. Does Round 1 creation change Change.lifecycle? If yes, where is that authorized by ADR-009? If no, state the correct lifecycle result.
14. Can an inactive historical policy or selector bundle be consulted on a replay? Under what durable binding?
15. What exact condition makes a new ledger-governed Change discoverable by GET/list?

---

## 7. Required decisions in the planning document

The planning document must make explicit decisions, not leave TODOs, for:

1. F3.1.2 cutover mechanism for genuinely new requests.
2. Whether `buildChange()` correction is part of F3.1.2 or a prerequisite micro-slice.
3. Transaction boundaries and recovery state machine.
4. Earliest durable authorization-artifact binding for idempotent replay.
5. Round 1 + requirements + audit insertion boundary.
6. A/B resolved-person distinctness rule.
7. Whether additional mandatory requirements are included or deferred.
8. Whether a migration is necessary.
9. Dependency-injection/wiring shape.
10. Rollback compatibility rule for outstanding `LEDGER_REQUIRED` reservations.
11. Minimal error taxonomy additions, if any.
12. Exact implementation slice decomposition.

Avoid generic “TBD during implementation” for any of these.

---

## 8. Preferred decomposition discipline

The planner must evaluate whether F3.1.2 should be:

- **one bounded implementation slice**, or
- **two explicitly ordered micro-slices** if a prerequisite such as the canonical single-build correction can be independently landed and proven without prematurely enabling ledger submission.

Do not split for aesthetics. Split only if it materially reduces correctness risk, review scope, or rollback hazard.

Likewise, do not create extra phases for tests/docs that belong naturally to the implementation slice.

---

## 9. Required planning document

Write:

`docs/backstage/f3-1-2-implementation-plan.md`

Required structure:

1. Status / authority / baselines
2. Current source reality at ADO baseline
3. Objective and explicit non-goals
4. Accepted invariants carried forward
5. Proposed submission state machine
6. Authorization-regime cutover decision
7. Single canonical Change construction
8. Policy/selector binding and principal resolution
9. Emergency separation-of-duty rule
10. Effective requirement + Round 1 mapping
11. Transaction boundaries and crash-recovery matrix
12. Visibility/finalization invariant
13. Idempotent replay and concurrency
14. Model C / provider boundary
15. Wiring / dependency changes
16. Migration decision
17. Error/HTTP contract
18. Observability
19. Deployment and rollback sequencing
20. Test matrix
21. Exact source paths expected to change
22. Slice decomposition / implementation order
23. Risks and rejected alternatives
24. Answers to all 15 challenge scenarios
25. Implementation acceptance criteria
26. GO / NO-GO recommendation for a separate implementation checkpoint
27. STOP

The document must be sufficiently concrete that a separate reviewer can challenge it and an implementation agent could later execute it without redesigning the architecture.

---

## 10. Acceptance criteria for the planning checkpoint

The planning checkpoint itself may return:

```text
F3.1.2 planning: READY_FOR_REVIEW
F3.1.2 implementation: NO-GO
```

only when:

- actual ADO source was inspected;
- P1–P20 are all resolved;
- all 15 challenge scenarios are answered;
- no unresolved architecture question is hidden behind implementation;
- transaction/recovery behavior is explicit;
- cross-version rollback with outstanding `LEDGER_REQUIRED` reservations is explicitly safe;
- the plan does not require changing ADR-009/ADR-012;
- source paths and tests are concrete.

If source access is unavailable or a blocking architecture question remains:

```text
F3.1.2 planning: BLOCKED
F3.1.2 implementation: NO-GO
```

State the smallest missing evidence or decision.

This checkpoint does not return implementation GO by itself. A separate architecture review of the plan is required before any implementation prompt is created.

---

## 11. Canonical documentation updates

After producing the plan:

- write `docs/backstage/f3-1-2-implementation-plan.md`;
- update `docs/backstage/current-state.md` only to point to the planning result;
- update `docs/backstage/implementation-progress.md` with a concise planning checkpoint;
- update `prompts/README.md` so the planning prompt moves to completed/historical and the next activity is the **independent F3.1.2 plan review**, not implementation.

Do not alter ADR-009 or ADR-012 unless the planning work proves a real contradiction. If such a contradiction is found, stop with `BLOCKED`; do not silently edit the ADR to fit the plan.

Commit documentation only.

---

## 12. Final report contract

Return:

```text
Docs baseline reviewed: <sha>
ADO baseline inspected: <sha>
F3.1.2 planning: READY_FOR_REVIEW | BLOCKED
Planning gates resolved: <N>/20
Challenge scenarios answered: <N>/15
Migration required: YES | NO
Planned implementation slices: <N>
F3.1.2 implementation: NO-GO
ADO implementation modified: NO
Canonical plan: docs/backstage/f3-1-2-implementation-plan.md
Final docs SHA: <sha>
```

Then summarize only the key architecture decisions, blockers (if any), and the next review gate.

---

## 13. STOP

STOP after the planning document and required canonical documentation updates are committed.

Do not:

- implement F3.1.2;
- fix `buildChange()`;
- add migrations/routes;
- wire policy/selector resolution into submission;
- create Round 1 in code;
- create an implementation prompt;
- start F3.1.3/F3.1.4;
- reopen Delivery/Deployments/production-rollout work.

The next checkpoint after `READY_FOR_REVIEW` is an **independent architecture review of the F3.1.2 plan**.
