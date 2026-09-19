# F3.1.2 — Final Focused Architecture Re-Review (Concurrency + ADR-013)

## 1. Status / verdict

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

The concurrency-corrected, ADR-013-aligned F3.1.2 plan is an internally
consistent, source-compatible implementation contract. All four original
architecture blockers remain closed. The Round-1 healthy-race convergence rule
is exact. F3.1.1c is correctly scoped as a narrow immutable-policy prerequisite.
F3.1.2a remains canonical-Change-only. F3.1.2b remains autonomy-free while
requiring normal-low primary + CAB under the future CAB-safe policy.

This review is documentation-only. No ADO code was modified. No implementation
prompt was authored.

---

## 2. Reviewed baselines

| Item | Value |
|---|---|
| Docs baseline reviewed (`main`) | `diegofernandes-dev/backstage-docs@7bfb8bc727c65a01251291208fe626af57c9e0f6` |
| Plan under review | `docs/backstage/f3-1-2-implementation-plan.md` |
| Authority | ADR-009 (partially superseded by ADR-013), ADR-013, F3.1 / F3.1.1 plans |
| Historical REJECT #1 | `docs/backstage/f3-1-2-plan-architecture-review.md` (preserved) |
| Historical REJECT #2 | `docs/backstage/f3-1-2-revised-plan-architecture-rereview.md` (preserved) |
| Review prompt | `prompts/f3-1-2-final-architecture-rereview.md` |
| ADO expected baseline | `188d8e9cc43423f3644b3cacfb9849257838a583` |
| ADO branch tip verified | `188d8e9cc43423f3644b3cacfb9849257838a583` (`feat/ado-repo-governance`) |
| Independent ADO source verification | **YES** — HTTPS `git ls-remote` + isolated worktree at exact tip |

Remote tip verification used
`https://dev.azure.com/diegolab/platform-devops/_git/platform-devops-developer-portal`
(`refs/heads/feat/ado-repo-governance` = `188d8e9...`). SSH fetch was unavailable
in this environment; HTTPS tip equality plus exact-SHA worktree inspection is
sufficient independent verification. Zero commits ahead of the accepted baseline.

---

## 3. Independent source-verification scope

Inspected at exact SHA `188d8e9` (worktree `/private/tmp/platform-devops-f311b-review`):

| Surface | Confirmed fact |
|---|---|
| Idempotency | Explicit requested-mode mismatch remains repository `CONFLICT`; omitting mode returns stored row; `find()` already exists |
| Create service | Still calls `buildChange()` twice (lines 190 + 214); no ledger branch; reserve omits mode |
| Ledger | `createRound(..., trx?)` / `appendAuditEvent(..., trx?)` already accept caller-owned `trx`; omit opens independent path |
| DevelopmentProvider | `createWithTransaction(trx, change)` joins caller-owned txn |
| Published policy | `default-change-authorization@2026-09-02.1` — `normal.low` is **primary only** (historical ADR-009); medium/high already primary + CAB |
| Registry | `createPolicyRegistry` does **not** yet enforce per-rule `requirementRole` uniqueness |
| Selectors | `cab-authority` present and reusable |
| Architecture guard | Service must not contain `createRound(` today (to be relaxed only for LEDGER path in 2b) |
| Post-baseline drift | **None** on create / idempotency / ledger txn / policy / selector / finalize surfaces |

Source reality matches the plan inventory. No contradictory drift blocks ACCEPT.

---

## 4. Mandatory gate matrix (G1–G12)

| Gate | Result | Evidence |
|---|---|---|
| G1 Repository mode | **PASS** | §6 keeps explicit mismatch `CONFLICT`; stored-mode-wins is service orchestration; `newSubmissionAuthorizationMode` first-insert only; concurrent first-insert loser recovers with mode omitted |
| G2 Caller-owned transaction | **PASS** | §5/§11 mandate one outer `trx` for DevProvider + `createRound` + every audit append + finalize + complete; external provider remains outside; omit-`trx` on LEDGER path is a defect |
| G3 Requirement identity | **PASS** | §10 locks `requirementId = requirementRole`; no hash fallback; uniqueness fail-closed at registration/publication/startup |
| G4 Rollback correctness | **PASS** | §19 mandatory runbook query; zero pending `LEDGER_REQUIRED` before pre-F3.1.2 binary rollback; not optional |
| G5 Concurrent Round-1 convergence | **PASS** | §13 healthy-race is deterministic: rollback → coherence re-read → same logical success; transient → existing retryable; contradiction → `INTERNAL_ERROR`; never Round 2; never CONFLICT for same payload; C1–C4 with authoritative PostgreSQL |
| G6 ADR-013 target alignment | **PASS** | normal.low/medium/high → primary + CAB; emergency unchanged; autonomy deferred to F3.2; old F3.1.1a policy identity immutable |
| G7 F3.1.1c prerequisite | **PASS** | §4a + §22: new immutable policy version only; change only normal.low; reuse `cab-authority`; no autonomy; not a third F3.1.2 slice; required before LEDGER enablement |
| G8 F3.1.2a isolation | **PASS** | §7/§21/§22: single canonical Change + pending-snapshot reuse only; no auth wiring; independent of CAB policy/autonomy |
| G9 F3.1.2b autonomy-free scope | **PASS** | Consumes CAB-safe policy; materializes primary + CAB for normal-low; no `skipCab`, `CabAutonomyGrant`, waiver engine, CAB Workbench, or autonomy RBAC |
| G10 Future F3.2 separation | **PASS** | Non-goals + ADR-013 mapping defer grant storage/commands/RBAC/applicability/all-covered/race/Workbench |
| G11 No migration claim | **PASS** | §16: F3.1.2 needs no migration; no F3.2 storage claim |
| G12 Source/test completeness | **PASS** | Paths implementable against `188d8e9`; Q5 normal-low proof; C1–C4 concurrency; SQLite + Postgres + guards + lint/build/tsc baseline coherent |

