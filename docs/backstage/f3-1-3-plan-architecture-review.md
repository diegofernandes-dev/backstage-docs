# F3.1.3 — Decision / New-Round Plan Architecture Review

## 1. Status / verdict

```text
F3.1.3 plan architecture review: REJECT
F3.1.3 plan: REVISION REQUIRED
Migration required: NO
F3.1.3a implementation-prompt authoring: NO-GO
F3.1.3a implementation: NO-GO
F3.1.3b implementation-prompt authoring: NO-GO
F3.1.3b implementation: NO-GO
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
Production cutover: NOT AUTHORIZED
```

The published plan is a strong, source-accurate F3.1.3a decision-command contract
and a coherent Model C composition for current projection vs immutable Round
history. It is **not** an implementation-ready contract because two F3.1.3b
governance-authority decisions remain open:

1. **Resubmission actor authority is invented.** ADR-009 defines same-`changeId`
   resubmission semantics but does not grant `change.create` plus requester /
   current `ownerRef` member / `platform_admin`. That set must not be silently
   blessed.
2. **Same-changeId identity boundary is unsound.** Allowing every user-editable
   create field — including `targetRef` / `ownerRef` / `systemRef` — can still
   represent a fundamentally different business Change under the same `changeId`.

F3.1.3a decision-command contracts (transport, individual/CAB authority, RBAC,
trx-aware reads, milestones, rejection projection, post-execution fail-closed,
PostgreSQL proofs D1–D6) independently match live source. They are not rewritten
by this REJECT. The next activity is the **narrowest 3b plan correction**, not a
3a redesign.

This review is documentation-only. No ADO code was modified. No
`ApprovalDecision` fact was created. No implementation prompt was authored.

---

## 2. Reviewed docs + ADO baselines

| Item | Value |
|---|---|
| Docs fetched (`main`) | `diegofernandes-dev/backstage-docs@915bd93f96ad763b4ed54621f698eba41d3ec99c` |
| Plan under review | `docs/backstage/f3-1-3-implementation-plan.md` |
| Plan’s claimed docs baseline | `7725217abb7648de237ecb653c31ec458c2e8754` |
| Review prompt | `prompts/f3-1-3-plan-architecture-review.md` |
| ADRs | ADR-006, ADR-007 (Model C), ADR-008, ADR-009 as partially superseded by ADR-013, ADR-013 |
| Prior plans / acceptances | F3.1, F3.1.2, F3.1.2b ACCEPT, non-prod LEDGER_REQUIRED ACCEPT |
| ADO baseline verified | `platform-devops-developer-portal@22495229502dabf2d99588599a156d862c5114fa` |
| ADO branch tip | `origin/feat/ado-repo-governance` = `2249522` (no post-baseline drift) |
| Independent source verification | **YES** |
| Laptop LEDGER facts | Read-only supporting evidence only; **zero** decisions created |

SSH `git fetch` to Azure DevOps failed in-session. Remote tip was confirmed
independently via `az repos ref list`
(`refs/heads/feat/ado-repo-governance` `objectId` =
`22495229502dabf2d99588599a156d862c5114fa`). Local HEAD already contained that
exact SHA.

Working tree (ADO): untracked `.vscode/` only. Overlay and SQLite remain
gitignored. Committed `app-config.yaml` still
`newSubmissionAuthorizationMode: LEGACY_PRE_F3`.

---

## 3. Independent source-verification scope

Inspected at exact SHA `2249522`:

