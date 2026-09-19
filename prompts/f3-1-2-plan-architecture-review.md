# F3.1.2 — Independent Architecture Review of the Submission Plan

## Status

**REVIEW-ONLY — NO IMPLEMENTATION AUTHORIZED.**

This checkpoint independently reviews:

`docs/backstage/f3-1-2-implementation-plan.md`

against the accepted Change Management / authorization architecture and the actual ADO baseline.

The plan under review was published at:

- canonical docs commit: `backstage-docs@6ae99a530450058e30815c21d855eb87d90431b8`
- ADO implementation baseline inspected by the planner:
  `platform-devops-developer-portal@188d8e9cc43423f3644b3cacfb9849257838a583`
- branch: `feat/ado-repo-governance`

F3.1.1b remains the accepted implemented baseline.

This review decides whether the F3.1.2 plan is safe and concrete enough to become the implementation contract for two ordered micro-slices:

1. F3.1.2a — single canonical Change construction;
2. F3.1.2b — ledger-governed submission + Round 1.

The review itself must not implement either slice.

---

## 1. Objective

Answer exactly this question:

> Does the F3.1.2 plan define a correct, fail-closed, crash-recoverable and reviewable path from the current F2 create flow to first-round ledger governance without reinterpreting existing logical submissions, violating Model C, inventing distributed atomicity, or weakening ADR-009 / ADR-012 boundaries?

Return exactly one verdict:

```text
F3.1.2 plan architecture review: ACCEPT
```

or

```text
F3.1.2 plan architecture review: REJECT
```

There is no generic `CONDITIONAL_ACCEPT`.

A non-blocking note may accompany `ACCEPT`, but every unresolved correctness, authority, transaction, replay, rollback, or contract ambiguity is a blocker and therefore requires `REJECT`.

---

## 2. Mandatory fresh-baseline procedure

Before reviewing:

1. Fetch the latest `main` from `diegofernandes-dev/backstage-docs`.
2. Record the exact fetched docs SHA.
3. Read at minimum:
   - `docs/adr/ADR-006-change-management-backend-contract.md`
   - `docs/adr/ADR-007-change-record-authority.md`
   - `docs/adr/ADR-008-multi-activity-change-execution-plan.md`
   - `docs/adr/ADR-009-change-authorization-model.md`
   - canonical ADR-012
   - `docs/backstage/f3-1-implementation-plan.md`
   - `docs/backstage/f3-1-1-implementation-plan.md`
   - `docs/backstage/f3-1-1a-architecture-acceptance.md`
   - `docs/backstage/f3-1-1b-architecture-acceptance.md`
   - `docs/backstage/f3-1-2-implementation-plan.md`
   - `docs/backstage/current-state.md`
   - `docs/backstage/implementation-progress.md`
   - this prompt.
4. Inspect the actual ADO source tree at exact SHA
   `188d8e9cc43423f3644b3cacfb9849257838a583`.
5. Verify the current ADO branch tip. If it has advanced, classify all drift touching the F3.1.2 review surface before deciding.
6. Do not accept a proposed method signature, transaction shape, repository capability, or recovery behavior merely because the plan says it exists. Verify it in source.

If implementation source is unavailable and an architecture-critical claim cannot be verified from accepted canonical evidence, return `REJECT`. Do not fill the gap from memory.

---

## 3. Strict review-only boundary

You MAY:

- inspect source/history/config/migrations/tests;
- run existing tests read-only in an isolated worktree;
- use disposable SQLite/Postgres for verification;
- inspect method signatures and transaction behavior;
- update canonical review documentation only.

You MUST NOT:

- modify ADO implementation;
- fix `buildChange()`;
- change idempotency semantics;
- add a transaction overload;
- add config;
- wire `POST /changes`;
- create an AuthorizationRound;
- add routes/migrations/permissions/frontend;
- create an F3.1.2a/F3.1.2b implementation prompt;
- start F3.1.3/F3.1.4;
- touch Delivery/Kargo/Argo/GitOps;
- alter ADR-009 or ADR-012 just to make the plan pass.

If a correction is required, describe the smallest exact correction to the plan and return `REJECT`.

---

## 4. Review principle

This is a review of an **implementation plan**, not an implementation.

Therefore the question is not whether the proposed design sounds reasonable. The plan must be sufficiently precise that a later implementation agent can execute it without making a new architecture decision.

Any sentence of the form:

