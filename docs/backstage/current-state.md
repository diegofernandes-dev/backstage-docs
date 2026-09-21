# Backstage IDP — current state snapshot

> **Documentation bridge:** `diegofernandes-dev/backstage-docs`  
> **Canonical architectural branch:** `main`  
> **Implementation repository (ADO):** `platform-devops-developer-portal`  
> **Active branch:** `feat/ado-repo-governance`  
> **Last updated:** 2026-09-21 (F3.1.3b implementation **PASS / PUBLISHED** at ADO `5e70d88`, pending independent acceptance; F3.1.3a remains **CLOSED / ACCEPTED IMPLEMENTED BASELINE** at parent `6bad066`; F3.1.2 remains **CLOSED / ACCEPTED IMPLEMENTED SUBMISSION BASELINE** at `2249522`; committed default `LEGACY_PRE_F3`; F3.1.4/F3.2 **NO-GO**; production cutover unauthorized)

## Stack

| Item         | Value                                                 |
| ------------ | ----------------------------------------------------- |
| Platform     | Backstage 1.51.0                                      |
| Auth         | Microsoft Entra ID + MS Graph catalog ingestion       |
| RBAC         | Community RBAC + ownership on Systems                 |
| Azure DevOps | Project access, repo governance, pipeline integration |
| TechDocs     | AWS S3 (production path)                              |

## GMUD change management

### Frontend (F2.2)

| Item                    | State                                                                                                                                                                                                                                                                                                  |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Route                   | `/gmud` (Minhas GMUDs) · `/gmud/new` (create, moved from `/gmud`) · `/gmud/:changeId` (detail) — one `page:change-management` extension, nested `<Routes>` via `GmudRouter`                                                                                                                            |
| Plugin                  | `@internal/plugin-change-management`                                                                                                                                                                                                                                                                   |
| API (client)            | `ChangeManagementApi` → `ChangeManagementClient` via Backstage discovery/fetch (`createChangeRequest`, `listChanges`, `getChange`); mock retained for tests/fixtures only                                                                                                                              |
| Domain model (frontend) | `Change` / `ChangeSummary` mirrored from backend `types.ts`; `CreateChangeHttpBody` — `targetRef`, `classification`, `requestedWindow`, `risk`, `rollbackPlan`, `evidence`, `executionPlan` (required, ordered, 1–20 activities per [ADR-008](../adr/ADR-008-multi-activity-change-execution-plan.md)) |
| UI                      | Create: five numbered sections, Catalog Group executor + optional Component target per activity. List: compact `Table` (Minhas GMUDs). Detail: read-only `InfoCard`s, ordered execution plan. Post-create: Ver GMUD / Criar outra GMUD / Voltar para Minhas GMUDs                                      |

### Backend (F2.2)

| Item                       | State                                                                                                                                |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Plugin                     | `change-management` (`createBackendPlugin`)                                                                                          |
| Routes                     | `POST /api/change-management/changes` · `GET /api/change-management/changes` · `GET /api/change-management/changes/:changeId`        |
| Service                    | `ChangeManagementService` — `createChange`, `listChanges`, `getChange`                                                               |
| Platform index             | `ChangeIndexRepository` → `change_index` table (identity, routing, audit snapshot incl. `executionPlan`)                             |
| Idempotency                | `IdempotencyRepository` → `change_idempotency` table (platform-owned, crash-safe recovery; carries an immutable `authorization_mode` since F3.1.0-V, see below)                                           |
| Sequence                   | `DatabaseChangeIdGenerator` → `change_id_sequences` table                                                                            |
| Provider registry          | `ProviderRegistry` — immutable `providerKey` routing per change                                                                      |
| Provider                   | `IChangeManagementProvider` → `DevelopmentProvider` (`providerKey: development`, non-production; `development_change_records` table) |
| Persistence                | SQLite (dev) / Postgres (prod) via `coreServices.database`, real knex migrations                                                     |
| Frontend wiring            | **Connected** — create, list, and detail all call the real backend                                                                   |
| RBAC                       | `change-management.change.create` / `.read` (contributor, template_executor, platform_admin) — `.read` also gates `GET /changes`     |
| Canonical backend contract | [ADR-006](../adr/ADR-006-change-management-backend-contract.md) (HTTP contract + participant read scope)                             |
| Record authority           | [ADR-007](../adr/ADR-007-change-record-authority.md) — Model C (hybrid index + provider record) + discovery/detail clarification     |
| Execution plan domain      | [ADR-008](../adr/ADR-008-multi-activity-change-execution-plan.md) — F2.1.2, read-visibility clause superseded by F2.2.1              |

