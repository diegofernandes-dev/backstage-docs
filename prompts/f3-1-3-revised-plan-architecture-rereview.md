# F3.1.3 — Revised Plan Architecture Re-Review

## Status

INDEPENDENT REVIEW ONLY — NO IMPLEMENTATION AUTHORIZED.

Purpose: independently decide whether the narrowly revised F3.1.3 plan is now an implementation-ready contract after ADR-014 closes the two blockers from the prior 18/20 REJECT.

Canonical authority:
- docs/backstage/f3-1-3-implementation-plan.md
- docs/backstage/f3-1-3-plan-architecture-review.md
- docs/adr/ADR-007-change-record-authority.md
- docs/adr/ADR-008-multi-activity-change-execution-plan.md
- docs/adr/ADR-009-change-authorization-model.md
- docs/adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md
- docs/adr/ADR-014-change-resubmission-authority-and-identity-boundary.md
- docs/backstage/f3-1-2-ledger-required-nonprod-activation-acceptance.md
- docs/backstage/current-state.md

Accepted implementation/runtime baseline:
- ADO `platform-devops-developer-portal`
- branch `feat/ado-repo-governance`
- expected accepted source SHA `22495229502dabf2d99588599a156d862c5114fa`
- accepted laptop demo target with effective `LEDGER_REQUIRED` overlay
- committed repository default `LEGACY_PRE_F3`

Historical review result:

```text
F3.1.3 plan architecture review: REJECT
Gates: 18/20 PASS
G13 resubmission actor authority: FAIL
G14 same-changeId identity boundary: FAIL
All F3.1.3a decision-command gates: PASS
```

## 1. Mandatory fresh baseline

Before reviewing:
1. Fetch latest `backstage-docs@main`; record exact SHA.
2. Read all authority files above, implementation-progress, prompts/README, and this prompt.
3. Fetch/verify the actual ADO branch tip.
4. Inspect any source drift after `2249522`. If drift affects decision/resubmission/index/provider/Catalog identity/RBAC semantics, evaluate it explicitly.
5. Compare the revised plan and ADR-014 against the historical REJECT. Verify the revision changed only the required F3.1.3b governance/identity contracts plus necessary cross-references/tests.

Do not rely only on the revision report. Verify source claims directly where relevant.

## 2. Review boundary

Do not:
- modify ADO code;
- create ApprovalDecision facts;
- change laptop config/data;
- author implementation prompts;
- implement F3.1.3a/F3.1.3b;
- implement F3.1.4/F3.2;
- alter production or Delivery/Kargo/Argo/GitOps state.

Return exactly `ACCEPT` or `REJECT`.

## 3. Regression of the 18 previously passing gates

Confirm the narrow revision did not regress the already-passing F3.1.3 contracts.

At minimum re-check:
- source inventory still matches ADO;
- F3.1.3a / F3.1.3b decomposition remains valid;
- decision transport/idempotency unchanged;
- individual authority unchanged;
- CAB live-membership + `cab.record` unchanged;
- decision permissions do not collapse business authority into `platform_admin`;
- trx-aware ledger-read contract unchanged;
- caller-owned transaction and exactly-once milestones unchanged;
- rejection lifecycle projection unchanged;
- no `authorized` lifecycle value;
- eligibility no longer fabricates a ledger Round in the planned 3a behavior;
- post-execution decisions remain fail-closed without completion evidence;
- provider replacement/current-projection composition remains coherent;
- `Migration required: NO` is still source-correct;
- PostgreSQL D1-D6 / R1-R3 proof contracts remain implementable;
- F3.1.4/F3.2/Teams/production remain excluded.

If the revision accidentally changed a frozen 3a semantic or creates a new contradiction, REJECT.

## 4. Blocker G13 re-review — resubmission actor authority

ADR-014 now defines:

```text
actor has change-management.change.resubmit
AND
(actorRef == original requestedBy
 OR actor is a current member of immutable ownerRef)
```