**Gates: 12 / 12 PASS.**

---

## 5. Regression — four original blockers

| Prior blocker | Result | Where closed |
|---|---|---|
| Repository mode-mismatch semantics | **PASS** | Explicit CONFLICT retained; service orchestration for existing reservations |
| Caller-owned transaction | **PASS** | Normative pass-`trx` contract; visibility = shared platform txn commit |
| Requirement identity | **PASS** | Option A only + publication uniqueness |
| Rollback control | **PASS** | Mandatory runbook correctness gate (zero pending LEDGER) |

**Prior critical blockers closed: 4 / 4.**

---

## 6. Concurrency convergence gate

The prior Round-2 REJECT ambiguity (“re-read **or** fail closed”) is gone.

Normative healthy same-actor / same-key / same-payload contract (§13):

1. loser rolls back its platform transaction;
2. re-reads outside that transaction;
3. coherent winner facts → same logical success;
4. transient lock/serialization not yet observable → existing retryable semantics;
5. genuine committed contradiction → `INTERNAL_ERROR` fail-closed;
6. never Round 2;
7. never CONFLICT merely for losing the healthy race.

Test contract C1–C4 is sufficient; disposable PostgreSQL is authoritative.

**Concurrency convergence gate: PASS.**

---

## 7. ADR-013 / F3.1.1c / slice isolation

| Concern | Finding |
|---|---|
| Target baseline | normal.low/medium/high = primary + CAB; emergency unchanged |
| Historical policy | `2026-09-02.1` remains immutable evidence; tip still primary-only for low — correctly forces F3.1.1c |
| F3.1.1c | Narrow new immutable publication; integrity via existing F3.1.1a mechanisms; reuse `cab-authority`; no autonomy |
| F3.1.2a | Canonical Change only; safe after this ACCEPT with separate implementation authorization |
| F3.1.2b | LEDGER path after 2a + 1c; no bypass/grant/waiver/Workbench/autonomy RBAC |
| F3.2 | Fully deferred |

Recommended execution order after separate authorization:

```text
1. F3.1.2a canonical Change
2. F3.1.1c CAB-safe policy publication
3. F3.1.2b ledger submission integration
4. F3.1.3 decisions
5. F3.1.4 read/RBAC
6. F3.2 CAB Governance & Delegated Autonomy
```

F3.1.2a and F3.1.1c may be planned independently (orthogonal surfaces). F3.1.2b
must not enable `LEDGER_REQUIRED` until both accepted prerequisites exist.

---

## 8. Decision

```text
F3.1.2 plan architecture re-review: ACCEPT
F3.1.2 plan: ACCEPTED IMPLEMENTATION CONTRACT
```

No open implementation-semantic choice remains for the F3.1.2 contract as
published. Implementation and F3.1.2b remain gated as recorded above.

Historical REJECT documents are preserved and not rewritten.

---

## 9. Final report

```text
Docs baseline reviewed: 7bfb8bc727c65a01251291208fe626af57c9e0f6
ADO baseline expected: 188d8e9cc43423f3644b3cacfb9849257838a583
ADO branch tip verified: 188d8e9cc43423f3644b3cacfb9849257838a583
Independent ADO source verification: YES
Prior critical blockers closed: 4/4
Concurrency convergence gate: PASS
ADR-013 alignment gate: PASS
F3.1.1c prerequisite gate: PASS
F3.1.2a isolation gate: PASS
F3.1.2b autonomy-free scope gate: PASS
F3.1.2 plan architecture re-review: ACCEPT
ADO implementation modified: NO
```

---

## 10. STOP

```text
STOP
ADO implementation modified: NO
No F3.1.2a/F3.1.1c/F3.1.2b implementation
No implementation prompt authored from this review
No policy publication
No ADR-009/012/013 change
Historical REJECT documents preserved
```