| Surface | Paths / confirmation |
|---|---|
| HTTP | `changeManagementPlugin.ts` — create/list/get/eligibility only; no decision/resubmit route |
| Service | `ChangeManagementService` — `createChange` / `listChanges` / `getChange`; architecture guard forbids `appendDecision` |
| Status | `ChangeStatus = 'submitted'` only (backend + frontend `STATUS_LABELS`) |
| Index | `KnexChangeIndexRepository.finalize()` updates `is_finalized` / `external_*` only; no status-update method; `authorization_mode` trigger-immutable; `status` string(32) with **no CHECK** |
| Provider | `IChangeManagementProvider` = `create`/`get`; `DevelopmentProvider.createWithTransaction` **upserts** `record_json` when `change_id` exists |
| Ledger writes | `createRound(trx?)`, `appendDecision(trx?)`, `appendAuditEvent(trx?)` |
| Ledger reads | `findRound` / `findCurrentRound` / `listRequirements` / `listAuditEvents` always use `this.knex`; current round = `ORDER BY round_number DESC LIMIT 1`; `appendDecision` does not lock; `FOR UPDATE` only inside `insertRound` |
| Decision storage | unique `(change_id, round_number, requirement_id)`; unique `(actor_ref, idempotency_key)`; rejection CHECK; **no** unique on audit `(change_id, round_number, event_type)` |
| Domain | `ApprovalDecision.actingAuthorityRef` (not `authorityRef`); `authorizationEvidence?: Record<string, unknown>` |
| Evaluators | `evaluateAuthorization` / `evaluateGovernance` pure; no `executionCompletedAt` producer in service |
| Eligibility | `ensureRound` still fabricates sandbox Round 1 from `sandbox-policy-v1.json` when no round exists; window from **index** snapshot |
| Identity | `ownershipEntityRefs` = user ref + Entra TP groups prefixed `group:default/cloud_azure_devops_` only |
| Catalog membership | live User `diego.fernandes_outlook.com`: `spec.memberOf` and `relations.memberOf` both `group:default/cloud_azure_devops_platform_devops` |
| Selector | `CatalogPrincipalResolver` service credentials; no member expansion; architecture test forbids `memberOf` / `getEntities` / `.relations` in selector sources |
| CAB / primary bindings | `app-config.yaml` `cab-authority` → `group:default/cloud_azure_devops_platform_devops`; `normal-primary-approver` → `user:default/diego.fernandes_outlook.com` |
| Permissions / RBAC | only `.create` / `.read`; CSV grants both to contributor and `platform_admin`; CAB group also has `platform_admin` |
| Idempotency | operation `change.create` only; PK `(operation, requested_by, idempotency_key)`; `reserve(trx?)` already exists; `operation` varchar(64) |
| Unique helper | `ledgerSubmission.ts` treats Postgres `23505` and SQLite unique/PK/serialization codes as contention |
| Round-1 audits | `ledgerSubmission.ts` hardcodes `roundNumber: 1` in requirement/audit payloads |
| GET detail | `getChange` returns `toPublicChange(provider.get())` with no index-status overlay |
| Execution completion | **none** in Change Management service/status/audit |

Laptop SQLite re-read (read-only; unchanged by this review):

| changeId | mode | rounds | decisions | notes |
|---|---|---|---|---|
| `CHG-2026-000001` | `LEDGER_REQUIRED` | 1 sandbox | 1 historical | Untouched |
| `CHG-2026-000002` | `LEGACY_PRE_F3` | 0 | 0 | Control |
| `CHG-2026-000003` | `LEDGER_REQUIRED` | 1 CAB-safe | **0** | Demo target; five canonical audits |
| `CHG-2026-000004` | `LEGACY_PRE_F3` | 0 | 0 | Same-binary backout |

Source drift after accepted F3.1.2: **NONE**.

---

## 4. G1–G20 matrix

