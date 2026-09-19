# F3.1.2 — Plan Architecture Review

## 1. Status / verdict

```text
F3.1.2 plan architecture review: REJECT
F3.1.2 plan: REVISION REQUIRED
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO
```

The published plan is directionally aligned with ADR-006/007/008/009/012 and the
F3.1.1b baseline, but it is **not** an implementation-ready contract. Three
architecture-critical decisions remain wrong or deferred inside the plan text,
and two further contract gaps prevent safe execution by a later implementation
agent.

This review is documentation-only. No ADO code was modified.

---

## 2. Reviewed docs + ADO baselines

| Item | Value |
|---|---|
| Docs fetched (`main`) | `diegofernandes-dev/backstage-docs@4b28eb20969dad7b7273464e28ae809f0da87a39` |
| Plan under review | `docs/backstage/f3-1-2-implementation-plan.md` (published planning checkpoint) |
| Plan’s claimed docs baseline | `d65bf5e1446e682576557583b841ddde7e5a890c` |
| Review prompt | `prompts/f3-1-2-plan-architecture-review.md` |
| ADRs | ADR-006, ADR-007, ADR-008, ADR-009, ADR-012 |
| Prior plans / acceptances | F3.1, F3.1.1, F3.1.1a/b acceptance + evidence |
| ADO baseline verified | `platform-devops-developer-portal@188d8e9cc43423f3644b3cacfb9849257838a583` |
| ADO branch tip (F3.1.2 surface) | `origin/feat/ado-repo-governance` = `188d8e9` (no post-baseline drift) |
| Unrelated local tip observed | workspace `feat/delivery-mvp-slice@170af45` — **outside** F3.1.2 review surface; classified and ignored |
| Independent source verification | **YES** (isolated worktree at exact `188d8e9`) |

---

## 3. Independent source-verification scope

Inspected at exact SHA `188d8e9` (worktree `/private/tmp/platform-devops-f311b-review`):

| Surface | Paths |
|---|---|
| Create pipeline | `ChangeManagementService.ts` (`createChange`, `finalizeCreate`, `buildChange`) |
| Idempotency | `KnexIdempotencyRepository.ts`, `idempotencyAuthorizationMode.test.ts`, `IdempotencyRepository.ts` |
| Index | `KnexChangeIndexRepository.ts`, `ChangeIndexRepository.ts` |
| Ledger | `AuthorizationLedgerRepository.ts`, `KnexAuthorizationLedgerRepository.ts` |
| Provider | `IChangeManagementProvider.ts`, `DevelopmentProvider.ts` |
| Policy / selector | published `default-change-authorization@2026-09-02.1`, `registry.ts`, `selector/config.ts`, `bootstrapAuthorization.ts` |
| Guards / plugin | `architecture.test.ts`, `changeManagementPlugin.ts` (wiring inventory via grep) |

No memory-only claims were accepted for transaction, reserve, rollback, or
idempotency behavior.

---

## 4. G1–G20 matrix

