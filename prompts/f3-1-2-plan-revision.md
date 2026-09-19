# F3.1.2 — Plan Revision After Architecture REJECT

## Status

**PLANNING / DOCUMENTATION REVISION ONLY — NO IMPLEMENTATION AUTHORIZED.**

The independent architecture review of the first F3.1.2 plan returned:

```text
F3.1.2 plan architecture review: REJECT
F3.1.2 plan: REVISION REQUIRED
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO
```

Canonical review:

`docs/backstage/f3-1-2-plan-architecture-review.md`

The purpose of this checkpoint is to revise:

`docs/backstage/f3-1-2-implementation-plan.md`

so it embodies the review's accepted architecture decisions exactly and is internally consistent enough for a fresh independent re-review.

Do not implement ADO code.

---

## 1. Authority and baseline

Before editing:

1. Fetch the latest `main` from `diegofernandes-dev/backstage-docs`.
2. Record the exact docs SHA.
3. Read in full:
   - `docs/backstage/f3-1-2-plan-architecture-review.md`
   - `docs/backstage/f3-1-2-implementation-plan.md`
   - `docs/backstage/f3-1-1b-architecture-acceptance.md`
   - `docs/backstage/f3-1-1a-architecture-acceptance.md`
   - `docs/backstage/f3-1-implementation-plan.md`
   - `docs/adr/ADR-006-change-management-backend-contract.md`
   - `docs/adr/ADR-007-change-record-authority.md`
   - `docs/adr/ADR-008-multi-activity-change-execution-plan.md`
   - `docs/adr/ADR-009-change-authorization-model.md`
   - canonical ADR-012
   - `docs/backstage/current-state.md`
   - `docs/backstage/implementation-progress.md`
   - `prompts/README.md`
   - this prompt.
4. Re-verify the actual ADO baseline and current branch tip:
   - repo: `platform-devops-developer-portal`
   - branch: `feat/ado-repo-governance`
   - accepted baseline: `188d8e9cc43423f3644b3cacfb9849257838a583`
5. If the ADO branch has advanced on the F3.1.2 surface, stop and report the drift instead of revising a stale plan.

The independent review decisions below are **normative for this revision**. Do not reopen them unless fresh source evidence directly contradicts the review.

---

## 2. Strict boundary

You MAY:

- inspect ADO source read-only;
- run non-mutating source/tests for verification;
- edit canonical documentation;
- revise the existing F3.1.2 plan;
- update current-state, construction progress, and prompt index.

You MUST NOT:

- modify `platform-devops-developer-portal`;
- fix `buildChange()`;
- change repository idempotency behavior;
- add transaction code;
- add config;
- add migrations/routes;
- wire `POST /changes`;
- create AuthorizationRounds;
- create the F3.1.2a implementation prompt;
- start F3.1.3/F3.1.4;
- alter ADR-009/ADR-012 to make the plan fit.

---

# 3. Mandatory revision decisions

The revised plan must embody every decision below as a single unambiguous contract.

## R1 — Preserve repository mode-mismatch CONFLICT

Do **not** change the accepted repository invariant.

The revised plan must state:

```text
KnexIdempotencyRepository.reserve:
- payload mismatch => CONFLICT
- explicit authorizationMode mismatch on an existing reservation => CONFLICT
- stored authorization_mode is immutable
```

Delete all plan language proposing to:

- ignore a requested mode mismatch;
- replace the F3.1.0-V mismatch-CONFLICT test;
- weaken repository fail-closed behavior.

### Required service orchestration

`newSubmissionAuthorizationMode` is a **new-reservation default only**.

For an already existing logical reservation, the service must use the stored mode and must not re-request the deployment's current default.

The revised plan must define a race-safe algorithm, including the concurrent-first-insert case.

At minimum, define behavior equivalent to:

```text
desiredMode = config.newSubmissionAuthorizationMode
existing = idempotency.find(lookup)

if existing:
  verify payload using repository/reserve semantics
  reserve/recover using existing.authorizationMode or omit the mode
  mode = stored mode
else:
  attempt first reservation with desiredMode
  if this process wins:
    mode = desiredMode
  if another process wins the unique race:
    re-read the winner's reservation
    verify same payload
    continue using winner.authorizationMode
```