#### Backend capabilities (F2.2.1)

- Durable platform canonical index with immutable `providerKey`, `externalId`, creation-time snapshot
- GET routes: index → stored `providerKey` → provider adapter (not global config)
- Durable idempotency with early reserve + atomic finalize (`Idempotency-Key` header); crash-safe recovery (`pending`/`completed` state machine, synchronous resume on retry, provider idempotent `create` by `changeId`)
- Durable platform-owned `changeId` sequence (`CHG-{YYYY}-{seq}`)
- `DevelopmentProvider` operational records in `development_change_records` (logically isolated)
- Fail-closed provider errors (503); unfinalized index rows not visible on GET or list
- `executionPlan` / `ExecutionActivity` on canonical `Change`; catalog-validated `responsibleRef` (Group) and optional activity `targetRef`
- Actor-scoped idempotency: `(operation, requested_by, idempotency_key)` — different actors may independently reuse the same key
- `GET /changes` (F2.2) — participant-scoped discovery from the canonical index, identical authorization predicate as `GET /:changeId`, zero provider calls; `ChangeSummary` projection with no provider metadata
- **New in F2.2.1:** `canReadChange` (shared by list and detail) grants read to any actor whose `ownershipEntityRefs` includes `change.ownerRef` **or any** `executionPlan.activities[].responsibleRef` — the **change participant** read policy. Read only; no ownership/approval/CAB/edit/execution authority. Backed by a derived, non-authoritative discovery index (`change_index_activity_participants`), backfilled by migration; `canReadChange` against the immutable snapshot remains the sole authorization truth.

#### Explicitly not implemented (F2.2.1)

- Real ITSM providers (SharePoint, Jira, ServiceNow)
- Azure DevOps / pipeline / deployment correlation
- Approvals, workflow, activity/lifecycle statuses beyond `submitted`
- Teams, CAB scheduling
- Evidence upload, editing, deletion
- Team-wide / enterprise-wide search, filters, sorting frameworks (deferred to F3)
- New governance roles (`change.read.all`, Change Manager, Auditor) (deferred to F3)

### Backend (F3.1.0 — Authorization Ledger Foundation)

| Item                       | State                                                                                                                                |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Status                     | **ACCEPTED IMPLEMENTED BASELINE** — ADO `4bad41d` (F3.1.0-H typing-only closure, direct child of `7663883`, which is a direct child of `6e28611`), published `feat/ado-repo-governance`. **F3.1.0 CLOSED.** |
| Domain types                | `AuthorizationRound`, `ApprovalRequirement`, `ApprovalDecision`, `AuthorizationAuditEvent` — append-only, immutable                   |
| Pure evaluators             | `AuthorizationEvaluation` (mandatory pre-execution requirements) and `GovernanceEvaluation` (mandatory post-execution + SLA) — pure, derived only |
| Persistence                 | Four new append-only ledger tables + `authorization_mode` on `change_index` and `change_idempotency`; SQLite and Postgres migrations with dialect-specific UPDATE/DELETE-blocking triggers |
| Legacy/cutover semantics    | `LEGACY_PRE_F3` / `LEDGER_REQUIRED`; the idempotency reservation is the earliest durable authority for a logical submission's regime; `change_index.authorization_mode` inherits from it; one logical retry never changes regime |
| F3.1.0 submission behavior  | Still F2 submission path — **every new Change remains `LEGACY_PRE_F3`; no `AuthorizationRound` is created by `POST /changes`** |
| Repository contracts        | `AuthorizationLedgerRepository` (insert/read only, no update/delete) with a Knex implementation                                       |
| Architecture guards          | Extended to prohibit provider/ADO/Teams canonical fields, workflow repositories, mutable current/evaluation state, and a policy DSL in the ledger |
| Explicitly not implemented  | First-round submission integration (`POST /changes` is still unwired from authorization), approval/decision commands, decision-time authority membership, CAB Workbench, Teams, Azure DevOps execution enforcement (all deferred to F3.1.2+). Policy publication and selector resolution runtimes now exist as of F3.1.1a/F3.1.1b, but nothing calls them from the submission path. |
| Known deviation             | Double-`buildChange()` defect **closed and accepted** by F3.1.2a at ADO `ccee1e1`. Provider finalize and pending index share one durable snapshot. |

