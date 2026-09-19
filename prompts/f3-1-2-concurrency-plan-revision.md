# F3.1.2 — Narrow Plan Revision: Concurrent Round 1 Convergence

## Status

PLANNING / DOCUMENTATION REVISION ONLY — NO IMPLEMENTATION AUTHORIZED.

The latest re-review returned REJECT with 18/20 gates PASS. The four original architecture blockers are closed. The only remaining blocker is the ambiguous behavior of the losing worker when two identical logical submissions concurrently reach Round 1 finalization.

Canonical inputs:
- docs/backstage/f3-1-2-implementation-plan.md
- docs/backstage/f3-1-2-plan-architecture-review.md
- docs/backstage/f3-1-2-revised-plan-architecture-rereview.md
- docs/backstage/current-state.md
- docs/backstage/implementation-progress.md
- prompts/README.md

Accepted ADO implementation baseline:
platform-devops-developer-portal@188d8e9cc43423f3644b3cacfb9849257838a583
branch feat/ado-repo-governance.

## Objective

Revise only the concurrency contract and its proof. Do not redesign F3.1.2 and do not reopen any architecture decision already accepted by the latest re-review.

The final plan must define deterministic convergence when two requests have:
- same actor
- same Idempotency-Key
- same payload
- same logical changeId
- stored authorization mode LEDGER_REQUIRED

Both workers may evaluate policy and resolve principals before commit. If worker A commits first and worker B loses at the Round 1 uniqueness / serialization / locking boundary, B must not arbitrarily return CONFLICT or INTERNAL_ERROR merely because it lost the healthy race.

## Mandatory fresh-baseline procedure

1. Fetch latest main from diegofernandes-dev/backstage-docs and record exact SHA.
2. Read all canonical inputs above.
3. If ADO source is available, verify the current feat/ado-repo-governance tip and classify drift touching create/idempotency/ledger/concurrency surfaces.
4. If material source drift exists, STOP with BLOCKED_BY_SOURCE_DRIFT.
5. If ADO source is unavailable, explicitly state that limitation. Do not invent new source facts. This docs-only correction may still proceed because the concurrency ambiguity is already established canonically.

## Strict boundary

Allowed:
- documentation-only edits
- changes to the concurrency/replay/test parts of the F3.1.2 plan
- minimal consistency edits elsewhere that are directly required by the concurrency contract
- current-state, implementation-progress and prompts README updates

Forbidden:
- ADO code changes
- F3.1.2a or F3.1.2b implementation
- changing repository mode-mismatch semantics
- changing the caller-owned transaction decision
- changing requirementId = requirementRole
- changing rollback policy
- adding migrations, routes, runtime wiring, queues, locks or workers
- authoring an implementation prompt
- starting F3.1.3/F3.1.4
- changing ADR-009 or ADR-012

## Frozen decisions that MUST remain unchanged

1. Explicit authorization-mode mismatch at repository level remains CONFLICT.
2. Stored-mode-wins is service orchestration for existing reservations.
3. newSubmissionAuthorizationMode applies only to genuinely new reservations.
4. One canonical Change snapshot.
5. Round 1 + requirements + required audit + index finalize + idempotency complete share one caller-owned platform transaction.
6. DevelopmentProvider joins that transaction.
7. External provider remains outside the platform transaction and converges by idempotent create/orphan retry.
8. requirementId is exactly requirementRole.
9. requirementRole uniqueness is validated at policy registration/publication.
10. No migration.
11. Rollback to a pre-F3.1.2 binary with pending LEDGER_REQUIRED reservations is forbidden by mandatory runbook correctness gate.
12. Exactly two implementation slices.
13. No F3.1.3/F3.1.4 behavior.

If fixing concurrency requires changing any frozen decision, STOP with BLOCKED_ARCHITECTURE_CONTRADICTION.

## Required healthy-race contract

For two workers A and B processing the same logical submission:

A commits the platform transaction first.

B reaches the same finalization region and loses due to the expected same-logical-submission concurrency boundary, such as Round 1 unique conflict or equivalent transaction serialization/locking conflict.

B MUST:

1. Roll back its own transaction.
2. Re-read committed state outside the rolled-back transaction.
3. Validate all of the following:
   - idempotency reservation exists
   - reservation.changeId equals the expected changeId
   - reservation.authorizationMode is LEDGER_REQUIRED
   - reservation.state is completed
   - finalized index exists for the same changeId
   - index authorization mode is LEDGER_REQUIRED
   - Round 1 exists for the same changeId and roundNumber 1
   - any already-available immutable routing/snapshot invariants exposed by the accepted repository contracts remain coherent
4. If all committed facts are coherent, return the same logical success result as the winner: same changeId, status submitted.
5. Never create Round 2.
6. Never mutate the winner's Round.
7. Never return CONFLICT for this healthy same-payload race.
8. Never return INTERNAL_ERROR merely because this worker lost the expected race.

## Transient winner-not-yet-observable rule

If the database exposes a transient locking / serialization / busy condition and the winner is not yet observable:

- rollback the local transaction
- perform only a bounded immediate coherence re-read when safe
- if the winner is now committed and coherent, return idempotent success
- otherwise use the existing retryable storage/provider-unavailable error semantics
- client retry must converge through the normal idempotency flow