- “prefer X, otherwise Y”;
- “possibly”;
- “if needed”;
- “optional hard guard”;
- “pick one scheme during implementation”;

must be examined to determine whether it hides a real design decision. If it does, the plan is not implementation-ready.

---

# 5. Mandatory architecture gates

Evaluate every gate below as `PASS` or `FAIL`. All must pass for `ACCEPT`.

## G1 — Baseline/source reality

Verify the plan's inventory against ADO `188d8e9`, including:

- current `ChangeManagementService.createChange` sequence;
- both `buildChange()` call sites;
- idempotency reservation semantics;
- index pending/finalize/read semantics;
- DevelopmentProvider transaction capability;
- external-provider path;
- AuthorizationLedgerRepository transaction behavior;
- policy runtime;
- selector runtime;
- plugin wiring;
- current errors;
- current architecture guard prohibiting submission wiring.

Any wrong source assumption that materially changes the proposed design is a blocker.

---

## G2 — Two-slice decomposition

Review the split:

### F3.1.2a
single canonical Change only, ledger still dark.

### F3.1.2b
cutover + runtime wiring + policy/selector + Round 1 + requirements + audit + replay/recovery.

Accept the split only if:

- 2a can land independently without changing authorization regime;
- 2a produces no AuthorizationRound;
- 2a preserves legacy observable behavior except fixing snapshot divergence;
- 2b can rely on the exact invariant established by 2a;
- there is no hidden third prerequisite.

If an additional repository-transaction capability must be introduced before 2b, decide whether it belongs explicitly in 2b or requires its own bounded prerequisite. The plan must say so.

---

## G3 — Single canonical Change invariant

Confirm the proposed 2a contract is exact:

- `buildChange()` is called at most once only when no durable pending snapshot exists;
- pending snapshot becomes the canonical object reused by provider, index, hash and Round;
- recovery never rebuilds a competing snapshot;
- server-generated `activityId` and `createdAt` are stable across retry;
- hash uses the existing canonical hashing primitive.

Reject any design where provider and Round can observe different Change values.

---

## G4 — Idempotency cutover semantics: resolve the canonical contradiction

This gate is mandatory and must not be waved through.

At the current accepted baseline, canonical documentation/source evidence contains two statements that appear to be in tension:

1. F3.1.0-V behavior/tests say a same-payload retry that explicitly requests a different `authorization_mode` conflicts.
2. Carried-forward F3.1.2 invariant says an existing reservation's stored mode wins and a deployment-default change must not produce a misleading conflict.

The F3.1.2 plan resolves this by changing `reserve()` semantics so a requested-mode mismatch on an existing row is ignored and stored mode wins.

Independently decide whether that is the correct architecture.

The review must distinguish:

- **explicit caller attempts to reinterpret an existing reservation**, versus
- **the application merely has a new default for new reservations after deployment**.

Explicitly evaluate a narrower alternative:

> preserve repository fail-closed mode-mismatch behavior, but make the service recover/read an existing reservation's stored mode before applying the new-submission default, so deployment defaults never “re-request” a different mode for an existing logical submission.

The review must choose one exact contract and justify it.

Do not accept both.

If the accepted plan changes an already accepted repository invariant, state why that change is necessary and which prior planning statement it supersedes. If no change is necessary, require the plan to remove it.

---

## G5 — New-submission mode cutover

Review the proposed:

`changeManagement.authorization.newSubmissionAuthorizationMode`

Accept only if:

- it applies solely at first reservation creation;
- default on capability deployment is `LEGACY_PRE_F3`;
- invalid values fail startup;
- flipping the pin cannot reinterpret existing reservations;
- no generalized feature-flag platform is introduced;
- ownership is consistent with authorization config ownership;
- enablement and disablement behavior are explicit.

Determine whether this config belongs in selector config files/types or in a broader authorization config boundary. The plan currently lists selector config paths; verify this does not semantically make submission cutover a selector concern.

---

## G6 — Earliest durable authorization binding

The plan currently says policy/bundle/principals become durable only when Round 1 commits.

Review whether this is safe for every pre-Round crash window.

Specifically decide whether a crash after external provider creation but before Round 1 may legitimately retry under a newer active policy/bundle/principal resolution.

Accept that only if:

- the Change remains invisible;
- no canonical authorization facts were committed;
- provider side effects are idempotently reconcilable;
- business semantics permit the not-yet-submitted/incomplete logical request to bind later;
- audit does not falsely imply an earlier policy was canonical.