| Gate | Result | Summary |
|---|---|---|
| G1 Baseline/source reality | **FAIL** | Inventory mostly accurate, but material omissions/assumptions: (a) ledger already accepts caller-owned `trx?` and self-opens a **separate** txn when omitted; (b) plan treats repository mode-mismatch CONFLICT as something that “must” be removed. |
| G2 Two-slice decomposition | **PASS** | 2a (canonical Change, ledger dark) then 2b (cutover + Round 1) is sufficient. No third prerequisite for ledger txn API — it already exists. Slice *content* of 2b must change (see G4/G9/G11). |
| G3 Single canonical Change | **PASS** | Proposed 2a contract is exact and necessary before Round hashing. |
| G4 Idempotency cutover semantics | **FAIL** | Plan wrongly changes repository fail-closed mode-mismatch into “stored wins / ignore request”. Correct architecture is service orchestration (see §5). |
| G5 New-submission mode cutover | **PASS** | Config pin under `changeManagement.authorization`, default `LEGACY_PRE_F3`, first-insert only is correct. Path `selector/config.ts` is the existing broader authorization config reader (historical filename); not a semantic selector concern if typed on `AuthorizationConfig`. |
| G6 Earliest durable auth binding | **PASS** | Binding at Round 1 commit is acceptable: Change invisible; no canonical auth facts; provider reconcilable by `changeId`; re-bind before Round allowed. |
| G7 Policy/selector binding | **PASS** | One immutable runtime policy + bundle per attempt; freeze on Round; no re-eval after Round 1. |
| G8 Emergency SoD | **PASS** | Resolved effective User refs distinct per `separationOfDutyKey` before Round; not decision-time SoD. |
| G9 Requirement identity | **FAIL** | Plan still says prefer `requirementRole` / else hash / “pick one scheme during implementation”. Not implementation-ready. |
| G10 Round/requirement/audit mapping | **FAIL** | Field mapping is otherwise sound, but blocked by unresolved `requirementId` (G9). |
| G11 Transaction participation | **FAIL** | Capability exists, but plan does not mandate passing the caller-owned `trx` into `createRound` / `appendAuditEvent`. Omitting `trx` opens an independent ledger transaction and **breaks** the claimed atomicity. |
| G12 External-provider recovery | **PASS** | No XA; orphan/retry by `changeId` matches F2 + `IChangeManagementProvider` idempotent-create contract already documented in source. |
| G13 Visibility/finalization | **FAIL** | Intent correct, but “check-then-finalize” via `findRound` (no `trx` overload) is not a same-transaction guarantee. Authority must be: Round insert + finalize + complete in **one** caller-owned txn (depends on G11 correction). |
| G14 Replay after Round 1 | **PASS** | Short-circuit on Round 1 / finalized / completed; DB constraints; no in-memory mutex. |
| G15 Rollback safety | **PASS** | Threat model matches actual `188d8e9` behavior (see §8). Binary rollback forbid while pending `LEDGER_REQUIRED` is a **correctness** requirement. |
| G16 No migration | **PASS** | F3.1.0 schema already holds required durable facts; no gap found after replay/rollback review. |
| G17 Error contract | **PASS** | SoD as existing `CONFLICT` + `details.reason=separation_of_duty` is decided and adequate. |
| G18 Dependency wiring | **PASS** | Inject `AuthorizationRuntime` + ledger + mode enum; no ad-hoc Catalog in service; acceptable boundary. |
| G19 Testability | **FAIL** | Matrix is strong, but I5 / “replace old CONFLICT test” encodes the wrong G4 contract; must also prove caller-owned trx participation (pass `trx`, and that omitting it is forbidden on the LEDGER finalize path). |
| G20 Scope discipline | **PASS** | No decisions/CAB/Teams/Delivery/eligibility/SLA scheduler absorption; Round 1 leaves lifecycle `submitted`. |

**Gates PASS: 12 / 20. Gates FAIL: 8 / 20.**

---

## 5. Critical decision: idempotency mode semantics

### What source does today (`188d8e9`)

`KnexIdempotencyRepository.reserve`:

1. Inserts the requested `authorization_mode` (default `LEGACY_PRE_F3`).
2. On unique conflict, payload mismatch → `CONFLICT`.
3. If caller **explicitly** passes `authorizationMode` and it differs from the
   stored row → `CONFLICT` (`authorization mode mismatch`).
4. If caller **omits** `authorizationMode`, the mode-mismatch check is skipped
   and the existing row (stored mode) is returned.

`ChangeManagementService.createChange` currently always reserves **without**
passing a mode, then copies `reserved.authorizationMode`. There is still no
LEDGER branch; every create finalizes the F2 path.

F3.1.0-V tests intentionally assert repository CONFLICT when a retry
**re-requests** a different mode.

### What the plan proposes

Change repository semantics so a requested-mode mismatch is ignored and stored
mode always wins; update/replace the F3.1.0-V CONFLICT tests.

### Independent decision (required)

**Reject the plan’s repository change.**

Accept this exact contract instead:

| Layer | Contract |
|---|---|
| Repository | **Keep** fail-closed mode-mismatch CONFLICT when a caller explicitly re-requests a different `authorizationMode`. Payload CONFLICT unchanged. |
| Service | **Orchestrate** so deployment defaults never re-request a different mode for an existing logical submission. |
| Cutover meaning | `newSubmissionAuthorizationMode` applies only to **first insert** of a new reservation. |

**Required service algorithm (normative for plan revision):**

```text
desiredMode = config.newSubmissionAuthorizationMode
existing = idempotency.find(lookup)   # or equivalent pre-read

if existing:
  # never pass a mismatched desiredMode
  reserved = idempotency.reserve({
    ...lookup,
    payloadHash,
    authorizationMode: existing.authorizationMode,  # or omit mode
  })
else:
  reserved = idempotency.reserve({
    ...lookup,
    payloadHash,
    authorizationMode: desiredMode,
  })

mode = reserved.authorizationMode   # durable authority
branch LEGACY vs LEDGER on mode only
```

Concurrent first-insert races remain arbitrated by the idempotency PK; losers
re-read the winner’s stored mode.

### Why this preserves both goals

