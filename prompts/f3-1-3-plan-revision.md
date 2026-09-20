# F3.1.3 — Narrow Plan Revision: Resubmission Authority + Same-Change Identity

## Status

PLANNING / DOCUMENTATION REVISION ONLY — NO IMPLEMENTATION AUTHORIZED.

Purpose: revise only the two F3.1.3b blockers identified by the independent architecture review. Preserve the accepted/source-verified F3.1.3a contract unchanged.

Canonical authority:
- docs/backstage/f3-1-3-implementation-plan.md
- docs/backstage/f3-1-3-plan-architecture-review.md
- docs/adr/ADR-007-change-record-authority.md
- docs/adr/ADR-008-multi-activity-change-execution-plan.md
- docs/adr/ADR-009-change-authorization-model.md
- docs/adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md
- docs/backstage/current-state.md

Review verdict to close:

```text
F3.1.3 plan architecture review: REJECT
Gates PASS: 18/20
G13 resubmission actor authority: FAIL
G14 same-changeId identity boundary: FAIL
F3.1.3a contracts: source-accurate and must not be redesigned
```

## 1. Mandatory fresh baseline

Before revising:
1. Fetch latest `backstage-docs@main` and record exact SHA.
2. Read every authority document above, implementation-progress, prompts/README, and this prompt.
3. Verify the actual ADO `platform-devops-developer-portal/feat/ado-repo-governance` tip.
4. Expected accepted source baseline is `22495229502dabf2d99588599a156d862c5114fa`; inspect any later drift.
5. If drift materially changes resubmission/index/provider/identity assumptions, STOP with `BLOCKED_BY_SOURCE_DRIFT`.

Do not modify ADO code.

## 2. Frozen contracts — do not reopen

The following F3.1.3a decisions already passed the independent review and are frozen for this correction:
- nested decision POST route + required Idempotency-Key;
- individual actor equality with snapshotted principal;
- CAB live Catalog membership + dedicated `cab.record` permission;
- no `platform_admin` CAB shortcut;
- actor-scoped decision idempotency + one terminal decision per requirement;
- caller-owned transaction with `change_index FOR UPDATE`;
- trx-aware ledger reads;
- exactly-once `decision_recorded` / `authorization_reached` / `round_rejected` milestones through the serialized parent lock;
- rejected lifecycle as index status projection + detail status overlay;
- no `authorized` Change lifecycle state;
- fail closed for post-execution decisions without execution-completion evidence;
- PostgreSQL D1-D6 as authoritative decision concurrency proof;
- F3.1.3a contains no resubmission implementation;
- migration remains NO for 3a.

Do not redesign these unless fresh source drift proves one impossible.

## 3. Architecture decision A — resubmission authority

Record an explicit architecture decision; do not infer resubmission authority from read/create/admin roles.

For the F3.1.3b MVP, use this bounded rule:

```text
actor may resubmit the rejected Change iff

  actor has server permission
    change-management.change.resubmit

AND

  (actorRef == original Change.requestedBy
   OR actor is a current member of the Change's immutable ownerRef group)
```

Hard rules:
- `platform_admin` alone is NOT resubmission business authority;
- CAB membership/`cab.record` alone is NOT resubmission authority;
- `responsibleRef`/participant read alone is NOT resubmission authority;
- `change.create` alone is NOT enough;
- permission and domain actor proof are both mandatory;
- owner membership must use a current live Catalog membership proof, prefix-agnostic and fail closed on unavailable membership source;
- the owner identity checked is the immutable ownerRef of the original Change identity, not a newly retargeted owner.

RBAC may grant the literal `change-management.change.resubmit` capability to broad technical roles such as contributor/template_executor/platform_admin only if the service still enforces the requester-or-owner domain proof. Permission alone must never permit resubmission.

If actual Backstage permission wiring makes a separate literal permission impossible without disproportionate scope, STOP and document the concrete blocker rather than silently reusing `change.create`.

## 4. Architecture decision B — same-changeId identity boundary

Use the conservative ADR-009 interpretation:

> Resubmission corrects the same business Change. Retargeting to a different governed target is a new Change and therefore requires a new `changeId`.

For F3.1.3b, these identity fields are immutable across rounds:
- `changeId`;
- original `requestedBy`;
- original identity `createdAt`;
- `targetRef`;
- snapshotted `ownerRef` associated with that Change identity;
- snapshotted `systemRef` associated with that Change identity.

