# Backstage implementation / construction progress

> **Canonical documentation bridge:** `diegofernandes-dev/backstage-docs` (`main`)  
> **Implementation source of truth:** Azure DevOps `platform-devops-developer-portal`  
> **Active implementation branch:** `feat/ado-repo-governance`  
> **Migration baseline:** legacy bridge `diegofernandes-dev/poc-teams-approval@fe4f8073f2a8785673e32ce51e5f70b7c322ad68`  
> **Current GMUD implementation baseline:** F3.1.2 **CLOSED / ACCEPTED IMPLEMENTED SUBMISSION BASELINE** at ADO `2249522` (full SHA `22495229502dabf2d99588599a156d862c5114fa`), parent product-convergence `f48dc82`. Product convergence **PASS** at `f48dc82`; F3.1.1c **CLOSED / ACCEPTED** at `3b302ab`; F3.1.2a **CLOSED / ACCEPTED** at `ccee1e1`; F3.1.2b **CLOSED / ACCEPTED**. Non-production LEDGER_REQUIRED activation **ACCEPT**; F3.1.3 demo target readiness **READY**.  
> **Current GMUD architecture baseline:** ADR-009 Accepted (partially superseded by ADR-013); ADR-014 Accepted for resubmission authority/identity; F3.1.0 + F3.1.1a + F3.1.1b + F3.1.1c + F3.1.2a + F3.1.2b accepted; product convergence PASS; F3.1.2 plan ACCEPTED IMPLEMENTATION CONTRACT; committed default `LEGACY_PRE_F3`; non-production LEDGER_REQUIRED activation ACCEPT on the operator-laptop runtime; F3.1.3 plan architecture review REJECT preserved; F3.1.3 plan revision READY_FOR_REREVIEW; F3.1.3 implementation NO-GO; F3.2 NO-GO

## How to use this log

This file is the active construction handoff for Backstage implementation checkpoints. After every meaningful implementation slice in Azure DevOps, append a concise checkpoint here containing:

- implementation branch and SHA;
- architecture/ADR decisions applied;
- source paths changed in ADO;
- migrations/domain changes;
- tests and functional verification;
- deviations from the normative docs;
- unresolved questions;
- explicit GO/NO-GO for the next slice.

ADO answers **what is implemented**. `backstage-docs` answers **what should be implemented** and records the reviewed construction state. A divergence must be recorded as a deviation; do not silently rewrite architecture to justify code.

## Migration note — 2026-09-01

The active Backstage bridge moved from `diegofernandes-dev/poc-teams-approval` to this repository. The legacy repository remains frozen historical evidence for the original Teams/Azure DevOps POC and the detailed pre-migration construction diary. New architecture, UI contracts, construction handoffs, and cross-workstream documentation must be written here.

The canonical architecture and current construction state were imported at legacy commit `fe4f8073f2a8785673e32ce51e5f70b7c322ad68`. Historical binary screenshots may remain in the legacy archive until intentionally re-captured/migrated; they are supporting evidence, not normative authority.

## Imported GMUD construction timeline

| Checkpoint  | Outcome                     | Implementation / architecture reference                                                |
| ----------- | --------------------------- | -------------------------------------------------------------------------------------- |
| F1          | Frontend GMUD shell         | Dedicated Backstage plugin and `/gmud` route                                           |
| F1.1        | Visual polish               | Backstage/MUI-first composition                                                        |
| F1.2        | Backstage-first layout      | Single form surface + informational rail                                               |
| F1.3        | Semantic UX                 | Generic production-change language; removed deployment-only assumptions                |
| F1.4        | Integrity cleanup           | Provider opacity, explicit classification, evidence semantics; ADO `52e01ca`           |
| F2.0        | Backend contract scaffold   | `ChangeManagementService` + provider contract; ADO `b2bed17`                           |
| F2.0 review | Record authority            | ADR-007 Model C accepted; accidental Model B drift reverted                            |
| F2.1        | Durable Model C persistence | canonical index + DevelopmentProvider; ADO `0dc3ed4`                                   |
| F2.1.1      | Crash-safe idempotency      | retry-driven recovery; ADO `ed6810b`                                                   |
| F2.1.2      | Multi-activity plan         | ADR-008; ADO `5e4f30e`                                                                 |
| F2.1.3      | Real frontend wiring        | create → backend → Model C persistence; ADO `75da44fb46d308e23b1c987e2093636fa4811b92` |
| F2.2        | My Changes + Detail         | index-backed list + provider-routed detail; ADO `0b9cb38`                              |
| F2.2.1      | Participant read            | requester/owner/responsible-team/admin; ADO `6e28611`                                  |
| F3.0        | Authorization architecture  | ADR-009 proposed; no implementation                                                    |
| F3.0.1      | Authorization convergence   | ADR-009 Accepted; no implementation                                                    |
| F3.1.0      | Authorization ledger foundation | domain types, pure evaluators, append-only ledger tables, legacy/cutover semantics; published ADO `7663883` (review evidence: `be16ffb`→`7a9347e`→`06ec9cf`) |

The original detailed F1–F3.0.1 diary is preserved in the legacy bridge at `docs/backstage/implementation-progress.md` for historical investigation only. This file is the active continuation point.

---

## Current implemented baseline — GMUD F2.2.1

### Domain and persistence

- Provider-neutral canonical `Change`.
- Model C: platform canonical index + provider operational record.
- `DevelopmentProvider` is non-production and remains behind `IChangeManagementProvider`.
- Durable platform-owned `changeId` sequence and actor-scoped idempotency.
- Full immutable `ExecutionPlan` snapshot with 1–20 ordered `ExecutionActivity` items.
- `ownerRef` is whole-change governance ownership; `responsibleRef` is activity execution responsibility.

### Read model

`GET /changes` is canonical-index-backed discovery and makes zero provider calls. `GET /changes/:changeId` routes through the immutable stored `providerKey` to the original provider.

Participant read policy:

```text
platform_admin
OR requestedBy == actor
OR actor belongs to Change.ownerRef
OR actor belongs to any ExecutionActivity.responsibleRef
```

`responsibleRef` grants READ only. It grants no approval, ownership, edit, CAB, or execution authority.

F2.2.1 added derived discovery table `change_index_activity_participants`; the immutable Change/index snapshot plus shared `canReadChange` predicate remains the authorization truth.

### Verification recorded at handoff

- backend Change Management: 9 suites / 68 tests PASS;
- frontend Change Management: 13 suites / 58 tests PASS;
- backend and frontend lint PASS;
- participant-index migration/backfill validated against a copy of the real development SQLite database;
- no frontend source change was required for F2.2.1.

### Implementation SHA

ADO `platform-devops-developer-portal`, branch `feat/ado-repo-governance`: **`6e28611`**.

---

## Current accepted architecture — GMUD F3.0.1

ADR-009 is **Accepted**. It defines a platform-owned, provider-neutral authorization ledger and four orthogonal concepts:

| Dimension               | Values / result                                                |
| ----------------------- | -------------------------------------------------------------- |
| Change lifecycle        | `submitted`, `executing`, `completed`, `rejected`, `cancelled` |
| AuthorizationEvaluation | `PENDING`, `AUTHORIZED`, `REJECTED`                            |
| GovernanceEvaluation    | `NOT_APPLICABLE`, `PENDING`, `COMPLIANT`, `NON_COMPLIANT`      |
| ExecutionEligibility    | point-in-time `ALLOW` / `DENY` with reasons                    |

`AUTHORIZED` is not a Change lifecycle state. A Change may be `submitted` and already `AUTHORIZED` while its execution window is closed. `completed` means actual execution completed and may coexist with pending post-execution governance.

### Accepted F3 MVP policy baseline

- Normal + low risk: one configured mandatory pre-execution approval.
- Normal + medium/high: configured primary approval + CAB pre-execution approval.
- Emergency: two distinct generic mandatory pre-execution approvers + mandatory non-blocking post-execution CAB retrospective.
- Additional mandatory requirements are additive only and require a dedicated backend permission.
- A rejected round is immutable; correction/resubmission uses the same `changeId` and a new monotonic AuthorizationRound.
- CAB is one collective authority decision by default, recorded by an actor authorized for the snapshotted CAB authority.
- Corporate names/job titles/e-mails are not canonical authorization semantics; versioned selectors resolve to platform principals and are snapshotted per round.
- Teams is a future individual-decision channel, not a system of record.
- Backstage is the preferred future CAB Workbench UI, not the authorization authority.
- Azure DevOps is a future execution-eligibility consumer/enforcer, not canonical business authorization.
- DevOps/Platform owns policy, controls, reliability, observability, and exceptions; it is absent from happy-path per-deployment approval.

### F3.1 implementation-plan gate

The reviewed [F3.1 implementation plan](./f3-1-implementation-plan.md) decomposes
the Authorization Ledger Foundation into five checkpoints.

`be16ffb` is an **unreviewed candidate**, created after a planning-only checkpoint
and therefore not an accepted implementation baseline. Corrective commit
`7a9347e` is a direct local child that makes the pre-F3.1.2 cutover safe, closes
the remaining F3.1.0 storage/evaluator gaps, and adds both-dialect evidence. Neither
commit has been pushed to ADO. The accepted implementation baseline remains F2.2.1
at `6e28611`.

**NO-GO for F3.1.1–F3.1.4** until the preceding checkpoint has passed its STOP
condition and received explicit review.

Explicitly out of F3.1: Teams, CAB Workbench, Azure DevOps enforcement/public eligibility transport, execution lifecycle integration, automatic SLA escalation, real ITSM providers, generic DSL/BPM, break-glass, decision reversal/expiry/abstention, and technical stop-execution behavior.

Planning findings recorded for implementation:

- ADO local HEAD is `6e28611`, while `origin/feat/ado-repo-governance` remains at
  `0b9cb38`.
- Existing finalized and unfinalized F2 records, plus every Change created by the
  still-F2 submission path, remain `LEGACY_PRE_F3`; no authorization facts are
  backfilled.
- The F2 create path builds the Change twice, which can diverge server-generated
  snapshot fields; this must be fixed before F3.1.2, not in F3.1.0.
- Configured RBAC policy files are absent from ADO HEAD; resolve before F3.1.4.

---

## GMUD F3.1.0-C — Authorization Ledger Cutover Safety Correction

Implementation repository/branch/SHA:
`platform-devops-developer-portal`, local branch
`review/f3-1-0-cutover-safety`, `7a9347e318b3a3a0b5bf4d457f206ce033a6f833`

Documentation baseline SHA: `4cdc9b1`

### Objective

Correct the unreviewed `be16ffb` candidate without rewriting its evidence. Ensure
that introducing empty ledger tables cannot make the still-F2 submission path
produce Changes that the F3 model considers inconsistent.

### Architecture applied

- `authorization_mode` defaults to and is explicitly inserted as
  `LEGACY_PRE_F3` for all pre-cutover F2 traffic.
- Existing finalized and pending rows are both backfilled as legacy; no ledger
  row is fabricated.
- The marker accepts only `LEGACY_PRE_F3 | LEDGER_REQUIRED` and is immutable.
- Future F3.1.2 must explicitly insert `LEDGER_REQUIRED` in the path that creates
  round 1; it must not flip the global default.
- Round creation is atomic and strictly monotonic; canonical artifacts/hashes,
  audit identity/references, bigint ordering, and governance evaluation are
  enforced without submission integration or runtime policy behavior.

### Tests / functional verification

- Change Management: 14 suites / 98 tests PASS.
- SQLite migration up/down, legacy cutover/default, constraints, append-only
  triggers, rollback, canonical hashing, evaluators, and F2 regression PASS.
- Disposable PostgreSQL 16: migration up/down, round/decision concurrency,
  parent-row locking, FKs/checks, bigint audit ordering, and triggers PASS.
- Backend lint PASS; backend build PASS.
- After Jest reported success, the known open handle remained and the process was
  stopped explicitly. No clean-exit claim is made.

### Deviations and process record

- `be16ffb` remains an unreviewed candidate because the preceding checkpoint was
  planning-only and did not authorize implementation.
- ADO remote `origin/feat/ado-repo-governance` remains `0b9cb38`; neither candidate
  commit was pushed or merged.
- Unrelated modifications in the primary ADO worktree were preserved untouched.

### Gate

**GO for architecture review of F3.1.0-C only.** `be16ffb` and `7a9347e` are local
review evidence, not an accepted baseline. **NO-GO** for ADO publication/merge and
for F3.1.1–F3.1.4 until explicit acceptance.

---

## GMUD F3.1.0-V — Cutover & Verification Closure

Implementation repository/branch/SHA:
`platform-devops-developer-portal`, local branch
`review/f3-1-0-cutover-safety`, `06ec9cf`

Local candidate lineage: `6e28611` (accepted baseline) → `be16ffb` (unreviewed) →
`7a9347e` (cutover-safety correction) → `06ec9cf` (this checkpoint).

Documentation baseline SHA: `ef1e97253e6e0fb11e1a9d69368f3332ad5ffa74`

### Objective

Close the two remaining architecture-review blockers on F3.1.0-C: the durable
authorization regime was not yet captured at the idempotency reservation boundary,
and the previously reported test-process non-exit was recorded but not classified.
No other design decision was reopened.

### Architecture applied

**Cross-cutover idempotency regime.** The idempotency reservation
(`change_idempotency`) is written before `change_index` necessarily exists, so it
is the earliest durable boundary for a logical submission
(`ChangeManagementService.createChange`, reserve at the top of the method, index
insert later in the same call). A new migration
(`20260901220000_add_authorization_mode_to_change_idempotency.cjs`) adds
`change_idempotency.authorization_mode` (`LEGACY_PRE_F3 | LEDGER_REQUIRED`,
default `LEGACY_PRE_F3`, immutable via a dialect-specific trigger identical in
shape to the existing `change_index` guard). Existing rows are backfilled to
`LEGACY_PRE_F3` — no ledger regime is fabricated for history that predates the
ledger.

`KnexIdempotencyRepository.reserve()` persists the caller's requested mode (or
`LEGACY_PRE_F3` if none is given, which is what the still-F2 path always does
today) on first insert, and on a retry against an existing row now checks the
requested mode against the stored one in addition to the existing payload-hash
check — a mismatch on either fails closed with `CONFLICT`. The stored mode is
never updated on a retry.

`change_index.authorization_mode` (added by F3.1.0-C) is no longer hardcoded in
`changeIndexMapper.changeToIndexRow` — it is threaded through from the
reservation's mode via `PendingIndexInput.authorizationMode`, so the index row
inherits from the reservation rather than the two being independently decided.
`ChangeManagementService` asserts index/reservation consistency at every point it
re-reads an existing index record alongside a reservation, throwing
`INTERNAL_ERROR` on disagreement — no background repair.

The future F3.1.2 boundary is unchanged from the F3.1.0-C plan and is now
mechanically enforced: a submission path that reserves with `LEDGER_REQUIRED`
must build one canonical Change, create `AuthorizationRound` 1, and finalize the
index with the same `LEDGER_REQUIRED` value; a retry whose reservation says
`LEGACY_PRE_F3` cannot be reinterpreted as ledger-governed merely because the
caller now defaults to or requests the new mode — the reservation's regime is
authoritative for that logical submission for its lifetime.

