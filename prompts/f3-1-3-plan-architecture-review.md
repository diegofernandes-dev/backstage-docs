# F3.1.3 — Decision / New-Round Plan Architecture Review

## Status

INDEPENDENT REVIEW ONLY — NO IMPLEMENTATION AUTHORIZED.

Purpose: independently decide whether `docs/backstage/f3-1-3-implementation-plan.md` is an implementation-ready contract for F3.1.3 against the actual ADO source and accepted live-ledger product baseline.

Canonical authority:
- docs/adr/ADR-009-change-authorization-model.md
- docs/adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md
- docs/backstage/f3-1-implementation-plan.md
- docs/backstage/f3-1-2-implementation-plan.md
- docs/backstage/f3-1-2b-architecture-implementation-acceptance.md
- docs/backstage/f3-1-2-ledger-required-nonprod-activation-acceptance.md
- docs/backstage/f3-1-3-implementation-plan.md
- docs/backstage/current-state.md

Accepted implementation/runtime baseline:
- ADO `platform-devops-developer-portal`
- branch `feat/ado-repo-governance`
- accepted SHA `22495229502dabf2d99588599a156d862c5114fa`
- accepted laptop demo target with effective `LEDGER_REQUIRED` overlay
- committed repository default `LEGACY_PRE_F3`

## 1. Mandatory fresh baseline

Before reviewing:
1. Fetch latest `backstage-docs@main`; record exact SHA.
2. Read every authority document above, implementation-progress, prompts/README, and this review prompt.
3. Independently fetch/inspect the live ADO branch tip.
4. Verify whether the tip still equals `2249522`; inspect any later drift.
5. Inspect the actual source for every factual claim the plan relies on, especially ledger read/write APIs, index/provider status behavior, identity/Catalog membership, RBAC, eligibility, idempotency, and current migrations.
6. Re-read the accepted laptop ledger facts only as supporting product evidence; do not create decisions during this review.

If source drift materially changes the plan's assumptions, evaluate the plan against current source and REJECT if any implementation-semantic choice becomes open.

## 2. Review boundary

Do not:
- modify ADO code;
- create ApprovalDecision rows;
- change laptop overlay/config;
- author an implementation prompt;
- implement F3.1.3a/F3.1.3b;
- implement F3.1.4/F3.2;
- change ADR-009/ADR-013;
- touch production or Delivery/Kargo/Argo/GitOps desired state.

Return exactly `ACCEPT` or `REJECT`.

## 3. Mandatory gates

G1 — Source inventory accuracy
- Every claimed current-source limitation/capability is correct at the verified ADO tip.
- `ApprovalDecision`, ledger uniques, audit tables, `change_index.status`, provider APIs, current router, permissions, and eligibility behavior match the plan.
- The plan does not assume APIs/constraints that do not exist.

G2 — Slice decomposition
- F3.1.3a decision recording and F3.1.3b same-changeId resubmission are materially distinct and the split is justified.
- F3.1.3a can be accepted independently without hidden dependency on 3b.
- F3.1.3b is correctly gated on 3a acceptance.
- No F3.1.4/F3.2 concern is smuggled into either slice.

G3 — Decision transport contract
Verify the proposed nested decision route, body, required Idempotency-Key, command hash, first-commit/replay response behavior, and public error mapping are deterministic and provider/channel-neutral.

Pay special attention to whether returning `ApprovalDecision` directly exposes any internal-only or sensitive evidence that should instead be a bounded command response. If source/domain makes the proposed response unsafe or unstable, REJECT rather than hand-wave it.

G4 — Individual actor authority
- Individual decision authority is exact current authenticated actor equality with the snapshotted principal.
- No requester/owner/platform-admin override is invented.
- Read/create permissions do not imply decision authority.

G5 — CAB / authority actor proof
Verify the plan's live decision-time membership proof is compatible with current Backstage Catalog identity data.

Must prove:
- snapshotted authority remains the historical authority identity;
- current membership is checked live;
- membership logic is prefix-agnostic and does not misuse `ownershipEntityRefs`;
- Catalog unavailable/missing actor fails closed;
- one CAB authority decision remains one decision, not N member decisions;
- persisted `actingAuthorityRef` / actor / authorizationEvidence are truthful and bounded.

Review whether `relations.memberOf` plus `spec.memberOf` is semantically correct for the actual Catalog entity shape and whether duplicates/normalization are handled deterministically.

G6 — Permission/RBAC boundary
Verify literal permissions and RBAC bindings are safe:
- individual `decide` and CAB `cab.record` are distinct;
- `platform_admin` is not automatically CAB business authority;
- CAB role binding to the configured CAB group works with the actual RBAC plugin/policy syntax;
- an actor needs both server permission and domain actor proof.

Review the plan's choice to grant general `decide` to contributor/platform_admin. Because individual equality is also mandatory, verify this is least-privilege-enough and not an unintended future escalation surface.

