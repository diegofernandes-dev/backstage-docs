# Backstage implementation / construction progress

> **Canonical documentation bridge:** `diegofernandes-dev/backstage-docs` (`main`)  
> **Implementation source of truth:** Azure DevOps `platform-devops-developer-portal`  
> **Active implementation branch:** `feat/ado-repo-governance`  
> **Migration baseline:** legacy bridge `diegofernandes-dev/poc-teams-approval@fe4f8073f2a8785673e32ce51e5f70b7c322ad68`  
> **Current GMUD implementation baseline:** F3.1.0 — ADO `7663883` (full SHA `766388393458f82fbdc2e0502b8c193d0a85e605`), published `feat/ado-repo-governance`  
> **Current GMUD architecture baseline:** F3.0.1 — ADR-009 Accepted; F3.1.0 published as ACCEPTED IMPLEMENTED BASELINE; F3.1.1 planning authorized, implementation not authorized

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
