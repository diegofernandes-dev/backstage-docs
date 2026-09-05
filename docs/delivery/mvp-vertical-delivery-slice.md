# MVP Vertical Delivery Slice — Result

## Verdict

**CONDITIONAL PASS**

The end-to-end flow works and was independently verified against raw cluster/Git/ledger state, not
just the tooling's own status reporting. The conditional qualifier reflects carried gaps explicitly
authorized by the execution prompt (production RBAC/credential separation, no dedicated test
coverage for the new `delivery` module, and a Kargo no-op-PR template limitation hit twice during
the run) — none of which the prompt required to be closed for this checkpoint.

## Baselines

- Canonical docs SHA: `f1685d2e0019ec57715cc2837201a2cfd71d7629` (backstage-docs `origin/main`)
- Backstage repo/branch/SHA: `platform-devops-developer-portal` / `feat/delivery-mvp-slice`
  (branched from `feat/ado-repo-governance`@`4bad41d`)
- GitOps repo/branch/SHA: `diegolab/platform-engineering/d0-gitops-sandbox` /
  `d1/desired-state`@`1a463a2d662569afe469fdea5b0900a5b1b6772f`
- Kargo version: v1.11.4 (reused D1 install)
- Argo CD version: v3.5.2 (reused D0/D1 install)
- Kubernetes version: v1.33.6+k3s1 (Rancher Desktop, reused D0/D1 sandbox cluster)

## Demo Flow Achieved

The full contract flow was exercised live against the real sandbox — not simulated:

```
ReleaseCandidate registered
  → DEV requested (dispatchable, no Change) → dispatched
  → HML requested (dispatchable, no Change) → dispatched
  → PRD requested → CHANGE_REQUIRED
  → GMUD created (CHG-2026-000001) and bound to the PRD request
  → ExecutionEligibility: DENY / PENDING_AUTHORIZATION (fresh, real ledger read)
  → PRD dispatch attempted while pending → 409 NOT_AUTHORIZED, no Promotion CR created
  → Authorization decision recorded on the immutable ledger (approved)
  → ExecutionEligibility: ALLOW / AUTHORIZED (fresh re-evaluation)
  → PRD dispatch succeeded → real Kargo Promotion created
  → Kargo: Git clone → yaml-update → commit → push → real ADO PR → human merge → argocd-update
  → Argo CD reconciled d1-prd: Synced / Healthy
  → Kubernetes running the same immutable artifact in d1-prd
  → Backstage Deployments tab reflects DEV/HML/PRD status, binding, and provider projection
```

## ReleaseCandidate

- `releaseCandidateId`: `72932ae5-caad-458d-9d37-d267f5ce9751`
- `componentRef`: `component:default/idp-showcase-api`
- `artifactRepoUrl`: `docker.io/library/nginx`
- `artifactFingerprint` (digest): `sha256:05b8cb60c354a44ab824ea6e7dc69b46d50762cdbe728a347a5b656e6fb3d7c4`
  (the same D1-proven Freight, reused rather than rebuilt)
- `freightName`: `cba145d3d41877ef906137f733399ace2573240d`

## DEV

- `DeploymentRequest` `b8bf50cd-0d2f-4e78-9bae-2fcdba4e8852`: `dispatchable` (no Change required) →
  `dispatched`. Kargo Promotion `dev.01m1sq4xnfh4qas8y7wms6z7jq.cba145d` hit the documented
  no-op-PR gap (see Simplifications) and errored; **correctly mirrored to `failed`** by the
  Delivery projection — DEV runtime was never disturbed.
- Independently verified live: `d1-dev` Argo app `Synced`/`Healthy`; running pod digest
  `sha256:05b8cb60...` (unchanged, matches Freight).

## HML

- `DeploymentRequest` `6e60b421-3ffb-4e3c-b9b8-77e558245c28`: same shape and same no-op-PR outcome
  as DEV, for the identical reason (both stages' Git content already matched the target digest from
  prior D1 evidence).
- Independently verified live: `d1-hml` Argo app `Synced`/`Healthy`; running pod digest
  `sha256:05b8cb60...`.

## PRD / Change Gate

- `DeploymentRequest` `25a5b052-8d6e-47db-8adf-73790b36fa2c` against target `prd`
  (`requiresChange: true`).
- Request with no Change bound → `status: change_required`.
- Dispatch attempted with no binding → `409 CHANGE_REQUIRED`; **verified no PRD Promotion CR
  existed** (`kubectl get promotion -n d1-sandbox`).
- GMUD `CHG-2026-000001` created (`targetRef: component:default/idp-showcase-api`,
  `authorizationMode: LEDGER_REQUIRED`) and bound to the request.