| Gate | Result | Summary |
|---|---|---|
| G1 Source inventory | **PASS** | Every claimed current-source limitation/capability is correct at `2249522`. No assumed API that does not exist. Trx-aware **read** gap is real and correctly identified. |
| G2 Slice decomposition | **PASS** | Decision recording vs same-`changeId` resubmission are distinct persistence/composition problems. 3a can be accepted without 3b. 3b is correctly gated on 3a. No F3.1.4/F3.2 smuggling. |
| G3 Decision transport | **PASS** | Nested POST, required Idempotency-Key, server-owned actor/hash/evidence, 201 first-commit / 200 exact replay, existing error codes. Returning `ApprovalDecision` on the **command response** is bounded (plan-specified evidence; no table names; not an F3.1.4 read model). |
| G4 Individual authority | **PASS** | Exact `actor.userEntityRef == resolvedPrincipalRef`. No requester/owner/`platform_admin` override. Read/create do not imply decide. |
| G5 CAB membership | **PASS** | Live Catalog `memberOf` against snapshotted Group; snapshot remains historical identity; prefix-agnostic (explicitly not `ownershipEntityRefs`); Catalog error → `PROVIDER_UNAVAILABLE`; missing user → `FORBIDDEN`; one collective decision; `actingAuthorityRef` truthful. `relations.memberOf` plus `spec.memberOf` matches the actual User entity shape (verified live). Implementation must `stringifyEntityRef`/dedupe; fail-closed `includes` is acceptable. New `decisionMembership.ts` correctly stays outside selector sources. |
| G6 Permission/RBAC | **PASS** | Distinct `decide` vs `cab.record`. `platform_admin` does not receive `cab.record`. CSV `g, group:…, role:default/change_cab_recorder` matches existing Casbin syntax. Granting `decide` to contributor/`platform_admin` is least-privilege-enough **because** Q2 equality remains mandatory; RBAC cannot know the selector-resolved user. Two-layer (permission + domain proof) is required. |
| G7 Decision idempotency | **PASS** | Requirement unique + actor-scoped key; exact replay; payload mismatch; different-key conflict; concurrent approve/reject; unique-loser rollback then re-observe. Global `(actor_ref, idempotency_key)` is explicit and acceptable: callers mint a fresh key per requirement. |
| G8 Transaction / trx-aware read | **PASS** | PostgreSQL `READ COMMITTED` visibility gap is real: `listRequirements` joins decisions on `this.knex` and would miss the uncommitted row. Optional `trx` on the named reads is sufficient; no DDL. One caller-owned txn owns lock, reads, insert, audits, optional status projection. Unique-violation loser must rollback before re-observe. Catalog I/O stays outside the lock. |
| G9 Milestone audit | **PASS** | No audit `(change_id, round_number, event_type)` unique exists. Exactly-once via `change_index FOR UPDATE` + trx-aware audit exists-check is sound. Current writers of `authorization_reached` / `round_rejected` do not exist; F3.1.2b submission emits other event types only; `EligibilityService.ensureRound` does not emit these milestones. All planned emitters serialize on the same parent lock. |
| G10 Rejection lifecycle | **PASS** | Canonical fact = rejecting decision + `round_rejected`; `change_index.status='rejected'` is a rebuildable projection; no CHECK blocks the value; list already reads index status; GET overlay of **status only** from index is required and must not rewrite provider `record_json` in 3a; no `authorized` lifecycle value. Overlaying status onto LEGACY rows is a no-op (`submitted`). Do not overlay non-status index fields in 3a. |
| G11 Eligibility safety | **PASS** | Disabling sandbox fabrication for `LEDGER_REQUIRED` is correct and fail-closed (`NO_LEDGER_ROUND`). Historical `CHG-2026-000001` already has a round, so evidence is preserved. Post-3b window authority = current Round snapshot; index remains discovery projection. Decision commands never call `ensureRound`. |
| G12 Post-execution timing | **PASS** | Generic command + `execution_completion_required` is implementable. Source has **no** accepted completion event, lifecycle `completed`, or sandbox completion surrogate in Change Management. The requirement row and sandbox `CHG-2026-000001` decision must not count. |
| G13 Resubmission authority | **FAIL** | See §5. Actor set is not already justified by accepted architecture. |
| G14 Same-changeId snapshot | **FAIL** | See §6. Unconstrained `targetRef` / owner / System under the same `changeId` is not an implementation-ready identity contract. |
| G15 Provider replacement | **PASS** | Explicit `DevelopmentProvider.replaceCurrent(change, trx)` is a valid bounded Model C operation. Reusing `create()` / `createWithTransaction()` would look like create-retry of a different body (source upserts `record_json`). Same Knex client: replacement rolls back with the caller-owned transaction. External providers fail closed until a later slice. Model C remains coherent: Round snapshots are immutable history; index is rebuildable **current** discovery; provider is current operational detail. This PASS does **not** rescue G14’s identity boundary. |
| G16 Resubmission idempotency | **PASS** | `change.resubmit` fits `operation` varchar(64); actor-scoped PK; payload hash includes `changeId` + corrected body; reserve/complete inside the locked trx prevents dangling pending rows; original `change.create` reservation untouched; two-actor race serializes on `FOR UPDATE` + expected round number. After lock, a lost race whose max round already advanced must map to `resubmission_conflict` (not a non-deterministic mix with `round_not_terminal`). |
| G17 Round terminality | **PASS** | Resubmit only when `evaluateAuthorization() === 'REJECTED'`. `AUTHORIZED` is not resubmittable. Undecided pre requirements cannot repair a rejected round. Post-execution rejection is `GovernanceEvaluation`, not Authorization `REJECTED`, and does not make a pre-rejected round resubmittable. `terminal_round` vs Q10 for undecided post on an `AUTHORIZED` round is resolved. |
| G18 Migration | **PASS** | Existing uniqueness, append-only triggers, round PK, unconstrained `change_index.status`, and idempotency PK already satisfy the contract. No CHECK/column/index blocks `rejected`, Round N, `change.resubmit`, or current-projection rewrite. `Migration required: NO` stands. |
| G19 PostgreSQL concurrency plan | **PASS** | D1–D6 and R1–R3 are sufficient, implementable, and database-authoritative. No sleep/timing-only proof. Reuse existing disposable Postgres 16 harness. |
| G20 Live product / scope | **PASS** | 3a happy path on `CHG-2026-000003` uses real Catalog membership (independently verified) plus `change_cab_recorder`; no fabricated membership; lifecycle stays `submitted`; eligibility remains window-governed. Rejection/resubmission uses a disposable Change, not 000003. F3.1.4 UI, Workbench, F3.2, Teams, execution lifecycle, and production cutover remain excluded. |

