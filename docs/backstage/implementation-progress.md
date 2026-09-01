# Backstage implementation / construction progress

> **Canonical documentation bridge:** `diegofernandes-dev/backstage-docs` (`main`)  
> **Implementation source of truth:** Azure DevOps `platform-devops-developer-portal`  
> **Active implementation branch:** `feat/ado-repo-governance`  
> **Migration baseline:** legacy bridge `diegofernandes-dev/poc-teams-approval@fe4f8073f2a8785673e32ce51e5f70b7c322ad68`  
> **Current GMUD implementation baseline:** F2.2.1 — ADO `6e28611`  
> **Current GMUD architecture baseline:** F3.0.1 — ADR-009 Accepted; F3.1 planning allowed, implementation not yet authorized

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

| Checkpoint | Outcome | Implementation / architecture reference |
|---|---|---|
| F1 | Frontend GMUD shell | Dedicated Backstage plugin and `/gmud` route |
| F1.1 | Visual polish | Backstage/MUI-first composition |
| F1.2 | Backstage-first layout | Single form surface + informational rail |
| F1.3 | Semantic UX | Generic production-change language; removed deployment-only assumptions |
| F1.4 | Integrity cleanup | Provider opacity, explicit classification, evidence semantics; ADO `52e01ca` |
| F2.0 | Backend contract scaffold | `ChangeManagementService` + provider contract; ADO `b2bed17` |
| F2.0 review | Record authority | ADR-007 Model C accepted; accidental Model B drift reverted |
| F2.1 | Durable Model C persistence | canonical index + DevelopmentProvider; ADO `0dc3ed4` |
| F2.1.1 | Crash-safe idempotency | retry-driven recovery; ADO `ed6810b` |
| F2.1.2 | Multi-activity plan | ADR-008; ADO `5e4f30e` |
| F2.1.3 | Real frontend wiring | create → backend → Model C persistence; ADO `75da44fb46d308e23b1c987e2093636fa4811b92` |
| F2.2 | My Changes + Detail | index-backed list + provider-routed detail; ADO `0b9cb38` |
| F2.2.1 | Participant read | requester/owner/responsible-team/admin; ADO `6e28611` |
| F3.0 | Authorization architecture | ADR-009 proposed; no implementation |
| F3.0.1 | Authorization convergence | ADR-009 Accepted; no implementation |

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

| Dimension | Values / result |
|---|---|
| Change lifecycle | `submitted`, `executing`, `completed`, `rejected`, `cancelled` |
| AuthorizationEvaluation | `PENDING`, `AUTHORIZED`, `REJECTED` |
| GovernanceEvaluation | `NOT_APPLICABLE`, `PENDING`, `COMPLIANT`, `NON_COMPLIANT` |
| ExecutionEligibility | point-in-time `ALLOW` / `DENY` with reasons |

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

**F3.1.0 implemented locally:** domain types, pure evaluators, four append-only
ledger tables/repositories, explicit `LEGACY_PRE_F3` marking, and SQLite
verification. ADO commit: `be16ffb`. The push to the configured ADO remote was
rejected for lack of repository access and remains a publication prerequisite.

**NO-GO for F3.1.1–F3.1.4** until the preceding checkpoint has passed its STOP
condition and received explicit review.

Explicitly out of F3.1: Teams, CAB Workbench, Azure DevOps enforcement/public eligibility transport, execution lifecycle integration, automatic SLA escalation, real ITSM providers, generic DSL/BPM, break-glass, decision reversal/expiry/abstention, and technical stop-execution behavior.

Planning findings recorded for implementation:

- ADO local HEAD is `6e28611`, while `origin/feat/ado-repo-governance` remains at
  `0b9cb38`.
- Existing finalized F2 records remain read-only `LEGACY_PRE_F3`; no authorization
  rounds or decisions are backfilled.
- The F2 create path builds the Change twice, which can diverge server-generated
  snapshot fields; this must be fixed before F3.1.2, not in F3.1.0.
- Configured RBAC policy files are absent from ADO HEAD; resolve before F3.1.4.

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