- **Stored mode wins across deployment-default changes** without a misleading
  CONFLICT on ordinary retries after the pin flips.
- **Explicit reinterpretation attempts remain fail-closed** at the repository
  boundary (defense in depth; keeps F3.1.0-V tests meaningful).
- Supersedes plan §6 “Reserve semantics refinement”, §23 rejected-alt row
  “Keep mode-mismatch CONFLICT”, Appendix A decision #1’s “refine reserve”,
  and test I5 as written.

The plan’s claim that keeping repository CONFLICT “breaks legacy retry after
cutover” is **false** under service orchestration.

---

## 6. Critical decision: transaction participation

### What source does today (`188d8e9`)

| API | Caller-owned `trx?` | If `trx` omitted |
|---|---|---|
| `KnexAuthorizationLedgerRepository.createRound` | **Yes** | Opens **its own** `knex.transaction(...)` and commits independently |
| `appendAuditEvent` | **Yes** | Writes on `this.knex` (not the outer create txn) |
| `index.finalize` | **Yes** | Updates on `this.knex` |
| `idempotency.complete` | **Yes** | Updates on `this.knex` |
| `DevelopmentProvider.createWithTransaction` | Required arg | N/A |
| `findRound` | **No** | Always reads on `this.knex` |

Same database ≠ same transaction. Nested/independent ledger commit before
index finalize would allow Round 1 to survive a rolled-back finalize txn, or
(worse for visibility) allow finalize without a durable Round if an implementer
only “checks” with `findRound` outside the txn.

### Independent decision (required)

1. **No new ledger repository capability is required** for F3.1.2 — caller-owned
   `trx` already exists.
2. The plan **must** state, as a hard requirement of F3.1.2b:

```text
Inside the platform create transaction (trx):
  DevelopmentProvider: provider.createWithTransaction(trx, canonicalChange)
  ledger.createRound(round, trx)
  ledger.appendAuditEvent(event, trx)  // each event
  index.finalize(..., trx)
  idempotency.complete(..., trx)
```

3. Calling `createRound(round)` or `appendAuditEvent(event)` **without** `trx`
   on the LEDGER finalize path is a defect — architecture guards / tests must
   forbid it.
4. Visibility authority is **not** a pre-txn `findRound` check. Authority is:
   Round 1 rows are inserted in the **same** txn that sets `is_finalized=true`.
   Recovery may read committed Round 1 with `findRound` **before** opening the
   finalize txn; that is fine because Round 1 is already durable.
5. No third micro-slice is required solely for txn API — fold the explicit pass-`trx`
   requirement into F3.1.2b.

Plan §2 inventory must be corrected to record the existing `trx?` behavior.
Plan §5/§11/§12 must stop implying that “listing operations under one txn” is
enough without naming the `trx` argument.

---

## 7. Critical decision: requirement identity

### What source does today

Published policy `default-change-authorization@2026-09-02.1` uses unique
`requirementRole` values **within each matched rule** (by inspection).
`createPolicyRegistry` does **not** currently enforce per-rule role uniqueness.

### What the plan proposes

Prefer `requirementId = requirementRole`; if collision appears, maybe hash;
“pick one scheme and test it”.

### Independent decision (required) — **Option A**

```text
requirementId = requirementRole
```

With a **hard uniqueness invariant**, not a runtime fallback:

1. Within one published rule’s effective `requirements[]`, every
   `requirementRole` must be unique.
2. Enforce at policy registration / publication validation (extend
   `createPolicyRegistry` or equivalent publication gate) — fail closed at
   startup/publication, never at “maybe hash during submit”.
3. Round construction uses `requirementId = requirementRole` only.
4. Tests must prove: duplicate role in a published rule fails registration;
   Round 1 materialization is stable across pre-commit retries.

**Option B (canonical hash) is rejected for F3.1.2** unless a future policy
publication ADR explicitly needs composite identity. No dual scheme. No
“if collision ever appears” branch in submission code.

Plan §10 and Appendix A must be rewritten to this single scheme.

---

## 8. Rollback / cutover review

### Actual old-binary behavior (`188d8e9`) for pending `LEDGER_REQUIRED`

Verified path:

1. Service reserves **without** `authorizationMode` → mode-mismatch check
   skipped → existing `LEDGER_REQUIRED` row accepted.
2. Service sets `authorizationMode` from the reservation.
3. Service **never branches** on mode; always `buildChange` + `finalizeCreate`
   (provider + finalize + complete) with **no** Round 1.

Therefore a pre-F3.1.2 binary that recovers a pending `LEDGER_REQUIRED`
reservation **will finalize and make the Change discoverable without Round 1**.
It does **not** CONFLICT merely because the stored mode is `LEDGER_REQUIRED`.