G7 — Idempotency and terminal-decision semantics
Verify:
- one terminal decision per requirement/round;
- exact replay returns original logical result;
- same key/changed command conflicts;
- different key/same requirement conflicts;
- concurrent duplicate approve converges;
- concurrent approve vs reject yields exactly one terminal decision;
- retry after post-commit network loss converges;
- actor-scoped idempotency key behavior matches actual DB uniqueness.

Check the specific plan choice that one actor cannot reuse the same Idempotency-Key on a different requirement because `(actor_ref,idempotency_key)` is global. This must be explicit and acceptable, not accidental.

G8 — Transaction + trx-aware read contract
This is critical.

Verify the plan correctly identifies the PostgreSQL visibility gap when ledger reads use `this.knex` instead of the caller-owned transaction.

Review must establish that adding optional `trx` to the relevant read methods is sufficient and does not require DDL.

One transaction must consistently own:
- change_index row lock;
- current-round/requirement/existing-decision reads;
- decision insert;
- decision audit;
- milestone audit;
- rejection lifecycle projection.

Unique-violation loser must rollback before re-observe.

G9 — Exactly-once milestone audit
Verify `authorization_reached` / `round_rejected` exactly-once behavior is sound without a new unique index.

The plan relies on one `change_index FOR UPDATE` serialization point plus trx-aware audit reads. Confirm all code paths that can emit those milestones are serialized by that same lock. If another current/future accepted path can emit them outside the lock, REJECT or require a stronger contract.

G10 — Rejection lifecycle projection
Review the ADR-009 lifecycle requirement and Model C composition carefully.

Verify that:
- rejecting a mandatory pre-execution requirement atomically produces the canonical decision/audit facts;
- `change_index.status='rejected'` is only a rebuildable projection;
- list shows rejected;
- detail overlays index status onto provider detail without mutating provider record in 3a;
- no `authorized` lifecycle value is added;
- provider/index truth does not become contradictory in a way that breaks ADR-007.

Specifically inspect whether the current GET/detail mapper can safely overlay status without accidentally overlaying stale/non-current fields or leaking this projection rule to LEGACY_PRE_F3 Changes.

G11 — Eligibility safety cleanup
Verify removal/disablement of sandbox Round fabrication for `LEDGER_REQUIRED` is correct and does not break historical sandbox evidence or legacy behavior.

Missing ledger Round for a ledger-governed Change must fail closed rather than fabricate governance evidence.

Verify post-3b eligibility can safely use the current Round snapshot window while list/detail may use the current index projection.

G12 — Post-execution decision timing
Verify the generic decision command cannot record a mandatory post-execution retrospective before accepted execution-completion evidence exists.

Because current source has no accepted execution-completion event, the fail-closed `execution_completion_required` rule must be implementable without inventing evidence.

Check whether current source has any legacy/sandbox completion surrogate that must explicitly NOT count.

G13 — Resubmission authorization contract
This is a high-scrutiny gate.

The plan proposes `change.create` permission plus actor is requester OR current ownerRef member OR platform_admin.

ADR-009 defines same-changeId resubmission semantics but does not by itself clearly grant those actor classes resubmission authority.

Determine whether this authorization set is already justified by accepted source/architecture. If it is an invented governance rule, REJECT the plan and require an explicit architecture decision rather than silently blessing it.

No CAB-only resubmit unless architecture explicitly grants it.

G14 — Same-changeId corrected snapshot semantics
Verify F3.1.3b preserves history and has a single coherent current projection:
- same `changeId`;
- original `requestedBy` and identity `createdAt` preserved;
- corrected editable fields in new Round snapshot;
- new activity IDs where plan says so;
- new current published policy/selectors/principals;
- Round 1 snapshot immutable;
- max round is current;
- current index projection rebuilt from Round N snapshot;
- participant index rebuilt;
- GET detail aligns with current operational provider record;
- prior decisions/audits remain untouched.

Check carefully whether changing owner/system/target under the same `changeId` is compatible with ADR-007/ADR-008 and the business meaning of 'correction' versus 'fundamentally different business change'. If the boundary is undefined, REJECT or require the plan to constrain editable fields.

G15 — Provider replacement contract
Verify introducing explicit `DevelopmentProvider.replaceCurrent(change,trx)` is a valid bounded Model C operation and does not reinterpret ordinary create idempotency.

External provider behavior must fail closed until explicitly supported.

Check transactionality with the index/participants/Round N and whether provider replacement can be rolled back with the caller-owned transaction in DevelopmentProvider.

G16 — Resubmission idempotency/atomicity
Verify `change.resubmit` can safely use the existing idempotency table without DDL.

Review:
- operation namespace;
- actor-scoped key;
- payload hash includes changeId + corrected body;
- reservation lifecycle occurs inside the locked transaction;
- no dangling reservation after losing a Round race;
- exact replay behavior is defined;
- two actors racing to resubmit converge deterministically;
- old `change.create` reservation remains untouched.

G17 — Round terminal semantics
Verify exact meaning of terminal is consistent:
- `REJECTED` terminal for resubmission;
- `AUTHORIZED` terminal for new pre-execution decisions, but not equivalent to rejected/resubmittable;
- undecided pre requirements on a rejected round cannot later repair it;
- post-execution requirements do not accidentally make a rejected pre-execution round resubmittable in contradictory ways.