Independently verify this is complete and implementable against current source.

Required checks:
- dedicated `change-management.change.resubmit` permission can be registered using current permission infrastructure;
- `change.create` is not reused as business authority;
- original requester still needs the explicit permission;
- immutable-owner member still needs the explicit permission;
- current membership is checked live and prefix-agnostically;
- membership provider unavailable fails closed;
- missing/unresolvable immutable owner fails closed;
- no-owner Change permits only original requester + permission;
- `platform_admin` alone is not sufficient;
- CAB membership/`cab.record` alone is not sufficient;
- participant/read/responsibleRef alone is not sufficient;
- no generic governance override exists;
- permission assignment may be technically broad only because the server still enforces requester-or-owner proof.

Challenge these cases explicitly:
1. requester has no resubmit permission;
2. platform_admin has permission but is neither requester nor immutable-owner member;
3. CAB member has permission but neither domain proof;
4. original owner membership is removed before resubmit;
5. Catalog ownership drifts to a new owner;
6. immutable owner Group disappears from Catalog.

G13 PASS only if no actor-authority ambiguity remains.

## 5. Blocker G14 re-review — same-changeId identity boundary

ADR-014 freezes across rounds:
- `changeId`;
- original `requestedBy`;
- identity `createdAt`;
- `targetRef`;
- original snapshotted `ownerRef`;
- original snapshotted `systemRef`.

Verify:
- same-changeId correction cannot retarget;
- changed target/owner/System requires a new Change/new changeId;
- Catalog ownership drift does not rewrite existing Change identity;
- allowed non-identity corrections are bounded and compatible with current create contract;
- new Round uses current policy/selectors/principal snapshots while preserving identity;
- prior Round snapshots remain immutable;
- index/provider current projection updates non-identity fields only;
- `DevelopmentProvider.replaceCurrent` cannot mutate frozen identity;
- detail/list semantics remain coherent under Model C.

### Execution-plan hidden retarget

Source allows optional per-activity Component `targetRef`. Verify ADR-014's guard is source-correct:
- omitted activity target remains allowed;
- present activity Component resolves to a System;
- resolved System must equal immutable Change `systemRef`, including absent/absent;
- cross-System activity target fails before Round/provider/index mutation;
- activity target never rewrites Change identity.

Challenge:
1. changed Change `targetRef` with otherwise identical payload;
2. Catalog target owner changed from Team A to Team B;
3. corrected execution activity points to Component in another System;
4. corrected execution activity points to Component with no System while Change has one;
5. Change has no System and corrected activity introduces one;
6. title/risk/window/rollback/plan-only correction preserves identity and legitimately creates Round N+1.

G14 PASS only if 'same business Change' now has an enforceable source-level boundary.

## 6. ADR-014 quality gate

Verify ADR-014 is narrow and authoritative enough to close the gap without creating new scope.

It must:
- explicitly fill the ADR-009 gap rather than silently reinterpret old text;
- remain consistent with ADR-007 snapshot authority and ADR-008 execution-plan semantics;
- not grant CAB autonomy or reopen ADR-013;
- not authorize implementation by itself;
- define both positive authority and explicit non-authorities;
- define Catalog ownership drift semantics;
- define retargeting as new Change/new changeId;
- avoid provider-specific identities.

## 7. Resubmission transaction/idempotency re-check

Confirm the identity/authority revision composes with the existing F3.1.3b mechanics:
- permission and Catalog/domain proof happen before write transaction;
- identity mismatch fails before Round/provider/index mutation;
- one caller-owned transaction then locks `change_index`;
- current round must be REJECTED;
- `change.resubmit` idempotency reservation uses authenticated actor + new key;
- payload hash includes changeId + normalized corrected body;
- Round N + requirements + audits + index non-identity projection + participant rebuild + provider replacement commit atomically;
- identity columns remain untouched;
- loser of concurrent resubmission maps deterministically to `resubmission_conflict`;
- original create reservation is untouched.

