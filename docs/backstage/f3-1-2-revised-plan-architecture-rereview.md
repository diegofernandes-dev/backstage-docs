# F3.1.2 — Revised Plan Architecture Re-Review

## 1. Status / verdict

```text
F3.1.2 revised-plan architecture re-review: REJECT
F3.1.2 revised plan: REVISION REQUIRED
F3.1.2a implementation prompt authoring: NO-GO
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO
```

The revision correctly closes the four blockers from the first architecture
review. The plan is materially stronger and directionally correct. However, one
remaining concurrency contract is still ambiguous enough to prevent the plan
from being an implementation-ready idempotency contract:

> when two workers concurrently reach the Round 1 platform transaction for the
> same logical submission, the plan currently allows the loser to “re-read
> completed/finalized state **or** fail closed on invariant mismatch” without
> defining the exact convergence rule.

That ambiguity also leaves the test matrix one step short: it proves uniqueness
of Round 1, but not that two concurrent same-key/same-payload submissions
converge to the same successful logical result with exactly one Round/audit set.

No ADO implementation was modified by this re-review.

---

## 2. Reviewed baselines and evidence limits

| Item | Value |
|---|---|
| Docs baseline reviewed | `diegofernandes-dev/backstage-docs@3bea2459b5afaa40396ac673e17d07197fd32359` |
| Revised plan | `docs/backstage/f3-1-2-implementation-plan.md` at revision commit `6284195a41bb428862c057ee9c0291014ccba61d` |
| Historical REJECT | `docs/backstage/f3-1-2-plan-architecture-review.md` at `0daa8fc0719bafd4d8b1e2f95c1ad0abeaa03422` |
| ADO accepted implementation baseline | `platform-devops-developer-portal@188d8e9cc43423f3644b3cacfb9849257838a583` |
| ADO current branch tip | **NOT independently re-verified in this ChatGPT checkpoint** |
| Independent source verification | **PARTIAL** — this re-review relies on the immediately preceding independent source review that inspected exact ADO `188d8e9`; no Azure DevOps connector is available in this environment |

The source-verification limitation is explicit and is not the reason for the
REJECT. The blocker below is visible in the revised canonical plan itself.

---

## 3. Regression review of the four prior blockers

| Prior blocker | Re-review | Result |
|---|---|---|
| Repository mode-mismatch semantics | Revised plan keeps explicit `authorizationMode` mismatch as repository `CONFLICT` and moves stored-mode-wins to service orchestration | **PASS** |
| Caller-owned transaction | Revised plan explicitly passes one outer `trx` to DevelopmentProvider, `createRound`, every audit append, index finalize, and idempotency complete | **PASS** |
| Requirement identity | Revised plan locks `requirementId = requirementRole` and adds fail-closed per-rule uniqueness at policy registration/publication | **PASS** |
| Rollback control | Revised plan removes optional-guard language and makes zero pending `LEDGER_REQUIRED` a mandatory runbook correctness gate before pre-F3.1.2 binary rollback | **PASS** |

The corrected crash matrix, visibility authority, no-migration decision, Model C
external-provider convergence, and two-slice decomposition also remain coherent.

---

## 4. G1–G20 re-review matrix

| Gate | Result | Re-review finding |
|---|---|---|
| G1 Baseline/source reality | **PASS*** | Revised inventory matches the facts established by the prior exact-source review. *Current ADO tip not reverified here; evidence is PARTIAL.* |
| G2 Two-slice decomposition | **PASS** | F3.1.2a canonical snapshot then F3.1.2b ledger submission remains sufficient. |
| G3 Single canonical Change | **PASS** | One build, durable pending snapshot, reuse on recovery. |
| G4 Idempotency cutover semantics | **PASS** | Repository mismatch-CONFLICT preserved; service applies deployment default only to genuinely new reservation creation. |
| G5 New-submission mode cutover | **PASS** | Explicit config pin, default LEGACY, existing reservations unaffected. |
| G6 Earliest durable authorization binding | **PASS** | Round platform-transaction commit remains the first durable authorization binding; pre-Round rebind remains invisible/reconcilable. |
| G7 Policy/selector binding | **PASS** | Immutable runtime objects per attempt; no reinterpretation after commit. |
| G8 Emergency SoD | **PASS** | Same effective User ref fails before Round commit; decision-time actor SoD remains later. |
| G9 Requirement identity | **PASS** | Exactly `requirementRole`; uniqueness validated before submission. |
| G10 Round/requirement/audit mapping | **PASS** | Deterministic field sources; no decision/Delivery leakage. |
| G11 Transaction participation | **PASS** | Caller-owned `trx` is normative; same-database is no longer confused with same-transaction. |
| G12 External-provider recovery | **PASS** | No XA; idempotent provider create + orphan/retry convergence. |
| G13 Visibility/finalization | **PASS** | Visibility authority is the shared platform transaction, not `findRound`. |
| G14 Replay / concurrency | **FAIL** | Sequential replay is precise; concurrent Round-1 loser behavior is still expressed as an unresolved alternative rather than an exact convergence contract. |
| G15 Rollback safety | **PASS** | Mandatory correctness gate matches the previously verified old-binary hazard. |
| G16 No migration | **PASS** | Revision introduces no missing durable state requiring schema change. |
| G17 Error contract | **PASS** | Existing codes remain adequate; SoD reason remains stable and provider-neutral. |
| G18 Dependency wiring | **PASS** | Runtime/ledger/config injection remains bounded and provider-neutral. |
| G19 Testability / proof | **FAIL** | T6 proves Round uniqueness but not end-to-end concurrent same-key convergence; no test currently requires both concurrent callers to obtain the same logical successful result. |
| G20 Scope discipline | **PASS** | No F3.1.3/F3.1.4, Delivery, Teams/CAB, or workflow-engine expansion. |

