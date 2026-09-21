# F3.1.4 — Composed Authorization/Governance Read Representation Planning

## Status

PLANNING / SOURCE-VERIFICATION ONLY — NO IMPLEMENTATION AUTHORIZED.

Purpose: produce the implementation-ready plan for **F3.1.4**, the remaining F3.1 checkpoint: permission-filtered authorization/governance representation composed into Change detail.

Canonical authority:
- docs/adr/ADR-007-change-record-authority.md
- docs/adr/ADR-009-change-authorization-model.md
- docs/adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md
- docs/adr/ADR-014-change-resubmission-authority-and-identity-boundary.md
- docs/backstage/f3-1-implementation-plan.md
- docs/backstage/f3-1-3-implementation-plan.md
- docs/backstage/f3-1-3-revised-plan-architecture-rereview.md
- docs/backstage/f3-1-3a-architecture-implementation-acceptance.md
- docs/backstage/f3-1-3b-architecture-implementation-acceptance.md
- docs/backstage/current-state.md

Accepted implementation/runtime baseline:
- ADO repo: `platform-devops-developer-portal`
- branch: `feat/ado-repo-governance`
- accepted F3.1.3b / F3.1.3 SHA: `5e70d8818f55072d8568b5eb15bae754abb943d1`
- accepted F3.1.3a parent: `6bad066d945d49feaf642313ec37467e2658dc3f`
- accepted laptop demo target: isolated local runtime with gitignored `LEDGER_REQUIRED` overlay
- committed repository default remains `LEGACY_PRE_F3`

F3.1.3 is CLOSED / ACCEPTED IMPLEMENTED DECISION + RESUBMISSION BASELINE. F3.1.4 implementation remains NO-GO until this plan is independently ACCEPTed and a later implementation prompt is explicitly launched.

## 1. Mandatory fresh baseline

Before planning:
1. Fetch latest `backstage-docs@main` and record exact SHA.
2. Read every authority file above, implementation-progress, prompts/README, and this prompt.
3. Independently verify live ADO `feat/ado-repo-governance` tip. Expected `5e70d88`. If later accepted drift exists, inspect it; if conflicting unpublished drift exists, STOP `BLOCKED_BY_SOURCE_DRIFT`.
4. Verify committed `app-config.yaml` still defaults `newSubmissionAuthorizationMode: LEGACY_PRE_F3`.
5. Inspect actual source for:
   - `GET /changes` and `GET /changes/:changeId` contracts and status overlay;
   - `Change` / `ChangeSummary` frontend and backend types;
   - `GmudDetailPage` / `GmudListPage` current fields;
   - `ChangeManagementClient` (`getChange`, `recordDecision`; absence of resubmit/read-model APIs);
   - `AuthorizationLedgerRepository` read surface (`findRound`, `findCurrentRound`, `listRequirements`, `listAuditEvents`, decision lookups);
   - `AuthorizationEvaluation` / `GovernanceEvaluation` / `EligibilityService`;
   - permissions.ts + `rbac-policy.csv` + template-executor seed (create/read/decide/cab.record/resubmit);
   - Model C provider detail vs index discovery;
   - existing architecture guards that prohibit an F3.1.4 read model leaking into decision/resubmit responses.
6. Independently inspect durable laptop facts when useful, especially:
   - `CHG-2026-000003` AUTHORIZED Round 1 / submitted;
   - `CHG-2026-000005` Round 2 PENDING / submitted;
   - a `LEGACY_PRE_F3` Change such as `CHG-2026-000002`.
   Do not create, repair, approve, reject, or resubmit Changes merely to plan.

If source reality contradicts ADR-009 or the accepted F3.1 decomposition, do not invent a workaround. Record the contradiction and return BLOCKED.

## 2. Planning boundary

F3.1.4 owns the remaining F3.1 checkpoint named in ADR-009 and `f3-1-implementation-plan.md`:

```text
permission-filtered authorization/governance representation
composed into Change detail
```

That includes choosing:
- the public read transport;
- the composed DTO (what is current vs historical, derived vs stored);
- permission boundaries for seeing rounds/requirements/decisions/audits/evaluations;
- GMUD detail representation of that read model;
- whether command actions (decide / resubmit) appear on the same detail surface, without becoming CAB Workbench.

F3.1.4 does **not** own:
- CAB Workbench / CAB inbox (F3.2);
- F3.2 autonomy grants/revoke history;
- Teams as interaction channel;
- additive requirements after submission;
- decision reversal, abstention, expiry;
- execution start/completion lifecycle;
- production `LEDGER_REQUIRED` cutover;
- generic workflow/BPM;
- changing F3.1.3a/F3.1.3b command semantics;
- widening decision/resubmit HTTP responses into this read model;
- Kargo/Argo/GitOps mutation.