**Open-handle classification.** The previously reported "Jest reports success but
the process retains an open handle" is Jest **watch mode**, not a resource leak.
`@backstage/cli-module-test-jest`'s `repo test` and `package test` commands both
auto-append `--watch`/`--watchAll` whenever `CI` is unset, no `--since` is given,
and neither `--coverage` nor `--watch(All)` was passed explicitly. Reproduced
identically on accepted baseline `6e28611` and on the F3.1.0-V candidate: `yarn
test` without `CI` hangs waiting for input in both, and must be interrupted
manually. `@backstage/cli` is `^0.36.2` (unchanged) at both commits, and the
change-management suites' own teardown was audited and found correct — every
`createTestKnex()` has a matching `destroyTestKnex()`, and `postgres.test.ts`
destroys both its scoped and admin Knex instances in `afterAll`. No `--forceExit`
was added.

### Tests / functional verification

New/extended, `packages/backend/src/modules/changeManagement/`:

- `persistence/idempotencyAuthorizationMode.test.ts` (new) — reservation defaults
  to `LEGACY_PRE_F3`; identical retry preserves `changeId` and regime; a retry
  requesting a different mode (simulating a post-cutover caller) fails closed and
  leaves the stored regime unchanged; an independent key for the same actor may
  use a different mode; a different actor with the same key is independent;
  `insertPending` inherits the reservation's mode and the column rejects direct
  updates.
- `authorization/migration.test.ts` (extended) — SQLite backfill of pre-existing
  idempotency rows to `LEGACY_PRE_F3`, invalid-value rejection, and up/down of the
  new migration.
- `authorization/postgres.test.ts` (extended) — same backfill/immutability
  assertions and up/down against disposable PostgreSQL 16.
- `ChangeManagementService.*.test.ts`, `testHelpers.ts` — updated call sites for
  the new required `authorizationMode` parameter; all pre-existing F2 recovery,
  crash-safe retry, actor-scoped idempotency, and payload-mismatch-conflict
  behavior unchanged and still green.

Results:

- Change Management (SQLite): 15 suites / 109 tests PASS (`CI=1`,
  `--runInBand`), including the architecture guard.
- Change Management (SQLite + disposable PostgreSQL 16, `--detectOpenHandles`):
  15 suites / 109 tests PASS, **0 open handles reported**, process **exit code 0**
  on both runs.
- Baseline `6e28611` (same command, `--detectOpenHandles`, `CI=1`): 9 suites / 68
  tests PASS, 0 open handles, exit code 0.
- Baseline `6e28611` and candidate `06ec9cf`, `yarn test` without `CI`: both enter
  Jest watch mode and do not exit on their own — identical pre-existing behavior,
  confirmed by process inspection on both, manually terminated in both cases.
- Backend lint PASS (`CI=1`). Backend build PASS (`CI=1`).
- `tsc` (repo-wide): identical 6 pre-existing errors before and after this
  checkpoint's changes (`KnexAuthorizationLedgerRepository.test.ts` missing
  `Knex` type import; `changeManagementPlugin.ts` duplicate-`knex`-package
  structural mismatch) — both present at `7a9347e`, neither touched by this
  checkpoint, neither in the Change Management test path.

### Deviations and process record

- `buildChange()` is still called twice in `ChangeManagementService.createChange`
  — unchanged from F3.1.0-C, **MUST FIX BEFORE F3.1.2**, not addressed here.
- Configured RBAC policy files remain absent from ADO HEAD — unchanged from
  F3.1.0-C, resolve before F3.1.4.
- ADO remote `origin/feat/ado-repo-governance` state not re-verified (unreachable
  from this environment); irrelevant, as nothing was pushed.
- Unrelated modifications in the primary ADO worktree were preserved untouched;
  all work happened in the isolated `review/f3-1-0-cutover-safety` worktree.

### Gate

**GO for architecture acceptance and publication review of F3.1.0-V.** Local
candidate `06ec9cf` is the full lineage `be16ffb` → `7a9347e` → `06ec9cf`; none of
the three commits is accepted, merged, or published to ADO by this checkpoint. The
accepted implementation baseline remains F2.2.1 at `6e28611`.

**NO-GO for F3.1.1–F3.1.4** until F3.1.0 is explicitly accepted.

---

## F3.1.0-P — Publication & baseline promotion

Implementation repository/branch/SHA: `platform-devops-developer-portal` /
`feat/ado-repo-governance` / **`7663883`** (full SHA
`766388393458f82fbdc2e0502b8c193d0a85e605`)
Documentation baseline SHA (start of checkpoint): `f5df375fb332bbb9eb2d8f76c973674b1d53c87f`

### Objective

Promote the architecture-accepted F3.1.0 final state (`06ec9cf`) from local review
candidate to an official, clean, published ADO implementation baseline — without
carrying the three-commit review lineage (`be16ffb` → `7a9347e` → `06ec9cf`) into
official history, and without any force operation.

### Pre-publication verification

- ADO remote `origin/feat/ado-repo-governance` was at `0b9cb38` at checkpoint
  start — one commit **behind** the already-accepted F2.2.1 baseline `6e28611`
  (F2.2.1 itself had never been pushed). `0b9cb38` confirmed an ancestor of
  `6e28611` via `git merge-base --is-ancestor` → publication was fast-forward-safe.
- Candidate ancestry independently re-verified: `6e28611` → `be16ffb` → `7a9347e`
  → `06ec9cf`, each a direct ancestor of the next.
- SSH transport to Azure DevOps (`git@ssh.dev.azure.com`) failed with
  `remote: One or more errors occurred` despite successful publickey
  authentication (server-side condition, not a credentials problem). HTTPS with
  an already-provisioned PAT worked for both read and write and was used via an
  ephemeral `credential.helper`, without altering the persistent `origin` remote
  URL. **Follow-up: repair ADO SSH access separately.**

### Clean candidate construction

- Isolated worktree/branch `publish/f3-1-0-ledger-foundation` created **exactly**
  from `6e28611` (not from any review commit).
- Final state transferred by `git checkout 06ec9cf -- .` — the three review
  commits were **not** cherry-picked, replayed, or merged.
- Equivalence proof: `git diff 06ec9cf` empty; `git write-tree` on the staged
  result equalled `ad970c288f7d983de1210e34987de52851fb0c86`, `06ec9cf`'s exact
  tree hash. `git diff --name-status 6e28611` showed exactly the 22 F3.1.0-scoped
  paths (migrations + `changeManagement`/`authorization`), confirming no
  unrelated candidate-branch files were included.

### Tests / functional verification (clean candidate, before commit)

- Change Management module (15 test files under `modules/changeManagement/`):
  **153/153 tests PASS**, `CI=true`, natural exit code 0, no `--forceExit`.
- SQLite: PASS (in-process).
- PostgreSQL 16 (disposable `postgres:16` Docker container, torn down after the
  run, `CHANGE_MANAGEMENT_TEST_POSTGRES_URL` set so `authorization/postgres.test.ts`
  actually ran rather than skipping): PASS.
- `yarn workspace backend lint`: PASS. `yarn workspace backend build`: PASS.
- `yarn tsc` repo-wide: 6 errors, and the exact error identifiers (file:line:col
  + code) are byte-identical across `06ec9cf`, this clean candidate, and (after
  push) the published commit. 5 are pre-existing `Knex` cross-package
  type-identity errors in `changeManagementPlugin.ts`, independently reproduced
  on baseline `6e28611` (5 errors there too). The 6th (`TS2304` missing `Knex`
  type import in `KnexAuthorizationLedgerRepository.test.ts:389`) was already
  present in the architecture-accepted `06ec9cf` candidate itself — not a
  regression introduced by this publication. **No new TypeScript error.**

### Publication

- One commit created on the clean branch: `feat(gmud): add authorization ledger
  foundation`, parent exactly `6e28611`, SHA `766388393458f82fbdc2e0502b8c193d0a85e605`.
- Immediately before push, `origin/feat/ado-repo-governance` re-verified via
  `git ls-remote` over HTTPS: still `0b9cb38`, unchanged.
- Pushed as a plain fast-forward: `0b9cb38..7663883`. **No `--force`, no
  `--force-with-lease`.**
- Post-push verification via two independent channels — `git ls-remote` and the
  Azure DevOps REST API (`repo_branch.get`) — both returned
  `766388393458f82fbdc2e0502b8c193d0a85e605` for `refs/heads/feat/ado-repo-governance`.
- Local branch `feat/ado-repo-governance` (previously at `be16ffb`) moved to the
  published SHA via `git reset --soft` (branch was checked out in the primary
  worktree, which also carries substantial unrelated uncommitted work from
  another workstream — `git branch -f` was refused for safety). Working-tree
  content for the `changeManagement`/migration paths was then restored from the
  new HEAD via `git checkout HEAD -- <paths>` to remove the stale pre-fix
  `be16ffb`-era migration content that the reset had left staged; content
  verified byte-identical to the pre-reset working tree before the fix, and the
  primary worktree's unrelated 40 files of in-progress work were confirmed
  untouched throughout. `be16ffb` / `7a9347e` / `06ec9cf` remain fully reachable
  via local branch `review/f3-1-0-cutover-safety` and reflog — preserved as
  review/audit evidence, not deleted.

### Deviations and process record (carried forward, unchanged)

- `buildChange()` is still called twice in `ChangeManagementService.createChange`
  — **MUST FIX BEFORE F3.1.2**. Not addressed in F3.1.0-P; equivalence with
  `06ec9cf` did not require it.
- Configured RBAC policy files remain absent from ADO HEAD — prerequisite for
  F3.1.4, not F3.1.0/F3.1.1.
- **F3.1.2 mandatory design invariant:** one logical idempotent submission never
  changes authorization regime across a retry, including across a deployment
  boundary. For an *existing* reservation the persisted mode wins — a
  `LEGACY_PRE_F3` reservation must resume legacy semantics; F3.1.2 must not
  attempt to re-reserve it as `LEDGER_REQUIRED` and must not return a misleading
  conflict merely because the application version changed. Only a *new*
  reservation may select `LEDGER_REQUIRED`. Regime must never be inferred from
  application version, deployment date, schema version, current default, or the
  mere existence of ledger tables.

### Model C / invariant reconfirmation

`ChangeIndexRepository`, `ProviderRegistry`, `IChangeManagementProvider`, and
`DevelopmentProvider` all remain active and unmodified in the published commit;
the authorization ledger sits adjacent to the canonical index, not inside a
monolithic platform-owned Change repository. No provider-specific authorization
semantics, Azure DevOps approval identifiers, Teams identifiers, or CAB
implementation were introduced. `ChangeManagementService` still only sets
`authorizationMode = 'LEGACY_PRE_F3'` on `POST /changes` — no `AuthorizationRound`
is created by F3.1.0.

### Gate

**F3.1.0 promoted from LOCAL REVIEW CANDIDATE to ACCEPTED IMPLEMENTED BASELINE**
at ADO `7663883` (full SHA `766388393458f82fbdc2e0502b8c193d0a85e605`). `06ec9cf`
and its ancestors (`be16ffb`, `7a9347e`) are now historical review evidence only,
retained on `review/f3-1-0-cutover-safety`, and excluded from official
first-parent history.

Superseded below by **F3.1.0-H** — see next section. `7663883` remains the
architectural F3.1.0 Authorization Ledger Foundation content; only its TypeScript
typing was corrected.

---

## F3.1.0-H — TypeScript baseline restoration (typing hotfix)

Implementation repository/branch/SHA: ADO `platform-devops-developer-portal`,
`feat/ado-repo-governance`, `4bad41d` (full SHA
`4bad41d058edf5c5314d17275e0c8bdb5abf690f`), direct child of `7663883`.
Documentation baseline SHA: `af215f775b270277a6887edf931b7337f6cb8b5e`.

### Objective

The F3.1.0 publication checkpoint compared the wrong baseline pair (`06ec9cf` vs
`7663883`) and incorrectly waived a new TypeScript error as pre-existing. The
correct comparison is the accepted F2.2.1 baseline `6e28611` vs the published
`7663883`, under which the error is a genuine F3.1.0 regression. This checkpoint
is a typing-only micro-hotfix: restore the repository-wide TypeScript error set
to exactly the historical five and re-verify all F3.1.0 gates. No architectural,
domain, or behavioral change.

### Regression

`TS2304: Cannot find name 'Knex'` in
`KnexAuthorizationLedgerRepository.test.ts:389:37` (`insertIndexRow(knex: Knex, ...)`),
a file added by `7663883`, with no `knex` import. Correction: added
`import type { Knex } from 'knex';`, matching the convention already used by 15+
files in the module (e.g. `ChangeManagementService.list.test.ts`,
`KnexAuthorizationLedgerRepository.ts`). One file, one line changed.

### TypeScript baseline comparison

Repository-wide `tsc --noEmit`, errors compared by file/line/column/code, not
just count, in an isolated worktree per commit:

| Commit | Errors | Notes |
| --- | --- | --- |
| `6e28611` (F2.2.1, accepted) | 5 | Pre-existing duplicate-Knex-types mismatch in `packages/backend/src/plugins/changeManagementPlugin.ts` (unrelated to this ledger work; not touched) |
| `7663883` (F3.1.0, published) | 6 | The 5 above + the new `TS2304` regression |
| `4bad41d` (F3.1.0-H, hotfix) | 5 | **Set-identical** to `6e28611` — confirmed by diff, not just count |

No historical TypeScript debt was fixed in this checkpoint.

### Tests / functional verification

- Change Management backend suites, CI mode (`CI=1`, `--ci --runInBand`, no
  `--forceExit`): 15 suites, 109 tests, all passed, exit code 0.
- SQLite (in-memory `better-sqlite3`): covered by the same run, passed.
- PostgreSQL: disposable `postgres:16-alpine` container,
  `CHANGE_MANAGEMENT_TEST_POSTGRES_URL` exported; `authorization/postgres.test.ts`
  ("F3.1.0 PostgreSQL contract") confirmed executed (not skipped) and passed —
  migration behavior, authorization ledger tests, idempotency authorization mode
  coverage. Container removed after the run (clean teardown).
- `yarn workspace backend lint`: passed, no output.
- `yarn workspace backend build`: passed, `dist/bundle.tar.gz` produced.

### Model C / invariant reconfirmation

Re-verified after the hotfix: `ChangeIndexRepository` (`KnexChangeIndexRepository`)
and `AuthorizationLedgerRepository` (`KnexAuthorizationLedgerRepository`) remain
separate, adjacent classes — no monolithic Change repository. `ProviderRegistry`,
`IChangeManagementProvider`, and `DevelopmentProvider` all remain active and
unmodified. No Teams, CAB Workbench, Azure DevOps approval integration, policy
runtime, selector resolution, or F3.1.1 behavior was introduced.

### Deviations (unchanged, carried forward)

- `buildChange()` is still called twice in `ChangeManagementService.createChange`
  — **MUST FIX BEFORE F3.1.2**. Not touched by F3.1.0-H.
- Configured RBAC policy files remain absent from ADO HEAD — prerequisite for
  F3.1.4, not F3.1.0/F3.1.1.

### Publication

Pushed as a normal fast-forward: pre-push remote was verified as exactly
`766388393458f82fbdc2e0502b8c193d0a85e605` (`7663883`); push
`7663883..4bad41d` was a fast-forward (no `+`, no force); post-push fetch
confirmed `origin/feat/ado-repo-governance` == `4bad41d058edf5c5314d17275e0c8bdb5abf690f`
with `7663883` as its direct parent. `06ec9cf`/`7a9347e`/`be16ffb` remain outside
official first-parent history.