### Architecture review outcome (cumulative through F2.2.1)

| Decision                         | Outcome                                                                                                                                                                                                                          |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Canonical record ownership       | **Model C** — platform canonical index + ITSM provider operational record                                                                                                                                                        |
| Provider replaceability          | API/frontend contract stable; multi-provider read routing via immutable `providerKey`                                                                                                                                            |
| `changeId` ownership             | Platform (`ChangeManagementService` / `changeIdGenerator`) — confirmed in ADO `b2bed17`, durable as of F2.1                                                                                                                      |
| Idempotency ownership            | Platform service store — confirmed in ADO `b2bed17`, durable/crash-safe as of F2.1.1                                                                                                                                             |
| Orphan record handling           | Log `change.create.orphan` (with `idempotencyKey`) + synchronous retry reconciliation; no background worker                                                                                                                      |
| Execution plan domain            | `ExecutionPlan`/`ExecutionActivity` accepted per ADR-008 (F2.1.2) — provider-neutral, no per-activity status                                                                                                                     |
| Catalog `ownerRef`/`systemRef`   | Creation-time snapshots — not live catalog refs on GET or list                                                                                                                                                                   |
| Frontend wiring                  | Complete as of F2.1.3 — real create client, no create-time GET                                                                                                                                                                   |
| List vs. detail authority (F2.2) | List = index snapshot (discovery only); detail = provider-authoritative, unchanged routing                                                                                                                                       |
| Read visibility (F2.2.1)         | **Change participant** policy — `platform_admin` OR requester OR `ownerRef` team OR any activity `responsibleRef` team; read only, no other authority; supersedes ADR-008's F2.1.2 "responsibleRef grants no read access" clause |

### Architecture and implementation status