Do not turn a rolling-deploy race between two different defaults into a false ordinary-retry conflict.

The DB unique key remains the correctness arbiter.

---

## R2 — Caller-owned transaction is mandatory on the LEDGER finalize path

Correct the plan's source inventory to record:

- `createRound(..., trx?)` already supports a caller-owned transaction;
- `appendAuditEvent(..., trx?)` already supports a caller-owned transaction;
- omitting `trx` uses an independent/default database execution path and must not be used by the F3.1.2b finalize flow;
- `index.finalize(..., trx)`, `idempotency.complete(..., trx)`, and `DevelopmentProvider.createWithTransaction(...)` participate in the platform create transaction.

The revised implementation contract must state exactly:

```text
Inside ONE caller-owned platform transaction (trx):

DevelopmentProvider path:
  provider.createWithTransaction(trx, canonicalChange)
  ledger.createRound(round1, trx)
  ledger.appendAuditEvent(event1, trx)
  ...
  index.finalize(..., trx)
  idempotency.complete(..., trx)
```

For an external provider:

```text
provider.create(canonicalChange)   # outside platform DB transaction
then ONE platform transaction:
  ledger.createRound(round1, trx)
  ledger.appendAuditEvent(..., trx)
  index.finalize(..., trx)
  idempotency.complete(..., trx)
```

No 2PC/XA claim.

### Visibility authority

Delete wording where a non-transactional `findRound()` “check-then-finalize” is the authority for visibility.

For a new LEDGER submission, the authority is:

> Round 1 + requirements + required submission audit + index finalization + idempotency completion commit together in the same platform transaction.

A committed pre-existing Round may be read during recovery, but a fresh finalize must never rely on an out-of-transaction existence check as the thing that makes finalization safe.

---

## R3 — Correct the crash matrix to match the transaction contract

The original crash matrix contains states that become impossible when Round + finalize + idempotency.complete truly commit in one transaction.

Rewrite the crash/recovery matrix so it distinguishes:

### Before platform transaction commit
Nothing from Round/finalize/complete is durable.

### After platform transaction commit
Round 1, requirements, audit, finalized index, and completed idempotency are durable together.

For DevelopmentProvider, provider state joins that same transaction.

For an external provider, provider side effect may exist before the platform transaction and is recovered through idempotent `create(changeId)` + orphan/retry convergence.

Remove or correct any scenario equivalent to:

> “Round 1 + finalized index committed, then crash before idempotency.complete”

if the normal F3.1.2 path places all of those writes in the same transaction.

The plan must not teach an implementation agent an impossible intermediate state.

---

## R4 — Lock requirement identity to one scheme

The revised plan must state exactly:

```text
requirementId = requirementRole
```

No hash fallback.
No dual scheme.
No “pick during implementation”.

Add a hard policy invariant:

> Within the effective requirements of any published rule, `requirementRole` is unique.

The plan must require fail-closed validation at policy registration/publication/startup before submission can use such a policy.

Update the expected F3.1.2b change surface to include the relevant policy registry/validation implementation and tests.

Tests must prove:

- duplicate `requirementRole` in one rule is rejected before runtime submission;
- Round 1 maps `requirementId` deterministically to `requirementRole`;
- retry-before-commit cannot choose another identity scheme.

---

## R5 — Make rollback policy explicit: runbook correctness gate

Resolve the rejected plan's “optional hard guard” ambiguity.

For this F3.1.2 scope, choose the smallest control that actually addresses the verified threat:

**The rollback compatibility control is a mandatory operational/runbook correctness gate, not an optional code guard.**

Reason:

- the dangerous binary is the **older pre-F3.1.2 binary**;
- adding a guard only to the new F3.1.2 binary does not make the old binary understand pending `LEDGER_REQUIRED`;
- therefore the safe control at this slice is to forbid rollback past F3.1.2 while pending ledger reservations exist.

The revised plan must state:

```text
Before rollback to a pre-F3.1.2 binary:

SELECT COUNT(*)
FROM change_idempotency
WHERE authorization_mode = 'LEDGER_REQUIRED'
  AND state = 'pending';

Required result: 0
```

Classification: **correctness gate**.

The plan may note that future deployment automation can enforce this operational check, but such automation is outside F3.1.2 unless already present and directly in scope.

Delete “optional hard guard” language.

---

## R6 — Preserve the accepted old-binary threat model

Retain the review's verified behavior:

- `188d8e9` calls reserve without explicitly requesting a mode;
- an existing `LEDGER_REQUIRED` reservation is therefore returned, not rejected for mode mismatch;
- the old service has no ledger branch;
- it can finalize the Change without Round 1.

Do not regress to the incorrect claim that the old binary necessarily fails with mode conflict.

This is why R5 is a correctness gate.

---

## R7 — Keep two slices; do not invent a third one

The review accepted:

### F3.1.2a
single canonical Change construction and recovery reuse only.

### F3.1.2b
new-submission cutover + authorization runtime wiring + policy/selector + SoD + Round 1 + requirements + audit + transaction/recovery.

Keep exactly those two slices unless fresh source drift proves otherwise.

The ledger transaction API already exists; it is **not** a third prerequisite slice.

F3.1.2b must explicitly include:

- caller-owned `trx` usage;
- policy `requirementRole` uniqueness validation;
- service-orchestrated new-reservation mode selection;
- rollback runbook evidence.

---

# 4. Required consistency sweep

Do not patch only the four paragraphs cited by the review.

Perform a whole-document consistency sweep of:

`docs/backstage/f3-1-2-implementation-plan.md`

At minimum update/reconcile:

- header/status and revision history;
- §2 source inventory;
- §5 state machine;
- §6 cutover;
- §10 requirement mapping;
- §11 transaction/crash matrix;
- §12 visibility;
- §13 replay/concurrency;
- §15 wiring if needed;
- §19 rollback;
- §20 tests;
- §21 expected source paths;
- §22 slice decomposition;
- §23 rejected alternatives;
- §24 challenge answers;
- §25 acceptance criteria;
- §26 gate wording;
- Appendix A decisions;
- Appendix B references if section numbering changes.

After revision, search the document for stale concepts. There must be no surviving architecture text that says or implies:

- repository mode-mismatch CONFLICT should be removed;
- “replace old CONFLICT test”;
- “pick one requirement ID scheme during implementation”;
- runtime fallback from `requirementRole` to hash;
- “optional hard guard”;
- non-transactional `findRound` check is the authority allowing finalize;
- Round/finalize can commit normally while idempotency.complete remains uncommitted under the same platform transaction.

Historical prose describing the **rejected first plan** may exist only in a clearly labeled revision-history section; it must not read as current architecture.

---

# 5. Revised test contract

Correct the test matrix so it proves the revised architecture.

At minimum include:

## Idempotency/cutover

- repository explicit mode mismatch remains `CONFLICT`;
- existing LEGACY reservation resumes legacy after config changes to LEDGER;
- existing LEDGER reservation resumes ledger after config changes to LEGACY;
- new reservation gets current config mode;
- concurrent first insert across different desired-mode processes uses the DB winner's stored mode;
- payload mismatch remains CONFLICT before policy/Catalog work.

## Transaction

- all DevelopmentProvider ledger-submit writes use the same caller-owned `trx`;
- injected failure after Round insert but before commit leaves **no** Round/finalize/complete/provider mutation durable;
- injected failure after audit append but before finalize leaves the transaction fully rolled back;
- successful commit makes Round/requirements/audit/finalize/complete visible together;
- tests/architecture guard prevent calling `createRound` / `appendAuditEvent` without the outer `trx` on this path.

## Requirement identity

- duplicate role in a published rule fails policy registration/publication;
- `requirementId === requirementRole`;
- deterministic materialization.

## Rollback

- runbook query / operational evidence is part of implementation acceptance;
- old-binary behavior is captured as a compatibility regression/test or documented source proof;
- no claim that a pre-F3.1.2 binary safely understands pending ledger reservations.