### Gate

**F3.1.0-H promoted — official implementation baseline is now**
`4bad41d` (full SHA `4bad41d058edf5c5314d17275e0c8bdb5abf690f`), a direct child of
`7663883`. This remains the same F3.1.0 Authorization Ledger Foundation
architectural slice with a typing correction only — not a new slice.

**F3.1.0 is CLOSED.**

**GO for F3.1.1 architecture/implementation planning.**
**NO-GO for F3.1.1 implementation** until its own planning checkpoint.

---

## F3.1.1 — Published Policy & Selector Resolution (planning checkpoint)

Implementation repository/branch/SHA: ADO `platform-devops-developer-portal`,
`feat/ado-repo-governance`, `4bad41d` (full SHA
`4bad41d058edf5c5314d17275e0c8bdb5abf690f`) — verified unchanged from F3.1.0-H
closure; `origin/feat/ado-repo-governance` fetched and confirmed identical.
Documentation baseline SHA: `f4dc268b6f7ac09600710e59d9c7d35d918fdc15`.

### Objective

Design the deterministic mechanism that answers, for an immutable Change
snapshot: which published policy version applies, which
`ApprovalRequirement`s it produces, which selectors those need, what
principal each selector resolves to, and what provenance a future
`AuthorizationRound` must snapshot. Planning only — no
`AuthorizationRound` is created by `POST /changes` in this checkpoint (that
remains F3.1.2).

### Architecture applied

Full design recorded in
[`f3-1-1-implementation-plan.md`](./f3-1-1-implementation-plan.md), derived
by reading the actual F3.1.0 implementation (types, canonical hashing,
evaluators, repository contract, migration columns, architecture guards) at
ADO `4bad41d`, not re-derived from ADR-009 alone. Key decisions:

- Policy matrix as versioned, frozen TypeScript; selector bindings as
  validated app-config (hybrid publication model).
- Policy input narrowed to `{classification, risk}` only.
- Policy output is a pure `RequirementDefinition[]` — the policy never
  persists `ApprovalRequirement` rows.
- One immutable selector bundle per version (not per-selector versioning),
  matching the round-level `selectorBundleKey/Version` columns already
  persisted by F3.1.0.
- Emergency `emergency-approver-a`/`emergency-approver-b` selectors
  restricted to `user`-typed principals in F3 MVP, so distinctness is
  provable at submission — flagged as an explicit open question for
  architecture review, not a silent ADR-009 amendment.
- No new database tables, routes, UI, Teams, or Azure DevOps integration.
- Reuses `authorization/canonical.ts` verbatim for every hash; no second
  canonicalization scheme introduced.

### ADO files changed

**None.** This checkpoint is documentation-only; the ADO working tree is
unmodified (`git status --porcelain` empty before and after).

### Tests / functional verification

Not applicable — no code was written. Test strategy (policy matrix,
determinism, version isolation, invalid publication, selector resolution,
emergency separation, generic-semantics guard, no-provider-leakage guard, SLA
validation, regression) is specified in the plan for the first authorized
implementation slice to satisfy.

### Deviations

Carried forward, unchanged by this checkpoint:

- `ChangeManagementService` still calls `buildChange()` twice — **must fix
  before F3.1.2**.
- Committed app-config still references RBAC CSV/conditional-policy files
  absent from ADO HEAD — prerequisite for F3.1.4.

### Open questions

1. Should emergency Approver A/B ever support authority-typed selectors? This
   plan narrows F3 MVP to individual (`user`) selectors only, to keep
   separation-of-duty provable at submission. Left for architecture review;
   would require its own ADR if reversed.
2. `selectorBundleVersion` string convention — left to F3.1.1b implementation
   time as a config-authoring detail.

### Gate

**F3.1.1 planning is COMPLETE.** Recommended decomposition: two slices,
F3.1.1a (pure policy domain + registry) and F3.1.1b (selector config +
Catalog-backed resolver) — a two-way split, not the three-way split
originally suggested, since a standalone "validation + provenance" slice has
no independent I/O boundary of its own.

**GO for implementation review of slice F3.1.1a** (see the plan's §29 for
exact file boundaries, types, tests, and STOP condition).

**F3.1.1 implementation itself remains NO-GO** until a separate, constrained
implementation prompt explicitly authorizes F3.1.1a. No ADO source was
modified in this checkpoint.

---

## F3.1.1-R — Policy Publication Integrity & Plan Correction (corrective checkpoint)

Implementation repository/branch/SHA: ADO `platform-devops-developer-portal`,
`feat/ado-repo-governance`, `4bad41d` (full SHA
`4bad41d058edf5c5314d17275e0c8bdb5abf690f`) — verified unchanged since F3.1.0-H
closure and since the prior F3.1.1 checkpoint below; `origin/feat/ado-repo-governance`
fetched and confirmed identical to local `HEAD`.
Documentation baseline SHA (the checkpoint this correction supersedes):
`75abd9d60facc6ff1dfc54af4a46192f4c287c9e`.

### Objective

Architecture review found three defects in the F3.1.1 plan published at
`75abd9d` and required a corrective revision before any implementation
authorization could be considered. This checkpoint corrects
[`f3-1-1-implementation-plan.md`](./f3-1-1-implementation-plan.md) in place —
no redesign from scratch, no parallel competing plan document. F3.1.1a
implementation remains unauthorized.

### Architecture applied

Three defects corrected (full detail in the plan's §0):

1. **Self-declared digest is not immutability.** A digest stored next to the
   mutable artifact it describes only detects "content changed, digest
   forgotten" — not "content and digest changed together under the same
   `key@version`," i.e. historical identity reuse. **Fix:** a small, explicit,
   checked-in append-only **publication manifest**
   (`published-manifest.json`) plus a pure `validatePublishedManifest(baseline,
   candidate)` validator, invoked via a repository command
   (`yarn validate:policy-publication --baseline-ref <git-ref>`) that reads the
   baseline from a **previous trusted git ref**, not the working tree.
   Confirmed the repository has **no CI pipeline at `4bad41d`**, so pipeline
   wiring is explicitly deferred, not invented here.
2. **Hashed artifact must determine behavior.** The prior
   `AuthorizationPolicyVersion.evaluate()` was executable code outside the
   hashed matrix, so behavior could change without the digest changing. **Fix:**
   published policy versions become pure serializable data
   (`PublishedAuthorizationPolicy { policyKey, version, provenance, rules }`);
   one shared, unversioned `evaluatePolicy(policy, input)` function replaces
   per-version `evaluate()`. `policyArtifactSha256 = sha256Canonical(rules)`
   now covers all policy-specific behavior; identity/provenance fields are
   deliberately excluded from the hash.
3. **Fragile job-title source scanning.** The prior guard grepped source text
   for `cto|director|manager|superintendent` — `manager` is an ordinary
   software word. **Fix:** replaced with guards over domain shape and
   published-artifact *data* (forbidden canonical field names extending the
   existing `FORBIDDEN_FIELDS` idiom; no e-mail-shaped values in the published
   artifact; generic selector-key naming). Explicit test requirement: a
   variable named `manager` must pass all guards.

Two corrections fall out of the above:

- **Active vs. historical policy validation separated** — an inactive
  historical policy is no longer validated against the *active* selector
  bundle at startup; only the `(activePolicy, activeSelectorBundle)` pair is.
- **Three failure domains formally separated** — CI/publication, runtime
  startup, submission — each with its own table (plan §16), replacing one
  blended table.

Selector bundles (F3.1.1b, still unimplemented) are recommended to reuse the
identical manifest + validator mechanism rather than a second, incompatible
model — one publication-integrity concept for both artifact kinds.

No contradiction with ADR-009 was found; ADR-009 already requires
"deterministic reviewed application code/configuration," rejects a policy
DSL, and states published versions are "never edited or reused for materially
different content" — the manifest mechanizes that sentence. **ADR-009 was not
modified.**

### ADO files changed

**None.** This checkpoint is documentation-only; the ADO working tree is
unmodified (`git status --porcelain` unchanged before and after; `HEAD` still
`4bad41d`).

### Documentation files changed (backstage-docs)

- `docs/backstage/f3-1-1-implementation-plan.md` — corrected in place (not a
  new parallel plan). Status changed to `PLAN CORRECTED / UNDER REVIEW`.
- `docs/backstage/current-state.md` — F3.1.1 status line and header updated to
  reflect the correction; ADO baseline unchanged (`4bad41d`).
- `docs/backstage/implementation-progress.md` — this checkpoint appended.

### Tests / functional verification

Not applicable — no code was written; this is a planning-only correction. The
revised test matrix (plan §26.A–M) is specified for the first authorized
implementation slice to satisfy, including new coverage the correction adds:
manifest append-only rules (§26.G), artifact/manifest digest agreement
(§26.H), and the `manager`-variable-passes regression case (§26.I).

### Deviations

Carried forward, unchanged by this checkpoint:

- `ChangeManagementService` still calls `buildChange()` twice — **must fix
  before F3.1.2**.
- Committed app-config still references RBAC CSV/conditional-policy files
  absent from ADO HEAD — prerequisite for F3.1.4.

### Open questions

1. Emergency Approver A/B authority-typing — carried forward unchanged from
   the prior checkpoint; not addressed by this correction.
2. `selectorBundleVersion` / environment-scoped bundle key string convention —
   left to F3.1.1b implementation time.
3. **New:** if `node --experimental-strip-types` proves unworkable when
   F3.1.1a is actually implemented, the plan's §6 documented fallback
   (plain-jest-tested validator + a synchronized `.mjs` re-implementation)
   should be used instead — flagged so implementation does not silently
   choose a third, undocumented option.

### Gate

**F3.1.1 planning is CORRECTED and under review.** The two-slice decomposition
(F3.1.1a pure policy domain + registry + publication integrity; F3.1.1b
selector config + Catalog resolver + selector publication integrity) is
re-evaluated and kept — the manifest/validator work absorbs into F3.1.1a
rather than forming an independent third slice.

**GO for architecture review of the corrected slice F3.1.1a** (see the plan's
§24 for exact file boundaries, types, tests, and STOP condition).

**F3.1.1 implementation itself remains NO-GO** until a separate, constrained
implementation prompt explicitly authorizes F3.1.1a. No ADO source was
modified in this checkpoint.

## F3.1.1-R2 — Genesis, Policy Model Versioning & Runtime Immutability (corrective checkpoint)

Implementation repository/branch/SHA: `platform-devops-developer-portal` /
`feat/ado-repo-governance` / **`4bad41d058edf5c5314d17275e0c8bdb5abf690f`**
(unchanged; **verified against the Azure DevOps REST API**, not merely a local
`origin/*` ref, because SSH fetch was unavailable in the review environment).

Documentation baseline SHA: `d5fc3ff93a8916932860f0c18ad5f3863f9f163c`
(the F3.1.1-R checkpoint this revision corrects).

### Objective

Close the three remaining contract gaps architecture review found in the R plan,
without redesigning the accepted R direction, and without implementing anything.

### Architecture applied

**Gap A — genesis / first publication.** R required the baseline manifest to be
read via `git show <baseline-ref>:<path>`, but the trusted baseline `4bad41d`
**predates the manifest**. Generic "missing file ⇒ empty baseline" is **forbidden**
— it would be a permanent append-only bypass. An empty baseline may be
synthesized only when **all five** hold: explicit `--allow-genesis-from <sha>`;
that SHA equals the compiled-in `AUTHORIZED_GENESIS_BASELINE_SHA`
(`4bad41d058edf5c5314d17275e0c8bdb5abf690f`); the resolved `--baseline-ref` also
equals it; the manifest is absent at that ref; and the candidate passes full
structural + duplicate validation.

The flag is an **acknowledgement, not an authority** — if a caller-supplied flag
alone could authorize genesis, deleting the manifest and naming any SHA would
reopen the bypass. Genesis is **one-time and self-closing**: once the manifest is
committed, every later baseline has the file present, so condition 4 can never
hold again. Post-genesis, `manifest missing → FAIL` is permanent. New violation
codes: `BASELINE_MANIFEST_MISSING`, `GENESIS_NOT_AUTHORIZED`. Seven security
cases (A–G) are contract, gated by tests §26.J1–J7.

**Gap B — `policyModelVersion`.** R's shared, *unversioned* `evaluatePolicy()`
meant a future semantics change could silently reinterpret an unchanged,
identically-hashed artifact. Added an explicit interpretation contract
`policyModelVersion: 1`, dispatched by an explicit `switch` with a fail-closed
default, and **included in the artifact digest**:

```
policyArtifactSha256 = sha256Canonical({ policyModelVersion, rules })
```

`version` (business publication identity) and `policyModelVersion` (evaluator/rule
semantics) are defined as separate, never-conflated axes. **V1 semantics are
append-only**: they are never edited in place; a semantics change introduces
`policyModelVersion = 2`. Only model version 1 exists in F3 MVP — one `case`, one
function, explicitly **not** a plugin registry, dynamic loader, schema
interpreter, or rules engine. No manifest for evaluator code in F3 MVP.

**Gap C — deep runtime immutability.** `Object.freeze(policy)` is shallow;
`policy.rules[0].requirements.push(…)` still succeeded, and `readonly` is
compile-time only. Added `deepFreezeSerializable(value)` in
`authorization/immutable.ts` — beside `canonical.ts`, so F3.1.1b's selector
bundles reuse it. It accepts **exactly** `canonicalJson`'s value domain (throwing
on functions, class instances, `Map`/`Date`, `undefined`) so "hashable" and
"freezable" remain one invariant, and is `WeakSet` cycle-safe rather than
short-circuiting on `Object.isFrozen` (which would wrongly skip a shallow-frozen
subtree). The published artifact module now exports a **plain literal**, not a
pre-frozen one.

Freeze order: load → validate → compute/check digest → **deep freeze** →
register. **Freezing cannot change the digest** — verified against
`authorization/canonical.ts` at `4bad41d`: freezing alters neither the prototype,
`Array.isArray`, key enumeration, nor values, which are the only four things
`canonicalJson` reads.

**Historical retention — corrected, and reconciled with ADR-009.** R's claim that
the registry "never evicts a registered version" and that every version is
"always available … for the lifetime of the deployed application" is
**withdrawn**. ADR-009 was read in full: it requires evidence sufficient to
*reproduce* outcomes and to let an auditor *reconstruct* the sequence, and forbids
consulting a mutable current policy to *reinterpret* history — but **no clause
requires executable replay of every historical policy**. Verified in ADO
`authorization/types.ts`: `AuthorizationRound` already persists policy identity,
artifact hash, provenance, input + fingerprint, matched-rule provenance, the
**full effective requirements**, resolved principal snapshots, and the Change
snapshot + hash. The ledger is self-contained.

Adopted **Model B**:

| Layer | Retains | Duration |
|---|---|---|
| Publication manifest | publication identity + digest history | forever, append-only |
| Authorization ledger | actual round evidence | forever, append-only |
| Runtime registry | active version + deliberately retained rollback versions | current build only |

Active pin naming a non-shipped version → **startup fails**. Shipped policy ⇒
must have a manifest entry; manifest entry ⇏ must have a shipped module.

