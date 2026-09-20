# F3.1.2 — Non-Production LEDGER_REQUIRED Activation / Cutover

## Status

OPERATIONAL ACTIVATION CHECKPOINT — DO NOT EXECUTE WITHOUT EXPLICIT USER LAUNCH.

Purpose: activate the already accepted F3.1.2 ledger-governed submission path on one clearly identified non-production Backstage product environment, prove it with real GMUD submissions, and leave the committed repository default unchanged.

This is not a new F3 implementation slice. It is the controlled operational activation of the accepted F3.1.2 capability.

Canonical authority:
- docs/backstage/f3-1-2b-architecture-implementation-acceptance.md
- docs/backstage/f3-1-2b-implementation-evidence.md
- docs/backstage/f3-1-2-implementation-plan.md
- docs/backstage/current-state.md
- docs/adr/ADR-009-change-authorization-model.md
- docs/adr/ADR-013-cab-governance-delegated-low-risk-autonomy.md
- docs/backstage/product-convergence-deployments-evidence.md

Accepted implementation baseline:
- platform-devops-developer-portal / feat/ado-repo-governance
- accepted F3.1.2b SHA: 22495229502dabf2d99588599a156d862c5114fa
- committed default: newSubmissionAuthorizationMode = LEGACY_PRE_F3

## 1. Mandatory fresh baseline

Before any activation:
1. Fetch latest backstage-docs main and record exact SHA.
2. Read all canonical authority files above plus implementation-progress, prompts/README, and this prompt.
3. Fetch/verify the live ADO tip for feat/ado-repo-governance.
4. Verify the runtime candidate contains accepted F3.1.2b SHA 2249522 and the accepted Deployments convergence baseline.
5. If source drift exists after 2249522, inspect it. If it touches Change submission, authorization runtime, ledger persistence, app-config authorization semantics, execution eligibility, Delivery/GMUD integration, or database migrations, STOP with BLOCKED_BY_SOURCE_DRIFT unless that drift has its own accepted evidence.

Do not activate a stale or unreviewed binary.

## 2. Identify the non-production target — no fabrication

Before changing configuration, identify exactly one real non-production Backstage product environment that the user can exercise.

Record:
- environment name;
- runtime/deployment location;
- exact running implementation SHA;
- database engine and database identity/name at a non-secret level;
- configuration source/override mechanism;
- restart/redeploy mechanism;
- how the user reaches the UI;
- whether Delivery reads are available there;
- operator responsible for rollback during this checkpoint.

Preferred target order:
1. existing shared DEV/non-production Backstage runtime, if one is actually present and safely isolated;
2. otherwise the existing developer/laptop runtime used to demonstrate the product, if it uses an isolated non-production database and represents the current product line.

Do NOT invent a Kubernetes namespace, production runtime, cloud environment, database, or deployment target merely to make this checkpoint green.

If no safely isolated non-production runtime is identifiable, return BLOCKED_BY_NO_NONPROD_TARGET and STOP.

Production is forbidden.

## 3. Activation mechanism

Activate only the selected non-production runtime using the existing supported environment-specific configuration/override mechanism.

Effective runtime value after activation:

```text
changeManagement.authorization.newSubmissionAuthorizationMode = LEDGER_REQUIRED
```

Hard constraints:
- do not change the committed app-config.yaml default from LEGACY_PRE_F3;
- do not commit LEDGER_REQUIRED as a repository-wide default;
- do not alter the accepted policy artifact or selector bundle;
- do not add a feature-flag framework;
- do not change source code solely to activate the value;
- do not modify production config;
- do not reuse production credentials/secrets;
- do not change Delivery/Kargo/Argo/GitOps desired state.

If the current project has no supported environment-specific override capable of changing this setting without changing the committed safe default, STOP with BLOCKED_BY_ACTIVATION_MECHANISM. Do not improvise a new config system in this checkpoint.

## 4. Preflight before flip

Before activation prove:
- accepted F3.1.2b binary is running or ready to run;
- startup active policy is default-change-authorization@2026-09-19.1;
- active selector bundle passes startup validation;
- normal-primary and cab-authority selectors resolve in this environment;
- database migrations required by F3.1.0 already exist and are healthy;
- product-convergence Deployments tab/runtime is present;
- GMUD create/list/detail is reachable;
- current effective newSubmissionAuthorizationMode is LEGACY_PRE_F3.

Capture current counts/identities for the non-production database:
- pending LEGACY_PRE_F3 reservations;
- pending LEDGER_REQUIRED reservations;
- completed LEDGER_REQUIRED reservations;
- existing Round-1 count.

Do not delete or rewrite existing facts to obtain a clean baseline.

## 5. Establish a real legacy control before activation

Create or identify one safe, disposable/non-production logical submission under the current LEGACY_PRE_F3 mode with a unique Idempotency-Key.

Prefer creating a fresh normal-low test GMUD through the real HTTP/UI path immediately before activation.