Retain SQLite + disposable PostgreSQL, external fake-provider recovery, SoD, policy/bundle replay, regressions, lint/build, architecture guards and set-identical TypeScript baseline.

---

# 6. Expected source-path plan correction

The revised plan must correct the expected F3.1.2b surface.

Do **not** claim `KnexIdempotencyRepository` semantics must be weakened.

Include only source changes actually implied by the revised contract.

Expected areas should include, as appropriate after source verification:

- `ChangeManagementService.ts`
- `changeManagementPlugin.ts`
- authorization config/types
- policy registry/publication validation for role uniqueness
- policy registry/publication tests
- architecture guards
- ledger-submit service tests
- idempotency/cutover orchestration tests
- existing ledger repository may be **used with trx without implementation change** if source still matches the review
- app config default remains `LEGACY_PRE_F3`

Be explicit about whether `KnexIdempotencyRepository.ts` changes at all. Under the accepted review decision, its explicit mismatch-CONFLICT semantics should remain intact; service orchestration may make a repository code change unnecessary.

---

# 7. Required revised-plan status

At the top of the revised plan, record that:

- the first plan was rejected at docs commit `0daa8fc0719bafd4d8b1e2f95c1ad0abeaa03422`;
- this is a revised candidate incorporating the review;
- implementation remains NO-GO.

If and only if every rejected gate is concretely corrected, end the revised plan with:

```text
F3.1.2 revised planning: READY_FOR_REREVIEW
F3.1.2 implementation: NO-GO
F3.1.2a implementation prompt authoring: NO-GO pending re-review ACCEPT
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO
```

Do not declare the plan accepted. Only a new independent review may do that.

---

# 8. Canonical documentation updates

After revising the plan:

1. Update `docs/backstage/current-state.md`:
   - F3.1.2 plan revised / READY_FOR_REREVIEW if complete;
   - implementation still NO-GO.
2. Append a concise revision checkpoint to `docs/backstage/implementation-progress.md`.
3. Correct its stale top header if it still says the F3.1.2 plan is merely `READY_FOR_REVIEW`; it must reflect the latest REJECT → revised-candidate state.
4. Update `prompts/README.md`:
   - move this revision prompt to completed/historical;
   - next activity = **fresh independent F3.1.2 plan re-review**;
   - implementation prompt authoring remains NO-GO until re-review ACCEPT.

Do not delete or rewrite the rejected review document. It remains audit history.

Commit documentation only.

---

# 9. Final report contract

Return:

```text
Docs baseline reviewed: <sha>
ADO baseline verified: <sha>
ADO branch tip: <sha>
F3.1.2 plan revision: READY_FOR_REREVIEW | BLOCKED
Rejected gates corrected: <N>/8
Critical review decisions embodied: <N>/4
Repository mode-mismatch semantics changed: NO
Caller-owned transaction made normative in plan: YES | NO
requirementId scheme: requirementRole | unresolved
Rollback control: RUNBOOK_CORRECTNESS_GATE | unresolved
F3.1.2 implementation: NO-GO
ADO implementation modified: NO
Canonical revised plan: docs/backstage/f3-1-2-implementation-plan.md
Final docs SHA: <sha>
```

For `Critical review decisions embodied`, count:

1. service-orchestrated stored-mode semantics while repository mismatch remains fail-closed;
2. mandatory caller-owned transaction;
3. `requirementId = requirementRole` + uniqueness validation;
4. explicit runbook correctness gate for rollback.

---

# 10. STOP

STOP after committing the revised planning documentation.

Do not:

- implement F3.1.2a;
- implement F3.1.2b;
- change ADO source;
- fix `buildChange()`;
- change repository mode-mismatch semantics;
- add runtime transaction wiring;
- add config;
- create Round 1;
- create an implementation prompt;
- start F3.1.3/F3.1.4;
- resume Delivery/production rollout work.

The next checkpoint after `READY_FOR_REREVIEW` is a **fresh independent architecture re-review of the revised F3.1.2 plan**.
