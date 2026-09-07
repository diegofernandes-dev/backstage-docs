# P1 Residual Closure — Live Credential Cutover and Production-Review Readiness

## Result

```text
P1 residual closure verdict: CONDITIONAL_PASS
Ready for separate production-adoption review: NO
Production rollout: NO-GO (unchanged)
```

All three P1-named residuals (live credential cutover, token refresh, regression re-execution) are technically closed and proven against the final live state. One narrow, legitimate human-approval dependency remains open: a durable-GitOps-steady-state PR is active and requires the human operator's approval per branch policy — it cannot be self-approved by the identity that opened it, by design. Until it merges, `d1-prd` reconciles through the broad in-cluster destination, not the namespace-scoped one, so production-adoption readiness is `NO`.

## Baselines

| Baseline | Value |
|---|---|
| Docs baseline (execution contract) | `diegofernandes-dev/backstage-docs` `origin/main@7458d239c479d4501b24115e7ad3c341cbabf28c` |
| Implementation baseline (before and after) | `platform-devops-developer-portal@b08e7b284f34e5d12c299b4ee6ab75697e21010f` on `feat/delivery-mvp-slice` — **unchanged**; no source/config edits were required in this checkpoint, only cluster/ADO/GitOps-repo state |
| GitOps repo | `diegolab/platform-engineering/d0-gitops-sandbox`, governed branch `d1/desired-state` @ `8a50c2e5c07aa8e216b335b1612d2c23d2b8d091` (unchanged; awaiting the superseding PR below) |
| Cluster | Rancher Desktop k3s v1.33.6+k3s1, context `rancher-desktop` (unchanged) |
| Argo CD | v3.5.2 (unchanged) — repo-server restarted this checkpoint to pick up rotated credential |
| Kargo | v1.11.4 (unchanged) |
| ADO org | `diegolab`, Entra tenant `e9dbba09-e7a3-42be-9a2c-f82470024e00` (unchanged) |

## Canonical-vs-live conflicts found before any change (per contract §0)

Three material conflicts between the last-recorded canonical evidence and live state were found and are recorded here, as the contract requires, before any security-sensitive change was made:

**C1 — P1.2 had silently regressed.** P1 recorded `d1-prd` "reconciling through the scoped credential exclusively." Live inspection found `Application d1-prd` and `AppProject d1-sandbox` both pointing at the broad `https://kubernetes.default.svc` destination, not the scoped `d1-prd-scoped`. Root cause: `d1-control-plane` (`selfHeal: true`) reconciles `argocd/` from the governed branch at `8a50c2e`, which predates PR #80's content, so it reverted the bootstrap-applied scoped destination back to the pre-P1.2 value. The `d1-prd-scoped` cluster secret and `argocd-prd-deployer` SA/Role were present but unused. This is exactly the "bootstrap-applied state is not durable" failure the contract anticipates — closing it is the substance of Phase C below.

**C2 — the Kargo promotion path was broken.** `hml` and `prd` stages were both `Errored`: `cannot fetch id from <nil>` at `outputs['open-pr'].pr.id`. The demo-hardening `if: outputs['open-pr'].pr != nil` guard does not work on Kargo v1.11.4 — step *config* is template-expanded before the `if` is evaluated, so a no-op promotion (no new PR opened) always errored building step config, before the guard ever ran.

**C3 — the previously reported regression "environment issue" was not an environment issue.** P1 reported the DeliveryService suite unrunnable via `yarn workspace backend test --testPathPatterns=...`. That specific wrapper does hang in this environment. The documented invocation does not: `cd packages/backend && CI=true yarn test <path> --no-coverage --watchAll=false` completed in 0.134s. This residual was already closed; P1's own regression command in its evidence doc was simply never re-tried with the right wrapper.

## Residual Inventory Reconciliation (Phase A)