| Slice                                | Status                                                                                                                                    |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| F2.2.1 implementation                | **ACCEPTED IMPLEMENTED BASELINE** at ADO `6e28611` (published to `origin/feat/ado-repo-governance` as part of F3.1.0-P)                    |
| F3.0.1 authorization architecture    | **ACCEPTED ARCHITECTURE** by ADR-009                                                                                                         |
| **F3.1.0 Authorization Ledger Foundation** | **ACCEPTED IMPLEMENTED BASELINE — CLOSED** at ADO `4bad41d` (full SHA `4bad41d058edf5c5314d17275e0c8bdb5abf690f`), the F3.1.0-H typing-only hotfix, a direct child of published commit `7663883` (full SHA `766388393458f82fbdc2e0502b8c193d0a85e605`), which is itself a direct child of `6e28611`. `7663883` remains historically accurate architectural content; only its TypeScript typing was corrected by `4bad41d`. The review lineage `be16ffb` → `7a9347e` → `06ec9cf` is retained as **review-only local evidence** on `review/f3-1-0-cutover-safety`, excluded from official history. |
| F3.1.1a — Policy domain, registry & publication integrity | **CLOSED / ACCEPTED IMPLEMENTED BASELINE** at ADO `d3c0751` (full SHA `d3c0751a15b908cec8f5595c97e52f41226344ed`), direct child of `4bad41d`, on `feat/ado-repo-governance`. Pure, I/O-free published authorization-policy foundation. See [`f3-1-1a-architecture-acceptance.md`](./f3-1-1a-architecture-acceptance.md). |
| F3.1.1b — Selector bundle, resolver & selector publication integrity | **CLOSED / ACCEPTED IMPLEMENTED BASELINE** at ADO `188d8e9` (full SHA `188d8e9cc43423f3644b3cacfb9849257838a583`), direct child of `d3c0751`, on `feat/ado-repo-governance`. Accepted by independent architecture review on 2026-09-19 (`backstage-docs` review baseline `cf9f96c`): G1–G17 all PASS. Selector/config half of F3.1.1 is closed. See [`f3-1-1b-architecture-acceptance.md`](./f3-1-1b-architecture-acceptance.md) and [`f3-1-1b-implementation-evidence.md`](./f3-1-1b-implementation-evidence.md). |
| F3.1.2a — Canonical Change construction / recovery | **CLOSED / ACCEPTED IMPLEMENTED BASELINE** at ADO `ccee1e1` (full SHA `ccee1e1676a2763e68880e5383ce1e5e48742843`), direct child of `188d8e9`. Independent acceptance ACCEPT on 2026-09-19 ([`f3-1-2a-architecture-implementation-acceptance.md`](./f3-1-2a-architecture-implementation-acceptance.md)). One `buildChange()` per logical create; recovery reuses durable pending snapshot; provider/index structurally identical. **`POST /changes` still `LEGACY_PRE_F3`; no `AuthorizationRound`.** Evidence: [`f3-1-2a-implementation-evidence.md`](./f3-1-2a-implementation-evidence.md). |
| F3.1.1c — CAB-safe policy baseline publication | **CLOSED / ACCEPTED IMPLEMENTED BASELINE** at ADO `3b302ab` (full SHA `3b302ab5c9caab38f96491b389b7ea9fe0b66c2f`), direct child of F3.1.2a `ccee1e1`. Independent acceptance ACCEPT on 2026-09-19 ([`f3-1-1c-architecture-implementation-acceptance.md`](./f3-1-1c-architecture-implementation-acceptance.md)). New immutable policy `default-change-authorization@2026-09-19.1`: normal-low → primary + CAB; medium/high/emergency unchanged; historical `2026-09-02.1` untouched; activePolicy pin updated; selector bundle unchanged. Evidence: [`f3-1-1c-implementation-evidence.md`](./f3-1-1c-implementation-evidence.md). |
| Product convergence — Catalog Deployments UX onto GMUD line | **PASS / PUBLISHED** at ADO `f48dc82` (full SHA `f48dc825ab5d1d16fafc3f70ef772d28613df1a0`), direct child of `3b302ab`. Root cause `BRANCH_DIVERGENCE` vs `feat/delivery-mvp-slice@170af45` (accepted surface `b08e7b2`). Minimal Deployments tab + Delivery runtime restored; Delivery branch not merged wholesale. Evidence: [`product-convergence-deployments-evidence.md`](./product-convergence-deployments-evidence.md). |
| F3.1.2b — Ledger-governed submission integration | **CLOSED / ACCEPTED IMPLEMENTED BASELINE** at ADO `2249522` (full SHA `22495229502dabf2d99588599a156d862c5114fa`), direct child of product-convergence `f48dc82`. Independent acceptance ACCEPT on 2026-09-20 ([`f3-1-2b-architecture-implementation-acceptance.md`](./f3-1-2b-architecture-implementation-acceptance.md)). Committed default remains `LEGACY_PRE_F3`. Evidence: [`f3-1-2b-implementation-evidence.md`](./f3-1-2b-implementation-evidence.md). |
| F3.1.2 — First-round submission integration | **CLOSED / ACCEPTED IMPLEMENTED SUBMISSION BASELINE** at ADO `2249522`. Plan remains the ACCEPTED IMPLEMENTATION CONTRACT ([`f3-1-2-final-architecture-rereview.md`](./f3-1-2-final-architecture-rereview.md)). F3.1.2a accepted. F3.1.1c accepted. Product convergence PASS. F3.1.2b accepted. Canonical plan: [`f3-1-2-implementation-plan.md`](./f3-1-2-implementation-plan.md). |
| F3.1.2 non-production LEDGER_REQUIRED activation | **ACCEPT.** Independent review 2026-09-20 ([`f3-1-2-ledger-required-nonprod-activation-acceptance.md`](./f3-1-2-ledger-required-nonprod-activation-acceptance.md)). Live operator-laptop overlay at accepted SHA `2249522`; committed default remains `LEGACY_PRE_F3`; F3.1.3 demo target readiness **READY**. Evidence: [`f3-1-2-ledger-required-nonprod-activation-evidence.md`](./f3-1-2-ledger-required-nonprod-activation-evidence.md). Production cutover remains unauthorized. |
| F3.1.3 — Decision command and new-round semantics | Historical plan review **REJECT** preserved ([`f3-1-3-plan-architecture-review.md`](./f3-1-3-plan-architecture-review.md)). Revised-plan re-review **ACCEPT** ([`f3-1-3-revised-plan-architecture-rereview.md`](./f3-1-3-revised-plan-architecture-rereview.md)). Plan is **ACCEPTED IMPLEMENTATION CONTRACT** ([`f3-1-3-implementation-plan.md`](./f3-1-3-implementation-plan.md), [ADR-014](../adr/ADR-014-change-resubmission-authority-and-identity-boundary.md)). `Migration required: NO`. |
| F3.1.3a — Server-authoritative decision command | **CLOSED / ACCEPTED IMPLEMENTED BASELINE** at ADO `6bad066` (full SHA `6bad066d945d49feaf642313ec37467e2658dc3f`), direct child of `2249522`. Independent acceptance ACCEPT on 2026-09-21 ([`f3-1-3a-architecture-implementation-acceptance.md`](./f3-1-3a-architecture-implementation-acceptance.md)). Nested POST decision route, required `Idempotency-Key`, exact individual principal, live Catalog CAB membership + dedicated `cab.record`, caller-owned txn, rejected lifecycle projection, eligibility `NO_LEDGER_ROUND` safety, PostgreSQL D1–D6, live laptop proof on `CHG-2026-000003` / disposable `CHG-2026-000005`. Evidence: [`f3-1-3a-implementation-evidence.md`](./f3-1-3a-implementation-evidence.md). |
| F3.1.3b — Rejected-Change resubmission / new Round | **IMPLEMENTATION PASS / PUBLISHED** at ADO `5e70d88` (full SHA `5e70d8818f55072d8568b5eb15bae754abb943d1`), direct child of accepted `6bad066`. Not independently accepted; F3.1.3b is **not CLOSED**. `POST /changes/:changeId/resubmissions`, dedicated `change.resubmit`, requester-or-immutable-owner authority, frozen identity, Round N+1, `DevelopmentProvider.replaceCurrent`, PostgreSQL R1 + R2–R13, live Round 2 on `CHG-2026-000005`. Evidence: [`f3-1-3b-implementation-evidence.md`](./f3-1-3b-implementation-evidence.md). Next gate: independent architecture/implementation acceptance review. F3.1.4/F3.2 remain **NO-GO**. |
| F3.1.4 | **NOT AUTHORIZED** |
| F3.2 — CAB Governance & Delegated Autonomy | **ARCHITECTURE ACCEPTED / implementation NOT AUTHORIZED** — ADR-013 defines bounded Group + System low-risk autonomy, separate CAB decision vs autonomy RBAC, append-only grant/revoke history, all-covered multi-activity rule, prospective revocation, and future CAB Workbench. |

