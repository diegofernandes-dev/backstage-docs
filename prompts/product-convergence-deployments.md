# Product Convergence — Restore Accepted Deployments UX onto Active GMUD Branch

## Status

IMPLEMENTATION CHECKPOINT PROMPT — DO NOT EXECUTE WITHOUT EXPLICIT USER LAUNCH.

Purpose: restore the already-accepted Catalog Component **Deployments** tab and only the Delivery runtime surface it genuinely requires onto the current active Backstage product line, while preserving the accepted GMUD/F3 baseline.

This is a branch-convergence checkpoint, not a new Delivery architecture milestone.

## Canonical context

Documentation authority:
- docs/backstage/current-state.md
- docs/delivery/README.md
- docs/delivery/deployments-ux-v2-implementation.md
- docs/delivery/deployments-ux-v2-responsive-polish.md
- docs/adr/ADR-012-delivery-management-gitops-promotion.md
- docs/backstage/f3-1-2-implementation-plan.md
- docs/backstage/f3-1-1c-architecture-implementation-acceptance.md

Known documented baselines at prompt authoring:
- active GMUD branch: platform-devops-developer-portal / feat/ado-repo-governance
- accepted GMUD/F3 tip: 3b302ab5c9caab38f96491b389b7ea9fe0b66c2f
- accepted Deployments UX implementation: feat/delivery-mvp-slice @ 0163a49
- accepted responsive polish: feat/delivery-mvp-slice @ b08e7b2

Do not assume those are still remote tips. Verify both branches live before changing anything.

## Problem statement

The accepted Deployments UX exists on the Delivery workstream, while the current GMUD/F3 product line is feat/ado-repo-governance. The user reports that the Deployments tab is absent from the currently running UI.

The goal is NOT to merge the entire Delivery branch.

The goal is:

```text
feat/ado-repo-governance
  + accepted Deployments product surface
  + only its required backend/frontend dependencies
  - unrelated sandbox / production-hardening / credential / GitOps rollout changes
```

Resulting product must expose the accepted component-scoped route:

```text
Catalog -> Component -> Deployments
```

Do not invent a new global sidebar/workbench in this checkpoint.

## 1. Mandatory fresh source verification

Before editing:

1. Fetch latest backstage-docs main and record exact SHA.
2. Read all canonical context files above and this prompt.
3. Fetch the actual ADO repository platform-devops-developer-portal.
4. Verify live tips for:
   - feat/ado-repo-governance
   - feat/delivery-mvp-slice.
5. Record the exact merge-base between the two branches.
6. Inspect the complete commit/path history for the accepted Deployments UX, including at least 0163a49 and b08e7b2.
7. Inspect the current target branch's catalog entity tab registration, app extensions, backend module wiring, change-management exports, and app-config.
8. Prove whether the user-observed missing Deployments tab is caused by branch divergence, a later regression, feature visibility logic, or another concrete source fact.

If the root cause is not branch divergence, do not force this plan. Report the actual cause and STOP with BLOCKED_BY_DIFFERENT_ROOT_CAUSE.

## 2. Source-selection rule

Do NOT:
- merge feat/delivery-mvp-slice wholesale;
- cherry-pick a long range blindly;
- bring production-rollout commits merely because they are descendants;
- import credential material, kubeconfigs, PAT/SP secrets, sandbox manifests, cluster RBAC, token refresh jobs, alerting, rollout runbooks, or production-target assumptions;
- reset or rebase away accepted GMUD history.

Build an explicit **convergence manifest** before editing:

```text
source path/commit
why required by Deployments UX
target action: PORT | ALREADY_PRESENT | REIMPLEMENT_MINIMALLY | EXCLUDE
risk/collision notes
```

Only PORT/REIMPLEMENT items needed for the accepted Deployments product surface may enter the target branch.

If the minimal required surface cannot be separated safely from unrelated Delivery/infra work, STOP with BLOCKED_BY_CONVERGENCE_RISK and document the exact coupling.

## 3. Target branch authority

feat/ado-repo-governance remains the product line for this checkpoint.

Preserve every accepted GMUD/F3 commit and behavior through the current live tip, including:
- F3.1.2a canonical Change baseline;
- F3.1.1c CAB-safe policy baseline;
- existing GMUD create/list/detail UI;
- Catalog integration;
- current authorization policy/selector artifacts;
- current app-config authorization pin.

No GMUD architecture decision may be weakened to make Delivery compile.

## 4. Accepted Deployments surface to restore

Port the smallest current implementation necessary to preserve the accepted v2 UX and responsive polish.

Expected functional surface includes, when source inspection proves still required:

### Frontend
- Catalog Component Deployments tab registration/loader;
- DeploymentsTab thin composition root;
- deployments/components/* required by the accepted screen;
- deployments/model/* including semantic status mapping;
- deployments/api/* client/API extension;
- deployments/hooks/* required for release candidates, RC-scoped deployments, history, events, eligibility, refresh/polling;
- accepted responsive layout/polish from b08e7b2.

### Backend Delivery reads/actions required by that tab
- Delivery backend plugin/module already established by ADR-012 workstream;
- release-candidate listing;
- RC-scoped deployment reads;
- promotion history;
- recent events projection/synthesis;
- eligibility read;
- only the existing promotion/binding actions actually invoked by the accepted UI.

### Cross-plugin contract
- additive change-management exports required by Deployments to type/read Change context, if still absent on the target branch.

Do not automatically port unrelated Delivery operational hardening just because it shares files. Reconcile at symbol/behavior level.

## 5. Critical collision guard — Backstage API extensions

The historical Deployments UX implementation found a real app-wide collision: registering the Delivery API extension as an unnamed extension in the Catalog module collided with catalogApiRef and broke GMUD/Catalog.

The converged implementation MUST preserve the corrected distinct extension identity equivalent to:

```text
api:catalog/delivery
```

using the explicit Delivery extension name required by the current Backstage APIs.

Mandatory negative proof:
- catalogApiRef remains registered and usable;
- GMUD pages still load;
- Catalog entity/list providers still load;
- Deployments API extension is separately registered;
- no duplicate extension-id warning/collision exists.

Do not copy an older pre-fix form from the Delivery branch.

## 6. UI acceptance contract

At minimum preserve the accepted Deployments UX behavior documented in deployments-ux-v2-implementation.md and responsive polish:

- component context tab is visible for the intended entity kind/type;
- release candidate selector, with latest/current behavior;
- DEV -> HML -> PRD environment composition and semantic statuses;
- GMUD/change governance context;
- eligibility context when a binding exists;
- promotion history;
- recent events;
- truthful unavailable values remain 'não disponível' rather than fabricated;
- accepted activity non-completion message remains truthful;
- refresh behavior works;
- laptop-responsive layout matches the accepted polish;
- no global Delivery workbench is invented.

If current domain contracts have legitimately evolved, adapt only enough to preserve semantic behavior; document every adaptation.

## 7. Delivery/GMUD architecture boundaries

Preserve ADR-012:
- Change Management authorizes;
- Delivery promotes;
- Git is desired-state authority;
- Argo reconciles;
- Kubernetes executes;
- Backstage composes the UX.

Do not add Kargo/Argo/ADO identifiers to canonical Change.

Do not make Delivery an authorization authority.

Do not make CAB approval a Kargo approval.

Do not implement F3.1.2b in this checkpoint.

Do not alter the new normal-low CAB-safe policy.

## 8. Forbidden scope

MUST NOT:
- implement F3.1.2b;
- implement F3.1.3/F3.1.4;
- implement F3.2 CAB autonomy;
- perform production rollout;
- select a production workload/cluster/namespace;
- import sandbox credentials or secrets;
- modify external GitOps desired-state repos;
- apply Kubernetes/Kargo/Argo resources;
- reopen ADR-012;
- redesign Deployments UX;
- redesign GMUD UX;
- create a generic Delivery workbench;
- add a new workflow engine;
- perform unrelated dependency upgrades.

## 9. Implementation strategy

Use an isolated worktree from the verified live feat/ado-repo-governance tip.

Preferred strategy:
1. build the convergence manifest;
2. port/reconcile required files or symbols deliberately;
3. resolve conflicts in favor of current accepted GMUD/F3 contracts;
4. keep one reviewable convergence commit (or a very small explicitly justified sequence);
5. no force push/history rewrite.

Do not preserve source-branch commit topology at the cost of importing unrelated work.

## 10. Mandatory tests

### Product navigation / extension safety
- existing catalogEntityTabs loader/smoke tests;
- explicit proof that the Component Deployments tab is registered/reachable;
- Catalog API extension still works;
- GMUD frontend smoke/render tests;
- no duplicate extension-id collision.

### Deployments frontend
- DeploymentsTab tests;
- semanticStatus tests;
- RC-scoped stale-selection/race test;
- release selector;
- governance/eligibility states;
- history/events rendering;
- responsive composition tests where existing test tooling supports them.

### Delivery backend
- full existing DeliveryService tests relevant to list RC / RC-scoped deployments / history / events / eligibility / binding/promotion actions;
- no regression of activity-scoped ChangeBinding;
- no regression of same-target concurrency if those codepaths are ported.

### Change Management regression
- full Change Management module;
- F3.1.2a canonical snapshot tests;
- F3.1.1 policy/selector/publication tests;
- verify POST /changes remains LEGACY_PRE_F3 and creates no Round.

### Repository quality
- app/frontend tests;
- backend tests;
- lint;
- build;
- repository-wide TypeScript baseline comparison.

No --forceExit. Do not hide failures.

## 11. Browser/product proof

Because the triggering defect is visible UI absence, source/unit proof alone is insufficient if a runnable local environment is available.

At minimum, against the converged target checkout:
1. start the Backstage app/backend using the normal dev path;
2. authenticate through the user's normal interactive flow if required; do not automate MFA or capture credentials;
3. open a representative Catalog Component;
4. confirm the Deployments tab is visible;
5. navigate into it;
6. confirm GMUD and Catalog tabs/pages remain reachable after navigation;
7. inspect browser console/network for extension-registration or API-ref errors.

If live Delivery/Kargo/Argo data is unavailable, do not fabricate it and do not apply infrastructure. The tab/navigation restoration may be proven with truthful empty/error/fixture state plus automated backend tests.

Capture the exact limitation in evidence.

## 12. Success criteria

PASS only if all are true:

- root cause is proven;
- target branch keeps all accepted GMUD/F3 history;
- Deployments tab is restored in Catalog Component context;
- accepted Deployments UX/runtime surface is present;
- Catalog still works;
- GMUD create/list/detail still works;
- Delivery API extension collision fix is preserved;
- no unrelated production/sandbox hardening is imported;
- no secrets/config credentials are imported;
- tests/build/lint/TS baseline are green or baseline-identical;
- F3.1.2b remains untouched;
- diff is explainable by the convergence manifest.

Otherwise return FAIL or BLOCKED with the smallest concrete reason.

## 13. Publication

If PASS:
1. commit only the convergence changes;
2. normal fast-forward push to feat/ado-repo-governance;
3. verify remote tip independently;
4. do not delete feat/delivery-mvp-slice in this checkpoint;
5. do not merge back into the Delivery branch.

## 14. Canonical evidence

After a real implementation attempt, create:
- docs/backstage/product-convergence-deployments-evidence.md

Update factually:
- docs/backstage/current-state.md
- docs/backstage/implementation-progress.md
- docs/delivery/README.md
- prompts/README.md

Record:
- docs baseline;
- both ADO source branch tips;
- merge-base;
- convergence manifest;
- exact target parent and final SHA;
- changed paths;
- test results;
- browser/product proof;
- any intentionally excluded Delivery branch commits/surfaces and why.

Do not rewrite historical Deployments evidence.

## 15. Next gate

If convergence PASS:

```text
Product convergence (GMUD + Deployments): PASS
Active product branch: feat/ado-repo-governance
F3.1.2a: CLOSED / ACCEPTED
F3.1.1c: CLOSED / ACCEPTED
F3.1.2b implementation-prompt authoring: GO
F3.1.2b implementation: still requires separate explicit authorization
F3.2: NO-GO
```

Then STOP.

Do not author or execute F3.1.2b from inside this convergence checkpoint.

## 16. Required final report

Return:

```text
Docs baseline reviewed: <sha>
Target ADO branch/tip before: feat/ado-repo-governance @ <sha>
Delivery source branch/tip: feat/delivery-mvp-slice @ <sha>
Merge-base: <sha>
Root cause: BRANCH_DIVERGENCE | DIFFERENT_ROOT_CAUSE | <precise>
Convergence manifest: <path/summary>
Product convergence: PASS | FAIL | BLOCKED_BY_CONVERGENCE_RISK | BLOCKED_BY_DIFFERENT_ROOT_CAUSE
Deployments tab restored: YES | NO
Catalog regression: PASS | FAIL
GMUD regression: PASS | FAIL
Delivery API extension collision guard: PASS | FAIL
Deployments frontend tests: <result>
Delivery backend tests: <result>
Change Management tests: <result>
Lint: <result>
Build: <result>
TypeScript baseline: <result>
Browser/product proof: PASS | PARTIAL | NOT_AVAILABLE
Production/sandbox infra imported: NO
Secrets imported: NO
F3.1.2b modified: NO
Target implementation commit: <sha | NOT_COMMITTED>
Remote target tip after publication: <sha | NOT_PUBLISHED>
Canonical evidence: docs/backstage/product-convergence-deployments-evidence.md
Next gate: F3.1.2b prompt authoring
```

## 17. STOP

STOP after convergence. Do not continue into F3.1.2b or any production rollout/hardening activity.