- Dispatch attempted with a bound-but-unauthorized Change → `409 NOT_AUTHORIZED
  (PENDING_AUTHORIZATION)`; **verified no PRD Promotion CR existed** — the gate holds even with a
  Change bound.
- Real authorization decision recorded on the ledger (`requirementId
  45ffec4b-d2ee-4f0c-9d09-6bcf82819106`, `outcome: approved`, `decisionId
  de297bf8-2f9e-489c-a37a-3fe82b6fd129`, actor `user:default/diego.fernandes_outlook.com`).
- Fresh eligibility re-evaluation → `ALLOW / AUTHORIZED`.
- Dispatch succeeded → Kargo Promotion `prd.01m1srpc60ra1zmbvda7by9c1d.cba145d` created.

## ExecutionEligibility

New `EligibilityService` in `changeManagement/authorization/`, deliberately never imported by
`ChangeManagementService` (preserves the existing `architecture.test.ts` guard against F3.1.0
ledger integration into the F2 submission path). Reuses the existing pure
`evaluateAuthorization(requirements)` primitive. Two new routes:
`GET /changes/:changeId/execution-eligibility` (fail-closed; `allow: ['user','service']` so Delivery
can call it as a service) and `POST /changes/:changeId/authorization/decisions`
(idempotency-key required, appends to the immutable ledger).

The one-time gap that made this possible: `ChangeManagementService` hardcoded
`authorizationMode = 'LEGACY_PRE_F3'` on every submission, and the ledger itself was never
instantiated outside tests (round creation was deferred to F3.1.2). Closing the smallest necessary
piece — a config-driven default (`changeManagement.authorizationMode: LEDGER_REQUIRED`, applied
to both the direct and idempotency-reservation code paths) plus a round-on-first-eligibility-check
builder reading a static, hash-verified policy file (`packages/backend/config/authorization/
sandbox-policy-v1.json`, one `authority`/`pre_execution` requirement) — was sufficient. No rules
engine, no DSL, no `ChangeStatus` enum change.

Round creation is one-shot and lazy: `EligibilityService.ensureRound` checks
`ledger.findCurrentRound` first and only builds round 1 the first time a `LEDGER_REQUIRED` change
needs eligibility evaluated. `architecture.test.ts`'s 6 guards were re-run after every related edit
and remained green throughout.

Both DENY-before and ALLOW-after were captured with the negative proof that matters most: **no
Promotion CR existed while dispatch was denied**, in both the unbound and bound-but-pending cases.

## Kargo / Git / Argo Evidence

- PRD Stage sources verified Freight from `hml` (`requestedFreight[].sources.stages: [hml]`),
  closing the D1-documented "not fully proven" stage-to-stage chaining gap rather than repeating
  the `direct: true` shortcut used for dev/hml.
- Real ADO pull requests: PR #75 (PRD infra bootstrap), PR #76 (PRD baseline digest seed, needed
  because the manifest was first bootstrapped with the already-promoted digest, producing a no-op
  Git diff), PR #77 (the actual Kargo-opened PRD promotion PR) — all merged, all requiring a
  genuinely separate approving identity (branch policy 103 on `d1/desired-state` does not count the
  requestor's own vote; confirmed via `az repos policy list` before asking for a real second
  approval rather than relaxing the policy).
- Git desired-state revision after PR #77 merge: `1a463a2d662569afe469fdea5b0900a5b1b6772f`
  (`stages/prd/deployment.yaml` updated to `sha256:05b8cb60...`).
- Argo CD reconciled `d1-prd` to that revision: `Synced` / `Healthy`.
- Kubernetes pod `d1-app-*` in namespace `d1-prd` running digest `sha256:05b8cb60...` — verified via
  `kubectl get pods -o jsonpath='{...imageID}'`, not merely inferred from Argo's own health check.

## Backstage UX

One new Component-level `Deployments` tab (`EntityContentBlueprint`, filter `kind:component`),
mirroring the existing `Platform`/`README`/`Changelog` tabs' `discoveryApi`+`fetchApi` pattern.
Shows per-stage (DEV/HML/PRD) status, bound Change, and provider projection (Kargo phase, Argo
sync/health); actions to request promotion, bind an existing Change ID, deep-link to `Create GMUD`,
and dispatch/retry. Polls `/refresh` every 5s only while a request is non-terminal. The dispatch
button's disabled-unless-ALLOW state is a UX affordance only — the server re-evaluates eligibility
unconditionally on every dispatch call regardless of what the client believes.

New backend plugin `delivery` (pluginId `delivery`), mirroring `change-management`'s
`createBackendPlugin` + express router + `knex.migrate.latest` structure exactly. Routes:
`POST /release-candidates`, `GET /components/:ns/:kind/:name/deployments`,
`POST /deployment-requests`, `GET /deployment-requests/:id`,
`POST /deployment-requests/:id/change-binding`, `GET /deployment-requests/:id/eligibility`,
`POST /deployment-requests/:id/dispatch`, `POST /deployment-requests/:id/refresh`.