If the logical submission must freeze authorization artifacts earlier than Round commit, identify the missing durable boundary.

---

## G7 — Policy/selector binding

Confirm one ledger submission attempt:

- uses one immutable runtime policy object;
- uses one immutable selector-bundle object;
- evaluates policy once before Round;
- resolves each requirement deterministically;
- persists exact policy/bundle identity and digest/provenance;
- never re-evaluates after Round 1 exists;
- can reconstruct/replay from ledger without requiring historical executable policy code.

No mutable “current policy” reinterpretation may occur after Round commit.

---

## G8 — Emergency separation of duty

Confirm F3.1.2b enforces submission-time distinctness of the **resolved effective User refs** for requirements sharing the relevant `separationOfDutyKey`.

Must fail before Round commit when A and B resolve to the same user.

Keep separate:

- selector-key distinctness at config/startup;
- resolved-principal distinctness at submission;
- actual decision-actor distinctness at F3.1.3.

The plan must not claim F3.1.2 solves decision-time SoD.

---

## G9 — Requirement identity is fully decided

The plan currently says:

- prefer `requirementRole` as `requirementId`;
- if collision appears, perhaps use a canonical hash;
- “pick one scheme and test it”.

That is not implementation-ready unless the review proves one exact scheme.

The review must choose one:

### Option A
`requirementId = requirementRole`, with a validated invariant that every effective requirement role is unique within a published rule/round and startup/publication/evaluation guarantees it.

### Option B
a deterministic canonical ID derived from exact fields defined now.

No fallback chosen at runtime or “if collision ever appears” architecture.

If Option A is accepted, require explicit validation/test of role uniqueness at the right boundary.

---

## G10 — Round 1 / requirement / audit mapping

Verify every persisted field has one deterministic source.

Confirm:

- `roundNumber = 1`;
- one canonical Change snapshot/hash;
- policy identity/version/digest/input/provenance;
- selector bundle identity/version/digest/provenance;
- requirement kind/phase/source/principal/SoD/SLA;
- one server-controlled Round timestamp;
- no decision facts;
- no mutable authorization evaluation persisted as authority;
- no Delivery/provider identifiers.

Review the proposed audit-event set. Ensure it is enough to reconstruct submission authorization setup without duplicating facts ambiguously.

---

## G11 — Transaction participation is real, not assumed

This is a critical gate.

The plan requires:

```text
DevelopmentProvider path:
  provider create
  Round 1 + requirements
  audit events
  index.finalize
  idempotency.complete
in ONE platform transaction
```

Inspect the actual ADO repository APIs.

Explicitly answer:

- Does `KnexAuthorizationLedgerRepository.createRound()` currently start its own transaction?
- Can it accept/use a caller-owned `Knex.Transaction`?
- Can `appendAuditEvent()` use the same caller-owned transaction?
- Do index.finalize and idempotency.complete already support caller-owned transaction?
- Does DevelopmentProvider use the same Knex/database instance?

If ledger methods currently self-open transactions, the plan must explicitly include the necessary repository API change and exact source/test paths. A nested/new independent transaction does **not** satisfy the proposed atomicity.

Do not accept “same database” as equivalent to “same transaction”.

---

## G12 — External-provider crash recovery

For a future external Model C provider, verify the plan remains correct without XA/2PC.

Review each state:

- reservation only;
- pending index;
- provider created / platform not committed;
- platform transaction failure;
- retry after orphan;
- Round exists but idempotency not complete;
- response lost after completion.

Require exact convergence rules and idempotent provider-by-`changeId` contract.

If the current `IChangeManagementProvider` contract does not guarantee the needed idempotency semantics for all future providers, the plan must state the provider capability/invariant required by F3.1.2 rather than assuming it.

---

## G13 — Visibility/finalization invariant

Confirm no `LEDGER_REQUIRED` Change becomes discoverable until Round 1 and mandatory requirements are durably present.

The review must verify how this is guaranteed **inside the same transaction**, not by a pre-transaction check susceptible to race/failure.

Reject a design that can commit:

`is_finalized = true`

without Round 1.

Also verify legacy finalized Changes remain unaffected.

---

## G14 — Replay after Round 1

Confirm an exact idempotent retry after Round 1:

