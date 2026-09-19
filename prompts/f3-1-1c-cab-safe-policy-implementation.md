# F3.1.1c — CAB-Safe Policy Publication Implementation

## Status

IMPLEMENTATION PROMPT — execute only after explicit user launch.

Architecture authority:
- ADR-009 as partially superseded by ADR-013
- docs/backstage/f3-1-1-implementation-plan.md
- docs/backstage/f3-1-2-implementation-plan.md
- docs/backstage/f3-1-2-final-architecture-rereview.md

Final review authorizes F3.1.1c prompt authoring. Implementation is not automatically authorized.

## Objective

Publish and activate one new immutable default-change-authorization policy version whose only business-policy change is:

normal.low: primary approval + CAB

while preserving:
- normal.medium: primary + CAB
- normal.high: primary + CAB
- emergency rules unchanged
- policyModelVersion unchanged
- existing selector identities unchanged
- existing published policy identities immutable

The currently accepted historical policy default-change-authorization@2026-09-02.1 must remain byte-for-byte historical evidence and must never be edited or reused for the new behavior.

F3.1.1c must not implement CAB autonomy. Autonomy belongs to F3.2.

## Mandatory fresh baseline

1. Fetch latest backstage-docs main and record exact SHA.
2. Read ADR-013, the F3.1.1 plan, F3.1.2 plan, final F3.1.2 architecture re-review, current-state, progress, and this prompt.
3. Fetch actual ADO repo platform-devops-developer-portal, branch feat/ado-repo-governance.
4. Record exact tip. Last accepted baseline before later authorized slices is 188d8e9cc43423f3644b3cacfb9849257838a583.
5. Inspect current policy directory, publication manifest, registry, activePolicy config pin, publication validator, selector runtime, and tests.
6. If the branch has advanced because F3.1.2a was implemented first, use the actual tip and verify that drift does not alter policy publication semantics.
7. If drift changes policy/manifest/registry/active-policy contracts, stop with BLOCKED_BY_SOURCE_DRIFT.

## Exact policy change

Create a NEW published policy artifact under the existing policy publication convention.

Choose a new unused business policy version consistent with the repository convention. Do not reuse 2026-09-02.1. Keep policyModelVersion = 1.

New matrix:
- normal.low => normal-primary + cab
- normal.medium => unchanged primary + cab
- normal.high => unchanged primary + cab
- emergency.low/medium/high => unchanged from the accepted policy

Reuse existing requirement roles and selector keys wherever behavior is unchanged.

For normal.low, reuse the existing normal primary requirement and the existing CAB requirement shape/selector already used by normal.medium/high. Do not invent a second CAB semantic.

## Publication integrity

Use the existing F3.1.1a append-only publication mechanism exactly:
- packages/backend/src/modules/changeManagement/authorization/policy/published-manifest.json
- policyArtifactDigest over policyModelVersion + rules
- validatePublishedManifest
- validate:policy-publication command
- existing registry/deep-freeze semantics

Append a new manifest entry. Never edit the digest/content entry for 2026-09-02.1.

Validate the new publication against the actual trusted baseline/ref immediately preceding this implementation, using the existing non-genesis path. Do NOT use genesis mode for F3.1.1c.

## Activation

Update the existing changeManagement.authorization.activePolicy config pin to the new policy key/version through the same config surface already used by F3.1.1b.

Do not change selector bundle identity unless source reality proves the existing active bundle lacks cab-authority. The final architecture review verified cab-authority exists and is reusable at 188d8e9.

Startup validation of the active policy + active selector bundle must pass.

Because POST /changes remains unwired until F3.1.2b, this policy activation must not create AuthorizationRounds or change legacy submission behavior by itself.

## Expected code surface

Use actual current source paths, expected to include:
- a new file under packages/backend/src/modules/changeManagement/authorization/policy/published/
- packages/backend/src/modules/changeManagement/authorization/policy/registry.ts only if explicit registration requires a new entry
- packages/backend/src/modules/changeManagement/authorization/policy/published-manifest.json
- focused policy publication/matrix tests
- the existing app-config activePolicy pin