The F3.1.0 note that RBAC CSV files were absent from HEAD is **closed**. `packages/backend/config/rbac/rbac-policy.csv` exists and is in force. F3.1.4 must not treat that as an open prerequisite.

Do not implement code in this checkpoint. Do not author the F3.1.4 implementation prompt inside planning unless the produced plan is already independently ACCEPTed — it will not be, in this same run.

## 3. Known source facts the plan must re-verify

These are starting observations from accepted `5e70d88`, not architecture decisions:

- `GET /api/change-management/changes/:changeId` returns `{ change }` with operational GMUD fields. Authorization rounds/requirements/decisions/audits/evaluations are **not** on the public `Change` DTO.
- Status overlay from `change_index` onto provider detail is already accepted. That overlay is not an F3.1.4 read model.
- Decision command returns `{ decision, authorizationEvaluation, changeStatus, roundNumber }` only. Resubmit returns `{ changeId, roundNumber, status }`. Architecture guards forbid expanding those into a composed authorization panel.
- Frontend detail (`GmudDetailPage`) is read-only operational GMUD. `recordDecision` exists on the client with the comment “No authorization UI”. There is no resubmit client method.
- Ledger already has insert+read repository methods. No DDL should be assumed; the plan must prove whether F3.1.4 can be a pure composition of existing reads.
- Permissions today: `change.create`, `change.read`, `change.authorization.decide`, `change.authorization.cab.record`, `change.resubmit`. ADR-009 still lists unimplemented F3.1 permission classes: audit read, add requirement, policy administration, governance read-all, cancellation.
- `canReadChange` remains participant read (platform_admin OR requester OR owner team OR any responsibleRef team). Participant read grants no decide/resubmit/CAB authority.

## 4. Required architecture/source questions

The plan must answer every question below from actual source + accepted architecture. Do not leave “TBD”. Do not invent admin overrides.

### Q1 — Read transport

Choose the minimum public HTTP contract that composes authorization/governance into Change detail.

Decide:
- extend `GET /changes/:changeId` vs a dedicated nested read such as `GET /changes/:changeId/authorization` (or equivalent);
- whether list (`GET /changes`) gains any authorization summary, or remains discovery-only;
- user credentials only vs service consumers (Delivery already reads eligibility as service);
- response boundedness: command responses stay command-bounded.

Do not put provider workflow, Teams, or Azure DevOps approval objects in this contract.

### Q2 — Composed representation

Define the exact DTO composed from existing ledger facts + pure evaluators.

At minimum decide inclusion and current-vs-historical treatment of:
- current `roundNumber`;
- `AuthorizationEvaluation`;
- `GovernanceEvaluation` (and what it shows when no post-execution requirements exist);
- current requirements (`requirementId = requirementRole`, kind, phase, mandatory, principal type/ref);
- decisions on those requirements;
- historical rounds (Round 1 after Round 2 on `CHG-2026-000005`);
- authorization audits;
- execution eligibility (already a separate GET — compose, link, or leave separate).

Invariant: derived evaluations are never stored. Historical rounds remain immutable. Identity freeze fields are not rewritten by the read model.

### Q3 — Permission filtering

Define, for each field class, who may see it:

```text
participant change.read
decide permission
cab.record
change.resubmit
new audit-read (if any)
new governance-read-all (if any)
platform_admin technical role
```

ADR-009 allows F3.1 to define audit-read and governance-read-all. The plan must either:
- introduce the minimum new permission(s) with RBAC bindings, or
- explicitly defer those permissions and filter using only existing ones, with a fail-closed default.

Hard rules:
- participant read must not become decide/CAB/resubmit authority;
- `cab.record` must not become a general authorization-history dump;
- comments/reasons/evidence on decisions need an explicit visibility rule (ADR-009: sensitive comments require access controls);
- do not leak other users’ pending individual-requirement identity beyond what the accepted architecture already snapshots as a platform principal ref.

### Q4 — LEGACY_PRE_F3 and missing-round

Specify the composed representation for:
- `LEGACY_PRE_F3` Changes (no Round);
- `LEDGER_REQUIRED` with missing round (`NO_LEDGER_ROUND` eligibility);
- rejected Round 1 vs pending Round 2.

Do not fabricate a sandbox Round for display.

### Q5 — Model C composition