- never creates Round 2;
- never resolves principals again;
- never evaluates a newly active policy;
- never changes requirement IDs;
- never changes snapshot/hash;
- converges provider/index/idempotency state;
- returns the same logical Change identity/result.

Concurrency correctness must rely on DB constraints/transactions, not in-memory locks.

---

## G15 — Rollback safety

Independently challenge the plan's statement:

> pre-F3.1.2 binary rollback is forbidden while pending LEDGER_REQUIRED reservations exist.

Inspect actual `188d8e9` behavior first.

Because current source reportedly treats an explicit mode mismatch as `CONFLICT`, determine whether an old binary would actually:

- corrupt/finalize a pending LEDGER reservation;
- fail closed with CONFLICT;
- or behave differently depending on the exact recovery path.

The review must not accept an inaccurate rollback threat model.

Choose and document the actual compatibility rule.

At minimum cover:

- new F3.1.2 binary + switch LEGACY;
- switch to LEDGER;
- switch back to LEGACY on same binary;
- binary rollback while pending LEDGER reservations exist;
- binary rollback after ledger submissions are completed;
- forward redeploy after fail-closed old-binary behavior.

If rollback is operationally forbidden even though old binary fails closed, explain whether the restriction is for correctness, availability, or both.

---

## G16 — No migration decision

Confirm the accepted F3.1.0 schema already contains all required durable facts.

Reject any new column/table proposed only for convenience.

If transaction/replay review reveals a missing durable binding or state, then `Migration required: NO` is wrong and the plan must be rejected/revised.

---

## G17 — Error contract

Review exact failure semantics for:

- validation;
- payload/idempotency conflict;
- principal not found;
- Catalog unavailable;
- policy/config invariant;
- emergency same-person;
- persistence failure;
- provider failure;
- recovery.

The public HTTP contract must remain provider-neutral.

Decide whether same-person SoD is correctly represented by an existing error code plus stable reason, or deserves a narrow new code. The plan must not leave that to implementation taste.

---

## G18 — Dependency wiring boundary

Confirm the plan injects an authorization submission capability cleanly into the service without:

- global mutable state;
- direct ad hoc Catalog calls in `ChangeManagementService`;
- re-reading config per request;
- letting frontend choose policy/mode;
- coupling selector configuration to execution/Delivery.

Review whether passing the entire `AuthorizationRuntime` is the right service dependency or whether a narrower immutable submission-facing interface is materially safer/easier to test.

Do not create abstraction for aesthetics; require only what protects the boundary.

---

## G19 — Testability and proof

Verify the proposed test matrix can prove the architecture.

Must include:

- F3.1.2a single-build/recovery;
- stored-mode/cutover contract chosen by G4;
- SQLite + disposable PostgreSQL;
- caller-owned transaction behavior for ledger/index/idempotency/provider;
- failure injection at each crash boundary;
- external fake provider orphan/recovery;
- Round 1 uniqueness;
- same-person SoD;
- policy/bundle drift before vs after Round commit;
- legacy regressions;
- architecture guards;
- lint/build;
- set-identical TypeScript baseline.

If a key invariant cannot be tested with the proposed interfaces, require the plan to change.

---

## G20 — Scope discipline / future-slice boundary

Confirm F3.1.2 does not absorb:

- decision commands;
- CAB authority-member authorization;
- new-round/resubmission decisions;
- additional mandatory requirements unless separately authorized;
- governance read models;
- RBAC role expansion;
- Teams/CAB Workbench;
- execution eligibility;
- Delivery correlation;
- SLA scheduler;
- generic workflow engine.

Round 1 creation must leave lifecycle `submitted`.
It must not imply `AUTHORIZED`.

---

# 6. Mandatory challenge review

The reviewer must independently answer these before deciding:

1. Does the plan really need to change repository mode-mismatch semantics, or can service orchestration preserve both “stored mode wins” and explicit mismatch fail-closed behavior?
2. Can the actual ledger repository participate in the caller-owned transaction the plan relies on?
3. If not, what exact API/file change is required and in which micro-slice?
4. Is `requirementRole` guaranteed unique enough to be the canonical requirement ID? Where is that invariant enforced?
5. Can a provider side effect exist while a retry later binds a newer authorization policy before Round commit? Is that acceptable?
6. Can a LEDGER index ever finalize if Round insertion silently rolled back?
7. What does an old `188d8e9` binary actually do when it sees an existing pending `LEDGER_REQUIRED` reservation?
8. Is the proposed rollback prohibition a correctness requirement or merely an availability/runbook choice?
9. Is `newSubmissionAuthorizationMode` located in the right config boundary?
10. Is there any crash window where a retry can regenerate a different canonical Change?
11. Is there any concurrency path that can produce duplicate Round 1 audit events?
12. Is no migration still correct after all replay/rollback questions are resolved?
13. Does completion mean exactly the same thing across idempotency, index, ledger and provider?
14. Can completed idempotency ever coexist with missing Round 1 under the proposed transaction model?
15. Does the plan leave any unresolved “implementation choice” that changes persisted semantics?