## End-to-End Artifact Correlation

The same Freight (`cba145d3d41877ef906137f733399ace2573240d`, digest
`sha256:05b8cb60c354a44ab824ea6e7dc69b46d50762cdbe728a347a5b656e6fb3d7c4`) was independently
confirmed running, at the same time, in all three namespaces:

| Target | Argo sync/health | Runtime `imageID` |
|---|---|---|
| `d1-dev` | Synced / Healthy | `sha256:05b8cb60c354a44ab824ea6e7dc69b46d50762cdbe728a347a5b656e6fb3d7c4` |
| `d1-hml` | Synced / Healthy | `sha256:05b8cb60c354a44ab824ea6e7dc69b46d50762cdbe728a347a5b656e6fb3d7c4` |
| `d1-prd` | Synced / Healthy | `sha256:05b8cb60c354a44ab824ea6e7dc69b46d50762cdbe728a347a5b656e6fb3d7c4` |

No rebuild occurred between stages. The PRD hop additionally carries a real, checkable business
authorization trail (`CHG-2026-000001` → round 1 → requirement `45ffec4b-...` → decision
`de297bf8-...` → eligibility decision `6bd5ef99-...` recorded on the dispatched
`DeploymentRequest`).

## Simplifications Used

- **Delivery-side eligibility evaluation lives outside `ChangeManagementService`, called over
  service-to-service HTTP** (`coreServices.auth` + `coreServices.discovery`), not an in-process
  import — by design, not merely to dodge the architecture guard, so Delivery stays swappable if
  the Change Management provider changes.
- **No dedicated automated test suite for the new `delivery` module.** All evidence above comes
  from exercising the real running system against the real sandbox, not from unit/integration
  tests. This is a real gap, not a demo shortcut avoided.