### Accepted authorization architecture (ADR-009 + ADR-013)

[ADR-009](../adr/ADR-009-change-authorization-model.md), as narrowly superseded by [ADR-013](../adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md), defines the current target without changing the accepted ADO baseline:

| Concern                          | Accepted decision                                                                                                                      |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Business authorization authority | Platform Change Management authorization ledger, not Azure DevOps or an ITSM provider                                                  |
| Policy                           | Deterministic, immutable, versioned policy evaluated at submission                                                                     |
| Requirements                     | Effective, snapshotted, provider-neutral; policy-generated plus additive mandatory requirements                                        |
| Decisions                        | Immutable/append-only `approved` or `rejected` human/governance facts                                                                  |
| Principal identity               | Configured selectors resolved to snapshotted platform principal refs; no titles/names in domain                                        |
| CAB                              | One collective Change-decision authority plus a separate governance authority over bounded low-risk autonomy; normal-low requires CAB by default until a valid future autonomy grant applies |
| Emergency                        | Multiple generic pre-execution approvals plus post-execution CAB retrospective                                                         |
| Orthogonal state                 | Lifecycle excludes `authorized`; `AUTHORIZED` is derived pre-execution authorization; post-execution governance has its own evaluation |
| Lifecycle                        | `submitted`, `executing`, `completed`, `rejected`, `cancelled`; start/completion require accepted execution evidence                   |
| Governance                       | `NOT_APPLICABLE`, `PENDING`, `COMPLIANT`, `NON_COMPLIANT`; retrospective rejection/SLA miss never rewrites historical authorization    |
| Authorization vs execution       | Authorization is pre-execution governance; executable now additionally requires lifecycle, window, target correlation, and no hold     |
| Teams                            | Future individual-decision interaction channel; never system of record                                                                 |
| Backstage                        | Preferred future CAB Workbench UI; backend authorization ledger remains authoritative                                                  |
| Pipelines                        | Future provider-neutral execution-eligibility consumers; no ADO object in canonical Change                                             |
| DevOps                           | Policy/control/integration/observability/exception owner; absent from happy-path per-deploy approval                                   |
| Model C                          | Retained; bounded platform authorization ledger sits beside the index while provider owns operational GMUD detail                      |

