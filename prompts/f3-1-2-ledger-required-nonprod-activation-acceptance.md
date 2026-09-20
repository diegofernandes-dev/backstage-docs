# F3.1.2 — Non-Production LEDGER_REQUIRED Activation Acceptance Review

## Status

REVIEW ONLY — NO IMPLEMENTATION, CONFIGURATION CHANGE, OR CUTOVER AUTHORIZED.

Purpose: independently decide whether the executed non-production `LEDGER_REQUIRED` activation recorded in `docs/backstage/f3-1-2-ledger-required-nonprod-activation-evidence.md` is trustworthy, correctly scoped, and sufficient to become the live F3.1.3 demonstration baseline.

Canonical authority:
- docs/backstage/f3-1-2-ledger-required-nonprod-activation-evidence.md
- docs/backstage/f3-1-2b-architecture-implementation-acceptance.md
- docs/backstage/f3-1-2b-implementation-evidence.md
- docs/backstage/f3-1-2-implementation-plan.md
- docs/backstage/current-state.md
- docs/adr/ADR-009-change-authorization-model.md
- docs/adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md
- docs/backstage/product-convergence-deployments-evidence.md

Accepted implementation baseline:
`platform-devops-developer-portal@22495229502dabf2d99588599a156d862c5114fa`

Executed activation target:
operator laptop Backstage runtime, local SQLite, gitignored `app-config.local.yaml` overlay, final effective mode `LEDGER_REQUIRED`.

## 1. Mandatory fresh baseline

Before reviewing:
1. Fetch latest `backstage-docs@main` and record exact SHA.
2. Read all authority files above plus `implementation-progress.md`, `prompts/README.md`, the activation prompt, and this review prompt.
3. Verify the live ADO branch tip for `feat/ado-repo-governance` still contains accepted SHA `2249522` with no unaccepted source drift relevant to Change submission/ledger behavior.
4. Independently inspect the local non-production runtime if it is still available.
5. Do not rely only on the evidence document when the factual runtime/database state can still be checked.

If the laptop runtime is no longer available, the review may still inspect durable evidence, local database artifacts, logs, and source, but it must explicitly downgrade any gate that requires live confirmation. Do not invent a live proof.

## 2. Review boundary

This review may only:
- inspect source, config, logs, database facts, HTTP/UI behavior, and canonical docs;
- return ACCEPT or REJECT;
- state whether this laptop runtime is adequate as the F3.1.3 demonstration target.

Do not:
- edit ADO source;
- change committed `app-config.yaml`;
- change `app-config.local.yaml` merely to make a gate pass;
- create/rewrite ledger facts;
- create ApprovalDecision rows;
- implement F3.1.3/F3.1.4/F3.2;
- touch production;
- modify Delivery/Kargo/Argo/GitOps desired state.

## 3. Mandatory gates

G1 — Source and runtime lineage
- ADO accepted SHA `2249522` is verified.
- No unaccepted source drift invalidates the activation evidence.
- Runtime under review uses that accepted binary or an independently accepted descendant.

G2 — Non-production isolation
- Target is local/operator-only and non-production.
- SQLite database is isolated from shared/prod data.
- No production cluster, namespace, cloud DB, or GitOps target was invented.
- No production credentials or secrets were introduced by the activation.

G3 — Activation mechanism correctness
- Activation uses only the existing gitignored `app-config.local.yaml` overlay.
- Committed `app-config.yaml` remains `LEGACY_PRE_F3`.
- Effective runtime can be shown as `LEDGER_REQUIRED` from startup/runtime diagnostics.
- No new config framework/source change was introduced.

G4 — Pre-cutover legacy control
Independently verify `CHG-2026-000002` (or the recorded equivalent) is:
- one real product-created Change;
- reservation mode `LEGACY_PRE_F3`;
- completed/submitted;
- zero AuthorizationRounds;
- created before the activation flip.

G5 — Live ledger submission
Independently verify `CHG-2026-000003` (or the recorded equivalent) has:
- reservation + index mode `LEDGER_REQUIRED`;
- exactly one finalized Change/index;
- exactly one Round 1;
- canonical Change snapshot/hash evidence;
- active policy `default-change-authorization@2026-09-19.1`;
- active selector-bundle identity/digest;
- exactly two normal-low requirements: primary + CAB;
- `requirementId = requirementRole` / sourceRef coherence;
- resolved principal snapshots;
- zero ApprovalDecision rows at submission.

G6 — Canonical submission audit
Verify exactly the expected submission audit set for the activation proof:
- one `change.authorization.round_created`;
- one `change.authorization.policy_selected`;
- one `change.authorization.selector_bundle_bound`;
- one `change.authorization.requirement_materialized` per requirement;
- no fabricated decision audit.

G7 — Idempotent replay
Verify replay of the same actor/key/payload:
- same `changeId`;
- same submitted result;
- one reservation/index/Round/provider record;
- no duplicate requirements;
- no duplicate canonical submission audit.

G8 — Different-payload conflict
Verify same actor + same key + changed harmless payload returns `CONFLICT` and creates no second Change/Round/provider/requirement/audit set.

G9 — Stored-mode-wins across cutover
Verify replay of the pre-cutover LEGACY control while runtime default is LEDGER:
- same original `changeId`;
- reservation remains `LEGACY_PRE_F3`;
- still no Round;
- current default does not reinterpret it.