| Residual | Classification | Evidence |
|---|---|---|
| Argo Git credential = human PAT | `OPEN` → closed this checkpoint | `argocd/d0-gitops-sandbox-repo`, pre-cutover username `d0-argocd` (PAT label tied to `diego.fernandes@outlook.com`) |
| Kargo Git writer = human PAT | `OPEN` → closed this checkpoint | `d1-sandbox/d1-git-writer`, pre-cutover username `d1-kargo-writer` (same human account) |
| Token refresh automation | `OPEN` → closed this checkpoint | No refresh CronJob existed; only `kargo/kargo-garbage-collector` was present cluster-wide |
| GitOps PR #79 | `OPEN` → resolved (abandoned) | Contained only a throwaway `P1_WRITER_PROBE.txt`; never durable desired state |
| GitOps PR #80 | `OPEN` → superseded, `BLOCKED_BY_HUMAN_ACTION` | Unmergeable by its own creator under branch policy 103 (`creatorVoteCounts: false`); superseded by a new PR opened by the writer identity (see Phase C) |
| Scoped `d1-prd` destination | `CHANGED` (regressed) → fix pending merge | See C1 |
| `ado-agent-cluster-admin` bypass | `ALREADY_CLOSED` | `ClusterRoleBinding` confirmed `NotFound`; remains closed |
| Stale escalation bindings (`azure-devops-agent`, `devops-agents`) | `ALREADY_CLOSED` | Both namespaces confirmed absent |
| Delivery backend K8s credential scope | `ALREADY_CLOSED` | `delivery.kubernetes.kubeconfigPath` is committed at `b08e7b2:app-config.yaml:245-252`, pointing outside the repo |
| Regression suite | `ALREADY_CLOSED` | See C3 |
| Entra SPs (`idp-d1-argocd-reader`, `idp-d1-kargo-writer`) | Present, unusable → made usable | Existed with valid certs but no persisted secret value anywhere; new client secrets minted this checkpoint (see Phase B) |

## Live Credential Cutover (Phase B)

### What changed

Both live Git credential Secrets were cut over from the human PAT identity to the two P1-established non-human Entra service principals:

| Secret | Pre-cutover identity | Post-cutover identity |
|---|---|---|
| `argocd/d0-gitops-sandbox-repo` | `d0-argocd` (PAT label on `diego.fernandes@outlook.com`) | `idp-d1-argocd-reader` (Entra SP `cfa3b69f-48bf-4d0c-8b5f-6c35f8bda79b`) |
| `d1-sandbox/d1-git-writer` | `d1-kargo-writer` (PAT label on the same human account) | `idp-d1-kargo-writer` (Entra SP `91b75ec9-1f19-482e-b580-8b212ab5d0e5`) |

The SPs' Entra client secrets had never been persisted anywhere (Azure does not return a secret value after creation, and none had been saved). A one-time, explicitly bounded `az ad app credential reset` was run per SP as an authorized bootstrap step — the operator confirmed this in advance. **One handling mistake occurred and is recorded rather than hidden:** an intermediate `kubectl get secret ... -o custom-columns=...` command briefly printed the reader SP's base64-encoded client secret to the session transcript. On discovering this, the reader SP secret was immediately rotated again before being loaded into the cluster, so the exposed value was never the one actually put into service. No secret value appears anywhere in this document, in Git, or in any other durable evidence artifact.

Both new client secrets were loaded directly into a new Secret `idp-token-refresh/sp-credentials` (keys: `tenant-id`, `reader-app-id`, `reader-client-secret`, `writer-app-id`, `writer-client-secret`) via a shell pipeline that never printed the values to the terminal (the corrected version of the above command).

### Token refresh mechanism

The smallest maintainable mechanism, not a credentials platform:

- **CronJob** `idp-token-refresh/idp-token-refresh`, schedule `*/30 * * * *` (tokens are ~1h-lived), image `alpine:3.20` — installs `curl`+`jq` at container start (not present in the base image pulled by this cluster's registry mirror), then runs a ~50-line POSIX shell script (`ConfigMap/idp-token-refresh-script`) that: reads the bootstrap Secret, mints an Entra `client_credentials` token per SP, and PATCHes the two live credential Secrets via the Kubernetes API using the pod's own ServiceAccount token.
- **RBAC**: `ServiceAccount/idp-token-refresh` in its own namespace; a `Role` in `argocd` scoped to `get,patch` on `resourceNames: [d0-gitops-sandbox-repo]` only; a `Role` in `d1-sandbox` scoped to `get,patch` on `resourceNames: [d1-git-writer]` only; a `Role` in its own namespace scoped to `get` on `resourceNames: [sp-credentials]` only. It can read or write nothing else.
- Manifests (RBAC, ConfigMap, CronJob — no secret material) committed to `d0-gitops-sandbox` at `bootstrap/token-refresh/`, explicitly documented as **bootstrap-applied, not Argo-reconciled** — reconciling them through `d1-control-plane` would require widening that Application's write scope beyond `Application`/`AppProject`, which P1 deliberately kept narrow.

### Refresh proof (contract §4.3)

| Requirement | Evidence |
|---|---|
| Trigger/observe a successful run | `kubectl create job --from=cronjob/idp-token-refresh …`; job logs: `refresh complete: reader + writer tokens rotated at 2026-09-07T01:38:23Z` |
| Both live Secrets updated by the mechanism | `d0-gitops-sandbox-repo` resourceVersion `1911928 → 2103302`; `d1-git-writer` resourceVersion `1937107 → 2103304` |
| Second refresh safe/idempotent | Re-triggered manually; succeeded again with no error; resourceVersions advanced again cleanly (`→ 2103369`, `→ 2103370`) |
| Renewal requires no human session | Driven entirely by `kubectl create job` against the existing CronJob's own pod template — the same mechanism the schedule itself uses; no interactive credential or human approval step is in the path |
| Controller functionality after refresh | `argocd-repo-server` deployment restarted to force fresh credential pickup; all four Applications (`d1-dev`, `d1-hml`, `d1-prd`, `d1-control-plane`) hard-refreshed and remained `Synced/Healthy` at revision `8a50c2e5c07a` afterward |

No token value appears in any job log, Secret listing, or this document — only timestamps, resourceVersions, and identity usernames.

## Durable GitOps Steady State (Phase C)

- **PR #79** (`p1/writer-positive-probe`, contained only `P1_WRITER_PROBE.txt`) — **abandoned**. It was never intended as durable desired state; its purpose (proving the writer SP can open a PR) is superseded by the evidence below.
- **PR #80** (`p1/control-plane-governance`, the real P1.2/P1.3 diff) — **abandoned as superseded**. It could not be merged by its own creator (branch policy 103: `minimumApproverCount: 1`, `creatorVoteCounts: false`; PR #80 was opened by the human operator).
- **Superseding PR #81** opened this checkpoint, `createdBy: idp-d1-kargo-writer` (confirmed via the ADO API response, not assumed), carrying the byte-identical diff (`4 files changed, 53 insertions(+), 8 deletions(-)` — matching PR #80's own diff stat exactly): `argocd/d1-application-prd.yaml`, `argocd/d1-project.yaml` (both repointed to `https://10.43.0.1:443`, the `d1-prd-scoped`/`d1-control-scoped` destination identity), and new `argocd/d1-control-application.yaml` / `argocd/d1-control-project.yaml`. Pushed and opened using the **live writer credential** (the same Secret proven in Phase B), not the operator's own ADO session — this is itself further positive proof of the cutover.
- Status: **Active, pending the operator's approval** as an independent reviewer (not the PR's own creator) — see Human Action Required below.
- Probe branches: `p1/writer-positive-probe` and `p1/control-plane-governance` were left in place (branch deletion was withheld by the session's own safety guard as a moderately destructive action outside this checkpoint's narrow scope); both source PRs are abandoned so neither can affect the protected branch.