**Gate:** the [F3 architect review brief](../architect-review-f3-change-authorization.md) records all nine architecture decisions as resolved and ADR-009 as Accepted. F3.1.0's review lineage — unauthorized/unreviewed `be16ffb` → cutover-safety correction `7a9347e` → F3.1.0-V closure `06ec9cf` — durably ties `change_idempotency.authorization_mode` (immutable, fail-closed on retry) to `change_index.authorization_mode` as the single source of a logical submission's regime, and classifies the previously reported test-process non-exit as pre-existing Jest watch-mode behavior (confirmed identical on `6e28611` and the candidate), not a leak. **F3.1.0-P published that architecture-accepted final state as one clean commit `7663883` directly on `6e28611`** — a final-state transfer, not a cherry-pick of the three review commits — and pushed it to `origin/feat/ado-repo-governance` as a plain fast-forward, with tree-hash equivalence proven against `06ec9cf`. Publication review then found a TypeScript regression in `7663883` against the accepted `6e28611` baseline; the one-line typing-only hotfix **F3.1.0-H, commit `4bad41d`**, restored the repository-wide error set to exactly the historical five. Before F3.1.2b cutover, all existing and newly created F2 Changes remain `LEGACY_PRE_F3`; no ledger facts are fabricated. **`4bad41d` is now the accepted F3.1.0 implementation baseline; `7663883`, `06ec9cf`, and their ancestors are historical evidence only. F3.1.0 is CLOSED.** F3.1.1a and F3.1.1b are **CLOSED / ACCEPTED** at ADO `d3c0751` and `188d8e9`. **F3.1.2 first plan review was REJECT** ([`f3-1-2-plan-architecture-review.md`](./f3-1-2-plan-architecture-review.md)); the revised plan's re-review was also **REJECT** on concurrency only ([`f3-1-2-revised-plan-architecture-rereview.md`](./f3-1-2-revised-plan-architecture-rereview.md)); the concurrency + ADR-013 final re-review is **ACCEPT** ([`f3-1-2-final-architecture-rereview.md`](./f3-1-2-final-architecture-rereview.md)) — **ACCEPTED IMPLEMENTATION CONTRACT**. **F3.1.2a is CLOSED / ACCEPTED** at ADO `ccee1e1` ([`f3-1-2a-architecture-implementation-acceptance.md`](./f3-1-2a-architecture-implementation-acceptance.md)). **F3.1.1c is CLOSED / ACCEPTED** at ADO `3b302ab` ([`f3-1-1c-architecture-implementation-acceptance.md`](./f3-1-1c-architecture-implementation-acceptance.md)). Product convergence is **PASS** at ADO `f48dc82`. **F3.1.2b is CLOSED / ACCEPTED** at ADO `2249522` ([`f3-1-2b-architecture-implementation-acceptance.md`](./f3-1-2b-architecture-implementation-acceptance.md)). **F3.1.2 is CLOSED / ACCEPTED IMPLEMENTED SUBMISSION BASELINE.** Non-production `LEDGER_REQUIRED` activation is **ACCEPT** on the operator-laptop runtime; committed default remains `LEGACY_PRE_F3`; production cutover remains NOT AUTHORIZED. F3.1.3 demo target readiness is **READY**. F3.1.3 plan architecture review **REJECT** is preserved ([`f3-1-3-plan-architecture-review.md`](./f3-1-3-plan-architecture-review.md)). F3.1.3 revised-plan re-review is **ACCEPT** ([`f3-1-3-revised-plan-architecture-rereview.md`](./f3-1-3-revised-plan-architecture-rereview.md)); the plan is **ACCEPTED IMPLEMENTATION CONTRACT** via [ADR-014](../adr/ADR-014-change-resubmission-authority-and-identity-boundary.md). **F3.1.3a is CLOSED / ACCEPTED IMPLEMENTED BASELINE** at ADO `6bad066` ([`f3-1-3a-architecture-implementation-acceptance.md`](./f3-1-3a-architecture-implementation-acceptance.md); evidence [`f3-1-3a-implementation-evidence.md`](./f3-1-3a-implementation-evidence.md)). **F3.1.3b implementation is PASS / PUBLISHED** at ADO `5e70d88` ([`f3-1-3b-implementation-evidence.md`](./f3-1-3b-implementation-evidence.md)) and remains open pending independent architecture/implementation acceptance. F3.1.4 and F3.2 remain NO-GO.