**Publication capability vs. enforcement.** Re-verified via `git ls-tree -r HEAD`
that the repository has **no CI pipeline at `4bad41d`** (no `.github/`, no
`.azuredevops/`, no root `azure-pipelines.yml`; root `scripts/` holds only
`build-showcase-techdocs.sh`). F3.1.1a therefore yields a publication-integrity
**mechanism + command + tests** — **not** a mandatory CI gate. The failure-model
heading was relabelled accordingly. Creating a pipeline remains out of scope.

### Revised F3.1.1a exact scope

Unchanged from R except for these additions: `policyModelVersion: 1`; generic
evaluator with explicit model-version dispatch; canonical hash over
`{policyModelVersion, rules}`; genesis constant, `BaselineResolution`,
`--allow-genesis-from`, and the two new violation codes; runtime
manifest/artifact agreement in the shipped ⇒ manifest direction;
`authorization/immutable.ts` + the freeze-ordering contract.

Still **not** in scope: selector config, Catalog, plugin auth wiring, DB,
migrations, routes, `ChangeManagementService`, `POST /changes`,
`AuthorizationRound` creation, permissions, frontend, Teams, CAB UI, ADO
enforcement, **and no CI pipeline**.

### ADO files changed

**None.** No source, test, config, script, migration, pipeline, `package.json`,
or runtime behavior was modified. The ADO working tree is byte-identical to the
start of this checkpoint and `HEAD` remains `4bad41d`.

### Tests / functional verification

No tests were run — this is a planning-only checkpoint with no code change. The
plan's test strategy was relabelled onto the canonical **A–R** scheme (R's ad-hoc
lettering collided once the new cases were added) and extended with: **E**
unsupported `policyModelVersion` fails closed; **F** hash includes
`policyModelVersion`; **J1–J7** genesis security cases; **L1–L5** nested-depth
mutation blocked (each asserting both a throw *and* an unchanged value, so the
proof survives non-strict emitted test code); **M** freeze/digest invariance.
**P** (TypeScript error set set-identical to the 5 known errors at `4bad41d`, by
file/line/column/code) and **Q** (all 15 F3.1.0 suites green) carry forward.

R's acceptance criterion to run the validator against "this branch's own prior
commit … trivially, nothing to compare against" was **withdrawn** — it is exactly
the undefined-genesis assumption of Gap A, and under the corrected rules it fails
`BASELINE_MANIFEST_MISSING`.

### Documentation files changed (backstage-docs)

- `docs/backstage/f3-1-1-implementation-plan.md` — corrected in place (§0a, §1,
  §2a, §5a, §6, §6a, §12, §13a, §16, §17a, §17b, §21a, §23, §24, §26, §28, §28a,
  §29, §30, §31, Gate).
- `docs/backstage/current-state.md` — status, F3.1.1 row, and the stale
  "planning is complete" claim.
- `docs/backstage/implementation-progress.md` — this checkpoint.

**ADR-009 was NOT modified.**

### Deviations

Carried forward, unchanged by this checkpoint:

- `ChangeManagementService` still calls `buildChange()` twice — **must fix before
  F3.1.2**.
- Cross-cutover idempotency: an existing `LEGACY_PRE_F3` reservation resumes the
  legacy path on retry; only a genuinely new logical submission may select
  `LEDGER_REQUIRED`. Policy evaluation must never reinterpret a legacy retry.
- Committed app-config still references RBAC CSV/conditional-policy files absent
  from ADO HEAD — prerequisite for F3.1.4.

### Open questions

1. Emergency Approver A/B authority-typing — carried forward unchanged.
2. `selectorBundleVersion` / environment-scoped bundle key convention — F3.1.1b.
3. `node --experimental-strip-types` fallback for the validator script — carried
   forward.
4. **New:** if any authorized commit lands on `feat/ado-repo-governance` before
   F3.1.1a is implemented, `AUTHORIZED_GENESIS_BASELINE_SHA` must be updated to
   that exact new pre-manifest SHA in the implementation PR — flagged so the
   implementer updates it deliberately rather than reaching for
   `--allow-genesis-from` with whatever SHA makes the check pass.

**Resolved in R2:** historical replay obligation (Model B); genesis code-path
lifetime (retained permanently with tests); genesis candidate strictness (any
structurally valid, duplicate-free candidate — entry count unconstrained).

### Gate

**F3.1.1 planning is CORRECTED (R2) and under FINAL architecture review.** The
two-slice decomposition is re-checked and unchanged: none of the three
corrections adds an I/O boundary, so F3.1.1a grows slightly rather than splitting.

**GO for final architecture acceptance of F3.1.1a**, subject to sign-off on the
genesis contract, the `policyModelVersion` contract, the deep-immutability
mechanism, the Model B retention decision, and the carried-forward emergency A/B
narrowing.

**F3.1.1a implementation remains NO-GO** until a separate, constrained
implementation prompt explicitly authorizes it. No ADO source was modified in
this checkpoint.

## F3.1.1a — Policy Domain, Registry & Publication Integrity (implementation checkpoint)

Implementation repository/branch/SHA: `platform-devops-developer-portal` /
`feat/ado-repo-governance` / **`d3c0751a15b908cec8f5595c97e52f41226344ed`**
(direct child of `4bad41d`; verified against the Azure DevOps REST API both
immediately before the push, and again after, independently).

Documentation baseline SHA: `fe4eb0635a4db72114a1c9ddb3accb8479f7afcc`
(re-fetched and confirmed unmoved both before implementation and again before
this documentation update).

Starting ADO baseline (verified via ADO REST API before any edit):
`4bad41d058edf5c5314d17275e0c8bdb5abf690f` — exact match to the F3.1.1-R2
authorization checkpoint. No reconciliation was required.

### Objective

Implement the F3.1.1a slice authorized by the F3.1.1a implementation
checkpoint: the pure, I/O-free published-policy domain, its generic evaluator,
canonical artifact digest, deep-freezing registry, and append-only publication
manifest with genesis — matching ADR-009's Normal/Emergency policy baselines
exactly, fully unwired from `POST /changes`.

### Architecture applied

Implemented exactly the F3.1.1-R2 plan's §24 slice, under
`packages/backend/src/modules/changeManagement/authorization/`:

- **`policy/types.ts`** — `PolicyInput`, `RequirementDefinition`, `PolicyRule`,
  `PublishedAuthorizationPolicy` (`policyModelVersion: 1`),
  `PolicyEvaluationResult` — exactly the approved shapes, no added fields.
- **`policy/totality.ts`** — the closed `classification × risk` domain, a
  compile-time `AssertTotalMatrix<DeclaredPair<Rules>>` exhaustiveness check
  (fails the build if a union member gains a value without a matching rule),
  and runtime `assertPolicyMatrixTotality` (zero or >1 matches per pair fails
  closed; no default/wildcard rule).
- **`policy/evaluatePolicy.ts`** — `evaluatePolicy` dispatches explicitly on
  `policyModelVersion` (`case 1: return evaluatePolicyModelV1(...)`, fail-closed
  `default`); `evaluatePolicyModelV1` is the permanent V1 semantics; one
  evaluator, shared and unversioned as application code, never per-version.
- **`policy/policyArtifactDigest.ts`** — `sha256Canonical({ policyModelVersion,
  rules })`, reusing `canonical.ts` verbatim; excludes `policyKey`, `version`,
  `provenance`.
- **`policy/published/default-change-authorization.2026-09-02.1.ts`** — the
  first published artifact: a plain literal, no functions, no
  `Object.freeze`, six explicit rules (`normal.low` … `emergency.high`)
  implementing ADR-009's Normal-change and Emergency policy baselines exactly.
  Emergency A/B are distinct-selector-key, `user`-typed individual
  requirements sharing one `separationOfDutyKey`; CAB is one collective
  `authority` requirement; the emergency CAB retrospective carries
  `sla: { durationSeconds: 432000, anchor: 'execution_completion' }` — 5
  *calendar* days of elapsed time, explicitly not "business days" (the current
  SLA contract is elapsed duration; true business-calendar semantics would be
  a separate policy-model evolution). This value was not specified by ADR-009
  or the plan and was set by explicit decision during implementation, recorded
  here for architecture review.
- **`policy/registry.ts`** — `createPolicyRegistry` in the reviewed order (load
  → validate totality/model-version/SLA → check manifest agreement → deep
  freeze → register); duplicate `policyKey@version` and unknown-version lookup
  both throw; Retention Model B (shipped ⇒ manifest entry required; manifest
  entry ⇏ shipped module required) implemented exactly, both directions tested.
- **`policy/published-manifest.json`** — seeded with one entry for
  `default-change-authorization@2026-09-02.1`; its digest
  (`a10560aedaf86b278f9a0963336da6a43d86114c0a63a9b783ff7b7d130e8252`) was
  computed from the actual artifact via `policyArtifactDigest`, never
  hand-typed, and is pinned by a registry test that recomputes and compares it.
- **`policy/publication/validateManifestHistory.ts`** — the pure
  `validatePublishedManifest`, self-contained with zero relative imports (so
  the standalone `.mjs` script can load it directly under
  `node --experimental-strip-types`); `AUTHORIZED_GENESIS_BASELINE_SHA =
  '4bad41d058edf5c5314d17275e0c8bdb5abf690f'` as a compiled-in constant.
  **Genesis violation-code resolution (recorded for architecture review):**
  the plan's §6/§6a table assigns `GENESIS_NOT_AUTHORIZED` to an unauthorized
  genesis attempt against an absent baseline, while §6a row D and test J5
  assign `BASELINE_MANIFEST_MISSING` to the same scenario — a genuine
  contradiction in the plan text. The implementation emits **both** codes:
  `BASELINE_MANIFEST_MISSING` always when the baseline is absent and genesis
  is not fully authorized, plus `GENESIS_NOT_AUTHORIZED` (with a
  `flag-mismatch` or `baseline-ref-mismatch` reason) whenever a genesis flag
  was supplied but failed a condition. This satisfies every §6/§6a/§26.J case
  exactly as written and stays fail-closed; the plan text itself should be
  reconciled in the next architecture pass.
- **`authorization/immutable.ts`** — `deepFreezeSerializable<T>`, beside
  `canonical.ts`; recurses bottom-up over `canonicalJson`'s exact value domain,
  `WeakSet` cycle-safe, never short-circuits on `Object.isFrozen`. Verified
  digest-before-freeze === digest-after-freeze (test M).
- **`scripts/validate-policy-publication.mjs`** — the `--baseline-ref`
  (required) / `--allow-genesis-from` (optional) CLI wrapper. Distinguishes
  path-absence (`git ls-tree` exits 0, empty stdout) from git failure (any
  non-zero exit is a hard error, never a synthesized empty baseline) before
  reading content via `git show`. Wired as root `package.json` script
  `validate:policy-publication`.
- **`architecture.test.ts`** — extended **additively** (a new nested
  `describe('F3.1.1a policy domain guards', ...)` block; the existing five
  guards are untouched) with: the extended `FORBIDDEN_FIELDS`-style check
  (`email`, `emailAddress`, `jobTitle`, `employeeName`, `employeeId`,
  `displayName`) over the new `policy/` sources; the rules-engine/DSL/workflow
  vocabulary ban extended to those sources; a regression case proving a
  variable/parameter/comment named `manager`/`director`/`cto`/`superintendent`
  passes every guard; and **data** guards over the loaded artifact object
  itself (every `selectorKey` matches `^[a-z][a-z0-9-]*$`; no e-mail-shaped
  string anywhere in `rules`; no ADO/Teams identifier).

### Node runtime finding

`node --experimental-strip-types` (Node 22.21.0, within the declared `22 || 24`
engine range) successfully imports a relative `.ts` module from a typeless
package with exit code 0, confirmed by direct probe before implementation.
The §26/§27 primary mechanism was used as specified; **the documented fallback
(hand-written `.mjs` re-implementation) was not needed.**

### ADO files changed

```
package.json                                                                        (M — one script added)
packages/backend/src/modules/changeManagement/architecture.test.ts                  (M — additive only)
packages/backend/src/modules/changeManagement/authorization/immutable.ts            (A)
packages/backend/src/modules/changeManagement/authorization/immutable.test.ts       (A)
packages/backend/src/modules/changeManagement/authorization/policy/**               (A — 13 files)
scripts/validate-policy-publication.mjs                                             (A)
```

18 files changed, matching the pre-commit scope audit exactly — no unexpected
file entered the candidate. `ChangeManagementService.ts`, `changeManagementPlugin.ts`,
`POST /changes`, `authorization_mode`, `IdempotencyRepository`,
`ChangeIndexRepository`, `AuthorizationLedgerRepository`, providers, frontend,
app-config, Catalog, permissions, Teams, CAB Workbench, and ADO deployment
enforcement are **all untouched**. No DB migration, route, endpoint, backend
service dependency, Catalog dependency, or CI pipeline was added.

### Tests / functional verification

- **New F3.1.1a tests: 76** across `immutable.test.ts` (17),
  `policy/published/…test.ts` (matrix A, totality B, determinism C, SLA R),
  `policy/evaluatePolicy.test.ts` (generic evaluator D, fail-closed model
  version E), `policy/policyArtifactDigest.test.ts` (hash inclusion F,
  identity-exclusion G, content-sensitivity H, freeze-invariance M),
  `policy/publication/validateManifestHistory.test.ts` (append-only table I,
  all seven genesis cases J1–J7), `policy/registry.test.ts` (duplicate/unknown
  K, manifest-agreement both directions, deep immutability L1–L5) — plus 14
  tests in the extended `architecture.test.ts` (N, O, and the `manager`
  regression case).
- **Full backend suite: 33 test suites, 237 tests, all green**, including all
  pre-existing F3.1.0 suites unmodified (only `architecture.test.ts` was
  extended, additively). Ran via `CI=true yarn workspace backend test` — no
  watch mode, no `--forceExit`, natural exit.
- **SQLite**: included in the 237 passing (the default in-memory
  `better-sqlite3` path).
- **PostgreSQL**: a disposable `postgres:16` Docker container was started,
  `CHANGE_MANAGEMENT_TEST_POSTGRES_URL` set, and the `F3.1.0 PostgreSQL
  contract` suite **genuinely executed** (5/5 passed, not skipped), then the
  full 33-suite/237-test run was repeated with Postgres included — still all
  green. Container removed via `docker rm -f` afterward (clean teardown, no
  leaked state).
- **Positive genesis command** (real invocation, not mocked):
  ```
  yarn validate:policy-publication \
    --baseline-ref 4bad41d058edf5c5314d17275e0c8bdb5abf690f \
    --allow-genesis-from 4bad41d058edf5c5314d17275e0c8bdb5abf690f
  ```
  Exit code **0**.
- **Negative genesis commands** (real invocations): omitting
  `--allow-genesis-from` against the pre-manifest baseline exits **1** with
  `BASELINE_MANIFEST_MISSING`; supplying a wrong genesis SHA exits **1** with
  `BASELINE_MANIFEST_MISSING` + `GENESIS_NOT_AUTHORIZED` (`flag-mismatch`).
- **Lint**: `yarn workspace backend lint` — exit 0, clean (an initial
  `jest/no-conditional-expect` batch of 12 findings in
  `validateManifestHistory.test.ts` was fixed by asserting the full
  `{ ok: false, violations }` shape unconditionally rather than narrowing
  inside an `if`).