**Architecture gates: 18 / 20 PASS.**

---

## 5. Remaining blocker — concurrent Round 1 convergence

### Current revised-plan text

The concurrency table currently says:

```text
Concurrent Round 1 platform transactions
  → DB uniqueness + one transaction wins;
    loser re-reads completed/finalized state
    or fails closed on invariant mismatch
```

The phrase “or” is not sufficiently precise for an implementation contract.

For the same actor + idempotency key + payload, two simultaneous callers are
the **same logical submission**. Once one worker commits a valid Round 1 and
completes the reservation, the losing worker must not turn an expected
uniqueness race into an arbitrary 500/409 merely because it entered the final
transaction concurrently.

### Required exact contract

The revised plan must define this deterministic behavior:

```text
Worker A and B process same logical submission.

Both may evaluate/resolve before commit.

A commits the platform transaction first.

B attempts Round 1 insert and loses on the expected
(change_id, round_number=1) uniqueness boundary.

B MUST:
  1. roll back its own platform transaction;
  2. re-read the idempotency reservation, finalized index, and Round 1;
  3. if:
       reservation.state == completed
       AND stored mode == LEDGER_REQUIRED
       AND index is finalized with matching mode
       AND Round 1 exists for the same changeId
     then return the same logical success result;
  4. if the winner is not yet observable because the database surfaced a
     transient lock/serialization condition, classify it as retryable according
     to the existing storage failure model; do not create Round 2;
  5. if committed facts disagree (wrong mode/changeId/missing Round after
     finalized completion), fail closed as INTERNAL_ERROR / invariant breach.
```

The uniqueness race is therefore separated from a true invariant mismatch.

The plan must not allow an implementation agent to choose between “return
idempotent success” and “fail closed” for the healthy winner-committed case.

### Why this is architecture, not implementation trivia

Without this rule, two implementations can both claim conformance while exposing
different public/idempotency behavior under the same concurrent request. That is
a persisted-submission semantic, not a local coding choice.

---

## 6. Required test correction

Add an explicit end-to-end concurrency case, for both SQLite where practical and
disposable PostgreSQL as the authoritative concurrency proof:

```text
same actor + same Idempotency-Key + same payload
two createChange calls concurrently

expected:
- same changeId/result for both callers after convergence
- exactly one finalized index
- exactly one Round 1
- exactly one requirement set
- exactly one canonical submission-audit set
- one completed idempotency reservation
- no Round 2
- no policy/principal reinterpretation after winner commit
```

Keep the existing DB-uniqueness test, but do not treat uniqueness alone as proof
of idempotent concurrent request behavior.

Also include a negative companion:

```text
same actor + same Idempotency-Key + different payload concurrently
→ one logical reservation wins; conflicting payload receives CONFLICT;
  authorization work for the conflicting payload does not proceed.
```

---

## 7. Findings that are now closed

The re-review explicitly accepts these previously disputed design choices:

- service pre-read / reserve-with-mode-only-for-new orchestration is a valid way
  to preserve both repository fail-closed semantics and deployment cutover;
- a race loser may recover the winner's stored mode by re-reserving with mode
  omitted when the payload matches;
- the ledger repository does not need a new transaction API;
- no migration is required;
- pre-Round policy/bundle/principal re-resolution remains acceptable because the
  Change is invisible and no canonical authorization facts exist yet;
- `requirementRole` is sufficient as Round-local identity once registry
  uniqueness is enforced;
- rollback safety belongs to the mandatory runbook gate in this slice;
- exactly two implementation slices remain the correct decomposition.

---

## 8. Decision

```text
F3.1.2 revised-plan architecture re-review: REJECT
F3.1.2 revised plan: REVISION REQUIRED
```

This is a **narrow rejection**, not a redesign.

The four original architecture blockers are closed. One remaining concurrency
contract and its proof must be made exact. ADR-009 and ADR-012 do not need to
change.

---

## 9. Next gate

```text
Next authorized activity: F3.1.2 plan revision — concurrent Round 1 convergence only
F3.1.2a implementation prompt authoring: NO-GO
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO
```

After that narrow correction, perform one fresh re-review. The next re-review
should focus primarily on the corrected concurrency contract plus regression of
the four already-closed blockers.

---

## 10. Final report

```text
Docs baseline reviewed: 3bea2459b5afaa40396ac673e17d07197fd32359
Revised plan candidate: backstage-docs@6284195a41bb428862c057ee9c0291014ccba61d
ADO baseline relied upon: 188d8e9cc43423f3644b3cacfb9849257838a583
ADO current tip independently verified in this checkpoint: NO
Independent source verification: PARTIAL
Architecture gates: 18/20 PASS
Original critical blockers closed: 4/4
Remaining blocker: concurrent Round 1 loser convergence
F3.1.2 revised-plan architecture re-review: REJECT
F3.1.2a implementation prompt authoring: NO-GO
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO
ADO implementation modified: NO
```

---

## 11. STOP

STOP.

Do not implement F3.1.2a/F3.1.2b and do not author an implementation prompt
until the concurrency contract is revised and independently accepted.