See [`implementation-progress.md`](./implementation-progress.md) §12–§19 for full checkpoint detail (F2.1 through F3.0.1 architecture convergence).

### Normative references

- UI contracts: [`gmud-create-screen.md`](../ui/gmud-create-screen.md) · [`gmud-my-changes-screen.md`](../ui/gmud-my-changes-screen.md) · [`gmud-detail-screen.md`](../ui/gmud-detail-screen.md)
- Architecture decisions: [`docs/adr/`](../adr/README.md) (ADR-001 through ADR-014 on `main`)
- Backend contract: [ADR-006](../adr/ADR-006-change-management-backend-contract.md)
- Record authority: [ADR-007](../adr/ADR-007-change-record-authority.md)
- Execution plan domain: [ADR-008](../adr/ADR-008-multi-activity-change-execution-plan.md)
- Authorization model: [ADR-009](../adr/ADR-009-change-authorization-model.md) — accepted base model, partially superseded by ADR-013
- CAB governance/autonomy: [ADR-013](../adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md) — accepted default-deny low-risk CAB governance and future delegated autonomy
- Resubmission authority/identity: [ADR-014](../adr/ADR-014-change-resubmission-authority-and-identity-boundary.md) — dedicated resubmit permission + requester-or-immutable-owner; frozen same-`changeId` identity
- Handoff detail: [`implementation-progress.md`](./implementation-progress.md) §9–§19

### Visual baseline

Historical F1/F2 screenshots remain supporting evidence in the legacy POC repository until intentionally re-captured or migrated. They are not architecture authority.

## Review gate — F2.2.1 complete, STOP before F3

**F2.2.1** closes the domain contradiction ADR-008 (F2.1.2) left open: an activity `responsibleRef` team could be assigned execution responsibility without being able to read the GMUD that assigns it. Adds the **change participant** read policy on top of the F2.2 discovery baseline:

1. `canReadChange` (shared by list and detail — unchanged sharing from F2.2) gains a fourth clause: any `executionPlan.activities[].responsibleRef` membership grants read, alongside `platform_admin` / requester / `ownerRef` team
2. New derived, non-authoritative discovery index `change_index_activity_participants` (backfilled by migration) narrows the list query's SQL candidates; the snapshot predicate remains the sole authorization truth
3. List/detail authorization proven identical for every participant class, including activity-responsible teams, by construction and by test
4. Read only — `responsibleRef` grants no ownership, approval, CAB, edit, or execution authority; ADR-008's F2.1.2 decision text is preserved with a superseded-by note, not rewritten
5. No workflow, approvals, activity status, Teams, CAB, new governance roles, or enterprise-wide search introduced

**STOP before F3** — historical gate; F3.0/F3.0.1 architecture work has since passed this checkpoint without implementing F3.1.

## Source-of-truth rules

| Question                    | Authority                                                                                                   |
| --------------------------- | ----------------------------------------------------------------------------------------------------------- |
| What is implemented?        | ADO `platform-devops-developer-portal` source code                                                          |
| What should be implemented? | ADRs and normative contracts in `diegofernandes-dev/backstage-docs@main`                                    |
| Divergence                  | Report as deviation in `implementation-progress.md` — do not silently alter architecture docs to match code |

## ADO implementation reference