- **Build**: `yarn workspace backend build` — exit 0.
- **TypeScript baseline**: captured inside the isolated worktree before any
  edit (5 errors, all in `changeManagementPlugin.ts`: `(67,61)`, `(77,64)`,
  `(78,64)`, `(79,60)` `TS2345`; `(80,11)` `TS2322`) and re-captured after
  implementation. **Set-identical by file/line/column/code** — `diff` reports
  no difference. No new TypeScript error was introduced.

### Documentation files changed (backstage-docs)

- `docs/backstage/f3-1-1-implementation-plan.md` — status header updated to
  record F3.1.1a as implemented/published, with the genesis violation-code
  contradiction flagged for architecture reconciliation.
- `docs/backstage/current-state.md` — last-updated line and the F3.1.1 status
  row split into F3.1.1a (implemented/published) and F3.1.1b
  (not implemented, not authorized).
- `docs/backstage/implementation-progress.md` — this checkpoint.

**ADR-009 was NOT modified.**

### Deviations

Carried forward, unchanged by this checkpoint:

- `ChangeManagementService` still calls `buildChange()` twice — **must fix
  before F3.1.2**.
- Cross-cutover idempotency: an existing `LEGACY_PRE_F3` reservation resumes
  the legacy path on retry; only a genuinely new logical submission may select
  `LEDGER_REQUIRED`. No policy evaluation is wired into either path in
  F3.1.1a.
- Committed app-config still references RBAC CSV/conditional-policy files
  absent from ADO HEAD — prerequisite for F3.1.4.

New from this checkpoint:

- **Genesis violation-code contradiction in the F3.1.1-R2 plan** (§6/§6a case C
  vs. §6a row D/§26.J5) — resolved in the shipped implementation by emitting
  both codes; the plan text itself should be reconciled by architecture
  review, not by a further implementation-side workaround.
- **CAB retrospective SLA duration** (`432000` seconds / 5 calendar days) was
  not specified by ADR-009 or the plan and was set by explicit decision during
  implementation. It is now permanently bound into the genesis-published
  artifact's digest; changing it requires a new policy version and manifest
  entry, never an edit to this one.

### Open questions

Carried forward from F3.1.1-R2, all still open and unaffected by this
implementation:

1. Emergency Approver A/B authority-typing.
2. `selectorBundleVersion` / environment-scoped bundle key convention —
   F3.1.1b.
3. (Resolved by this checkpoint) `node --experimental-strip-types` worked as
   the primary mechanism; the documented fallback was not needed.
4. (Resolved by this checkpoint) the ADO baseline had not moved past `4bad41d`
   before implementation, so `AUTHORIZED_GENESIS_BASELINE_SHA` needed no
   update.

New:

5. Reconcile the genesis violation-code contradiction in
   `f3-1-1-implementation-plan.md` §6/§6a/§26.J against the shipped
   `BOTH_CODES` behavior — either amend the plan text to match, or direct a
   future revision of the implementation.

### Gate

**F3.1.1a is IMPLEMENTED and PUBLISHED to `feat/ado-repo-governance` at
`d3c0751`.** All acceptance criteria in the F3.1.1a implementation
authorization were met: isolated clean worktree proven; full A–R test matrix
and all seven genesis security cases pass; the real
`validate:policy-publication` command passes for authorized genesis and fails
correctly (with the documented codes) for both negative cases; SQLite and
PostgreSQL both genuinely executed and passed; lint and build are clean; the
TypeScript error set is exactly the 5-error baseline; the pre-commit scope
audit found only approved paths; the push was a verified plain fast-forward
with no force operation; the final remote SHA was independently confirmed.

**GO for F3.1.1a implementation acceptance**, subject to architecture review of:
the shipped genesis both-codes resolution (recorded above, plan text
reconciliation needed); the CAB retrospective SLA value chosen during
implementation; and the implementation generally.

**NO-GO for F3.1.1b implementation** — not implemented, not authorized by this
checkpoint.

**NO-GO for F3.1.2+ implementation** — not authorized by this checkpoint.

`buildChange()`-twice remains **MUST FIX BEFORE F3.1.2**. RBAC CSV/conditional-
policy files remain an **F3.1.4 prerequisite**.

---

## F3.1.1b — Selector Bundle, Catalog Resolver & Publication Integrity (implementation checkpoint)

Implementation repository/branch/SHA: `platform-devops-developer-portal` /
`feat/ado-repo-governance` / **`188d8e9cc43423f3644b3cacfb9849257838a583`**
(direct child of `d3c0751`; verified against the Azure DevOps REST API both
before any edit and again after the push).

Documentation baseline SHA: `8de1ca430387c3f4b231f76b2c731347ec636c9a`.

Starting ADO baseline (verified live before any edit):
`d3c0751a15b908cec8f5595c97e52f41226344ed` — exact match to the accepted
F3.1.1a implemented baseline. No reconciliation was required. The Delivery
branch `feat/delivery-mvp-slice` was not used, read from, or merged.

Full evidence: [`f3-1-1b-implementation-evidence.md`](./f3-1-1b-implementation-evidence.md).

### Objective

Implement the F3.1.1b slice: the selector bundle domain, its typed
environment-scoped configuration reader, startup validation of the **active
policy + active selector bundle pair**, a Catalog-backed principal resolver
using backend service credentials, selector canonical digests, and the
selector-bundle publication identity appended to the existing F3.1.1a
manifest — fully unwired from `POST /changes`.

### Architecture applied

New `authorization/selector/` directory: `types.ts` (`SelectorEntry`,
`SelectorBundle` — one immutable versioned bundle, no per-selector versions),
`selectorDigest.ts` (`sha256Canonical(bundle.selectors)` and the per-resolution
digest over `{selectorKey, selectorVersion, principalType, principalRef}`,
reusing `canonical.ts` verbatim), `config.ts` (`readAuthorizationConfig`,
required reads throughout, selectors read as a *list* so duplicate keys stay
detectable), `bundle.ts` (`createSelectorBundle` in the `createPolicyRegistry`
order, ending in the reused `deepFreezeSerializable`), `startupValidation.ts`
(active pair only), `CatalogPrincipalResolver.ts` (produces the existing
`PrincipalResolutionSnapshot` with no added field, via `parseEntityRef` +
`catalog.getEntityByRef` under `auth.getOwnServiceCredentials()`), and
`bootstrapAuthorization.ts` (the whole startup path in one call).

Key architectural points:

- **Content in configuration, identity in the manifest.** Bindings are
  environment-scoped app-config; the identity and digest of a binding set is a
  published artifact. Rebinding without a version bump fails startup — proven
  against a real running backend.
- **Active-pair-only validation.** An inactive historical policy referencing a
  selector absent from the active bundle does **not** crash startup; pinned by a
  dedicated test.
- **Emergency A/B narrowing expressed generically** — enforced over shared
  `separationOfDutyKey` groups, never by hardcoded selector names.
- **`validateManifestHistory.ts` reused entirely unchanged** — the
  `selector-bundle` artifact kind already existed. One entry appended; the
  pre-existing policy entry is byte-identical.
- **Fail-closed resolver** — `INTERNAL_ERROR` / `NOT_FOUND` /
  `PROVIDER_UNAVAILABLE`, no cache, no member expansion, no address or
  human-readable-name fallback.

### ADO files changed

```
app-config.yaml                                                                     (M — +29)
packages/backend/src/modules/changeManagement/architecture.test.ts                  (M — +157, additive)
packages/backend/src/modules/changeManagement/authorization/policy/published-manifest.json  (M — +6, append only)
packages/backend/src/plugins/changeManagementPlugin.ts                              (M — +21, 0 deletions)
packages/backend/src/modules/changeManagement/authorization/selector/**             (A — 14 files)
```

All four modifications are pure insertions: 213 insertions, 0 deletions. The
pre-commit scope audit found `ChangeManagementService.ts`, Delivery, frontend,
migrations, routes, approval commands, Teams/CAB and
`app-config.production.yaml` all untouched. No migration, route, or CI pipeline
was added.

### Tests / functional verification

- **New F3.1.1b tests: 105** across 7 suites, plus a 4-test opt-in live suite.
- **Full backend suite: 40 suites / 328 tests passed**, 1 skipped (opt-in live).
- **PostgreSQL**: disposable `postgres:16` container; `authorization/postgres.test.ts`
  genuinely executed inside the green run; container removed afterwards.
- **Publication**: `yarn validate:policy-publication --baseline-ref d3c0751…`
  exits **0** as an ordinary append. **`--allow-genesis-from` was not used at
  any point.** Negatives against the new commit exit **1** with
  `DIGEST_CHANGED` (identity reuse with changed content) and `IDENTITY_REMOVED`.
- **Live Catalog** (running instance, untouched): a real `User` resolved; a real
  `Group` resolved to the Group ref while the test first proves that Group
  genuinely carries `hasMember` relations and then proves none appear in the
  snapshot; a non-existent principal failed closed with `NOT_FOUND`; service
  credentials used on every lookup.
- **Real startup**: a second backend booted from the committed `app-config.yaml`
  on port 7008 with its own SQLite DB and logged the validated active pair with
  the exact digest. A rebound selector and an unpublished bundle identity each
  **failed startup closed**; the valid config was restored and booted cleanly,
  with `/api/change-management/changes` behaving identically to the untouched
  instance.
- **Lint**: exit 0. **Build**: exit 0.
- **TypeScript baseline**: no error added, none removed. The 5 pre-existing
  `changeManagementPlugin.ts` errors shifted by exactly +21 lines — precisely
  the number of lines inserted into that file — with identical file, column,
  code and source expression. This is the first F3 slice that had to modify that
  file, so identity was proven at file+code+column+expression level rather than
  by a literal line-number diff.

### Deviations

New from this checkpoint:

1. **`app-config.production.yaml` deliberately has no authorization block.**
   Production overlays the base config, so the mandatory check is satisfied
   everywhere; inventing production Catalog refs would be fabrication.
   **Consequence:** a production deployment would currently inherit the dev
   bundle. Publishing a distinct `selector-bundle-prod` identity is a
   prerequisite for production rollout, which is separately gated on a real
   production target.
2. **`resolverProvenance` is colon-separated, not `key@version`** — the
   `key@version.n` form matches the address-shape guard, so the separator was
   changed to keep "no snapshot value is address-shaped" exception-free.
3. **`contentDigest` in configuration is optional** — always computed and always
   checked against the manifest; the declared value is a second operator
   statement, validated when present.
4. **Publication transport:** ADO SSH was failing all session; the push went over
   HTTPS with a PAT through a temporary remote that was removed immediately, and
   the resulting SHA was confirmed independently via the ADO REST API. `origin`
   is unchanged.

Carried forward, unchanged: `buildChange()` twice (**MUST FIX BEFORE F3.1.2**);
`LEGACY_PRE_F3` idempotency semantics; RBAC CSV/conditional-policy files as an
**F3.1.4 prerequisite**; decision-time authority membership is not selector
resolution; production rollout readiness gated on a real production target.

### Open questions

Carried forward:

1. Emergency Approver A/B authority-typing (unchanged; MVP remains user-typed).
2. Genesis violation-code contradiction in `f3-1-1-implementation-plan.md`
   §6/§6a/§26.J versus the shipped both-codes behavior — untouched here, still
   awaiting an architecture-side plan-text reconciliation.

Resolved by this checkpoint:

3. `selectorBundleVersion` / environment-scoped bundle key convention — settled
   as `selector-bundle-<env>@<date>.<n>` with content in configuration and
   identity in the manifest.

New:

4. When a real production target exists, a `selector-bundle-prod` identity must
   be published before production rollout; until then production inherits the
   dev bundle from the base config.

### Gate

**F3.1.1b is IMPLEMENTED and PUBLISHED to `feat/ado-repo-governance` at
`188d8e9`.** The publication validated as a normal append against the trusted
F3.1.1a baseline with no genesis path; startup validation, fail-closed Catalog
resolution and non-expansion of authorities were proven against a real running
Backstage and Catalog; lint, build and the full suite (SQLite and PostgreSQL)
are green; the TypeScript error set is unchanged.

```text
F3.1.1b implementation: PASS
Architecture implementation acceptance: PENDING SEPARATE REVIEW
F3.1.2: NO-GO
Production rollout: separate / deferred
```

**`POST /changes` remains completely unwired.** No `AuthorizationRound` is
created and `ChangeManagementService` is untouched.

**NO-GO for F3.1.2** — not authorized by this checkpoint. `buildChange()`-twice
remains **MUST FIX BEFORE F3.1.2**.

---

## F3.1.1b — Architecture Implementation Acceptance

Documentation baseline SHA: `cf9f96cbf0ae8488a356c32f6f13c0ebb62b2ee3`
(`diegofernandes-dev/backstage-docs@main` at review start).

Implementation candidate reviewed:
`platform-devops-developer-portal@188d8e9cc43423f3644b3cacfb9849257838a583`
(parent `d3c0751a15b908cec8f5595c97e52f41226344ed`, exact match).

Independent source verification: **PARTIAL** — exact SHA inspected and high-value
tests re-run in an isolated worktree; fresh Azure DevOps `git fetch` failed in
this session; live Catalog / disposable Postgres were not re-executed and remain
grounded in the prior implementation evidence.

### Gate

```text
F3.1.1b architecture implementation acceptance: ACCEPT
Architecture gates: 17/17 PASS
F3.1.1b: CLOSED / ACCEPTED IMPLEMENTED BASELINE
F3.1.2 planning: GO
F3.1.2 implementation: NO-GO
ADO implementation modified by this review: NO
```

Canonical review document:
[`f3-1-1b-architecture-acceptance.md`](./f3-1-1b-architecture-acceptance.md).

Carried forward unchanged: `buildChange()`-twice **MUST FIX BEFORE F3.1.2**;
`LEGACY_PRE_F3` reservation semantics; emergency A/B resolved-person distinctness
at submission; production selector-bundle publication as a production-rollout
prerequisite; RBAC CSV / conditional policies as an F3.1.4 prerequisite.

---

## F3.1.2 — Fail-Closed Submission + First AuthorizationRound (planning checkpoint)

Implementation repository/branch/SHA: ADO `platform-devops-developer-portal` /
`feat/ado-repo-governance` / **`188d8e9cc43423f3644b3cacfb9849257838a583`**
(HTTPS `git ls-remote` tip equal to accepted F3.1.1b baseline; SSH fetch
unavailable in session; **no post-baseline drift** on the F3.1.2 surface).

Documentation baseline SHA (start): `d65bf5e1446e682576557583b841ddde7e5a890c`.

### Objective

Produce a reviewable implementation plan for composing `POST /changes`,
immutable idempotency regime selection, F3.1.1a/b policy/selector runtime, and
F3.1.0 ledger Round 1 — planning only.

### Architecture applied

Full design in
[`f3-1-2-implementation-plan.md`](./f3-1-2-implementation-plan.md). Key
decisions:

- Cutover via `changeManagement.authorization.newSubmissionAuthorizationMode`
  (`LEGACY_PRE_F3` \| `LEDGER_REQUIRED`), default `LEGACY_PRE_F3`; **stored
  reservation mode wins forever** (refine reserve semantics so mode re-request
  is not CONFLICT).
- Two micro-slices: **F3.1.2a** single canonical `buildChange`/snapshot reuse;
  **F3.1.2b** ledger submission wiring.