Record:
- changeId;
- idempotency key identifier (do not expose secrets);
- reservation authorization mode = LEGACY_PRE_F3;
- no AuthorizationRound created for that logical submission.

This provides a live cross-cutover control.

If the environment cannot create a safe legacy control without impacting real users/data, document the reason and STOP rather than fabricating DB rows manually.

## 6. Activate and restart/redeploy

Apply the environment-scoped LEDGER_REQUIRED override and restart/redeploy only the selected non-production Backstage runtime through its normal operational mechanism.

After startup prove from runtime logs/config diagnostics:
- effective mode = LEDGER_REQUIRED;
- active policy = default-change-authorization@2026-09-19.1;
- active selector bundle validated;
- change-management plugin initialized;
- Delivery plugin/Deployments product surface still initializes;
- no config/extension collision warnings.

Do not edit the committed repository default.

## 7. Live ledger-governed submission proof

Create a new **normal + low** GMUD through the real product path using a new unique Idempotency-Key.

Prefer the Backstage GMUD UI for the primary proof; use API/database inspection as supporting evidence.

Validate all of the following against canonical database/API facts:
- one new idempotency reservation;
- reservation.authorization_mode = LEDGER_REQUIRED;
- one canonical changeId;
- one finalized change_index row with authorization_mode = LEDGER_REQUIRED;
- exactly one AuthorizationRound with roundNumber = 1;
- Round snapshot hash matches the canonical Change snapshot contract;
- active policy identity is default-change-authorization@2026-09-19.1;
- active selector bundle identity is recorded;
- exactly the expected effective normal-low requirements are materialized: primary + CAB;
- requirementId equals requirementRole;
- principal snapshots are present and match resolved selectors;
- required submission audit events exist exactly once;
- Change lifecycle/status remains submitted;
- no ApprovalDecision exists merely because the Change was submitted.

Do not manually insert ledger rows to make this proof pass.

## 8. Live idempotency proof

Replay the exact same logical submission with the same actor, Idempotency-Key, and payload through the normal API path.

Prove:
- same changeId;
- same logical submitted response;
- still exactly one reservation;
- still exactly one finalized index;
- still exactly one Round 1;
- no duplicate requirements;
- no duplicate canonical submission audit events;
- no duplicate provider record.

Then submit the same actor + same key with a deliberately different harmless payload and prove CONFLICT before any second authorization artifact appears.

Do not use this checkpoint to stress-test high-volume concurrency; authoritative concurrency is already accepted in F3.1.2b.

## 9. Stored-mode-wins across the live cutover

After LEDGER_REQUIRED activation, replay the pre-activation LEGACY control using its original actor/key/payload.

Prove:
- it remains LEGACY_PRE_F3;
- same logical result/changeId;
- no AuthorizationRound is fabricated for it;
- current runtime default does not reinterpret the reservation.

This is mandatory live cross-cutover evidence.

## 10. Product-level proof

With LEDGER_REQUIRED active:
- `/gmud` list remains reachable;
- newly created ledger-governed GMUD detail is reachable;
- Catalog remains reachable;
- Catalog Component Deployments tab remains visible/reachable;
- Delivery reads do not regress;
- no `api:catalog/delivery` collision appears;
- browser console/startup logs show no new extension/config errors.

Because F3.1.3 decision commands do not exist yet, the ledger-governed normal-low Change must remain authorization-pending. Do not fabricate approvals.

If the existing execution-eligibility read path can evaluate this Change, prove it does not return execution ALLOW while mandatory primary/CAB requirements are undecided. Record the exact real response/state rather than assuming a label.

## 11. Backout behavior on the same accepted binary

Prove the operational rollback mechanism without downgrading the binary:
1. switch the environment-specific default back to LEGACY_PRE_F3 using the same supported override mechanism;
2. restart/redeploy the same accepted F3.1.2-capable binary;
3. prove a genuinely new reservation now stores LEGACY_PRE_F3;
4. replay the previously created LEDGER_REQUIRED submission and prove its stored LEDGER_REQUIRED regime remains authoritative and returns the same logical result;
5. prove no Round is removed or rewritten.

Then decide the final non-production steady state for this checkpoint:

**If all activation/backout proofs pass and the environment is the intended F3.1.3 demonstration environment, set the environment-specific override back to LEDGER_REQUIRED and leave that non-production runtime active in ledger mode.**

If it is only an ephemeral/local proof environment, it may finish in LEGACY_PRE_F3, but the evidence must state that clearly and F3.1.3 live-product validation remains blocked until a persistent non-production ledger target exists.

The committed repository default remains LEGACY_PRE_F3 in both cases.

## 12. Binary rollback hazard — mandatory

Do NOT downgrade to any binary predating F3.1.2 during this checkpoint.

Before any future downgrade to a pre-F3.1.2 binary, the accepted mandatory query is:

```sql
SELECT COUNT(*)
FROM change_idempotency
WHERE authorization_mode = 'LEDGER_REQUIRED'
  AND state = 'pending';
```

