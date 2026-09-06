# MVP Demo Hardening — Execution Result

## Verdict

**CONDITIONAL PASS**

Three genuine, reproduced demo blockers were found and fixed; all fixes are independently verified
against real repository, cluster, and test-run state. The MVP's full DEV→HML→PRD→Change→Kargo→
Argo→Kubernetes flow was **not** re-driven live end-to-end in this checkpoint — the interactive
Backstage session required for it (real Entra auth, no service token available for scripted calls)
was not completed during this execution window. The `CONDITIONAL PASS` therefore rests on: (a) the
original MVP's own proven end-to-end run (`mvp-vertical-delivery-slice.md`), which used the same
sandbox, same component, and same artifact fingerprint, and (b) fresh, independent verification of
every fix made in this checkpoint against live cluster/Git/test state. No PRD dispatch, Kargo
Promotion, or ADO PR was created or approved by the agent in this checkpoint.

## Baselines

- Canonical docs SHA at execution start: `3a06b6430a1d4734a546b7bf471d68e789a2e410`
  (`backstage-docs` `origin/main`)
- Implementation branch: `feat/delivery-mvp-slice`
- Implementation SHA before this checkpoint: `99e146ae229425278b127e09b32f9dd28f0fac1c`
- Implementation SHA after this checkpoint: `50ed1b0b72fdf3ddc5152fb47b22f20114238332`
- GitOps repo/branch: `diegolab/platform-engineering/d0-gitops-sandbox` / `d1/desired-state`
  @ `8a50c2e5c07aa8e216b335b1612d2c23d2b8d091` (after PR #78 merge)
- Kargo version: v1.11.4 (unchanged, reused from D1/MVP)
- Argo CD version: v3.5.2 (unchanged, reused from D0/D1/MVP)
- Kubernetes version: Rancher Desktop (`rancher-desktop` context), namespaces `d1-dev`, `d1-hml`,
  `d1-prd`, `d1-sandbox`, `kargo`, `argocd` all `Active`

## Demo Blockers Found

All three were reproduced against real state, not inferred.

### 1. The demo SHA did not build (critical)

Committed, clean-tracked files referenced modules that did not exist anywhere in `HEAD`:

- `packages/app/src/modules/catalogEntityTabs/index.ts` registered a `platform` tab importing
  `./PlatformTab` — `PlatformTab.tsx` was untracked.
- `packages/backend/src/index.ts` imported `./plugins/idpAssessorPlugin` — untracked.
- `packages/backend/src/index.ts` imported `./modules/catalogValidation/catalogValidationModule` —
  untracked.

Proof: `git archive HEAD` extracted into a clean directory contained
`ChangelogTab/DeploymentsTab/EntityMarkdownTab/ReadmeTab/index.ts` under `catalogEntityTabs/` but no
`PlatformTab.tsx`; `packages/backend/src/plugins/` contained only `adoProjectAccessPlugin`,
`catalogEntityFilesPlugin`, `changeManagementPlugin`, `deliveryPlugin`; `catalogValidation` and
`idpAssessor` were absent from `packages/backend/src/modules/`. The demo only ever worked because
the presenter's own working tree happened to hold these files as uncommitted, unrelated
`feat/ado-repo-governance` work — a clean clone of the demo SHA could not build.

### 2. The sandbox could not replay a promotion (Kargo no-op)

All three Kargo Stages already held the target digest
(`sha256:05b8cb60c354a44ab824ea6e7dc69b46d50762cdbe728a347a5b656e6fb3d7c4`), so every promotion
produced no Git diff. Reproduced live on the cluster:

```
phase: Errored
message: failed to build step context: failed to get step config: cannot fetch id from <nil> (1:24)
 |  outputs['open-pr'].pr.id
 | .......................^
```

Root cause, confirmed via `stepExecutionMetadata`: when `git-open-pr` has nothing to open, Kargo
marks both `commit` and `open-pr` `Skipped` and never populates `outputs['open-pr']` at all. The
`git-wait-for-pr` step's `prNumber: ${{ outputs['open-pr'].pr.id }}` config expression then fails to
even evaluate, erroring the whole Promotion — and the downstream `argocd-update` step is skipped as
a result, so a true no-op never even re-triggers an Argo sync check.

### 3. Zero automated coverage on the Delivery gate

`packages/backend/src/modules/delivery/` had 0 test files versus 15 suites (104 tests) in
`changeManagement`. The single most important thing the demo claims — PRD fail-closed until
authorized — was protected by nothing but live-system observation.

### 4. (Found during live verification) Deployments tab could show a stale/wrong request

While attempting to drive the live flow, a `DeploymentsTab` field genuinely misled: the
**"Release candidate ID"** input has no filtering effect on which request is shown per stage — this
is by design (`docs/delivery/README.md` documents "current/desired release by target," not an
RC-scoped view), so that in itself was not a bug. Investigating it surfaced a real defect one layer
down: `DeliveryService.listDeployments` picked **the first matching request per target** via
`Array.find`, but a target can accumulate one request per distinct release candidate (`requestDeployment`
only short-circuits on an exact RC+target match). With more than one release candidate ever
requested against a stage, the tab could silently display an arbitrary — not necessarily current —
request. In this sandbox it surfaced as DEV/HML showing `failed` (accurate history from blocker #2's
pre-fix runs) with no way to tell there was a path to a fresh, correct request without understanding
the underlying data model.

## Fixes Applied

All fixes are the smallest change that resolves the reproduced defect; no opportunistic cleanup.

### Fix 1 — commit the referenced-but-missing files

Commit [`a30f156`](../../packages/backend) in `platform-devops-developer-portal`. Path-scoped to
exactly what already-committed code imports:

- `packages/app/src/modules/catalogEntityTabs/PlatformTab.tsx`
- `packages/backend/src/plugins/idpAssessorPlugin.ts`
- `packages/backend/src/modules/idpAssessor/**`
- `packages/backend/src/modules/catalogValidation/**`
- `packages/backend/src/modules/idpProvisioner/idpPlatformYaml.ts` and `pipelineConsumerYaml.ts`
  (transitive dependencies of `assessRepository.ts`)
- Removed one unused `componentRef` variable in `DeploymentsTab.tsx` (`tsc` TS6133)

The other ~30 uncommitted `feat/ado-repo-governance` files (templates, ADRs 0013–0016,
`idpProvisioner` additions, `platform-pipeline-templates/`) were deliberately left untouched — they
are unrelated in-progress work, not referenced by any committed import, and out of this checkpoint's
scope.

Verified: `git archive HEAD` re-extracted into a clean directory after the commit resolves every
import with no missing files.

### Fix 2 — Kargo no-op handling

See "Kargo No-op Handling" below for the full rationale. Two tiers applied, per the prompt's
preferred order:

1. **Deterministic baseline reset** (primary): `d0-gitops-sandbox` PR #78 (merged
   `8a50c2e5c07aa8e216b335b1612d2c23d2b8d091`) reseeds `stages/prd/deployment.yaml` to a
   deliberately older, real, pullable digest (`sha256:98f8ec75657d21b924fe4f69b6b9bff2f6550ea48838af479d8894a852000e40`)
   so the demo's PRD promotion always has genuine content to move — the same approach the original
   MVP run used (`d0-gitops-sandbox@6b87aa3`), reused rather than reinvented. Also committed the
   `kargo.akuity.io/authorized-stage` annotation on `argocd/d1-application-prd.yaml` for
   documentation parity with the live cluster (this file is not Git-reconciled; applied imperatively,
   matching D0's established pattern).
2. **Narrow template guard** (secondary): all three Kargo Stages (`dev`, `hml`, `prd`) in
   `d1-sandbox` had their `git-wait-for-pr` step patched with
   `if: "${{ outputs['open-pr'].pr != nil }}"`. Applied imperatively via `kubectl apply` (Kargo Stage
   CRs are not Git-managed in this sandbox — confirmed no `kind: Stage` manifest exists in
   `d0-gitops-sandbox`). **Validated live**: a manually triggered no-op Promotion on `dev`
   (`dev.01m1szesy6dbf54ewgeszqcswh.cba145d`, same freight already at DEV's target digest) went from
   the documented `Errored` failure to `Succeeded`, with `git-wait-for-pr` correctly `Skipped` and
   `argocd-update` still running and succeeding. The real-PR path was independently confirmed
   unaffected: the earlier `Succeeded` promotion's `outputs['open-pr'].pr` (`{id: 74, url: ...}`)
   satisfies the new guard's `!= nil` check.

No workflow engine or custom state machine was introduced, per the prompt's explicit prohibition.

### Fix 3 — DeliveryService regression suite

Commit [`856290b`](../../packages/backend) in `platform-devops-developer-portal`. Added
`FakeDeliveryRepository` (in-memory, mirrors the established `changeManagement`
fake-provider/testHelpers pattern) and a 10-test `DeliveryService.test.ts` covering all 7 required
invariants (see "Regression Tests Added" below). Also added `catalogEntityTabs/index.test.ts`, a
4-test guard asserting every tab's dynamic `import()` resolves — the exact class of defect Fix 1
corrected would have failed this test before ever reaching a browser.

### Fix 4 — request-selection ordering bug

Commit [`50ed1b0`](../../packages/backend) in `platform-devops-developer-portal`.
`DeliveryService.listDeployments` now takes the **last** (most recently created) request per target
instead of the first `Array.find` match. `KnexDeliveryRepository.listRequestsForComponent` gained an
explicit `ORDER BY delivery_requests.created_at ASC` (previously unordered, relying on incidental
row-return order). One regression test added: `listDeployments across multiple release candidates`
verifies that with two release candidates requested against the same target, the newer one's request
is the one returned. Test passes deterministically because ordering by query/insertion order (not by
comparing `createdAt` strings at read time, which can tie at millisecond resolution) breaks ties
correctly.

## Regression Tests Added

`packages/backend/src/modules/delivery/DeliveryService.test.ts` — 11 tests, all passing:

1. `produces CHANGE_REQUIRED and denies dispatch when target requires a Change and none is bound`
2. `produces NOT_AUTHORIZED and denies dispatch when the bound Change is not eligible, creating no provider side effect`
3. `permits dispatch when the bound Change is authorized (ALLOW)`
4. `never creates a provider Promotion while dispatch is denied, in both the unbound and bound-but-pending cases`
5. `short-circuits an already-dispatched request without calling the provider again`
6. `requesting the same release-candidate/target pair twice returns the same request, not a duplicate`
7. `mirrors a Succeeded provider projection to the succeeded terminal status`
8. `mirrors Errored/Failed provider projections to the failed terminal status without disturbing the runtime`
9. `does not mirror a non-terminal phase (still promoting) away from dispatched`
10. `are immediately dispatchable and dispatch without any eligibility check` (DEV/HML, no Change)
11. `shows the most recently requested request per target, not an arbitrary earlier one`

`packages/app/src/modules/catalogEntityTabs/index.test.ts` — 4 tests, all passing: each of
README/Changelog/Platform/Deployments tab loaders resolves to a component.

**Full regression run** (`yarn backstage-cli repo test --no-watch --testPathPatterns
"delivery|changeManagement|catalogEntityTabs|idpAssessor|catalogValidation"`):

```
Test Suites: 1 skipped, 20 passed, 21 total
Tests:       5 skipped, 134 passed, 139 total
```

The 1 skipped suite (`changeManagement/authorization/postgres.test.ts`, 5 tests) requires a live
Postgres instance and is a pre-existing, unrelated skip. All 15 `changeManagement` suites (104
tests) remain green, confirming no regression to the F2/F3.1.x Change Management path.

`yarn tsc --noEmit -p tsconfig.json`: 8 pre-existing errors remain (a `knex` package-hoisting type
conflict between `@backstage/backend-plugin-api`'s bundled `knex` and the root `knex`, affecting
already-shipped `changeManagementPlugin.ts` and `deliveryPlugin.ts` equally) — not introduced by this
checkpoint, not fixed (out-of-scope dependency surgery), and does not block `tsc`'s emit-free
type-checking use in CI since these plugins already ran correctly in the original MVP's live
verification.

## Kargo No-op Handling

Per the prompt's preferred order, both tiers were applied:

1. **First**: made the demo deterministic via reset/seed state. `d0-gitops-sandbox` PR #78 reseeds
   PRD's manifest to a digest different from DEV/HML's, guaranteeing the next PRD promotion has a
   real Git diff. This alone would have sufficed for a single demo run, matching the exact mechanism
   already proven in the original MVP checkpoint.
2. **Also applied**: a narrow, safe Kargo Stage template fix (`if` guard on `git-wait-for-pr`) so
   that *any* no-op promotion — including a DEV/HML re-run during a live demo, or a second run of the
   same demo before a fresh reseed — succeeds idempotently instead of erroring. This closes the
   D1-carried gap for good rather than only working around it for one run, without introducing any
   workflow engine, custom state machine, or change to Kargo's own promotion-controller code.

Both are documented precisely so a future engineer can distinguish "the reset that guarantees a real
promotion" from "the safety net for when a promotion happens to be a no-op anyway."

## Demo Baseline / Reset

**Identity:**

- Component: `component:default/idp-showcase-api`
- Backstage implementation SHA: `50ed1b0b72fdf3ddc5152fb47b22f20114238332`
  (`feat/delivery-mvp-slice`, `platform-devops-developer-portal`)
- Release candidate: `releaseCandidateId 72932ae5-caad-458d-9d37-d267f5ce9751`,
  `freightName cba145d3d41877ef906137f733399ace2573240d`
- Artifact repository: `docker.io/library/nginx`
- Immutable artifact fingerprint (target digest for DEV/HML/PRD after a successful PRD promotion):
  `sha256:05b8cb60c354a44ab824ea6e7dc69b46d50762cdbe728a347a5b656e6fb3d7c4`

**GitOps:**

- Repository: `diegolab/platform-engineering/d0-gitops-sandbox`
- Branch: `d1/desired-state`
- Baseline revision: `8a50c2e5c07aa8e216b335b1612d2c23d2b8d091` (after PR #78 merge)

**Kargo:**

- Project: `d1-sandbox`
- Warehouse: `nginx-warehouse` (subscribes to `nginx:1.27.x`, currently one qualifying Freight:
  `cba145d3d41877ef906137f733399ace2573240d`)
- Stages: `dev`, `hml`, `prd` (all three carry the `git-wait-for-pr` `if`-guard fix, applied
  imperatively — not Git-managed)

**Argo CD:**

- Applications: `d1-dev`, `d1-hml`, `d1-prd` (namespace `argocd`)
- Target namespaces: `d1-dev`, `d1-hml`, `d1-prd`

**Expected initial desired-state digest per stage (the reset target):**

| Stage | Expected starting digest | Rationale |
|---|---|---|
| DEV | `sha256:05b8cb60...` (target) | Already converged; DEV/HML promotions during the demo are expected no-ops, now handled cleanly by the Kargo fix |
| HML | `sha256:05b8cb60...` (target) | Same as DEV |
| PRD | `sha256:98f8ec75...` (deliberately different/older) | Must differ from DEV/HML so the PRD promotion has a real Git diff to commit/PR — this is the one promotion the demo needs to show live |

**Required sandbox services/controllers:** Rancher Desktop Kubernetes cluster with Kargo v1.11.4 and
Argo CD v3.5.2 installed (namespaces `kargo`, `kargo-cluster-secrets`, `kargo-shared-resources`,
`kargo-system-resources`, `argocd`, `argo-rollouts` all `Active`); Backstage backend/frontend running
locally (`yarn start`) with a valid Microsoft Entra app registration for auth (`AZURE_CLIENT_ID`,
`AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID` — no secret values recorded here); an ADO PAT configured for
Kargo's Git write access to `d0-gitops-sandbox` (already provisioned per D1; not rotated in this
checkpoint).

**Reset procedure** (prefers declarative Git operations; two documented imperative steps are
preserved as-is per D0's established boundary and this prompt's explicit permission not to expand
scope by replacing them):

1. Confirm `d1/desired-state`'s `stages/prd/deployment.yaml` is on a digest different from
   `stages/dev`/`stages/hml`. If a prior demo already converged all three stages to the same digest
   (as this checkpoint found), open a small PR reseeding PRD to a different real, pullable digest
   (reuse `sha256:98f8ec75657d21b924fe4f69b6b9bff2f6550ea48838af479d8894a852000e40`, already validated
   pullable) and get it merged by a second approving identity — branch policy on `d1/desired-state`
   does not count the requestor's own vote.
2. Confirm the two documented imperative sandbox-only steps are still in place (both were present and
   verified during this checkpoint, no action needed unless the cluster is rebuilt from scratch):
   - `d1-prd` Argo Application carries the `kargo.akuity.io/authorized-stage: "d1-sandbox:prd"`
     annotation (now also recorded in Git for documentation parity, per Fix 2).
   - `d1-prd` namespace exists (created manually in the original MVP run — `CreateNamespace=true`
     needs `Namespace` in the AppProject's `clusterResourceWhitelist`, intentionally left empty).
3. Confirm the Kargo Stage `git-wait-for-pr` `if`-guard fix (Fix 2, tier 2) is present on all three
   Stages — it is applied imperatively (`kubectl apply`) and would need reapplying if the Kargo
   Stage CRs are ever recreated from a stale manifest or backup.
4. In Backstage's Deployments tab for `idp-showcase-api`, paste the release candidate ID and request
   DEV, then HML (both no-op-safe now), then PRD (the one real promotion).

## Demo Runbook

A 5–10 minute live sequence, following the contract's required 15 steps. Steps marked **[HUMAN]**
require a person, not the presenter alone — call these out explicitly before starting.

1. **Component in Backstage** — open `idp-showcase-api`, show Overview.
2. **Current DEV/HML/PRD state** — open the **Deployments** tab; show DEV/HML `succeeded` at the
   target digest and PRD not-yet-promoted (reseeded, older digest).
3. **ReleaseCandidate / artifact identity** — point out the release candidate ID field and the
   Freight name; explain it's the same immutable digest already running in DEV/HML.
4. **DEV promotion** — click Request promotion (if not already requested) → Dispatch. Expect
   `succeeded` quickly (no-op, now handled cleanly).
5. **HML promotion** — same as DEV.
6. **PRD request blocked by CHANGE_REQUIRED** — request PRD promotion; show the `change_required`
   status chip and the Change-ID/Create-GMUD UI that appears.
7. **GMUD creation or association [HUMAN]** — click "Create GMUD", fill in the form
   (`targetRef: component:default/idp-showcase-api`), submit. Paste the resulting Change ID back
   into the Deployments tab and click "Bind change".
8. **DENY while authorization is pending** — click "Retry after authorization"; expect a
   `409 NOT_AUTHORIZED` toast and the status remaining `awaiting_authorization`. Optionally show
   `kubectl get promotion -n d1-sandbox` has no new PRD Promotion CR as proof.
9. **Approval / authorization decision [HUMAN]** — in the GMUD UI, record the authorization decision
   (approve). This is a real business decision, never automated.
10. **Fresh ALLOW result** — back in Deployments, the eligibility re-evaluates on next dispatch
    attempt; no separate action needed.
11. **PRD dispatch** — click "Retry after authorization" again; expect it to succeed this time,
    creating a real Kargo Promotion.
12. **Kargo/Git/PR path** — show the Kargo Promotion in the Kargo UI or via
    `kubectl get promotion -n d1-sandbox`; show the real ADO PR that opens on `d0-gitops-sandbox`
    against `d1/desired-state`.
13. **PR merge [HUMAN]** — a second approving identity reviews and merges the PR (branch policy
    requires this; the requestor's own vote does not count).
14. **Argo reconciliation** — show `d1-prd` Argo Application transitioning to `Synced`/`Healthy` at
    the new revision (`kubectl get application d1-prd -n argocd` or the Argo UI).
15. **Kubernetes runtime state** — `kubectl get pods -n d1-prd -o jsonpath='{...imageID}'` shows the
    same digest now running in PRD as in DEV/HML — the same-artifact-fingerprint proof point.
16. **Backstage Deployments tab result** — refresh (or wait for the 5s poll) to show PRD as
    `succeeded` with the bound Change and Kargo/Argo projection.

**Fallback / recovery for likely failures:**

- **A promotion shows `Errored` with the `outputs['open-pr'].pr.id` message** — the Kargo Stage
  `if`-guard fix (Fix 2) is missing or was reverted; reapply it with `kubectl apply` against the
  Stage manifests captured in this checkpoint, or re-derive it: add
  `if: "${{ outputs['open-pr'].pr != nil }}"` to the `git-wait-for-pr` step.
- **PRD's request/dispatch cycle shows a stale or unexpected prior state** — this checkpoint fixed
  `DeliveryService.listDeployments` to show the latest request per target; if seeing an old
  `failed`/`succeeded` from a prior demo run, just request a fresh promotion for that stage — the
  new one will now correctly become "current."
- **The PR is not merged in time** — the demo can pause here; the Promotion CR remains `Running`
  waiting on `git-wait-for-pr`, Argo has not yet reconciled, nothing is broken. Resume after merge.
- **The Deployments tab looks stale** — it polls every 5s while any request is non-terminal; a
  refresh forces an immediate `/refresh` call.
- **Everything already converged (all three stages on the same digest) before the demo** — re-run
  step 1 of the reset procedure (reseed PRD to a different digest) before presenting.

## End-to-End Verification

**Not freshly re-executed live in this checkpoint** — the interactive session needed to drive PRD
through Change binding, a real authorization decision, dispatch, a real ADO PR, and a human merge
was not completed during this execution window (see Deviations below). The chain is verified via two
combined sources:

1. **The original MVP checkpoint's own live, independently-verified run**
   (`docs/delivery/mvp-vertical-delivery-slice.md`), using this same sandbox, same component, same
   Freight, executed end-to-end: ReleaseCandidate → DEV (dispatched) → HML (dispatched) → PRD
   `CHANGE_REQUIRED` → GMUD `CHG-2026-000001` bound → DENY (`409 NOT_AUTHORIZED`, verified no PRD
   Promotion CR existed) → real ledger authorization decision → ALLOW → PRD dispatch → real Kargo
   Promotion → real ADO PR #77 → human merge → Argo `d1-prd` `Synced`/`Healthy` → Kubernetes pod
   verified via `imageID` running `sha256:05b8cb60...` — the same digest independently confirmed
   running in `d1-dev` and `d1-hml` at the same time.
2. **Fresh verification performed in this checkpoint**, all against live state:
   - Clean-checkout build fix (`git archive HEAD` re-extraction, no missing imports).
   - Kargo no-op fix validated live: a real no-op Promotion created and observed transitioning from
     the exact `Errored` failure mode to `Succeeded`, with `git-wait-for-pr` `Skipped` and
     `argocd-update` still running.
   - PRD baseline reseed validated live: PR #78 merged, `d1-prd` pod's `imageID` confirmed running
     `sha256:98f8ec75...` (the reseeded digest, not the target), Argo `Synced`/`Healthy` at the new
     revision.
   - 134 passing tests including 11 new `DeliveryService` tests directly exercising the PRD gate
     invariants (CHANGE_REQUIRED, NOT_AUTHORIZED, ALLOW, no-side-effect-while-denied, idempotency,
     projection mirroring) against the exact service code now running.

The artifact fingerprint (`sha256:05b8cb60...`) traceability across DEV/HML/PRD is the one claim
resting entirely on source (1) rather than a fresh PRD promotion in this checkpoint — PRD currently
runs the reseeded `sha256:98f8ec75...` by design, awaiting the next real promotion to move it back to
the shared target digest.

## Remaining Gaps

**Demo risk (should be resolved or explicitly re-verified before presenting live):**

- A fresh, live, end-to-end PRD promotion through the fixed code has not been re-run in this
  checkpoint. The DeliveryService fix (request ordering) and the Kargo template fix have each been
  independently validated, but not yet in combination through a single live PRD dispatch. Recommend
  running the full sequence once, unhurried, before the actual stakeholder demo.
- The sandbox's Backstage backend session (`backstage-cli repo start`, PID 40387 as of this
  checkpoint) has hot-reloaded the `DeliveryService`/`KnexDeliveryRepository` changes per its normal
  dev-mode watch behavior; this was not independently confirmed by observing a live API response
  post-change (no live dispatch occurred to exercise it). A full server restart before the demo is a
  safe, cheap precaution.

**Production hardening (explicitly out of this checkpoint's scope, unchanged from the MVP report):**

```
production-grade Argo controller RBAC
production Git credential separation
multi-cluster topology
full provenance/signature/admission chain
production prune/deletion policy
HA/DR, break-glass automation, automatic production rollback
advanced retention
organization-wide branch migration
public ingress/callback architecture
Teams approval integration polishing, full CAB UX
multi-provider Delivery support
```

- The Kargo Stage `if`-guard fix is imperative-only (not Git-reconciled), matching D0's established
  boundary for Stage/Application/AppProject CRs — a cluster rebuild from a stale backup would need it
  reapplied. Documented above in the reset procedure, not otherwise addressed.
- `delivery`'s Kubernetes API access still uses the ambient kubeconfig's cluster-admin credential —
  unchanged, previously flagged as a production-hardening gap in the MVP report.
- 8 pre-existing `tsc` errors from a `knex` package-hoisting conflict remain (see Regression Tests
  Added) — not introduced or fixed in this checkpoint.

## Documentation Updated

- `docs/delivery/mvp-demo-hardening.md` — this document (new).
- `docs/delivery/README.md` — corrected the stale "Architecture direction only — implementation NOT
  authorized" status header and the "NO-GO: Delivery backend/frontend implementation..." Gate line,
  which had not been updated since before the MVP vertical slice checkpoint executed and passed.
- `docs/golden-paths/current-state.md` — noted that the assessor/Platform-tab dependency set
  referenced by already-committed code is now itself committed (previously "remains WIP until their
  full dependency set is reviewed and merged").
- Preserved unchanged, as historical evidence: `mvp-vertical-delivery-slice.md`,
  `d1-kargo-fit-evaluation.md`, `d0-architecture-review.md`, `d0-gitops-reconciliation-evidence.md`,
  `architecture-spike.md`.
- `ADR-012-delivery-management-gitops-promotion.md` status: **unchanged**, still `Proposed`.

## Deviations / Blockers

No hard STOP condition was triggered. One process deviation from the original plan: the live
end-to-end re-run was to pause for the user's authorization decision and PR merge (both correctly
requiring separate human action, matching the harness's prior-established discipline of never
self-approving business authorization or self-merging governance-gated PRs). The interactive
Backstage session needed to drive that flow was not completed in this execution window; rather than
block indefinitely or fabricate the result, the checkpoint was completed using the original MVP's
already-proven evidence for the parts not re-driven, with every fix made in this checkpoint
independently verified on its own. This is recorded honestly above rather than presented as a fresh
full re-run.

Two repository-visible actions were taken with the user's explicit approval before proceeding:
merging `d0-gitops-sandbox` PR #78 (the user reviewed and merged it themselves, a second identity
per branch policy) and opening that PR in the first place (confirmed as in-scope demo-hardening
reset work, not a production change).

## Gate

STOP.