- **Kargo's no-op-PR limitation was hit three times in this run** (dev, hml, and the first PRD
  attempt) — a promotion whose target manifest already matches the desired digest produces no Git
  diff, so `git-open-pr` has nothing to open and `git-wait-for-pr` errors trying to read a
  nonexistent PR's id. This is the exact gap D1 carried forward (`Do not fail merely because a
  PR-producing step was skipped`); it was not fixed in Kargo's own template, only worked around by
  seeding a different PRD baseline digest so the real governed promotion had genuine content to
  move. DEV/HML runtime state was never at risk during any of these no-op errors — they are
  visible, non-destructive failures, correctly mirrored to `failed` in the Delivery projection.
- **Two Argo/Kargo control-plane gaps were closed imperatively, matching D0's established
  pattern** (Application/AppProject CRs are documentation, not Git-reconciled): the
  `kargo.akuity.io/authorized-stage` annotation was missing on the newly created `d1-prd`
  Application (present on `d1-dev`/`d1-hml` from D1's own bootstrap), and the `d1-prd` namespace
  had to be created manually because `CreateNamespace=true` requires `Namespace` in the
  AppProject's `clusterResourceWhitelist`, which is intentionally empty (D0's cluster-resource
  boundary). Both are documented in code comments at their source (the Application manifest, and
  this doc) rather than silently patched.
- **Provider-neutral wiring bug found and fixed during the run, not before:** Kargo's admission
  webhook always mints its own `metadata.name` for a `Promotion` (its
  `<stage>.<ulid>.<freight-prefix>` convention), ignoring any name supplied at create time. The
  first implementation assumed the requested name would stick, breaking projection read-back
  silently. Fixed to use `generateName` and read the actual returned name, with idempotent retry
  via lookup on the `idp.laborit.io/deployment-request-id` annotation instead of a
  self-constructed name.
- **`ChangeManagementService`'s idempotency-reservation path did not originally honor the new
  `defaultAuthorizationMode`** — every real caller (the GMUD frontend always sends an
  `Idempotency-Key`) would have silently stayed on `LEGACY_PRE_F3` regardless of config. Found by
  actually creating a GMUD through the API and checking its persisted `authorization_mode`, not by
  reading the code. Fixed by passing `authorizationMode` into
  `idempotencyRepository.reserve()`.

## Carried Production Gaps

Per the execution prompt's explicit deferral list — none of these were required to close for this
checkpoint, and none were touched:

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

Additionally carried forward from this checkpoint specifically:

- The authorization ledger's round/decision model is now genuinely load-bearing for one sandbox
  policy (`delivery-mvp-sandbox-policy` v1, a single static `authority`/`pre_execution`
  requirement). It has not been exercised against F3.1.x's richer policy shapes
  (CAB, multi-round, post-execution governance) — those remain F3.1.2+ scope.
- Kargo's no-op-PR / idempotent-promotion gap (D1-carried) is still open in Kargo's own templates;
  this MVP worked around it via baseline seeding rather than hardening the promotion template
  itself, per the prompt's instruction not to spend the MVP solving Kargo's entire operational
  model.
- `delivery` has zero automated test coverage. `changeManagement`'s 14 existing suites (104 tests)
  remain green, confirmed by rerunning them after every change to that module.
- The sandbox Kubernetes API access used by the `delivery` backend plugin
  (`@kubernetes/client-node`, `kc.loadFromDefault()`) uses the ambient kubeconfig's full
  cluster-admin credential — the same posture D0 already flagged as a production-hardening gap, now
  additionally exercised by a second, independent code path (Delivery, not just Argo's own
  reconciler).

## Deviations / Blockers

No hard STOP condition (contract §20) was triggered. Two real blockers were hit and resolved without
weakening any guardrail (see Simplifications above for the Argo authorization annotation and
namespace gaps). One process blocker: the harness's own action classifier correctly declined to
let the agent silently self-approve pull requests or record a business authorization decision —
both required explicit, separate human action (a second ADO identity approving each PR; the user
explicitly confirming the authorization-decision API call) before the agent proceeded. This matches
the spirit of the contract's authorization model rather than working around it.

## Files Changed

**`platform-devops-developer-portal`** (branch `feat/delivery-mvp-slice`):
- New: `packages/backend/src/modules/delivery/**` (types, errors, permissions, service,
  persistence, Kargo/Argo provider, eligibility client)
- New: `packages/backend/migrations/delivery/20260906000000_initial.cjs`
- New: `packages/backend/src/plugins/deliveryPlugin.ts`
- New: `packages/backend/src/modules/changeManagement/authorization/EligibilityService.ts`
- New: `packages/backend/config/authorization/sandbox-policy-v1.json`
- New: `packages/app/src/modules/catalogEntityTabs/DeploymentsTab.tsx`
- Modified (minimal, guard-preserving): `packages/backend/src/modules/changeManagement/
  ChangeManagementService.ts` (config-driven `defaultAuthorizationMode`, passed into both the
  direct and idempotency-reservation paths)
- Modified: `packages/backend/src/plugins/changeManagementPlugin.ts` (wires `EligibilityService` +
  2 new routes, independently of `ChangeManagementService`)
- Modified: `packages/backend/src/index.ts` (registers `deliveryPlugin`)
- Modified: `packages/app/src/modules/catalogEntityTabs/index.ts` (registers `Deployments` tab)
- Modified: `packages/backend/package.json` (adds `@kubernetes/client-node` direct dependency,
  `config` to `files`)
- Modified: `app-config.yaml` (`changeManagement.authorizationMode: LEDGER_REQUIRED`, `delivery.*`
  config block, `delivery` added to `permission.rbac.pluginsWithPermission`)

**`d0-gitops-sandbox`** (branch `d1/desired-state`, via 3 merged PRs):
- PR #75: `stages/prd/{deployment,service}.yaml`, `argocd/d1-application-prd.yaml`,
  `argocd/d1-project.yaml` (AppProject destination extended)
- PR #76: `stages/prd/deployment.yaml` baseline digest seed
- PR #77 (opened by Kargo, not the agent): the actual PRD promotion commit

## Commits

- `platform-devops-developer-portal@feat/delivery-mvp-slice`: not yet committed — pending this
  report's review before committing (§25: narrow, path-scoped commits, never sweeping the 40
  unrelated uncommitted files from `feat/ado-repo-governance` also present in this working tree).
- `d0-gitops-sandbox@d1/desired-state`: PRs #75, #76 merged by the agent's branch + a second human
  approver; PR #77 opened by Kargo, merged the same way.

## Documentation Updated

This document. `docs/adr/ADR-012-delivery-management-gitops-promotion.md` status is **not**
changed — still `Proposed`, per the execution prompt's explicit instruction not to mark it Accepted
solely because the MVP works.

## Recommended Next Step

The user should decide the next checkpoint explicitly. Candidates, not started:

- Add automated test coverage for the `delivery` module (unit tests for `DeliveryService`'s CAS/
  idempotency logic at minimum; the live-system evidence in this report substitutes for it today
  but does not replace it going forward).
- Harden Kargo's promotion templates against the no-op-PR case (skip `git-wait-for-pr` cleanly when
  `git-open-pr` had nothing to open), closing the D1-carried gap this run hit three times.
- Decide whether the sandbox policy's single static requirement should evolve toward F3.1.2's
  richer ledger model (CAB, multi-round) before any further Delivery integration work, or whether
  Delivery should remain decoupled from that evolution.
- Any production-hardening item from the Carried Production Gaps list — none are implied as next
  by this report.

## Gate

STOP