- Round 1 + requirements + submission audit in the same platform DB transaction
  as index finalize + idempotency complete; DevelopmentProvider may join;
  external provider create remains outside (Model C orphan/retry — no 2PC).
- Emergency A/B same effective User fail-closed before Round commit.
- Additive mandatory requirements deferred; **no migration**.
- Binary rollback past F3.1.2 forbidden while pending `LEDGER_REQUIRED`
  reservations exist.

P1–P20 resolved; all 15 challenge scenarios answered. ADR-009 / ADR-012
untouched.

### ADO files changed

**None.** Documentation-only checkpoint.

### Tests / functional verification

Not applicable — no code. Concrete SQLite/Postgres/failure-injection matrix is
in the plan §20 for the future implementation checkpoint.

### Deviations

None new. Carried forward: production selector-bundle publication still a
production-rollout prerequisite; RBAC CSV files still F3.1.4.

### Gate

```text
F3.1.2 planning: READY_FOR_REVIEW
F3.1.2 implementation: NO-GO
Planning gates resolved: 20/20
Challenge scenarios answered: 15/15
Migration required: NO
Planned implementation slices: 2
ADO implementation modified: NO
```

**Next checkpoint:** independent architecture review of
[`f3-1-2-implementation-plan.md`](./f3-1-2-implementation-plan.md) — not
implementation.

---

## F3.1.2 — Plan architecture review (REJECT)

Implementation repository/branch/SHA verified: ADO
`platform-devops-developer-portal` / `feat/ado-repo-governance` /
**`188d8e9cc43423f3644b3cacfb9849257838a583`** (`origin` tip equal; no
post-baseline drift on the F3.1.2 surface). Unrelated local
`feat/delivery-mvp-slice@170af45` classified as outside surface.

Documentation baseline (review start): `4b28eb20969dad7b7273464e28ae809f0da87a39`.

### Objective

Independently decide whether
[`f3-1-2-implementation-plan.md`](./f3-1-2-implementation-plan.md) is a safe,
concrete implementation contract for F3.1.2a/b — review-only.

### Architecture applied

Canonical review:
[`f3-1-2-plan-architecture-review.md`](./f3-1-2-plan-architecture-review.md).

Verdict **REJECT**. Gates **12/20 PASS**. Three critical decisions resolved by
the review (plan must be revised to embody them):

1. **Idempotency mode:** keep repository explicit mode-mismatch `CONFLICT`;
   service orchestration makes stored mode win across deployment-default flips
   (do **not** change `reserve()` to ignore mismatches).
2. **Transactions:** `createRound` / `appendAuditEvent` already accept caller
   `trx?`; omitting opens an independent txn — plan must mandate passing the
   outer platform `trx` (same DB ≠ same txn).
3. **requirementId:** Option A only — `requirementId = requirementRole` plus
   publication/registry uniqueness validation; no runtime hash fallback.

Also confirmed: pre-F3.1.2 binary **will finalize** pending `LEDGER_REQUIRED`
without Round 1 (correctness-level binary-rollback forbid); external orphan/retry
OK; no migration needed; two-slice split remains structurally sufficient after
plan corrections.

### ADO files changed

**None.** Documentation-only checkpoint.

### Tests / functional verification

Source inspection at exact `188d8e9` (idempotency reserve, ledger trx behavior,
create/finalize path, provider idempotency contract, published policy roles).
No ADO tests executed as part of this review.

### Deviations

None new in ADO. Plan revision required before any implementation contract
acceptance.

### Gate

```text
F3.1.2 plan architecture review: REJECT
F3.1.2 plan: REVISION REQUIRED
Architecture gates: 12/20 PASS
Critical decisions resolved by review: 3/3
F3.1.2a implementation prompt authoring: NO-GO
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO
ADO implementation modified: NO
```

**Next checkpoint:** revise
[`f3-1-2-implementation-plan.md`](./f3-1-2-implementation-plan.md) only —
then re-review. Do not author an F3.1.2a implementation prompt yet.

---

## Next checkpoint template

```text
## <workstream> <slice> — <name>

Implementation repository/branch/SHA:
Documentation baseline SHA:

### Objective
...

### Architecture applied
...

### ADO files changed
...

### Tests / functional verification
...

### Deviations
...

### Open questions
...

### Gate
GO / NO-GO
```

For independent Backstage workstreams (for example Golden Paths / Software Templates), keep a focused workstream progress log under its own `docs/<workstream>/` folder and reference shared ADRs here only when the decision affects the overall platform.

---

## GMUD F3.1.2-R — Plan revision after architecture REJECT

Documentation checkpoint: `backstage-docs@6284195a41bb428862c057ee9c0291014ccba61d`.

### Outcome

```text
F3.1.2 revised planning: READY_FOR_REREVIEW
F3.1.2 implementation: NO-GO
F3.1.2a implementation prompt authoring: NO-GO pending re-review ACCEPT
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO
```

The rejected first plan was revised without changing ADO implementation. The revised contract now:

- preserves `KnexIdempotencyRepository.reserve` explicit authorization-mode mismatch as fail-closed `CONFLICT`;
- makes stored-mode-wins a service orchestration rule for existing reservations, including a race-safe first-insert recovery path;
- mandates one caller-owned Knex transaction for DevelopmentProvider + Round 1 + requirements + authorization audit + index finalization + idempotency completion;
- removes non-transactional `findRound` as visibility authority;
- fixes `requirementId = requirementRole` and requires fail-closed per-rule `requirementRole` uniqueness at policy registration/publication;
- classifies rollback to pre-F3.1.2 with pending `LEDGER_REQUIRED` reservations as a mandatory runbook correctness gate;
- corrects the crash matrix so Round/finalize/complete cannot appear as normal separately committed platform states;
- keeps exactly two implementation slices: F3.1.2a canonical Change, then F3.1.2b ledger submission.

ADO source was **not modified** by this revision. The immediately preceding architecture review independently verified ADO `188d8e9` and no F3.1.2-surface drift; the fresh re-review must verify the branch tip again before ACCEPT.

Next gate: **fresh independent architecture re-review of the revised F3.1.2 plan**.

---

## GMUD F3.1.2-RR — Revised-plan architecture re-review (REJECT)

Canonical review: [`f3-1-2-revised-plan-architecture-rereview.md`](./f3-1-2-revised-plan-architecture-rereview.md).

### Outcome

```text
F3.1.2 revised-plan architecture re-review: REJECT
Architecture gates: 18/20 PASS
Original critical blockers closed: 4/4
Remaining blocker: concurrent Round 1 loser convergence
F3.1.2a implementation prompt authoring: NO-GO
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO
```

The re-review accepted every correction from the first REJECT: repository mode-mismatch remains fail-closed while stored-mode-wins is service orchestration; caller-owned transaction is explicit; `requirementId = requirementRole` is locked with publication-time uniqueness; rollback is a mandatory runbook correctness gate.

One remaining implementation-contract ambiguity was found. When two workers concurrently reach the Round 1 transaction for the same logical submission, the plan says the loser may re-read completed state or fail closed, but does not define the exact healthy-race convergence behavior. The required correction is narrow: a loser of the expected Round-1 uniqueness race must roll back, re-read the winner's completed/finalized/Round-1 facts, and return the same logical success when those facts are coherent; true committed inconsistency remains fail-closed. Add an end-to-end concurrent same-key/same-payload test proving one Change, one Round, one requirement/audit set, one completed reservation, and the same logical result to both callers.

Independent ADO source verification in this ChatGPT re-review was **PARTIAL**: it relied on the immediately preceding exact-source review of ADO `188d8e9`; the current ADO tip could not be re-fetched from this environment. This limitation is not the reason for REJECT.

**Next checkpoint:** revise only the concurrency contract and proof, then perform a focused fresh re-review. No implementation prompt yet.

---

## GMUD F3.1.2-CR — Concurrency plan revision (READY_FOR_REREVIEW)

Documentation-only checkpoint against `backstage-docs` main after the 18/20 concurrency REJECT.

### Outcome

```text
F3.1.2 concurrency plan revision: READY_FOR_REREVIEW
F3.1.2 implementation: NO-GO
F3.1.2a implementation prompt authoring: NO-GO pending fresh ACCEPT
F3.1.2a implementation: NO-GO
F3.1.2b implementation: NO-GO
```

Canonical plan [`f3-1-2-implementation-plan.md`](./f3-1-2-implementation-plan.md) now defines a deterministic healthy Round-1 loser contract:

- same actor + Idempotency-Key + payload + `LEDGER_REQUIRED`: loser rolls back, re-reads completed reservation / finalized index / Round 1, returns the same logical success when coherent;
- never Round 2, never CONFLICT for same payload, never INTERNAL_ERROR merely for losing the race;
- transient winner-not-yet-observable uses bounded immediate re-read then existing retryable storage semantics (no polling/locks/queues);
- true committed inconsistency fails closed as INTERNAL_ERROR;
- concurrent different payload remains CONFLICT before authorization with no Round-race recovery.

Proof additions: authoritative PostgreSQL end-to-end cases C1/C2, concurrent different-payload C3, invariant-corruption negative C4. DB uniqueness (T6) retained but is not sufficient alone.

Frozen decisions 1–13 from the concurrency revision prompt were **not** reopened. ADO implementation was **not** modified. Local `origin/feat/ado-repo-governance` tip verified as exact accepted SHA `188d8e9`; live Azure DevOps remote fetch was unavailable in this checkpoint (no material create/idempotency/ledger drift relative to that tip).

Both historical REJECT review documents are preserved.

**Next gate:** one focused independent architecture re-review of the concurrency-corrected F3.1.2 plan. No implementation prompt yet.

---

## GMUD ADR-013 / F3.1.2-CAB — CAB-safe governance realignment

Architecture checkpoint after the F3.1.2 concurrency correction.

### Outcome

```text
ADR-013: ACCEPTED
F3.1.2 concurrency + CAB-safe plan: READY_FOR_REREVIEW
F3.1.1c CAB-safe policy publication: PLANNED PREREQUISITE / implementation NO-GO
F3.1.2a implementation: NO-GO pending plan ACCEPT
F3.1.2b implementation: NO-GO
F3.2 CAB Governance & Delegated Autonomy: architecture accepted / implementation NO-GO
```

ADR-013 changes the target normal-low governance baseline from primary-only to **primary + CAB by default**. A future bounded CAB-issued autonomy grant may omit only the normal-low CAB requirement, but that capability is explicitly deferred to F3.2.

Side effects recorded:

- the accepted F3.1.1a policy identity remains immutable historical evidence;
- a new immutable **F3.1.1c** policy publication is required before F3.1.2b: normal-low becomes primary + CAB; medium/high/emergency otherwise unchanged;
- F3.1.2a remains fully independent and still only fixes canonical Change construction/recovery reuse;
- F3.1.2b consumes the CAB-safe policy and contains no skipCab/autonomy/grant/waiver logic;
- F3.2 owns bounded Group + System autonomy grants, grant/revoke/renew RBAC, Round applicability, multi-activity all-covered semantics, grant/revoke concurrency, and future CAB Workbench;
- CAB Change-decision authority and CAB autonomy-governance authority are separate backend/RBAC powers;
- platform_admin does not automatically imply CAB business authority;
- autonomy history is append-only and revocation is prospective; committed Rounds are never rewritten.

Canonical ADR: [`../adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md`](../adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md).

Canonical plan: [`f3-1-2-implementation-plan.md`](./f3-1-2-implementation-plan.md).

**Next gate:** final focused independent architecture re-review of the concurrency-corrected, ADR-013-aligned F3.1.2 plan. No implementation is authorized by this checkpoint.

---

## GMUD F3.1.2-FR — Final architecture re-review (ACCEPT)

Independent review-only checkpoint against `backstage-docs@7bfb8bc` and live ADO `feat/ado-repo-governance` tip `188d8e9`.

### Outcome

```text
F3.1.2 plan architecture re-review: ACCEPT
F3.1.2 plan: ACCEPTED IMPLEMENTATION CONTRACT
F3.1.2a implementation-prompt authoring: GO
F3.1.1c implementation planning/prompt authoring: GO
F3.1.2a implementation: still requires separate explicit authorization
F3.1.1c implementation: still requires separate explicit authorization
F3.1.2b implementation: NO-GO until F3.1.2a + F3.1.1c accepted
F3.2 implementation: NO-GO
ADO implementation modified: NO
```

### Verification

- Independent ADO tip verification: **YES** (HTTPS `git ls-remote` + exact-SHA worktree); tip equals expected baseline `188d8e9`; no post-baseline drift on create/idempotency/ledger/policy/selector/finalize surfaces.
- Prior critical blockers closed: **4/4**.
- Concurrency convergence gate: **PASS** (deterministic healthy Round-1 loser; C1–C4; authoritative PostgreSQL).
- ADR-013 alignment: **PASS** (normal-low/medium/high = primary + CAB; autonomy deferred to F3.2).
- F3.1.1c prerequisite: **PASS** (narrow new immutable policy publication; not a third F3.1.2 slice).
- F3.1.2a isolation: **PASS** (canonical Change only).
- F3.1.2b autonomy-free scope: **PASS** (no skipCab / CabAutonomyGrant / waiver / Workbench / autonomy RBAC).

Canonical review: [`f3-1-2-final-architecture-rereview.md`](./f3-1-2-final-architecture-rereview.md).

Historical REJECT documents preserved:
[`f3-1-2-plan-architecture-review.md`](./f3-1-2-plan-architecture-review.md),
[`f3-1-2-revised-plan-architecture-rereview.md`](./f3-1-2-revised-plan-architecture-rereview.md).

**Next authorized documentation activity:** author the constrained F3.1.2a implementation prompt and/or F3.1.1c planning/prompt. Do not implement from this checkpoint.

---

## GMUD F3.1.2-NEXT — Implementation prompts authored

After final F3.1.2 architecture ACCEPT, two constrained implementation prompts were authored. No ADO implementation was executed by this documentation checkpoint at the time.

```text
F3.1.2a implementation prompt: later EXECUTED (see F3.1.2a checkpoint below)
F3.1.1c implementation prompt: READY_FOR_EXPLICIT_LAUNCH
F3.1.2b implementation: NO-GO
F3.2 implementation: NO-GO
```

Prompts:
- `prompts/f3-1-2a-canonical-change-implementation.md`
- `prompts/f3-1-1c-cab-safe-policy-implementation.md`

Recommended order: F3.1.2a first, then F3.1.1c. Each implementation must stop at its own independent acceptance-review gate. F3.1.2b remains blocked until both prerequisites are implemented and independently accepted.

---

## GMUD F3.1.2a — Canonical Change Construction (implementation)

Implementation repository/branch/SHA: `platform-devops-developer-portal` /
`feat/ado-repo-governance` / **`ccee1e1676a2763e68880e5383ce1e5e48742843`**
(direct child of `188d8e9`; verified via HTTPS `git ls-remote` and Azure DevOps
REST `az repos ref list` both before and after the push).

Documentation baseline SHA (start): `b36c725b8355cf377ac5d4ac3f6afd7fd7f27778`.

Starting ADO baseline (verified live before any edit):
`188d8e9cc43423f3644b3cacfb9849257838a583` — exact match to the accepted
F3.1.1b implemented baseline. **Source drift: NONE.** Delivery branch was not
used.

Full evidence: [`f3-1-2a-implementation-evidence.md`](./f3-1-2a-implementation-evidence.md).

### Objective