**Gates PASS: 18 / 20. Gates FAIL: 2 / 20.**

---

## 5. Blocking contract A — resubmission actor authority (G13)

### What the plan proposes

`change-management.change.create` **and** actor is requester **or** current
`ownerRef` member **or** `platform_admin`. No CAB-only resubmit.

### What accepted architecture actually says

ADR-009:

- defines `rejected --accepted resubmission--> submitted` (same `changeId`, new
  round) in the **passive voice**;
- does **not** name who may perform that transition;
- lists participant read as requester / `ownerRef` team / `responsibleRef` team /
  `platform_admin` and states that this is **read only**;
- lists cancellation actors explicitly: requester, Change owner, or
  governance/admin, **for cancel only**;
- lists conceptual permissions (read, decide, CAB record, add requirement,
  cancel, policy admin, governance read-all) with **no resubmit capability**;
- allows F3.1 planning to refine **literal permission strings**, not to invent
  a new lifecycle-actor class without recording the decision.

Accepted source:

- `canReadChange` uses requester / owner membership / responsible membership /
  `platform_admin` as **read**;
- `change.create` is granted to contributor / `template_executor` /
  `platform_admin` and authorizes **new** Changes, not rewrite of an existing
  identity;
- `platform_admin` is the Entra TP group that also happens to be the laptop CAB
  binding — ADR-013 forbids collapsing that coincidence into CAB business
  authority (the plan gets this right for `cab.record`; it must not smuggle the
  same coincidence into resubmit).

### Independent decision

**Do not bless this actor set inside F3.1.3 planning.**

| Candidate | Already justified? | Review finding |
|---|---|---|
| Requester | **Not recorded** | Strongest intuitive actor (the original submitter correcting the Change), but ADR-009 never grants it. |
| Current `ownerRef` member | **Not recorded** | Owner membership is the read/cancel *proof shape*, not resubmit authority. Combined with unconstrained `targetRef` (G14), an owner member can transfer the Change to another System under the same id. |
| `platform_admin` | **Not recorded** | Technical admin is not business-correction authority. Cancellation’s “governance/admin” is a different capability. |
| CAB-only resubmit | Correctly excluded | No architecture grant exists. |