Prove the read stays Model C:
- provider remains operational GMUD detail;
- ledger remains adjacent authorization authority;
- index remains discovery + current lifecycle projection;
- F3.1.4 must not move operational fields into the ledger or authorization fields into the provider.

### Q6 — GMUD UI boundary

F3.1.3 deferred “requirement/decision panel” to F3.1.4 and “approve/reject UI / CAB inbox” to “F3.1.4 / Workbench”.

The plan must split that cleanly:

| Surface | F3.1.4 | F3.2 Workbench |
|---|---|---|
| See current round/requirements/evaluation on `/gmud/:changeId` | decide | — |
| Record a decision from that detail (individual / CAB) | decide | — |
| Resubmit a rejected Change from that detail | decide | — |
| CAB inbox / scheduling / autonomy grants | NO | F3.2 |

If F3.1.4 includes command buttons, they must call the **existing** F3.1.3a/F3.1.3b APIs. Do not invent a second decision/resubmit protocol. Buttons must be permission- and authority-gated in the UI **and** still fail closed server-side.

If F3.1.4 is display-only, say so explicitly and leave actions for a later named slice.

Portuguese product copy remains required for any new GMUD chrome, matching existing list/detail.

### Q7 — Slice decomposition

Recommend the smallest correct implementation decomposition (for example F3.1.4a read API, F3.1.4b GMUD panel/actions) **only if** combining them would force reviewers to accept unrelated persistence and UI contracts in one gate.

`Migration required: YES/NO` must be explicit. Prefer **NO**. Do not add DDL “for future proofing”.

### Q8 — Live demonstration target

Use the accepted laptop `LEDGER_REQUIRED` overlay. Plan read-only proofs against existing durable Changes:

- `CHG-2026-000003` — AUTHORIZED, submitted, Round 1, two decisions;
- `CHG-2026-000005` — Round 2 PENDING, submitted, Round 1 rejected history immutable;
- one `LEGACY_PRE_F3` Change — no fabricated Round.

If the plan includes action UI, the live proof may use those existing Changes only where the accepted command contracts already allow it. Do not reject `CHG-2026-000003` to continue a story. Do not approve Round 2 of `CHG-2026-000005` unless that is an explicit F3.1.4 demonstration of the existing decision API from the new UI.

Do not fabricate CAB membership.

## 5. Error and empty-state taxonomy

Produce a concrete error/empty table using existing public codes wherever possible:

- Change not found / not readable;
- LEGACY / no ledger round;
- permission deny vs empty filtered authorization section (distinguish 403 from “you may open the GMUD but not see audits”);
- Catalog/membership unavailable on any UI action path (commands remain F3.1.3 fail-closed).

## 6. Required plan document

Write:

```text
docs/backstage/f3-1-4-implementation-plan.md
```

The plan must include:
- exact objective and non-goals;
- answers to Q1–Q8;
- permission/RBAC table;
- DTO + route contracts;
- UI boundary table;
- source paths expected after inspection;
- proof matrix (read filtering, LEGACY safety, history immutability, F3.1.3a/3b command regressions, GMUD/Catalog/Deployments);
- live laptop demonstration script;
- `Migration required`;
- STOP / next-gate text.

Update factually:
- `docs/backstage/current-state.md`
- `docs/backstage/implementation-progress.md`
- `prompts/README.md`

Do not mark F3.1.4 implementation GO.

## 7. Forbidden while planning

MUST NOT:
- modify ADO source;
- create/repair Round/Decision/Audit facts;
- change laptop overlay/config;
- implement F3.1.4 UI/API;
- author the F3.1.4 implementation prompt in the same run;
- implement F3.2, Teams, execution lifecycle, or production cutover;
- mutate Kargo/Argo/GitOps.

## 8. Required final report

Return:

```text
Docs baseline reviewed: <sha>
ADO baseline reviewed: <sha>
Source drift: NONE | ACCEPTED_ONLY | BLOCKED
F3.1.4 planning: READY_FOR_REVIEW | BLOCKED
Read transport chosen: <route>
Permission model: EXISTING_ONLY | NEW_PERMISSIONS <names>
UI actions in F3.1.4: DISPLAY_ONLY | DETAIL_COMMANDS | DEFERRED
Slice decomposition: SINGLE | 4a+4b | OTHER
Migration required: NO | YES
Canonical plan: docs/backstage/f3-1-4-implementation-plan.md
F3.1.4 implementation: NO-GO
F3.2: NO-GO
Next gate: independent F3.1.4 plan architecture review
```

## 9. STOP

STOP after the plan exists and docs are updated.

Do not implement F3.1.4 from inside planning. Independent architecture review of the plan is the next gate. Implementation remains separately authorized.