Implement only F3.1.2a: one logical create builds the canonical Change at most
once; recovery reuses the durable pending index snapshot instead of regenerating
server-generated `activityId` / `createdAt`.

### Architecture applied

In `ChangeManagementService.createChange`:

- When no pending index exists: `buildChange()` once → `insertPending` → re-read
  durable snapshot.
- When pending index already exists: reuse `indexRecord.snapshot`; never call
  `buildChange()`.
- `finalizeCreate` always receives the durable snapshot.

No AuthorizationRuntime, ledger, Round, policy/selector, migration, route,
frontend, or `authorization_mode` change. `POST /changes` remains
`LEGACY_PRE_F3` with no `AuthorizationRound`.

### ADO files changed

```
packages/backend/src/modules/changeManagement/ChangeManagementService.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.test.ts
packages/backend/src/modules/changeManagement/ChangeManagementService.recovery.test.ts
```

### Tests / functional verification

- A1/A2/A3 focused proofs PASS; A4 F2 regressions PASS.
- Change Management module: 287 tests PASS (4 skipped live Catalog); SQLite +
  disposable PostgreSQL 16 PASS.
- Lint PASS; build PASS; TypeScript error set set-identical to `188d8e9` (5
  pre-existing Knex duplicate-type errors in `changeManagementPlugin.ts`).

### Deviations

None new. Carried forward: F3.1.1c CAB-safe policy still required before
F3.1.2b; production selector-bundle publication still a production-rollout
prerequisite; RBAC CSV files still F3.1.4.

### Gate

```text
F3.1.2a implementation: PASS
Architecture/implementation acceptance: PENDING SEPARATE REVIEW
F3.1.1c: not implemented
F3.1.2b: NO-GO
```

**STOP.** Do not implement F3.1.1c or F3.1.2b from this checkpoint.

---

## GMUD F3.1.2a-AR — Acceptance review prompt prepared

F3.1.2a is implemented/published at ADO `ccee1e1676a2763e68880e5383ce1e5e48742843`, but remains pending independent acceptance.

Canonical review prompt:
`prompts/f3-1-2a-architecture-implementation-acceptance.md`

Current gate:
```text
F3.1.2a implementation: PASS
F3.1.2a architecture/implementation acceptance: PENDING
F3.1.1c implementation: NOT STARTED
F3.1.2b implementation: NO-GO
```

The acceptance review must inspect the exact ADO diff independently and may only ACCEPT or REJECT; it must not repair code.

---

## GMUD F3.1.2a — Architecture / implementation acceptance (ACCEPT)

Documentation baseline SHA (review start): `a3c5c8b8299eb5ca5ba22ab13f8443bc52d9abcb`.

Implementation candidate reviewed:
`platform-devops-developer-portal@ccee1e1676a2763e68880e5383ce1e5e48742843`
(parent `188d8e9cc43423f3644b3cacfb9849257838a583`, exact match).

Independent ADO source verification: **YES** — HTTPS `git ls-remote` tip equals `ccee1e1`; complete diff `188d8e9..ccee1e1` inspected in isolated worktree; no later tip drift.

### Outcome

```text
F3.1.2a architecture/implementation acceptance: ACCEPT
F3.1.2a: CLOSED / ACCEPTED IMPLEMENTED BASELINE
Accepted ADO SHA: ccee1e1676a2763e68880e5383ce1e5e48742843
Gates: G1–G8 PASS
F3.1.1c implementation: still requires separate explicit authorization
F3.1.2b implementation: NO-GO until F3.1.1c is implemented and independently accepted
ADO implementation modified by this review: NO
```

Canonical review:
[`f3-1-2a-architecture-implementation-acceptance.md`](./f3-1-2a-architecture-implementation-acceptance.md).

Independent proofs re-run at exact SHA: focused A1/A2/A3 suites PASS; full Change Management module 287 tests PASS with disposable PostgreSQL 16; lint/build PASS; TypeScript error set set-identical to `188d8e9`.

**Next authorized activity:** F3.1.1c CAB-safe policy implementation only after separate explicit user launch of its authored prompt. Do not start F3.1.2b.

---

## GMUD F3.1.1c — CAB-safe policy publication (IMPLEMENTED / PUBLISHED)

Documentation baseline SHA (implementation start): `982bf0516dab64d9beeb0d3b4fc7dee663561a98`.

Implementation published:
`platform-devops-developer-portal@3b302ab5c9caab38f96491b389b7ea9fe0b66c2f`
(parent exact F3.1.2a tip `ccee1e1676a2763e68880e5383ce1e5e48742843`).

### Outcome

```text
F3.1.1c implementation: PASS
Historical policy modified: NO
New policy: default-change-authorization@2026-09-19.1
normal.low primary + CAB: PASS
medium/high/emergency unchanged: PASS
Publication manifest append-only: PASS
Active policy pin updated: YES
Active selector bundle changed: NO
CAB autonomy implemented: NO
Migrations added: NO
Publication validation: PASS (non-genesis vs ccee1e1)
Policy/selector regressions: PASS
Change Management regressions: PASS (301 tests)
Lint/Build: PASS
TypeScript baseline: LINE_IDENTICAL to ccee1e1
Remote ADO tip: 3b302ab5c9caab38f96491b389b7ea9fe0b66c2f
```

Canonical evidence:
[`f3-1-1c-implementation-evidence.md`](./f3-1-1c-implementation-evidence.md).

**Next authorized activity:** independent F3.1.1c architecture/implementation
acceptance review. Do **not** start F3.1.2b until that acceptance ACCEPTs.

---

## GMUD F3.1.1c-AR — Acceptance review prompt prepared

F3.1.1c is implemented/published at ADO `3b302ab5c9caab38f96491b389b7ea9fe0b66c2f`. The independent acceptance review was subsequently executed (see next checkpoint).

Canonical review prompt:
`prompts/f3-1-1c-architecture-implementation-acceptance.md`

---

## GMUD F3.1.1c — Architecture / implementation acceptance (ACCEPT)

Documentation review baseline: `backstage-docs@d3c4b13915afc462df25fadb7a4cb294db303d13`.

Reviewed ADO commit: `3b302ab5c9caab38f96491b389b7ea9fe0b66c2f`
Parent verified: exact `ccee1e1676a2763e68880e5383ce1e5e48742843`.

### Outcome

```text
F3.1.1c architecture/implementation acceptance: ACCEPT
F3.1.1c: CLOSED / ACCEPTED IMPLEMENTED BASELINE
Accepted ADO SHA: 3b302ab5c9caab38f96491b389b7ea9fe0b66c2f
F3.1.2a: CLOSED / ACCEPTED IMPLEMENTED BASELINE
F3.1.2b implementation-prompt authoring: GO
F3.1.2b implementation: still requires separate explicit authorization
F3.2 implementation: NO-GO
Gates: G1–G8 PASS
ADO implementation modified by this review: NO
```

Canonical review:
[`f3-1-1c-architecture-implementation-acceptance.md`](./f3-1-1c-architecture-implementation-acceptance.md).

Independent proofs re-run at exact SHA: non-genesis publication validation PASS; focused policy/selector/architecture 120 tests PASS; full Change Management module 306 tests PASS with disposable PostgreSQL 16; lint/build PASS; TypeScript error locations identical to `ccee1e1`.

**Next authorized activity:** author a constrained F3.1.2b implementation prompt from the already accepted F3.1.2 implementation contract. Do **not** implement F3.1.2b, F3.1.3, F3.1.4, or F3.2 inside this gate.

---

## Product convergence checkpoint — Deployments missing from active GMUD line

User-observed product issue: the previously accepted Catalog Component **Deployments** tab is not present in the currently running product based on the active GMUD line.

Canonical evidence shows the accepted Deployments UX lived on the separate ADO workstream `feat/delivery-mvp-slice` (`0163a49` UX implementation; `b08e7b2` responsive polish), while the current GMUD/F3 line is `feat/ado-repo-governance` (`3b302ab` accepted F3.1.1c baseline). Live ADO verification confirmed **BRANCH_DIVERGENCE** (merge-base `4bad41d`; Deployments paths absent on governance tip).

Prepared canonical convergence prompt:
`prompts/product-convergence-deployments.md`

### Result — PASS (published)

- Root cause: `BRANCH_DIVERGENCE`
- Implementation: ADO `feat/ado-repo-governance@f48dc825ab5d1d16fafc3f70ef772d28613df1a0` (parent `3b302ab`)
- Minimal accepted Deployments UX/runtime restored; Delivery branch not merged wholesale
- `api:catalog/delivery` collision guard preserved; POST `/changes` remains `LEGACY_PRE_F3`
- No production/sandbox hardening, secrets, or GitOps/Kargo infra imported
- Evidence: [`product-convergence-deployments-evidence.md`](./product-convergence-deployments-evidence.md)

Gate:
```text
Product convergence (GMUD + Deployments): PASS
Active product branch: feat/ado-repo-governance @ f48dc82
F3.1.2a: CLOSED / ACCEPTED
F3.1.1c: CLOSED / ACCEPTED
F3.1.2b implementation-prompt authoring: GO
F3.1.2b implementation: still requires separate explicit authorization
F3.2: NO-GO
```

---

## GMUD F3.1.2b-NEXT — Ledger submission implementation prompt authored

After product convergence PASS at ADO `f48dc825ab5d1d16fafc3f70ef772d28613df1a0`, the constrained F3.1.2b implementation prompt is now authored.

Canonical prompt:
`prompts/f3-1-2b-ledger-submission-implementation.md`

Gate:
```text
F3.1.2a: CLOSED / ACCEPTED
F3.1.1c: CLOSED / ACCEPTED
Product convergence (GMUD + Deployments): PASS
F3.1.2b implementation prompt: READY_FOR_EXPLICIT_LAUNCH
F3.1.2b implementation: NOT STARTED
Operational LEDGER_REQUIRED cutover: NOT AUTHORIZED
F3.1.3/F3.1.4: NO-GO
F3.2: NO-GO
```

The implementation contract explicitly reconciles the accepted Deployments convergence drift, preserves the execution-eligibility route and `api:catalog/delivery` collision fix, and requires the committed runtime default to remain `LEGACY_PRE_F3`. LEDGER behavior is proven through controlled tests; production/runtime cutover is a later explicit gate.

---

## GMUD F3.1.2b — Ledger-governed submission integration (IMPLEMENTED / PUBLISHED)

Explicit launch of `prompts/f3-1-2b-ledger-submission-implementation.md` against docs `b3ea5ee` and ADO parent `f48dc82`.

- Implementation: ADO `feat/ado-repo-governance@22495229502dabf2d99588599a156d862c5114fa` (parent `f48dc82`)
- Stored-mode-wins orchestration; repository explicit mode mismatch remains CONFLICT
- Committed default `newSubmissionAuthorizationMode: LEGACY_PRE_F3`; LEDGER_REQUIRED only in controlled tests
- CAB-safe `normal.low` materializes primary + CAB; `requirementId = requirementRole`
- Duplicate `requirementRole` fails at policy registration
- Emergency SoD fail-closed; caller-owned DevelopmentProvider transaction; healthy Round-1 loser convergence
- PostgreSQL C1/C2 authoritative concurrency PASS
- Deployments/Catalog/GMUD regressions PASS
- Evidence: [`f3-1-2b-implementation-evidence.md`](./f3-1-2b-implementation-evidence.md)

Gate:

```text
F3.1.2b implementation: PASS / PUBLISHED
Independent F3.1.2b architecture/implementation acceptance: PENDING
Committed default newSubmissionAuthorizationMode: LEGACY_PRE_F3
Operational LEDGER_REQUIRED cutover: NOT AUTHORIZED
F3.1.3/F3.1.4: NO-GO
F3.2: NO-GO
```

---

## GMUD F3.1.2b-AR — Independent acceptance review prompt prepared

F3.1.2b is implemented/published at ADO `22495229502dabf2d99588599a156d862c5114fa` (parent `f48dc82`) and remains pending independent architecture/implementation acceptance.

Canonical review prompt:
`prompts/f3-1-2b-architecture-implementation-acceptance.md`

Current gate:
```text
F3.1.2a: CLOSED / ACCEPTED
F3.1.1c: CLOSED / ACCEPTED
Product convergence: PASS
F3.1.2b implementation: PASS / PUBLISHED
F3.1.2b architecture/implementation acceptance: PENDING
Committed default: LEGACY_PRE_F3
Operational LEDGER_REQUIRED cutover: NOT AUTHORIZED
F3.1.3/F3.1.4/F3.2: NO-GO
```

The review must independently inspect `f48dc82..2249522`, re-prove the PostgreSQL concurrency contract, transaction atomicity, CAB-safe requirement materialization, stored-mode-wins behavior, product-convergence preservation, and the mandatory rollback gate. It may only ACCEPT or REJECT and must not modify ADO code or activate LEDGER_REQUIRED.

---

## GMUD F3.1.2b — Architecture / implementation acceptance (ACCEPT)

Documentation review baseline: `backstage-docs@06c337aa7598a3de178ef84cbfcf061258a538ce`.

Reviewed ADO commit: `22495229502dabf2d99588599a156d862c5114fa`
Parent verified: exact `f48dc825ab5d1d16fafc3f70ef772d28613df1a0`.

### Outcome

```text
F3.1.2b architecture/implementation acceptance: ACCEPT
F3.1.2b: CLOSED / ACCEPTED IMPLEMENTED BASELINE
Accepted ADO SHA: 22495229502dabf2d99588599a156d862c5114fa
F3.1.2: CLOSED / ACCEPTED IMPLEMENTED SUBMISSION BASELINE
Committed default newSubmissionAuthorizationMode: LEGACY_PRE_F3
Operational LEDGER_REQUIRED cutover: NOT YET AUTHORIZED
F3.1.3 planning/prompt authoring: GO
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
Gates: G1–G19 PASS
ADO implementation modified by this review: NO
Operational LEDGER_REQUIRED cutover performed: NO
```

Canonical review:
[`f3-1-2b-architecture-implementation-acceptance.md`](./f3-1-2b-architecture-implementation-acceptance.md).

Independent proofs re-run at exact SHA: focused ledgerSubmit 30 tests PASS including disposable PostgreSQL 15 C1/C2/C3; Change Management + Delivery 374 tests PASS; Deployments frontend 20 tests PASS; GMUD frontend 58 tests PASS; lint/build PASS; TypeScript remains the historical five dual-package Knex errors in `changeManagementPlugin.ts`.

**Next authorized activity:** author a narrow LEDGER_REQUIRED activation/cutover checkpoint for the intended non-production product environment, then independently verify it, then plan/author F3.1.3 decision-command work. Do **not** implement F3.1.3, F3.1.4, or F3.2, and do **not** flip the committed default, inside this gate.

---

## GMUD F3.1.2-CUTOVER-NEXT — Non-production LEDGER_REQUIRED activation prompt authored

F3.1.2b and the complete F3.1.2 submission baseline are CLOSED / ACCEPTED at ADO `22495229502dabf2d99588599a156d862c5114fa`. The accepted binary still commits `newSubmissionAuthorizationMode: LEGACY_PRE_F3` by default.

Canonical operational prompt:
`prompts/f3-1-2-ledger-required-nonprod-activation.md`