Do not modify generic evaluator semantics, policyModelVersion dispatch, canonical hashing rules, immutable helper semantics, or selector resolver semantics unless source drift proves a blocking inconsistency.

## Hard non-goals

Do not:
- edit the historical 2026-09-02.1 policy artifact
- reuse its version string
- change policyModelVersion
- add a policy DSL/rules engine
- add a database/migration
- wire ChangeManagementService
- create AuthorizationRound
- change authorization_mode
- implement F3.1.2a or F3.1.2b
- add CabAutonomyGrant
- add skipCab/waiver logic
- add autonomy RBAC
- add CAB Workbench
- modify medium/high/emergency behavior
- change selector principal bindings
- change frontend/Delivery/Teams

## Mandatory tests

P1 — historical identity immutability:
- old 2026-09-02.1 artifact remains unchanged
- old manifest entry remains unchanged
- publication validator accepts only the appended new identity

P2 — exact new matrix:
- normal.low returns exactly primary + CAB pre-execution requirements
- normal.medium/high remain exactly as before
- emergency rows remain exactly as before

P3 — determinism/digest:
- new policy digest is deterministic
- policyModelVersion remains 1
- identity metadata remains outside behavioral digest exactly as existing contract defines

P4 — registry/startup:
- old and new versions can both be registered
- activePolicy resolves the new version
- active policy + active selector bundle compatibility passes
- cab-authority resolves through the existing selector contract

P5 — no autonomy:
- no skipCab/autonomy/grant field or branch appears in policy/runtime

P6 — regressions:
- all existing F3.1.1a/b policy/selector/publication tests remain green
- Change Management regressions remain green

## Validation

Run at minimum:
- focused new policy matrix tests
- policy registry tests
- publication manifest/history tests
- validate:policy-publication against the real pre-change baseline ref without genesis
- selector active-pair startup tests
- existing F3.1.1a/b suites
- Change Management regression suites
- backend lint
- backend build
- repository-wide TypeScript baseline comparison

Do not fix unrelated debt. Do not alter CI architecture just to run this validator.

## Publication

After all gates pass:
1. commit only F3.1.1c-scoped changes;
2. normal fast-forward push to feat/ado-repo-governance;
3. no history rewrite/force push;
4. record parent SHA and final SHA;
5. verify remote ADO tip.

If implementation runs after F3.1.2a, parent SHA is expected to be that accepted/published slice tip, not 188d8e9.

## Documentation after real implementation

Create docs/backstage/f3-1-1c-implementation-evidence.md.

Update current-state, implementation-progress, and prompts/README factually.

Next gate is an independent F3.1.1c architecture/implementation acceptance review.

F3.1.2b remains NO-GO until BOTH F3.1.2a and F3.1.1c are independently accepted.

## Final report

Return:
Docs baseline reviewed: <sha>
ADO parent/tip before implementation: <sha>
Source drift: NONE | NON_BLOCKING | BLOCKED
F3.1.1c implementation: PASS | FAIL | BLOCKED_BY_SOURCE_DRIFT
Historical policy modified: NO
New policy key/version: <key@version>
normal.low primary + CAB: PASS | FAIL
medium/high unchanged: PASS | FAIL
emergency unchanged: PASS | FAIL
Publication manifest append-only: PASS | FAIL
Active policy pin updated: YES | NO
Active selector bundle changed: NO | explain
CAB autonomy implemented: NO
Migrations added: NO
Publication validation: <result>
Policy/selector regressions: <result>
Change Management regressions: <result>
Lint: <result>
Build: <result>
TypeScript baseline: <result>
Implementation commit: <sha | NOT_COMMITTED>
Remote ADO tip after publication: <sha | NOT_PUBLISHED>
Canonical evidence: docs/backstage/f3-1-1c-implementation-evidence.md
Next gate: F3.1.1c independent acceptance review

## STOP

Stop after F3.1.1c. Do not implement F3.1.2b or F3.2 and do not continue into another slice.