Verify no new DDL is needed for these rules.

## 8. Revised proof matrix

Review whether R4-R13 are sufficient and internally consistent:
- R4 requester + permission success; requester without permission denied;
- R5 immutable-owner member + permission success; removed membership denied;
- R6 platform_admin alone denied;
- R7 CAB-only actor denied;
- R8 permission without requester/owner proof denied;
- R9 new Catalog owner alone cannot take over existing Change;
- R10 changed targetRef denied before mutation;
- R11 immutable owner/system mismatch denied before mutation;
- R12 allowed correction creates Round N+1 with frozen identity preserved;
- R13 hidden cross-System execution-plan retarget denied.

PostgreSQL remains authoritative for R1 concurrency. Identity/authority negatives may be functional SQLite/Postgres tests.

## 9. Decision

Return exactly one:
`ACCEPT` or `REJECT`.

Do not use CONDITIONAL_ACCEPT.

ACCEPT only if:
- prior G13 is now PASS;
- prior G14 is now PASS;
- the 18 previously passing gates remain materially valid;
- ADR-014 is coherent;
- no new implementation-semantic/governance ambiguity remains;
- `Migration required: NO` remains correct.

## 10. If ACCEPT

Record exactly:

```text
F3.1.3 revised plan architecture re-review: ACCEPT
F3.1.3 plan: ACCEPTED IMPLEMENTATION CONTRACT
ADR-014: ACCEPTED / governing F3.1.3b authority + identity
Migration required: NO
F3.1.3a implementation-prompt authoring: GO
F3.1.3a implementation: still requires separate explicit authorization
F3.1.3b implementation-prompt authoring: NO-GO until F3.1.3a independent acceptance
F3.1.3b implementation: NO-GO
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
Production cutover: NOT AUTHORIZED
```

Do not author the F3.1.3a implementation prompt from inside this review.

## 11. If REJECT

Identify only concrete remaining/new blockers.

Do not reopen previously accepted F3.1.3a contracts unless fresh source evidence proves an actual contradiction.

Set the next activity to the narrowest correction.

## 12. Canonical documentation

Create:
- `docs/backstage/f3-1-3-revised-plan-architecture-rereview.md`

Update factually:
- `docs/backstage/current-state.md`
- `docs/backstage/implementation-progress.md`
- `prompts/README.md`

Preserve:
- historical first plan review REJECT;
- ADR-014;
- revised plan history.

Commit documentation only and STOP.

## 13. Required final report

Return:

```text
Docs baseline reviewed: <sha>
ADO branch tip verified: <sha>
Independent ADO source verification: YES | NO
Source drift: NONE | NON_BLOCKING | BLOCKING
Previously passing gates regression: PASS | FAIL
G13 resubmission authority: PASS | FAIL
G14 same-change identity: PASS | FAIL
ADR-014 coherence: PASS | FAIL
Dedicated resubmit permission: PASS | FAIL
Requester/immutable-owner domain proof: PASS | FAIL
platform_admin standalone denied: PASS | FAIL
CAB standalone denied: PASS | FAIL
Catalog ownership drift rule: PASS | FAIL
Frozen identity fields: PASS | FAIL
Retarget => new Change rule: PASS | FAIL
Execution-plan hidden retarget guard: PASS | FAIL
Provider/index identity preservation: PASS | FAIL
Resubmission idempotency/transaction: PASS | FAIL
R4-R13 proof matrix: PASS | FAIL
Migration required: NO | CONTRADICTED
F3.1.3 revised plan architecture re-review: ACCEPT | REJECT
ADO implementation modified: NO
Final docs SHA: <sha>
```

## 14. STOP

STOP after the independent re-review.

Do not modify ADO implementation, create ApprovalDecision facts, author implementation prompts, or implement F3.1.3/F3.1.4/F3.2.