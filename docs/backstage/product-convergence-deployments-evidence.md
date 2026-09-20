# Product Convergence — Deployments UX Evidence

## 1. Baseline

| Item | Value |
|---|---|
| Docs baseline reviewed | `diegofernandes-dev/backstage-docs` `c0fb15f7c3fa67366bcdab574d4b4fa1d62abda1` |
| Target ADO branch/tip before | `feat/ado-repo-governance` `@ 3b302ab5c9caab38f96491b389b7ea9fe0b66c2f` |
| Delivery source branch/tip | `feat/delivery-mvp-slice` `@ 170af45dc011e0e0f19ed6e7444e46794e32daaf` |
| Accepted Deployments surface tip | `b08e7b284f34e5d12c299b4ee6ab75697e21010f` (v2 + responsive polish; not tip) |
| Merge-base | `4bad41d058edf5c5314d17275e0c8bdb5abf690f` |
| Target parent | `3b302ab5c9caab38f96491b389b7ea9fe0b66c2f` |
| Target implementation commit | `f48dc825ab5d1d16fafc3f70ef772d28613df1a0` |
| Remote target tip after publication | `f48dc825ab5d1d16fafc3f70ef772d28613df1a0` (independent `az repos ref` verify) |

Live tips were verified with Azure CLI `az repos ref list` (SSH `git fetch`/`push` to ADO failed with authenticated-but-rejected upload-pack; HTTPS + Azure AD bearer token succeeded for push/fetch).

## 2. Root cause

**`BRANCH_DIVERGENCE`.**

On `feat/ado-repo-governance@3b302ab`, Catalog Component entity tabs were only README + Changelog. The accepted Deployments tab, Delivery backend plugin/module, migrations, and eligibility read path existed only on `feat/delivery-mvp-slice` after the merge-base. No later governance regression or feature-visibility gate explained the absence.

## 3. Convergence manifest (summary)

Source tree for PORT content: **`b08e7b2`** (not wholesale tip `170af45`).

| source | action | notes |
|---|---|---|
| `packages/app/.../DeploymentsTab*` + `deployments/**` + `CompactEntityHeaderStyles` | PORT | Accepted v2 + polish; preserve `name: 'delivery'` → `api:catalog/delivery` |
| `catalogEntityTabs/index.ts` + `index.test.ts` | REIMPLEMENT_MINIMALLY | Register Deployments + delivery API + compact header; **exclude** Platform tab |
| `packages/backend/src/modules/delivery/**` + `deliveryPlugin` + migrations | PORT | Runtime required by the tab |
| `EligibilityService*` + `sandbox-policy-v1.json` | PORT | Eligibility panel / Delivery eligibility client fixture only |
| `changeManagementPlugin.ts` | REIMPLEMENT_MINIMALLY | Keep F3.1.1b bootstrap; add `GET .../execution-eligibility` only |
| `plugins/change-management/src/index.ts` | REIMPLEMENT_MINIMALLY | Additive type/label re-exports |
| `app-config.yaml` | REIMPLEMENT_MINIMALLY | Add `delivery` RBAC plugin + optional `delivery:` block; **exclude** `authorizationMode: LEDGER_REQUIRED` and operator kubeconfig path |
| `rbac-policy.csv` | REIMPLEMENT_MINIMALLY | Additive `delivery.deployment.*` for contributor + platform_admin |
| `packages/app` + `packages/backend` package.json / yarn.lock | REIMPLEMENT_MINIMALLY | Declare catalog-react/catalog-model; `@kubernetes/client-node`; pack `config/` |
| PlatformTab / catalogValidation / idpAssessor demo deltas | EXCLUDE | Unrelated demo surfaces |
| `POST .../authorization/decisions` | EXCLUDE | Sandbox decision path not used by accepted Deployments UX |
| `ChangeManagementService` LEDGER_REQUIRED default | EXCLUDE | Preserve F3.1.2a: POST `/changes` stays `LEGACY_PRE_F3` |
| `170af45` / `app-config.production.yaml` credential split | EXCLUDE | Production/laptop hardening |
| Secrets, kubeconfigs, GitOps/Kargo/Argo infra apply | EXCLUDE | Forbidden scope |

Lint-only adaptations inside ported UI (nested ternaries → if/else; remove `autoFocus`) preserved behavior.

## 4. Changed paths

55 files on ADO commit `f48dc82` (see `git show --stat f48dc82`). Representative:

- Frontend Deployments surface under `packages/app/src/modules/catalogEntityTabs/`
- Delivery backend under `packages/backend/src/modules/delivery/` + `plugins/deliveryPlugin.ts` + migrations
- EligibilityService + CM plugin eligibility route
- app-config / RBAC / package manifests / yarn.lock

## 5. Tests

| Gate | Result |
|---|---|
| Deployments frontend (`catalogEntityTabs`) | PASS — 5 suites / 20 tests |
| Delivery backend | PASS — 26 tests |
| EligibilityService | PASS |
| Change Management module (excl. CI-only postgres URL gate) | PASS — 333 tests (4 skipped) |
| GMUD frontend plugin | PASS — 13 suites / 58 tests |
| architecture.test (F3.1.1c / POST unwired / no Delivery in CM domain) | PASS |
| Lint | PASS (0 problems; also cleared pre-existing undeclared catalog-react imports by declaring deps) |
| `yarn build:all` | PASS |
| TypeScript baseline | IDENTICAL — exactly 5 historical knex dual-package errors in `changeManagementPlugin.ts` |

## 6. Browser / product proof

Against converged checkout `yarn start` (worktree → later main tip `f48dc82`):

- Backend initialized `delivery` + `change-management`; F3.1.1c policy pin validated at startup.
- Catalog Component list loaded for authenticated user.
- `IDP Showcase API` entity tabs include **Deployments** (selected).
- Route `/catalog/default/component/idp-showcase-api/deployments` served accepted empty-state copy: “Nenhuma release candidate registrada” (truthful; no Kargo/Argo sandbox data imported).
- Delivery HTTP reads returned 200 (`release-candidates`, `promotion-history`, `events`) with empty payloads; `delivery.deployment.read` ALLOW.
- Served bundle contains `path:"deployments",title:"Deployments"`, `name:"delivery"`, `plugin.catalog.delivery.service`.
- No duplicate extension-id / catalogApiRef collision warnings in startup logs.
- GMUD `/gmud` remained reachable (“Minhas GMUDs” empty-state) after Deployments navigation; Catalog sidebar intact.

**Limitation:** no live Kargo/Argo delivery data in this environment (intentionally excluded). Proof is tab/navigation + empty/truthful runtime + automated suites.

## 7. Intentionally excluded Delivery surfaces

- Wholesale merge of `feat/delivery-mvp-slice`
- `170af45` production vs laptop credential path / `app-config.production.yaml`
- Operator-local kubeconfig paths and secrets
- Platform demo tab and assessor/catalogValidation demo modules
- Sandbox `POST /authorization/decisions` and `authorizationMode: LEDGER_REQUIRED`
- Any Kubernetes/GitOps/Kargo/Argo resource apply or production rollout

## 8. Verdict

```text
Product convergence (GMUD + Deployments): PASS
Active product branch: feat/ado-repo-governance @ f48dc82
F3.1.2a: CLOSED / ACCEPTED
F3.1.1c: CLOSED / ACCEPTED
F3.1.2b implementation-prompt authoring: GO
F3.1.2b implementation: still requires separate explicit authorization
F3.2: NO-GO
```

STOP. Do not author or implement F3.1.2b from this checkpoint.