---

# 7. Decision rules

## ACCEPT

Return `ACCEPT` only if:

- G1–G20 all PASS;
- the three critical issues are fully resolved:
  1. idempotency mode contract;
  2. real transaction participation;
  3. deterministic requirement ID;
- rollback threat model matches actual source behavior;
- no architecture decision remains deferred to implementation;
- two-slice decomposition is sufficient;
- no ADR change is required.

An ACCEPT may authorize **authoring a separate F3.1.2a implementation prompt**.

It does not authorize code execution by itself.

## REJECT

Return `REJECT` if any correctness/contract issue remains.

For REJECT, provide:

- exact blocker;
- exact plan section affected;
- smallest required plan correction;
- whether the plan can be revised without changing ADR-009/012;
- keep all F3.1.2 implementation NO-GO.

Do not repair ADO code.

---

# 8. Required output document

Write:

`docs/backstage/f3-1-2-plan-architecture-review.md`

Required structure:

1. Status / verdict
2. Reviewed docs + ADO baselines
3. Independent source-verification scope
4. G1–G20 matrix
5. Critical decision: idempotency mode semantics
6. Critical decision: transaction participation
7. Critical decision: requirement identity
8. Rollback/cutover review
9. External-provider recovery review
10. Findings / required corrections
11. Decision
12. Next gate
13. STOP

If ACCEPT, include exactly:

```text
F3.1.2 plan architecture review: ACCEPT
F3.1.2 plan: ACCEPTED IMPLEMENTATION CONTRACT
F3.1.2a implementation prompt authoring: GO
F3.1.2a implementation: NO-GO until separate explicit authorization
F3.1.2b implementation: NO-GO
```

If REJECT:

```text
F3.1.2 plan architecture review: REJECT
F3.1.2 plan: REVISION REQUIRED
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO
```

---

# 9. Canonical documentation updates

After the review:

- create `docs/backstage/f3-1-2-plan-architecture-review.md`;
- update `docs/backstage/current-state.md`;
- append a concise checkpoint to `docs/backstage/implementation-progress.md`;
- update `prompts/README.md`.

Do not rewrite the original planning document silently.

If ACCEPT and minor wording clarifications are useful, record them in the review document rather than mutating history unless the review identifies a factual contradiction that must be corrected before implementation. Any material plan correction means REJECT + revision checkpoint.

If ACCEPT:

- mark this review prompt completed;
- set next authorized activity to **authoring the constrained F3.1.2a implementation prompt**;
- do not create that prompt in this checkpoint.

If REJECT:

- set next activity to **F3.1.2 plan revision only**;
- do not create an implementation prompt.

Commit documentation only.

---

# 10. Final report contract

Return:

```text
Docs baseline reviewed: <sha>
ADO baseline verified: <sha>
ADO branch tip: <sha>
Independent source verification: YES | PARTIAL | NO
Architecture gates: <N>/20 PASS
Critical decisions resolved: <N>/3
F3.1.2 plan architecture review: ACCEPT | REJECT
F3.1.2a implementation prompt authoring: GO | NO-GO
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO
ADO implementation modified: NO
Canonical review: docs/backstage/f3-1-2-plan-architecture-review.md
Final docs SHA: <sha>
```

Then summarize only material architecture findings and required follow-ups.

---

# 11. STOP

STOP immediately after committing the review documentation.

Do not:

- modify ADO source;
- implement F3.1.2a;
- implement F3.1.2b;
- fix `buildChange()`;
- change idempotency code;
- add transaction support;
- add config;
- wire `POST /changes`;
- create AuthorizationRound code;
- create the F3.1.2a implementation prompt;
- start F3.1.3/F3.1.4;
- resume Delivery/production rollout work.

The sole purpose of this checkpoint is to decide whether the current F3.1.2 plan is an acceptable implementation contract.