Do not add polling loops, sleeps, distributed locks, in-memory mutexes, worker queues, background reconcilers or new durable state.

PostgreSQL is the authoritative concurrency proof. SQLite coverage is desirable where meaningful, but the plan must not require identical driver errors or scheduling semantics across both dialects.

## True invariant mismatch rule

Fail closed only when committed facts are genuinely contradictory, for example:
- completed reservation points to a different changeId
- stored mode is not LEDGER_REQUIRED
- completed reservation exists but finalized index is missing
- finalized ledger index exists but Round 1 is missing
- index mode disagrees with reservation mode
- Round identity disagrees with the expected changeId / roundNumber
- another accepted invariant check fails

These are INTERNAL_ERROR / invariant failures, not idempotency CONFLICT.

The plan must explicitly separate a healthy concurrency loser from a true invariant mismatch.

## Different-payload concurrent rule

For same actor + same Idempotency-Key + different payload:

- one logical reservation wins
- conflicting payload returns CONFLICT
- conflicting payload must not reach policy/Catalog authorization work
- no second Change, provider record or AuthorizationRound is created
- no Round-race recovery path applies to a payload mismatch

## Required plan edits

Update the F3.1.2 implementation plan consistently, at minimum:
- submission state machine if needed
- transaction/crash recovery section
- idempotent replay and concurrency section
- error contract if needed
- test matrix
- challenge-scenario answers
- implementation acceptance criteria
- explicit decisions appendix if useful

Remove any healthy-race wording equivalent to:
- loser may succeed or fail closed
- either return success or error
- implementation may choose
- best effort
- retry as needed

No semantic choice may remain for the same-key/same-payload healthy race.

## Mandatory test contract

C1 — identical concurrent requests, authoritative PostgreSQL integration:
- same actor
- same Idempotency-Key
- same payload
- two createChange calls started concurrently
Expected:
- both callers converge to the same successful logical result
- same changeId
- status submitted
- exactly one idempotency reservation, completed
- exactly one finalized index
- exactly one Round 1
- exactly one effective requirement set
- exactly one canonical submission-audit set
- exactly one DevelopmentProvider operational record
- no Round 2
- no duplicate requirements
- no duplicate canonical audit events from the loser

C2 — deterministic finalization race:
Use barriers/hooks/failure injection where practical so both workers reach the finalization region. Do not rely only on timing sleeps. Prove the loser transaction rolls back before coherence re-read and cannot leave partial provider/Round/audit/index/idempotency state on the DevelopmentProvider shared-transaction path.

C3 — concurrent different payload:
Same actor and key, different payload. One wins; the other returns CONFLICT before authorization evaluation. No second Change/Round/provider record.

C4 — invariant corruption negative:
Using a controlled fixture, prove loser recovery does not convert a genuine committed inconsistency into success. Example: completed reservation with missing Round 1 or mismatching index authorization mode. Expected fail closed under existing invariant error contract.

Keep the existing database uniqueness test, but uniqueness alone is not sufficient proof of end-to-end idempotent convergence.

## Required output status

If the final ambiguity is completely removed:

F3.1.2 concurrency plan revision: READY_FOR_REREVIEW
F3.1.2 implementation: NO-GO
F3.1.2a implementation prompt authoring: NO-GO pending fresh ACCEPT
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO

Otherwise:
F3.1.2 concurrency plan revision: BLOCKED
F3.1.2 implementation: NO-GO

Do not declare the plan accepted. Only a separate fresh architecture re-review can ACCEPT it.

## Canonical documentation updates

After revising:
1. update docs/backstage/f3-1-2-implementation-plan.md
2. update docs/backstage/current-state.md
3. append a concise checkpoint to docs/backstage/implementation-progress.md
4. update prompts/README.md:
   - move this prompt to completed/historical
   - next authorized activity = focused fresh architecture re-review of the concurrency-corrected plan
   - implementation prompt authoring remains NO-GO until that review returns ACCEPT

Preserve both historical REJECT review documents.

Commit documentation only.

## Final report contract

Return:

Docs baseline reviewed: <sha>
ADO baseline relied upon: <sha>
ADO branch tip independently verified: YES | NO
F3.1.2 concurrency plan revision: READY_FOR_REREVIEW | BLOCKED | BLOCKED_BY_SOURCE_DRIFT | BLOCKED_ARCHITECTURE_CONTRADICTION
Healthy Round-1 loser contract deterministic: YES | NO
Same-payload concurrency proof specified: YES | NO
Different-payload concurrency proof specified: YES | NO
True-invariant negative proof specified: YES | NO
Original closed architecture decisions reopened: NO
F3.1.2 implementation: NO-GO
ADO implementation modified: NO
Canonical revised plan: docs/backstage/f3-1-2-implementation-plan.md
Final docs SHA: <sha>

Then summarize only the exact concurrency rule added, the test-proof additions, any source-verification limitation, and the next review gate.

## STOP

STOP after committing documentation.

Do not implement code, do not create an implementation prompt, do not start F3.1.3/F3.1.4, and do not reopen Delivery or production-rollout work.

The next checkpoint after READY_FOR_REREVIEW is one focused independent architecture re-review of the concurrency-corrected F3.1.2 plan.