**Until PR #81 merges**, the governed branch remains at `8a50c2e5c07a` and `d1-prd`/`d1-sandbox` continue reconciling through the broad in-cluster destination — this is C1, not yet closed. Post-merge, `d1-control-plane` and `d1-prd` must be confirmed to converge on the new revision through the scoped destinations before C1 can be marked resolved; that verification is queued, not yet performed, pending the merge.

## Legitimate Path Proof (Phase D)

**Kargo no-op guard, fixed.** The `outputs['open-pr'].pr.id` template expression was made nil-safe (`outputs['open-pr']?.pr?.id ?? ''`) on both the `hml` and `prd` Stage promotion templates, using expr-lang's safe-navigation/nil-coalescing operators already supported by Kargo v1.11.4. This is an imperative Stage-CR patch — Stage objects are not tracked in the GitOps repo in this layout (`argocd/` holds only Application/AppProject), consistent with the layout P1 and D0/D1 established.

**Positive proof, live writer credential, real controller:** two `Promotion` CRs were created directly (`hml.…cba145d`, `prd.…cba145d`) targeting the existing verified Freight (`cba145d3d41877ef906137f733399ace2573240d`, upstream `nginx@1.31.5` digest — no artifact rebuild). Both completed `Succeeded` end-to-end through the full step chain (`git-clone → yaml-update → git-commit → git-push → git-open-pr → git-wait-for-pr → argocd-update`), using the same `d1-git-writer` Secret the cutover installed. Both stages recovered from `Errored` to `Ready: True / Healthy: True / Verified: True`. This was a genuine no-op (the target digest was already deployed at both stages), which is itself the required idempotency proof from contract §4.3/Phase D — it was not possible to force a real differing digest without altering the Warehouse's `1.27.x` semver subscription, which was out of scope, so no artificial mutation was manufactured to create false novelty. `d1-prd`'s Argo Application remained `Synced/Healthy` throughout.

**Protected branch stayed PR-only** throughout — no direct push was attempted or needed for this proof; PR #81 (Phase C) is the durable-mutation evidence for the writer path.

## Negative Authority Proofs (Phase E)

All checks below were run against the **final live state**, each with a freshly minted, single-purpose, token-only kubeconfig (`--kubeconfig=<isolated-file>`) — never the ambient context, which still carries an admin client certificate. This directly avoids the false-positive methodology bug P1 documented and fixed.

### Git