Required result: `0`.

If nonzero, old-binary rollback is forbidden until those logical submissions are drained through supported F3.1.2 recovery.

The same-binary config backout described above does not reinterpret existing reservations and is the preferred rollback for this activation checkpoint.

## 13. Failure handling

If activation causes a runtime/product failure:
- capture the factual error;
- revert only the non-production override to LEGACY_PRE_F3;
- restart/redeploy the same accepted F3.1.2-capable binary;
- verify GMUD/Catalog/Deployments recover;
- do not delete ledger facts;
- do not downgrade the binary unless the zero-pending rule is independently satisfied and a separate downgrade action is explicitly authorized.

Return FAIL with the smallest concrete root cause. Do not fix unrelated product issues inside this checkpoint.

## 14. Forbidden scope

MUST NOT:
- modify F3.1.2b source code;
- commit LEDGER_REQUIRED as global/default app config;
- activate production;
- implement F3.1.3 decision commands;
- implement F3.1.4 read/RBAC;
- implement F3.2 CAB autonomy;
- add CAB Workbench;
- create ApprovalDecision manually;
- add Teams approval;
- change authorization policy or selector mappings merely to make the proof pass;
- add migrations;
- rewrite/delete ledger facts;
- apply Kargo/Argo/GitOps resources;
- modify Delivery desired state;
- invent an infrastructure target.

## 15. Acceptance criteria

Activation checkpoint PASS requires:
- real non-production target identified;
- exact accepted implementation lineage verified;
- environment-only activation mechanism proven;
- committed default remains LEGACY_PRE_F3;
- live normal-low submission stores LEDGER_REQUIRED;
- exactly one Round 1 with primary + CAB requirements;
- immutable policy/selector evidence present;
- idempotent replay creates no duplicates;
- payload mismatch remains CONFLICT;
- pre-activation LEGACY reservation remains legacy after flip;
- same-binary backout to LEGACY works;
- existing LEDGER reservation remains ledger after backout;
- GMUD/Catalog/Deployments product surface remains healthy;
- no approvals fabricated;
- no source/migration/production changes.

## 16. Evidence and documentation

After a real execution create:
- docs/backstage/f3-1-2-ledger-required-nonprod-activation-evidence.md

Update factually:
- docs/backstage/current-state.md
- docs/backstage/implementation-progress.md
- prompts/README.md

Record:
- docs baseline;
- ADO runtime SHA;
- exact target environment description;
- activation mechanism (without secrets);
- preflight counts;
- legacy control changeId/mode;
- ledger proof changeId and Round/requirement/audit counts;
- idempotent replay proof;
- payload-conflict proof;
- stored-mode-wins proof;
- backout proof;
- final effective non-production mode;
- product/browser proof;
- any limitations.

Do not mark the activation independently accepted merely because execution PASSed.

## 17. Next gate

After execution PASS, next gate is an independent **F3.1.2 non-production LEDGER_REQUIRED activation acceptance review**.

F3.1.3 planning/prompt authoring remains architecture-GO, but F3.1.3 implementation should not begin until the activation is independently accepted and a persistent/non-production ledger-governed target exists for product validation.

## 18. Required final report

Return:

```text
Docs baseline reviewed: <sha>
ADO/runtime SHA: <sha>
Non-production target: <description>
Target isolation: PASS | FAIL
Source drift: NONE | ACCEPTED_ONLY | BLOCKED
Activation mechanism: <non-secret description>
Committed repository default remains LEGACY_PRE_F3: YES | NO
Effective target mode after activation: LEDGER_REQUIRED | OTHER
Legacy pre-cutover control: PASS | FAIL
Live ledger submission: PASS | FAIL
normal.low primary + CAB Round 1: PASS | FAIL
Canonical audit evidence: PASS | FAIL
Idempotent same-payload replay: PASS | FAIL
Different-payload conflict: PASS | FAIL
Stored-mode-wins across cutover: PASS | FAIL
Same-binary config backout: PASS | FAIL
Existing LEDGER reservation preserved after backout: PASS | FAIL
GMUD/Catalog/Deployments product proof: PASS | FAIL
Execution eligibility pending/not-ALLOW proof: PASS | FAIL | NOT_AVAILABLE_EXPLAINED
Old-binary downgrade performed: NO
Source code modified: NO
Migrations added: NO
Production modified: NO
Final non-production mode: LEDGER_REQUIRED | LEGACY_PRE_F3
Activation checkpoint: PASS | FAIL | BLOCKED_BY_NO_NONPROD_TARGET | BLOCKED_BY_ACTIVATION_MECHANISM | BLOCKED_BY_SOURCE_DRIFT
Canonical evidence: docs/backstage/f3-1-2-ledger-required-nonprod-activation-evidence.md
Next gate: independent activation acceptance review
```

## 19. STOP

STOP after activation/backout/evidence.

Do not implement F3.1.3, F3.1.4, or F3.2 in this checkpoint.