Reusing `change.create` as the literal permission name is a defensible F3.1
naming choice **only after** the actor classes are decided. Analogizing
cancellation actors is an available *proposal* for the correction checkpoint,
not an already-accepted rule this review may silently adopt.

Required plan correction: record an explicit architecture decision naming who
may resubmit, with server-side proof, and keep CAB-only resubmit forbidden
unless a later ADR says otherwise.

---

## 6. Blocking contract B — same-changeId identity vs correction (G14)

### What the plan proposes

Same `changeId`; freeze `requestedBy` and identity `createdAt`; “user-editable
create fields may be corrected”; `ownerRef` / `systemRef` re-resolved from the
new `targetRef`; new activity IDs; new Round snapshot; index/provider become
the **current** projection; Round 1 remains immutable birth evidence.

### What accepted architecture actually says

ADR-009 Q19: same `changeId` for correction; **a fundamentally different
business change receives a new Change and `changeId`.**

ADR-007: `ownerRef` / `systemRef` are snapshots of the Catalog target. A GMUD
created for Team A must not historically appear owned by Team B because Catalog
drifted. Intentional retargeting is a different question the ADR never
authorized as “correction.”

ADR-008: the execution plan is the work to be done; correcting activities after
rejection is a natural correction of the **same** governed target.

### Independent decision

Model C **does** remain coherent if index/provider become the current
projection while prior Round snapshots stay immutable history (G15 PASS). That
composition is the right narrowing of ADR-007’s F2 “birth snapshot is the list
row” once rounds exist.

What is **not** coherent is treating a changed `targetRef` / owner / System as
the same business Change:

```text
CHG-42 Round 1: target = system:payments, owner = Team A
CHG-42 Round 2: target = system:hr,       owner = Team B
```

That is a new business change wearing the prior identity. Audit, participant
read, policy matching, and eligibility would follow the new System while the
public `changeId` pretends continuity.

The plan **defined** the editable set (all create fields) rather than leaving
it TBD. The defined set is still not implementation-ready, because it does not
draw the ADR-009 identity line.

Required plan correction — pick exactly one:

1. **Freeze identity fields** on resubmit: at least `targetRef`, and therefore
   `ownerRef` / `systemRef`. Allow correction of title, summary, window, risk,
   classification, rollback, evidence, and execution-plan content (new activity
   IDs remain OK).
2. **Explicitly decide** that same-`changeId` correction may retarget, with a
   written argument why that is not a fundamentally different business Change,
   plus the read-scope consequences after owner/system change.

This review does not choose (1) vs (2) beyond noting that (1) is the
conservative reading of ADR-009 Q19 and is the narrowest unblocking path.

---

## 7. Required challenges