| Item                                             | Value                                                                                |
| ------------------------------------------------ | ------------------------------------------------------------------------------------ |
| Branch                                           | `feat/ado-repo-governance`                                                           |
| F1.4 commit                                      | `52e01ca`                                                                            |
| F2.0 commit                                      | `b2bed17` (backend contract scaffold)                                                |
| F2.1 commit                                      | `0dc3ed4` (durable canonical index + `DevelopmentProvider`)                          |
| F2.1.1 commit                                    | `ed6810b` (idempotency recovery, crash-safe retry)                                   |
| F2.1.2 commit                                    | `5e4f30e` (multi-activity execution plan domain)                                     |
| F2.1.3 commit                                    | `75da44fb46d308e23b1c987e2093636fa4811b92` (execution plan wired to real backend)    |
| F2.2 commit                                      | `0b9cb38` (My Changes List + Change Detail)                                          |
| **F2.2.1 commit**                                | **`6e28611`** (participant read policy)                                              |
| F3.1.0 commit (superseded)                       | `7663883` (full SHA `766388393458f82fbdc2e0502b8c193d0a85e605`) — published, direct child of `6e28611`; superseded by F3.1.0-H below after a TypeScript typing regression was found in publication review |
| **F3.1.0-H official commit**                     | **`4bad41d`** (full SHA `4bad41d058edf5c5314d17275e0c8bdb5abf690f`) — direct child of `7663883`, typing-only hotfix, ACCEPTED IMPLEMENTED BASELINE |
| **F3.1.1a official commit**                      | **`d3c0751`** (full SHA `d3c0751a15b908cec8f5595c97e52f41226344ed`) — ACCEPTED IMPLEMENTED BASELINE |
| **F3.1.1b official commit**                      | **`188d8e9`** (full SHA `188d8e9cc43423f3644b3cacfb9849257838a583`) — ACCEPTED IMPLEMENTED BASELINE |
| **F3.1.2a official commit**                      | **`ccee1e1`** (full SHA `ccee1e1676a2763e68880e5383ce1e5e48742843`) — CLOSED / ACCEPTED IMPLEMENTED BASELINE |
| **F3.1.1c official commit**                      | **`3b302ab`** (full SHA `3b302ab5c9caab38f96491b389b7ea9fe0b66c2f`) — CLOSED / ACCEPTED IMPLEMENTED BASELINE |
| **Product convergence official commit**          | **`f48dc82`** (full SHA `f48dc825ab5d1d16fafc3f70ef772d28613df1a0`) — PASS / PUBLISHED |
| **F3.1.2b official commit**                      | **`2249522`** (full SHA `22495229502dabf2d99588599a156d862c5114fa`) — CLOSED / ACCEPTED IMPLEMENTED BASELINE; F3.1.2 submission baseline |
| **F3.1.3a implementation commit**                | **`6bad066`** (full SHA `6bad066d945d49feaf642313ec37467e2658dc3f`) — implementation PASS, **not independently accepted / not CLOSED** |
| F3.1.0 review-only candidates (not official history) | `be16ffb` (unreviewed) → `7a9347e` (cutover-safety correction) → `06ec9cf` (F3.1.0-V closure, architecture-accepted final state) — all on local branch `review/f3-1-0-cutover-safety`, retained as review/audit evidence only |
| Legacy bridge final architecture import baseline | `poc-teams-approval@fe4f807`                                                         |
| New documentation bridge                         | `diegofernandes-dev/backstage-docs@main`                                             |

## Superseded references

| Item                                                                | Status                                                                                                             |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Legacy bridge `diegofernandes-dev/poc-teams-approval`               | Historical POC/archive only after migration to `backstage-docs`                                                    |
| GitHub mirror `diegofernandes-dev/platform-devops-developer-portal` | Deprecated accidental mirror — do not use for development                                                          |
| ADR-008 "responsibleRef grants no read access" (F2.1.2)             | Superseded for read visibility by ADR-006 "Participant read scope (F2.2.1)"                                        |
| F3.0.1 architecture convergence gate                                | Superseded by F3.1.0 publication below                                                                             |
| F3.1.0 review candidate lineage (`be16ffb`/`7a9347e`/`06ec9cf`)     | Superseded as implementation reference by published commit `7663883` — retained only as review/audit history on `review/f3-1-0-cutover-safety` |
| F3.1.0 commit `7663883`                                              | Superseded as implementation reference by F3.1.0-H commit `4bad41d` — publication review found a TypeScript regression (`TS2304` in `KnexAuthorizationLedgerRepository.test.ts`) against the accepted `6e28611` baseline; `7663883` remains historically accurate architectural content, only its typing was corrected |
| F3.1.0-H publication gate                                            | **F3.1.0 CLOSED. GO for F3.1.1 planning; NO-GO for F3.1.1 implementation** — accepted implemented baseline is ADO `4bad41d` (child of `7663883`) |