Do not re-resolve owner/system from Catalog during same-changeId resubmission merely because Catalog ownership drifted. Historical identity remains tied to the original governed target snapshot.

If the operator needs a different `targetRef`, owner, or System, the correction is out of scope for the existing Change: create a new Change / new `changeId`.

## 5. Fields that may be corrected

F3.1.3b may accept corrections to non-identity business/execution fields, subject to the existing create validation contract, including:
- title;
- summary/description;
- classification;
- risk;
- requested execution window;
- rollback information;
- evidence/references supported by the current create contract;
- execution-plan content.

New execution activity IDs remain acceptable for the new Round snapshot.

However, inspect the actual execution-plan schema. If an activity can independently point to another Catalog target/System, the revised plan must state the fail-closed rule that prevents a corrected execution plan from crossing the immutable Change target/System identity boundary. Do not leave a hidden retarget path through activity fields.

Do not invent fields that do not exist in source.

## 6. Resubmission command contract revision

Update only F3.1.3b sections so the command is a correction of the same identity.

Expected transport remains:

```text
POST /api/change-management/changes/:changeId/resubmissions
Idempotency-Key: <required new key>
```

The body may reuse the create-shaped editable payload for ergonomic reasons, but the server must reject identity changes rather than accepting/re-resolving them.

Define deterministic errors for:
- targetRef differs from original -> `CONFLICT` or `VALIDATION_ERROR` using the existing taxonomy, with a stable `details.reason` such as `change_identity_mismatch`;
- any source-level identity field mismatch;
- actor has permission but is neither requester nor current member of immutable ownerRef -> `FORBIDDEN`;
- owner membership source unavailable -> `PROVIDER_UNAVAILABLE`;
- owner group missing/unresolvable -> fail closed.

Do not add a generic governance override.

## 7. Catalog ownership drift

Make the contract explicit for this case:

```text
Round 1 Change identity snapshot:
targetRef = component/system X
ownerRef  = Team A
systemRef = System X

Catalog later says target X is now owned by Team B
```

Same-changeId resubmission still uses Team A / System X as the immutable Change identity snapshot for authority/history in F3.1.3b.

Team B ownership drift does not silently transfer resubmission rights for the existing Change.

If governance wants ownership transfer to affect an in-flight/rejected Change, that is a separate future policy/administrative operation and is not part of F3.1.3b.

## 8. Provider/index/new-Round composition

Preserve the already reviewed Model C composition:
- prior Round snapshots immutable;
- max round is current authorization round;
- current index/provider may project the latest corrected non-identity fields;
- explicit `DevelopmentProvider.replaceCurrent(change,trx)` remains acceptable;
- external providers fail closed until supported;
- participants derived from the corrected execution plan may be rebuilt;
- identity fields in the current projection must remain the original immutable values.

Do not let `replaceCurrent` alter targetRef/ownerRef/systemRef/requestedBy/createdAt.

## 9. Resubmission idempotency and concurrency

Preserve the accepted 3b mechanics:
- new operation namespace `change.resubmit`;
- original `change.create` reservation untouched;
- actor-scoped required Idempotency-Key;
- payload hash includes `changeId` + normalized corrected body;
- reserve/complete in the serialized transaction;
- two resubmissions serialize on `change_index FOR UPDATE`;
- at most one Round N+1 wins;
- loser returns deterministic `resubmission_conflict`;
- prior rounds/requirements/decisions/audits remain immutable.

Add authority-specific negatives:
- requester loses role/technical permission -> forbidden even though requester identity matches;
- owner member loses live membership before command -> forbidden;
- platform_admin who is neither requester nor immutable-owner member -> forbidden;
- CAB member who is neither requester nor immutable-owner member -> forbidden.

## 10. Architecture record

Because the independent review identified these as governance/identity decisions not previously granted by ADR-009, create a narrow new ADR:

`docs/adr/ADR-014-change-resubmission-authority-and-identity-boundary.md`

Status: `Accepted`.