| # | Challenge | Answer |
|---|---|---|
| 1 | CAB membership removed between Round creation and decision | Plan Q3: snapshot remains historical authority identity; live Catalog membership is checked at decision time. Not a current member → `FORBIDDEN` `not_authority_member`; zero rows. |
| 2 | Catalog unavailable after permission passes but before SQL starts | Plan Q3/Q5: membership I/O is **outside** the write transaction. Catalog/credentials error → `PROVIDER_UNAVAILABLE` `membership_source_unavailable`; transaction never opens; zero decision. |
| 3 | Same actor reuses one idempotency key on a different requirement | Plan Q5 matrix: `CONFLICT` because `(actor_ref, idempotency_key)` is global. Callers mint a fresh key per requirement. Explicit and acceptable. |
| 4 | Two actors concurrently approve two different requirements, each seeing the other undecided | Plan Q6/Q7 + D4: both serialize on `change_index FOR UPDATE`; trx-aware `listRequirements` sees the in-transaction decision; exactly one `authorization_reached`. Without the trx-aware read this would be racy on PostgreSQL — the plan correctly closes that source gap. |
| 5 | Approve and reject race on the same requirement | Plan Q5/D3: requirement unique allows one terminal row; loser unique-violation → rollback → re-observe → `CONFLICT` `conflicting_decision`. No `ON CONFLICT UPDATE`. |
| 6 | Rejection commits but HTTP response is lost; retry uses same key | Exact replay: same actor/key/hash → 200 + original `decisionId` / `decidedAt`; no second decision, audit, or status write. |
| 7 | Detail GET after rejection while provider still says submitted | Plan Q8: 3a overlays **index** `status='rejected'` onto public Change; provider `record_json` unchanged. List already reads index status. Overlay is status-only and is a no-op for `LEGACY_PRE_F3`. |
| 8 | Resubmit changes `targetRef` to a different System/owner | **Blocking.** Plan Q11 currently allows it. This review rejects that as compatible with ADR-009 Q19 without an explicit identity freeze or an explicit retargeting decision (§6). |
| 9 | Two authorized actors concurrently resubmit after rejection | Plan Q11/R1: `FOR UPDATE` + expected next `round_number` + round PK → one Round N+1. Loser `CONFLICT` `resubmission_conflict`. Reservations live inside the same trx so a lost race does not leave a dangling `change.resubmit` pending row. |
| 10 | Resubmission fails after provider replacement but before commit | Plan Q11/G15: `replaceCurrent` joins the caller-owned Knex transaction on the same database as index/ledger. Rollback restores provider, index, Round N, and idempotency. Confirmed `DevelopmentProvider` uses that client. |
| 11 | Post-execution CAB retrospective before execution completion exists | Plan Q10: `CONFLICT` `execution_completion_required`; zero decision. No completion surrogate exists in current source; the requirement row must not count. |
| 12 | Decision targets an old Round after Round 2 exists | Plan error table: `CONFLICT` `stale_round` when `roundNumber` ≠ current max. Prior Round 1 decisions remain immutable (R3 / append-only triggers). |

**Challenges answered: 12 / 12.**

---

## 8. What remains accepted in the plan (not to be redesigned)

Do not reopen these in the correction checkpoint:

- F3.1.3a / F3.1.3b split
- Nested decision route, required Idempotency-Key, command hash, 201/200
- Individual equality-only authority
- Live Catalog CAB membership + dedicated `cab.record` (not `platform_admin`)
- Caller-owned `trx` + `change_index FOR UPDATE` + **trx-aware ledger reads**
- Exactly-once milestones without new unique index
- Rejection as rebuildable index projection; no lifecycle `authorized`
- Eligibility: no sandbox fabrication for `LEDGER_REQUIRED`
- Post-execution fail-closed without inventing completion evidence
- `Migration required: NO`
- `DevelopmentProvider.replaceCurrent` (not `create()` upsert)
- PostgreSQL D1–D6 and R1–R3
- Laptop demo: approve `CHG-2026-000003`; reject/resubmit a disposable Change

---

## 9. Narrowest next activity

```text
Next: F3.1.3 plan correction (3b only)
Must close:
  1. explicit resubmission actor authority (G13)
  2. same-changeId identity / editable-field boundary (G14)
Then: independent re-review of the corrected plan
F3.1.3a implementation-prompt authoring: still NO-GO until ACCEPT
```

Do not author the F3.1.3a implementation prompt from this REJECT. Do not
implement F3.1.3/F3.1.4/F3.2. Do not change ADR-009/ADR-013 in this checkpoint;
if the correction needs an ADR amendment, that is a separate authorized docs
activity.

---

## 10. Decision

```text
F3.1.3 plan architecture review: REJECT
F3.1.3 plan: REVISION REQUIRED
Migration required: NO
F3.1.3a implementation-prompt authoring: NO-GO
F3.1.3a implementation: NO-GO
F3.1.3b implementation: NO-GO
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
Production cutover: NOT AUTHORIZED
ADO implementation modified: NO
```