Any ambiguity between AuthorizationEvaluation terminality and GovernanceEvaluation must be resolved before ACCEPT.

G18 — Migration decision
Independently verify `Migration required: NO`.

Existing schema must be sufficient for:
- trx-aware reads;
- decision replay/conflict;
- lifecycle status `rejected` projection;
- multiple rounds;
- `change.resubmit` operation;
- current projection rewrite;
- provider replacement.

If any DB CHECK/column/index constraint blocks the proposed values/operations, REJECT.

G19 — PostgreSQL concurrency proof plan
Verify D1–D6 and R1–R3 are sufficient and implementable.

PostgreSQL must be authoritative for:
- duplicate decision races;
- approve-vs-reject;
- two requirements reaching AUTHORIZED;
- atomic rejection lifecycle;
- concurrent resubmission / monotonic Round N;
- history preservation.

No sleeps/timing-only proof.

G20 — Live product proof and scope boundaries
Verify the plan can demonstrate 3a on the accepted laptop target without fabricating membership:
- primary approval;
- CAB collective approval by a real current CAB member;
- derived AUTHORIZED;
- lifecycle remains submitted;
- eligibility still respects window.

Verify rejection uses a disposable Change, not the accepted happy-path `CHG-2026-000003`.

F3.1.4 UI, CAB Workbench, F3.2 autonomy, Teams, execution lifecycle, and production cutover remain excluded.

## 4. Required challenges

Before deciding, explicitly challenge at least these cases:
1. CAB group membership is removed between Round creation and decision.
2. Catalog is unavailable after permission passes but before SQL transaction starts.
3. Same actor reuses one idempotency key on a different requirement.
4. Two actors concurrently approve two different requirements and both observe the other as undecided.
5. Approve and reject race on the same requirement.
6. Rejection commits but HTTP response is lost; retry uses same key.
7. Detail GET after rejection when provider record still says submitted.
8. Resubmit changes targetRef to a different System/owner.
9. Two different authorized actors concurrently resubmit after rejection.
10. Resubmission transaction fails after provider replacement but before commit.
11. A post-execution CAB retrospective is attempted before execution completion exists.
12. A decision targets an old Round after Round 2 exists.

The review document must answer each challenge or reference the exact plan section that does.

## 5. Decision

Return exactly one:
`ACCEPT` or `REJECT`.

Do not use CONDITIONAL_ACCEPT.

ACCEPT only if no implementation-semantic or governance-authority decision remains open.

## 6. If ACCEPT

Record:

```text
F3.1.3 plan architecture review: ACCEPT
F3.1.3 plan: ACCEPTED IMPLEMENTATION CONTRACT
Migration required: NO
F3.1.3a implementation-prompt authoring: GO
F3.1.3a implementation: still requires separate explicit authorization
F3.1.3b implementation-prompt authoring: NO-GO until F3.1.3a acceptance
F3.1.3b implementation: NO-GO
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
Production cutover: NOT AUTHORIZED
```

Do not author the implementation prompt inside this review.

## 7. If REJECT

Identify only concrete blocking contracts. Do not redesign unrelated accepted F3.1.2 behavior.

Keep all F3.1.3 implementation NO-GO and set the next activity to the narrowest plan correction.

## 8. Canonical documentation

Create:
- `docs/backstage/f3-1-3-plan-architecture-review.md`

Update factually:
- `docs/backstage/current-state.md`
- `docs/backstage/implementation-progress.md`
- `prompts/README.md`

Preserve the planning document and planning history.

Commit documentation only and STOP.

## 9. Required final report

Return:

```text
Docs baseline reviewed: <sha>
ADO branch tip verified: <sha>
Independent ADO source verification: YES | NO
Source inventory gate: PASS | FAIL
Slice decomposition gate: PASS | FAIL
Decision transport gate: PASS | FAIL
Individual authority gate: PASS | FAIL
CAB membership gate: PASS | FAIL
Permission/RBAC gate: PASS | FAIL
Decision idempotency gate: PASS | FAIL
Transaction/trx-aware-read gate: PASS | FAIL
Milestone-audit gate: PASS | FAIL
Rejection lifecycle gate: PASS | FAIL
Eligibility safety gate: PASS | FAIL
Post-execution timing gate: PASS | FAIL
Resubmission authority gate: PASS | FAIL
Same-changeId snapshot gate: PASS | FAIL
Provider replacement gate: PASS | FAIL
Resubmission idempotency gate: PASS | FAIL
Round terminality gate: PASS | FAIL
Migration gate: PASS | FAIL
PostgreSQL concurrency plan gate: PASS | FAIL
Live-product/scope gate: PASS | FAIL
Challenges answered: <N>/12
F3.1.3 plan architecture review: ACCEPT | REJECT
ADO implementation modified: NO
Final docs SHA: <sha>
```

## 10. STOP

STOP after the plan review.

Do not modify ADO implementation, create ApprovalDecision facts, author implementation prompts, or implement F3.1.3/F3.1.4/F3.2.