Gate:
```text
F3.1.2: CLOSED / ACCEPTED IMPLEMENTED SUBMISSION BASELINE
Committed repository default: LEGACY_PRE_F3
Non-production LEDGER_REQUIRED activation: NOT STARTED
Operational production cutover: NOT AUTHORIZED
F3.1.3 planning/prompt authoring: GO
F3.1.3 implementation: NO-GO pending explicit prompt and preferably accepted live ledger target
F3.1.4: NO-GO
F3.2: NO-GO
```

The activation checkpoint is operational only: it must discover one real isolated non-production Backstage target, use an existing environment-specific override to activate `LEDGER_REQUIRED`, prove a real normal-low Round 1 (primary + CAB), idempotent replay, legacy-reservation continuity across the cutover, same-binary config backout, and preservation of GMUD/Catalog/Deployments. The committed repository default remains `LEGACY_PRE_F3`; production and source-code changes are forbidden.

---

## GMUD F3.1.2-CUTOVER — Non-production LEDGER_REQUIRED activation executed

Documentation baseline at start: `backstage-docs@80a68153701582745eef784811831576808d8bbd`.

Runtime: operator laptop `yarn start` at accepted ADO SHA `22495229502dabf2d99588599a156d862c5114fa` (independent `az repos ref list` match; source drift NONE). Isolated SQLite `packages/backend/data/change-management.sqlite`. Activation via gitignored `app-config.local.yaml` overlay only.

```text
Activation checkpoint: PASS
Independent activation acceptance: PENDING
Committed repository default: LEGACY_PRE_F3
Final laptop overlay: LEDGER_REQUIRED
Live proofs: CHG-2026-000002 LEGACY control; CHG-2026-000003 LEDGER Round 1 primary+CAB;
  idempotent replay; HTTP 409 conflict; stored-mode-wins; CHG-2026-000004 same-binary backout
Old-binary downgrade / source / migrations / production: NO
F3.1.3 implementation: NO-GO
```

Canonical evidence:
[`f3-1-2-ledger-required-nonprod-activation-evidence.md`](./f3-1-2-ledger-required-nonprod-activation-evidence.md).

**Next authorized activity:** independent activation-acceptance review. Do **not** implement F3.1.3, F3.1.4, or F3.2, and do **not** flip the committed default, inside this gate.

---

## GMUD F3.1.2-CUTOVER-AR — Independent activation acceptance prompt prepared

Operational non-production `LEDGER_REQUIRED` activation executed with PASS on the isolated operator-laptop Backstage runtime at accepted ADO `22495229502dabf2d99588599a156d862c5114fa`.

Canonical review prompt:
`prompts/f3-1-2-ledger-required-nonprod-activation-acceptance.md`

Current gate:
```text
F3.1.2: CLOSED / ACCEPTED IMPLEMENTED SUBMISSION BASELINE
Activation execution: PASS
Independent activation acceptance: PENDING
Final laptop overlay: LEDGER_REQUIRED
Committed repository default: LEGACY_PRE_F3
Production cutover: NOT AUTHORIZED
F3.1.3 implementation: NO-GO pending acceptance + demo-target readiness
F3.1.4/F3.2: NO-GO
```

The independent review must verify the live/durable activation facts rather than accepting the evidence document at face value, and must separately classify whether the laptop SQLite runtime is adequate as the F3.1.3 product-validation target (`READY` or `NOT_READY`).

---

## GMUD F3.1.2-CUTOVER-ACCEPT — Non-production LEDGER_REQUIRED activation accepted

Documentation review baseline: `backstage-docs@55735d1282b390ab968f16f5bb612282bfe46a2c`.

Reviewed runtime: operator laptop at accepted ADO `22495229502dabf2d99588599a156d862c5114fa` (independent `az repos ref list` match; source drift NONE). Isolated SQLite inspected read-only; same binary independently restarted with the existing gitignored overlay.

```text
F3.1.2 non-production LEDGER_REQUIRED activation acceptance: ACCEPT
Activation baseline: ACCEPTED
Accepted runtime SHA: 22495229502dabf2d99588599a156d862c5114fa
Committed repository default: LEGACY_PRE_F3
Accepted non-production effective mode: LEDGER_REQUIRED
Production cutover: NOT AUTHORIZED
F3.1.3 demo target readiness: READY
F3.1.3 planning/prompt authoring: GO
F3.1.3 implementation: still requires separate explicit authorization
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
Gates: G1–G15 PASS
Independent runtime/database verification: YES
ADO/source modified by review: NO
Production modified by review: NO
```

Canonical review:
[`f3-1-2-ledger-required-nonprod-activation-acceptance.md`](./f3-1-2-ledger-required-nonprod-activation-acceptance.md).

Independent proofs: live startup `LEDGER_REQUIRED`; SQLite identities `CHG-2026-000002` LEGACY / `CHG-2026-000003` LEDGER Round 1 primary+CAB / `CHG-2026-000004` backout LEGACY; canonical five-event audit set; original 201 replay + 409 conflict logs with no duplicate artifacts; stored-mode-wins; pending-LEDGER drain `0`; UI GMUD/Catalog/Deployments preserved; live eligibility `DENY` / `PENDING_AUTHORIZATION` / `roundNumber=1` without fabricating decisions.

**Next authorized activity:** F3.1.3 planning/prompt authoring. Do **not** implement F3.1.3, F3.1.4, or F3.2, and do **not** flip the committed default or perform production cutover, inside this gate.

---

## GMUD F3.1.3-NEXT — Decision/new-round planning prompt authored

After independent ACCEPT of the non-production `LEDGER_REQUIRED` activation, the operator-laptop runtime is the accepted F3.1.3 demonstration target (`READY`).

Canonical planning prompt:
`prompts/f3-1-3-planning.md`

Gate:
```text
F3.1.2: CLOSED / ACCEPTED
Non-prod LEDGER_REQUIRED activation: ACCEPT
F3.1.3 demo target readiness: READY
F3.1.3 planning prompt: READY_FOR_EXPLICIT_LAUNCH
F3.1.3 implementation: NO-GO
F3.1.4: NO-GO
F3.2: NO-GO
Production cutover: NOT AUTHORIZED
```

The planning checkpoint must resolve decision-command transport and permissions, individual vs CAB/authority decision authorization, decision idempotency/concurrency, transaction/audit semantics, rejection lifecycle, post-execution requirement timing, and whether same-changeId resubmission/new-round semantics need a separate F3.1.3b micro-slice or migration. It may not modify ADO code.

---

## GMUD F3.1.3 — Decision command and new-round planning (READY_FOR_REVIEW)

Documentation review baseline at this execution: `backstage-docs@7725217abb7648de237ecb653c31ec458c2e8754` (published planning contract + prior draft). Independent source re-verification confirmed the draft against live ADO `2249522` and the laptop LEDGER facts, and made the trx-aware ledger-read contract explicit.

ADO source independently verified at `platform-devops-developer-portal@22495229502dabf2d99588599a156d862c5114fa` (local HEAD, `origin/feat/ado-repo-governance`, and `az repos ref list` `objectId`). Source drift after accepted F3.1.2: NONE. Laptop overlay remains `LEDGER_REQUIRED`; committed default remains `LEGACY_PRE_F3`. `CHG-2026-000003` re-read read-only: Round 1 primary + CAB, zero decisions, five canonical audits. Catalog `relations.memberOf` proves `user:default/diego.fernandes_outlook.com` is a member of `group:default/cloud_azure_devops_platform_devops`. This checkpoint created zero decisions.

Canonical plan:
[`f3-1-3-implementation-plan.md`](./f3-1-3-implementation-plan.md)

```text
F3.1.3 planning: READY_FOR_REVIEW
F3.1.3 implementation: NO-GO
Migration required: NO
Recommended slices: F3.1.3a + F3.1.3b
Decision command transport: RESOLVED
Individual authority contract: RESOLVED
CAB/authority membership contract: RESOLVED
Permission boundary: RESOLVED
Decision idempotency/concurrency: RESOLVED
Decision transaction/audit: RESOLVED
Rejection lifecycle: RESOLVED
Post-execution requirement behavior: RESOLVED
Resubmission/new-round contract: RESOLVED
ADO implementation modified: NO
```

F3.1.3a owns the server-authoritative decision command, live Catalog CAB membership, dedicated decide vs cab.record permissions, idempotency/concurrency, caller-owned transaction/audit with **trx-aware ledger reads**, derived evaluation transitions, and rejection lifecycle projection without inventing `authorized`. F3.1.3b owns same-`changeId` resubmission / Round N after terminal rejection. No DDL. No ApprovalDecision fact was created by this checkpoint.

**Next authorized activity:** independent architecture review of the F3.1.3 plan. Do **not** author an implementation prompt, implement F3.1.3/F3.1.4/F3.2, fabricate decisions, or perform production cutover inside this gate.

---

## GMUD F3.1.3-AR — Independent plan architecture review prepared

F3.1.3 planning is complete and source-reverified against ADO `22495229502dabf2d99588599a156d862c5114fa` with `Migration required: NO` and recommended slices `F3.1.3a + F3.1.3b`.

Canonical review prompt:
`prompts/f3-1-3-plan-architecture-review.md`

Current gate:
```text
F3.1.3 planning: READY_FOR_REVIEW
F3.1.3 plan architecture review: PENDING
F3.1.3 implementation: NO-GO
F3.1.3a implementation-prompt authoring: NO-GO pending ACCEPT
F3.1.3b: NO-GO
F3.1.4: NO-GO
F3.2: NO-GO
Production cutover: NOT AUTHORIZED
```

The review explicitly challenges the plan's most consequential choices: decision-time CAB membership + RBAC, global actor-scoped decision idempotency, PostgreSQL transaction visibility/trx-aware ledger reads, exactly-once authorization/rejection audit milestones without new DDL, rejected lifecycle projection under Model C, post-execution anchor fail-closed behavior, resubmission actor authority, same-changeId target/owner correction boundaries, and transactional `DevelopmentProvider.replaceCurrent`.

---

## GMUD F3.1.3-AR — Plan architecture review REJECT

Independent review-only checkpoint of [`f3-1-3-implementation-plan.md`](./f3-1-3-implementation-plan.md) against live ADO `feat/ado-repo-governance` at accepted SHA `22495229502dabf2d99588599a156d862c5114fa`.

Canonical review:
[`f3-1-3-plan-architecture-review.md`](./f3-1-3-plan-architecture-review.md)

Docs baseline reviewed: `diegofernandes-dev/backstage-docs@915bd93f96ad763b4ed54621f698eba41d3ec99c` (`origin/main` at review start). Planning document preserved.

Independent ADO verification: **YES** — local HEAD + `az repos ref list` `objectId` `2249522`; no later drift. Laptop LEDGER facts re-read only; zero `ApprovalDecision` rows created.

```text
F3.1.3 plan architecture review: REJECT
F3.1.3 plan: REVISION REQUIRED
Migration required: NO
Gates: 18/20 PASS
Challenges answered: 12/12
F3.1.3a implementation-prompt authoring: NO-GO
F3.1.3a implementation: NO-GO
F3.1.3b implementation: NO-GO
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
Production cutover: NOT AUTHORIZED
ADO implementation modified: NO
```

PASS gates include source inventory, slice split, decision transport, individual/CAB authority, RBAC, decision idempotency, caller-owned trx + trx-aware ledger reads, exactly-once milestones without DDL, rejection lifecycle projection, eligibility safety, post-execution fail-closed, provider `replaceCurrent`, resubmission idempotency/atomicity, round terminality, no-migration, PostgreSQL D1–D6/R1–R3, and live-product/scope.

Blockers (F3.1.3b only; do not redesign accepted F3.1.2 or the passing 3a contracts):

1. **G13** — requester / current `ownerRef` member / `platform_admin` plus `change.create` is not already granted by ADR-009. Cancellation actors and participant-read are different capabilities. Do not silently bless this set.
2. **G14** — allowing every user-editable create field, including `targetRef` / owner / System, can still represent a fundamentally different business Change under the same `changeId`. Model C current-projection vs immutable Round history remains coherent (G15 PASS); the identity boundary is not.

**Next authorized activity:** narrow F3.1.3 plan correction of G13 and G14 only, then independent re-review. Do **not** author an implementation prompt, implement F3.1.3/F3.1.4/F3.2, fabricate decisions, or perform production cutover inside this gate.

---

## GMUD F3.1.3-REV — Narrow plan correction prepared after REJECT

Independent F3.1.3 plan review returned `REJECT` with **18/20 gates PASS**. The accepted/source-accurate F3.1.3a decision-command contract is preserved. Only two F3.1.3b governance contracts require correction:

1. explicit resubmission actor authority;
2. same-changeId identity boundary preventing target/owner/System retargeting.

Canonical correction prompt:
`prompts/f3-1-3-plan-revision.md`

Planned bounded decisions:
```text
resubmit permission + (original requester OR current member of immutable ownerRef)
platform_admin alone: NO
CAB alone: NO
targetRef across same changeId: IMMUTABLE
ownerRef/systemRef across same changeId: IMMUTABLE
retargeting: new Change/new changeId
```

The correction prompt also requires ADR-014, Catalog-ownership-drift semantics, hidden execution-plan retarget protection, and a fresh independent re-review. No ADO implementation is authorized.

---

## GMUD F3.1.3-REV — Narrow plan correction READY_FOR_REREVIEW

Documentation-only checkpoint against `backstage-docs@a94c2e266b8aa6cff618bc82889abf2779b71cc4` and ADO `feat/ado-repo-governance@22495229502dabf2d99588599a156d862c5114fa` (`git fetch` confirmed no later drift).

Canonical ADR:
[`../adr/ADR-014-change-resubmission-authority-and-identity-boundary.md`](../adr/ADR-014-change-resubmission-authority-and-identity-boundary.md)

Revised plan:
[`f3-1-3-implementation-plan.md`](./f3-1-3-implementation-plan.md)

Historical REJECT preserved:
[`f3-1-3-plan-architecture-review.md`](./f3-1-3-plan-architecture-review.md)

```text
F3.1.3 plan revision: READY_FOR_REREVIEW
F3.1.3 implementation: NO-GO
F3.1.3a implementation-prompt authoring: NO-GO pending fresh ACCEPT
F3.1.3a implementation: NO-GO
F3.1.3b implementation: NO-GO
Migration required: NO
ADO implementation modified: NO
```

Closed in this revision (F3.1.3b only):

1. **G13 / ADR-014** — dedicated server permission `change-management.change.resubmit` **and** (original `requestedBy` **or** current live Catalog member of the immutable `ownerRef`). `platform_admin` / CAB / `cab.record` / participant-read / `change.create` are not standalone resubmission authority. Owner proof is prefix-agnostic live `memberOf`, fail-closed on unavailable membership source.
2. **G14 / ADR-014** — same-`changeId` identity freezes `targetRef`, original `ownerRef`, original `systemRef`, `requestedBy`, and identity `createdAt`. Retargeting requires a new Change. Catalog ownership drift does not transfer rights. Activity `targetRef` cannot cross the Change System identity (source schema can express that path).

F3.1.3a contracts were not redesigned. Added proofs R4–R13. Source drift: **NONE**. Dedicated resubmit permission is wireable with existing `createPermission` / registry / CSV; `change.create` was not reused.

**Next authorized activity:** independent F3.1.3 plan re-review. Do **not** author an implementation prompt, implement F3.1.3/F3.1.4/F3.2, fabricate decisions, or perform production cutover inside this gate.