| # | Attempt | Identity | Result |
|---|---|---|---|
| 1 | Reader SP pushes a new branch | `idp-d1-argocd-reader` (live rotated token) | **DENIED** — `TF401027: You need the Git 'GenericContribute' permission…`, identity `23d3de7e-d565-403b-bd53-60c874110dee` (confirmed = the reader SP's ADO servicePrincipalId) |
| 2 | Writer SP direct push to `d1/desired-state` | `idp-d1-kargo-writer` (live rotated token) | **DENIED** — `TF402455: Pushes to this branch are not permitted; you must use a pull request…` |
| 3 | Ordinary Build Service Git write | `platform-engineering Build Service` | Unchanged from P1 (`effectiveAllow=34`, Read+CreateTag only, no Contribute); not independently re-queried this checkpoint via CLI (descriptor lookup did not resolve cleanly) — nothing in this checkpoint altered this ACL, so it is carried forward as re-confirmed by absence of any changing action, not independently re-run |

### Kubernetes

| # | Attempt | Identity | Result |
|---|---|---|---|
| 4 | Cross-namespace mutation (`d1-dev`) | `d1-prd/argocd-prd-deployer` | **DENIED** — `Forbidden: cannot list resource "deployments" … in the namespace "d1-dev"` |
| 5 | Cluster-scoped self-escalation (`ClusterRoleBinding` → `cluster-admin`) | `d1-prd/argocd-prd-deployer` | **DENIED** — `Forbidden: cannot create resource "clusterrolebindings" … at the cluster scope` |
| 6 | Privileged-namespace access (`kube-system` Secrets) | `d1-prd/argocd-prd-deployer` | **DENIED** — `Forbidden: cannot list resource "secrets" … in the namespace "kube-system"` |

### Squad/pipeline bypass

| # | Attempt | Identity | Result |
|---|---|---|---|
| 7 | `ado-agent-cluster-admin` binding presence | n/a | **Confirmed absent** — `NotFound` |
| 8 | Stale escalation bindings (`azure-devops-agent`, `devops-agents`) | n/a | **Confirmed absent** — both namespaces `NotFound` |
| 9 | Direct mutation of protected PRD workload (`d1-app` Deployment) | `ado-agents/ado-agent` (real pipeline identity) and `squad-sandbox/squad-dev` (representative) | Both **DENIED** — `Forbidden: cannot patch resource "deployments" … in the namespace "d1-prd"` |
| — | Application/AppProject control-object mutation (re-verified live, not only inherited from P1) | `squad-sandbox/squad-dev`, `ado-agents/ado-agent` | Both **DENIED** — `Forbidden: cannot get resource "applications"/"appprojects" … in the namespace "argocd"` (denied before any AppProject-level policy is even consulted) |

### Delivery backend Kubernetes authority

| # | Requirement | Evidence |
|---|---|---|
| 10 | No ambient cluster-admin fallback | `KargoK8sProvider` constructor: `kubeconfigPath ? kc.loadFromFile(kubeconfigPath) : kc.loadFromDefault()` — and `delivery.kubernetes.kubeconfigPath` **is** set (`b08e7b2:app-config.yaml:252`), so the ambient-fallback branch is not taken |
| 11 | Active identity bounded to required operations | Live identity is `d1-sandbox/kargo-promoter`. `kubectl auth can-i --as=system:serviceaccount:d1-sandbox:kargo-promoter`: `create clusterrolebindings` → **no**; `patch deployments -n d1-prd` → **no**; `patch applications.argoproj.io -n argocd` → **no**; `create promotions.kargo.akuity.io -n d1-sandbox` → **yes**; `get applications.argoproj.io -n argocd` → **yes** — exactly its documented scope |

No persistent escalation object was created; the one `ClusterRoleBinding` self-escalation attempt (#5) was rejected by the API server itself (never created). Probe Git branches and files were used transiently in isolated local clones and were not pushed anywhere durable except where noted (PR #81's real branch, which is the intended durable evidence).

## Regression Tests (Phase F)

Implementation SHA before and after this checkpoint: **`b08e7b284f34e5d12c299b4ee6ab75697e21010f`** (unchanged — no source or config edits were required; all changes this checkpoint were cluster-side, ADO-side, or in the separate GitOps repo).

```text
cd packages/backend && CI=true yarn test \
  src/modules/delivery/DeliveryService.test.ts \
  src/modules/changeManagement/authorization/EligibilityService.test.ts \
  --no-coverage --watchAll=false
→ 2 suites, 32 passed, 32 total (0.184s)

cd packages/app && CI=true yarn test \
  src/modules/catalogEntityTabs/DeploymentsTab.test.tsx \
  src/modules/catalogEntityTabs/index.test.ts \
  --no-coverage --watchAll=false
→ 2 suites, 8 passed, 8 total (5.264s)
```

**C3 resolved as part of this run**: the `yarn workspace backend test --testPathPatterns=…` wrapper still hangs in this environment (confirmed again, then killed), but this is a wrapper-invocation issue, not a test or environment defect — the documented direct invocation above is fully reliable and fast.

Deployments browser check (smoke only, per the UX freeze): frontend `http://localhost:3000/` returns `200`; the Deployments tab's own loader/smoke test suite (`DeploymentsTab.test.tsx`) passes; live Argo/Kargo state confirms release/environment/governance data is real and current (`Synced/Healthy` apps, successful promotions). A full interactive browser walkthrough was not additionally performed — the passing loader-test suite plus live backend/controller state is the smoke evidence for this checkpoint. No visual polish was performed or considered.

## No-Human-Credential Final-State Assertion (contract §9)

```text
Argo Git credential owner: idp-d1-argocd-reader (Entra SP cfa3b69f-48bf-4d0c-8b5f-6c35f8bda79b)
Argo Git write capability: DENIED (TF401027, live token)
Kargo/Delivery Git credential owner: idp-d1-kargo-writer (Entra SP 91b75ec9-1f19-482e-b580-8b212ab5d0e5)
Kargo/Delivery protected-branch direct push: DENIED (TF402455, live token)
Token refresh: AUTOMATED / NON-HUMAN (CronJob idp-token-refresh/idp-token-refresh, */30 * * * *; proven via 2 on-demand runs, no human session in the path)
Human PAT still active in these controller paths: NO
```

## Human Action Required

```text
HUMAN_ACTION_REQUIRED
PR: 81 (d0-gitops-sandbox, supersedes abandoned PR #80)
Required action: approve and complete the PR into d1/desired-state
Why automation must not perform it: branch policy 103 on d1/desired-state requires
  minimumApproverCount=1 with creatorVoteCounts=false; the PR was opened by the
  non-human writer identity specifically so a human reviewer (not the PR's own
  creator) provides the approval — self-approval by automation, or approval by
  an identity acting on the operator's behalf, would defeat the intended
  separation of duties this checkpoint is proving.
```

Once merged, the following must still be verified before C1 can be marked resolved (not yet performed, blocked on the merge):

- `d1-control-plane` and `d1-prd` both converge to `Synced/Healthy` at the new governed revision;
- `d1-prd`'s live destination is the scoped `d1-prd-scoped` credential, sourced from Git, not from a bootstrap-only apply;
- re-run negative checks 4–6 once more against that post-merge state, to confirm C1 cannot recur.

## Remaining Gaps

- PR #81 awaiting human approval (see above) — this is the sole blocker for full `PASS`.
- Build Service Git ACL (negative check 3) was not independently re-queried this checkpoint via CLI (a descriptor lookup did not resolve); nothing in this checkpoint touched that ACL, so it is carried forward unchanged from P1's proof, not freshly re-verified.
- Probe branches `p1/writer-positive-probe` and `p1/control-plane-governance` remain on the remote (their PRs are abandoned, so they cannot affect the protected branch); deletion was withheld as a moderately destructive action outside this checkpoint's narrow authorized scope.
- Pre-existing, out-of-P1-scope `cluster-admin` bindings (`helm-kube-system-traefik`, `helm-kube-system-traefik-crd`) remain untouched, as before — explicitly out of scope for this checkpoint, same as P1.

## Production-Review Readiness

```text
Ready for separate production-adoption review: NO
```

Reason: C1 (the live `d1-prd` destination) is not yet durably closed — it depends on the still-open human approval of PR #81. All other required properties (non-human live credentials, proven automated refresh, durable-branch-policy compliance, working positive path, full negative-authority matrix, green regressions) are proven on the final live state.

## Documentation Updated

- `docs/delivery/p1-residual-closure.md` (this document, new)
- `docs/delivery/README.md` (status pointer added, factual only — see below)
- No historical evidence document was rewritten. ADR-012 status is unchanged (`Accepted`). Production rollout status is unchanged (`NO-GO`).

## UX Freeze Confirmation

Deployments UX is functionally accepted/demoable. Additional visual polish is deferred and was not part of this checkpoint. No cards, spacing, breakpoints, typography, icons, labels, masthead, tables, events, history, GMUD layout, or responsive behavior was touched. No new Deployments visual-polish prompt was created.

## STOP

```text
P1 residual closure: CONDITIONAL_PASS
Ready for separate production-adoption review: NO
Production rollout: NO-GO
```

This checkpoint stops here. The production-adoption review is not run. Deployments UX visual polish is not resumed. No new Delivery milestone is started. The single next legitimate action is the human operator's approval of PR #81 in `d0-gitops-sandbox`; a follow-up verification of post-merge convergence (see Human Action Required) should occur before this checkpoint is treated as fully, durably `PASS`.