G10 — Same-binary backout
Verify the evidence/runtime proves:
- environment override was switched to `LEGACY_PRE_F3` on the same accepted binary;
- a genuinely new reservation then stored LEGACY;
- existing LEDGER reservation stayed LEDGER and retained its Round/evidence;
- no historical Round was deleted/rewritten.

G11 — Final steady state
Verify final laptop overlay is `LEDGER_REQUIRED` if this laptop is to be the F3.1.3 demo target.

Also verify the committed repository default is still `LEGACY_PRE_F3`.

G12 — Product preservation
With LEDGER active, independently verify as available:
- `/gmud` list;
- ledger-governed GMUD detail;
- Catalog;
- Catalog Component Deployments tab;
- Delivery reads;
- no `api:catalog/delivery` extension collision;
- no new startup/config-extension errors attributable to activation.

G13 — Execution eligibility
Verify the live ledger-governed normal-low Change with no decisions evaluates to not-ALLOW.

Recorded expected real result is:
`decision = DENY`, `reason = PENDING_AUTHORIZATION`, `roundNumber = 1`.

Do not accept a fabricated/mocked eligibility response.

G14 — Rollback safety
Verify recorded mandatory old-binary query returned zero pending LEDGER reservations at the backout checkpoint:

```sql
SELECT COUNT(*)
FROM change_idempotency
WHERE authorization_mode = 'LEDGER_REQUIRED'
  AND state = 'pending';
```

Old-binary downgrade must not have been performed.

G15 — Forbidden scope
Confirm:
- ADO source modified: NO;
- migration added: NO;
- production changed: NO;
- policy/selector mappings changed: NO;
- ApprovalDecision fabricated: NO;
- F3.1.3/F3.1.4/F3.2 behavior added: NO;
- Kargo/Argo/GitOps mutated: NO.

## 4. Adequacy of the laptop target for F3.1.3

Separately from activation correctness, determine whether the laptop runtime is adequate as the next F3.1.3 product-validation target.

Mark `F3.1.3 demo target readiness` as:
- `READY` if the same accepted product runtime can be reliably restarted, preserves its local ledger database across normal restarts, supports real Entra sign-in/Catalog selector resolution, and can exercise real HTTP/UI decision flows without production dependencies;
- `NOT_READY` if the environment is too ephemeral, cannot preserve ledger state, cannot authenticate/resolve principals reliably, or cannot support realistic product interaction.

A shared Kubernetes DEV environment is preferable later, but its absence alone does not make the laptop target invalid for F3.1.3 if the above product-validation properties are proven.

Do not turn this into a production-readiness review.

## 5. Decision

Return exactly one activation verdict:
`ACCEPT` or `REJECT`.

Do not use CONDITIONAL_ACCEPT.

Activation `ACCEPT` requires G1–G15 PASS or an explicitly non-applicable proof accepted by the contract. A missing mandatory proof is REJECT.

Report `F3.1.3 demo target readiness: READY | NOT_READY` separately.

## 6. If ACCEPT

Record:

```text
F3.1.2 non-production LEDGER_REQUIRED activation acceptance: ACCEPT
Activation baseline: ACCEPTED
Accepted runtime SHA: 22495229502dabf2d99588599a156d862c5114fa
Committed repository default: LEGACY_PRE_F3
Accepted non-production effective mode: LEDGER_REQUIRED
Production cutover: NOT AUTHORIZED
F3.1.3 demo target readiness: READY | NOT_READY
F3.1.3 planning/prompt authoring: GO
F3.1.3 implementation: still requires separate explicit authorization
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
```

If demo target readiness is READY, set the next activity to F3.1.3 planning/prompt authoring.

If activation is ACCEPT but demo target readiness is NOT_READY, next activity is only to establish a persistent non-production demo target; do not block the already accepted activation itself for topology preference alone.

## 7. If REJECT

Identify only concrete proof/operational defects.
Do not repair source/config/data during the review.
Keep F3.1.3 implementation NO-GO.

## 8. Canonical documentation

Create:
- docs/backstage/f3-1-2-ledger-required-nonprod-activation-acceptance.md

Update factually:
- docs/backstage/current-state.md
- docs/backstage/implementation-progress.md
- prompts/README.md

Commit documentation only and STOP.

## 9. Final report

Return:

```text
Docs baseline reviewed: <sha>
ADO/runtime SHA verified: <sha>
Independent runtime/database verification: YES | PARTIAL | NO
Source/lineage gate: PASS | FAIL
Isolation gate: PASS | FAIL
Activation mechanism gate: PASS | FAIL
Legacy control gate: PASS | FAIL
Live ledger submission gate: PASS | FAIL
Canonical audit gate: PASS | FAIL
Idempotent replay gate: PASS | FAIL
Different-payload conflict gate: PASS | FAIL
Stored-mode-wins gate: PASS | FAIL
Same-binary backout gate: PASS | FAIL
Final steady-state gate: PASS | FAIL
Product preservation gate: PASS | FAIL
Execution eligibility gate: PASS | FAIL
Rollback safety gate: PASS | FAIL
Forbidden-scope gate: PASS | FAIL
F3.1.2 non-production LEDGER_REQUIRED activation acceptance: ACCEPT | REJECT
F3.1.3 demo target readiness: READY | NOT_READY
ADO/source modified by review: NO
Production modified by review: NO
Final docs SHA: <sha>
```

## 10. STOP

STOP after the review.

Do not implement F3.1.3, F3.1.4, or F3.2 and do not perform any production cutover.