### Compatibility rule (accepted by this review)

| Action | Rule |
|---|---|
| Deploy F3.1.2 binary with pin `LEGACY_PRE_F3` | Allowed |
| Flip pin to `LEDGER_REQUIRED` | Allowed after binary deploy |
| Flip pin back to `LEGACY_PRE_F3` | Allowed; existing `LEDGER_REQUIRED` reservations still resume ledger path on F3.1.2 binary |
| Binary rollback to pre-F3.1.2 while any `change_idempotency` row has `authorization_mode='LEDGER_REQUIRED' AND state='pending'` | **FORBIDDEN — correctness** (not merely availability) |
| Binary rollback after all such pendings drained / completed | Operationally allowed; completed ledger submissions remain durable facts |
| Forward redeploy of F3.1.2 after a fail-closed ops stop | Allowed |

The plan’s rollback prohibition is therefore justified as a **correctness**
gate. “Optional hard guard” language in plan §19 must become either an explicit
in-scope hard guard or an explicit out-of-scope runbook-only control — not
optional taste.

---

## 9. External-provider recovery review

Plan reuse of F2 orphan/retry is correct and ADR-007 Model C–compatible:

| State | Convergence |
|---|---|
| Reservation only / pending index | Resume; single canonical snapshot |
| Provider created, platform not committed | `change.create.orphan`; retry `create(change)` by `changeId`; then platform txn |
| Platform txn failure | Nothing platform-durable; retry |
| Round exists, idempotency not complete | Heal complete / finalize as needed; never Round 2 |
| Response lost after complete | Idempotent return of same `{changeId,status}` |

`IChangeManagementProvider` already documents idempotent create by
`change.changeId`. F3.1.2 must treat that as a **required provider capability**
for any future external adapter (plan already states this — keep it).

Pre-Round provider orphan + later policy re-bind is acceptable (G6).

---

## 10. Findings / required corrections

### Blockers (must revise plan before any ACCEPT)

1. **§6 / §23 / Appendix A / I5 — idempotency mode**  
   Remove repository semantic change. Adopt service orchestration in §5.
   Keep F3.1.0-V explicit mismatch CONFLICT tests.

2. **§2 / §5 / §11 / §12 — transaction participation**  
   Document existing `trx?` APIs. Mandate passing caller-owned `trx` into
   `createRound` and `appendAuditEvent`. Replace “check-then-finalize via
   `findRound`” as the visibility authority with same-txn Round insert +
   finalize.

3. **§10 — requirementId**  
   Lock Option A (`requirementId = requirementRole`) + publication/registry
   uniqueness validation. Delete “pick one scheme” / hash fallback.

4. **§19 — optional hard guard**  
   Decide: in-scope hard guard **or** runbook-only. No optional architecture.

### Non-blocking notes (for revision quality; not alternate verdicts)

- Config may live in `authorization/selector/config.ts` only because that file
  already owns `readAuthorizationConfig`; extend `AuthorizationConfig`, do not
  invent a second config tree.
- Two-slice split remains valid after corrections; no third slice required for
  ledger txn API.
- No ADR-009/012 change required to revise the plan.
- Plan can be revised without ADO code changes in this checkpoint.

### What does **not** need correction

- F3.1.2a single-build prerequisite and ordering.
- Earliest durable auth binding at Round 1.
- SoD / error taxonomy / no-migration / scope exclusions / external no-2PC.
- Rollback correctness threat model (matches source).

---

## 11. Decision

```text
F3.1.2 plan architecture review: REJECT
F3.1.2 plan: REVISION REQUIRED
```

Reason: unresolved / incorrect critical contracts for (1) idempotency
mode-mismatch, (2) real caller-owned transaction participation, and
(3) deterministic `requirementId`, plus dependent visibility and test-matrix
failures.

ADR-009 and ADR-012 remain unchanged. The plan **can** be revised to ACCEPT
without ADR amendment.

---

## 12. Next gate

```text
Next authorized activity: F3.1.2 plan revision only
F3.1.2a implementation prompt authoring: NO-GO
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO
```

Do not author an F3.1.2a implementation prompt until a revised plan returns
`ACCEPT` from a fresh review checkpoint.

---

## 13. STOP

```text
STOP
ADO implementation modified: NO
F3.1.2a not implemented
F3.1.2b not implemented
No buildChange fix
No idempotency code change
No transaction/config/migration/route wiring
No AuthorizationRound creation
No F3.1.2a implementation prompt authored
No F3.1.3 / F3.1.4 started
```