The ADR must record only:
1. same-changeId resubmission business authority = dedicated server permission + (original requester OR current member of immutable ownerRef);
2. platform admin / CAB / participant-read are not standalone resubmission authority;
3. same-changeId correction freezes `targetRef`, original `ownerRef`, original `systemRef`, `requestedBy`, and identity `createdAt`;
4. target/System/owner retargeting requires a new Change/new `changeId`;
5. correctable non-identity fields may create a new Round with current policy/selectors and preserved history;
6. Catalog ownership drift does not rewrite the identity/authority of an existing Change;
7. no production/runtime behavior is authorized by the ADR itself.

Update ADR index/readme accordingly.

Do not reopen ADR-013 CAB autonomy.

## 11. Required plan edits

Revise `docs/backstage/f3-1-3-implementation-plan.md` narrowly:
- status -> `REVISION COMPLETE — READY FOR RE-REVIEW`;
- authority includes ADR-014;
- Q11 resubmission authority and identity fields updated;
- permission table includes dedicated resubmit permission for 3b;
- error taxonomy updated;
- F3.1.3b source/test expectations updated;
- R1-R3 expanded with authority/identity negatives;
- risks/rejected alternatives updated;
- explicit decisions checklist updated;
- GO/NO-GO remains implementation NO-GO pending re-review.

Do not rewrite F3.1.3a sections except cross-references required by ADR-014.

## 12. Mandatory revised tests/proofs

Add to the F3.1.3b implementation contract:
- R4 original requester with resubmit permission succeeds after rejected Round;
- R5 current immutable-owner member with resubmit permission succeeds;
- R6 platform_admin alone fails;
- R7 CAB authority member alone fails;
- R8 actor with resubmit permission but no requester/owner proof fails;
- R9 current Catalog owner changed from original owner: new owner alone cannot resubmit existing Change;
- R10 changed `targetRef` under same `changeId` fails before Round/provider/index mutation;
- R11 changed immutable owner/system identity fails closed;
- R12 correction of allowed fields creates Round N+1 and preserves immutable identity fields;
- R13 execution-plan hidden retarget/cross-System path fails closed if the source schema can express one.

PostgreSQL remains authoritative for concurrent R1. Functional authority/identity negatives may run on both SQLite and PostgreSQL.

## 13. Resulting gate

Return exactly one:

```text
F3.1.3 plan revision: READY_FOR_REREVIEW
F3.1.3 implementation: NO-GO
F3.1.3a implementation-prompt authoring: NO-GO pending fresh ACCEPT
F3.1.3a implementation: NO-GO
F3.1.3b implementation: NO-GO
Migration required: NO
```

or:

```text
F3.1.3 plan revision: BLOCKED
F3.1.3 implementation: NO-GO
Blocking contract: <precise>
```

Do not ACCEPT the plan from inside the revision checkpoint.

## 14. Canonical documentation updates

Create:
- `docs/adr/ADR-014-change-resubmission-authority-and-identity-boundary.md`

Update:
- `docs/adr/README.md`
- `docs/backstage/f3-1-3-implementation-plan.md`
- `docs/backstage/current-state.md`
- `docs/backstage/implementation-progress.md`
- `prompts/README.md`

Preserve:
- `docs/backstage/f3-1-3-plan-architecture-review.md` as historical REJECT evidence.

Set the next current activity to a fresh independent F3.1.3 plan re-review.

Commit documentation only and STOP.

## 15. Required final report

Return:

```text
Docs baseline reviewed: <sha>
ADO branch tip verified: <sha>
Source drift: NONE | NON_BLOCKING | BLOCKED
ADR-014 created: YES | NO
Resubmission actor authority: RESOLVED | BLOCKED
Same-changeId identity boundary: RESOLVED | BLOCKED
targetRef immutable across rounds: YES | NO
ownerRef/systemRef immutable across rounds: YES | NO
platform_admin standalone resubmit: NO
CAB standalone resubmit: NO
Catalog ownership drift rule: RESOLVED | BLOCKED
Execution-plan hidden retarget rule: RESOLVED | NOT_APPLICABLE | BLOCKED
Migration required: NO | BLOCKED
F3.1.3 plan revision: READY_FOR_REREVIEW | BLOCKED
F3.1.3 implementation: NO-GO
ADO implementation modified: NO
Final docs SHA: <sha>
```

## 16. STOP

STOP after the narrow plan/ADR revision.

Do not modify ADO implementation, create ApprovalDecision facts, author an implementation prompt, or implement F3.1.3/F3.1.4/F3.2.