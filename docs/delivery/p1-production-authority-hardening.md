# P1 — Production Authority Hardening

## Result

```text
P1 verdict: CONDITIONAL_PASS
```

All four authority properties have real positive and negative execution evidence. Two items are carried as named, bounded follow-ups rather than closed: (a) the two new Entra service-principal Git credentials use short-lived (~1h) OAuth tokens and were proven functionally correct, but were not wired into the *live* Argo/Kargo credential secrets with an automated refresh mechanism in this session; (b) two Argo destination service accounts required a broader-than-minimal, but still namespace-scoped and write-narrow, read grant to satisfy Argo's own cache architecture in this shared multi-tenant sandbox cluster — a real platform constraint, not a design compromise. Neither gap reopens the architecture or changes the core finding: **an ordinary squad identity cannot bypass the governed Delivery path**, proven negatively and repeatedly against real credentials.

## Baselines

| Baseline | Value |
|---|---|
| Docs baseline (execution contract) | `diegofernandes-dev/backstage-docs` `origin/main` @ `e64435eac124b64c593436397c82795ae8fea481` |
| Implementation baseline | `platform-devops-developer-portal` / `feat/delivery-mvp-slice` @ `ee114cfac59d7aa91a190c188d9e634547534c8d` (working tree clean except untracked `.vscode/`) |
| Implementation change (this checkpoint) | `app-config.yaml` — added `delivery.kubernetes.kubeconfigPath` only; no source code modified |
| GitOps repo | `diegolab/platform-engineering/d0-gitops-sandbox`, governed branch `d1/desired-state`, base `8a50c2e5c07aa8e216b335b1612d2c23d2b8d091` |
| GitOps PRs opened this checkpoint | PR #79 (writer-identity proof, `p1/writer-positive-probe` → `d1/desired-state`, merge status `succeeded`, awaiting human approval); PR #80 (P1.2/P1.3 control-plane governance, `p1/control-plane-governance` → `d1/desired-state`, includes a follow-up fix commit for AppProject description length, merge status `succeeded`, awaiting human approval) |
| Cluster | Rancher Desktop k3s v1.33.6+k3s1, context `rancher-desktop`, single-cluster sandbox (unchanged from D0/D1) |
| Argo CD | v3.5.2 (unchanged) |
| Kargo | v1.11.4 (unchanged) |
| ADO org | `diegolab`, backed by Entra tenant `e9dbba09-e7a3-42be-9a2c-f82470024e00` (a *different* tenant from the operator's default Azure CLI tenant `1c551f3e-b73b-4d35-8df4-dfbb90fb3e8a` — see Finding 0) |

Both GitOps PRs are technically proven independently of merge status: PR #79's writer identity was proven by direct authenticated Git operations against the real ADO API (not merely by the PR existing); PR #80's manifests were bootstrap-applied to the live cluster directly from the branch content (matching the D0/D1-established one-time-imperative-bootstrap pattern) so the control-plane governance proof did not wait on merge. Merge remains required before these changes are the durable, attributable Git record; **do not treat this document as recording a merged, governed steady state until both PRs are completed.**

## Finding 0 — ADO org / Entra tenant mismatch (methodology note, not a security gap)

The `diegolab` Azure DevOps organization is backed by Entra tenant `e9dbba09-e7a3-42be-9a2c-f82470024e00`, which is **not** the operator's default `az` CLI tenant (`1c551f3e-b73b-4d35-8df4-dfbb90fb3e8a`). The first attempt to create the two service principals succeeded in the wrong tenant and every onboarding call failed with `VS403283: Could not add user ... at this time` until this was discovered (via `az account tenant list` cross-referenced against the ADO `connectionData` API's `authorizedUser.descriptor`) and the session re-authenticated (`az login --tenant e9dbba09-...`) before recreating both SPs in the correct tenant. The original (wrong-tenant) app registrations were deleted. This is recorded because a future operator hitting `VS403283` should check tenant alignment first, not assume a permissions problem.

## Baseline authority matrix (before P1)

| Surface | Identity | Authority (verified live, not assumed) | Risk |
|---|---|---|---|
| Delivery/Kargo Git writer | ADO PAT, label `d1-kargo-writer`, Secret `d1-sandbox/d1-git-writer` | Code R&W scope; only real constraint was branch policy 103 | Tied to human account `diego.fernandes@outlook.com` (confirmed: no ADO graph identity named `d0-argocd`/`d1-kargo-writer` exists — these are just PAT display labels) |
| Argo Git reader | ADO PAT, label `d0-argocd`, Secret `argocd/d0-gitops-sandbox-repo` | **Live-tested, not assumed:** a throwaway branch push with this exact credential **succeeded** (`p1/negative-argo-write-probe`, immediately deleted after proof) | **Confirmed real write bypass** — the "reader" had full write authority prior to remediation |
| Argo Kubernetes identity | SA `argocd/argocd-application-controller` | ClusterRole `{apiGroups:["*"],resources:["*"],verbs:["*"]}` + `nonResourceURLs:["*"]`; zero cluster secrets registered; impersonation off | cluster-admin-equivalent; AppProject was the only boundary |
| Application/AppProject mutation | anyone with cluster access | Fully imperative; not reconciled by any Application | Realistic bypass path D0 flagged and P1 closes |
| Delivery backend's own K8s identity | `KargoK8sProvider` via `kc.loadFromDefault()` | Ambient kubeconfig = cluster-admin | The governed service was itself an unbounded bypass |
| Ordinary squad pipeline (real identity) | SA `ado-agents/ado-agent` | **`ClusterRoleBinding ado-agent-cluster-admin` → `cluster-admin`**, confirmed live: could patch `d1-prd` Deployments, create ClusterRoleBindings, read `kube-system` Secrets | **Confirmed total, live bypass** — not hypothetical |
| Stale bindings | `azure-devops-agent/default`, `devops-agents/rancher-pipelines-agent` | `cluster-admin`, namespaces confirmed absent (`NotFound`) | Dangling escalation path if either namespace/SA is ever recreated |
| Ordinary squad pipeline (Git) | ADO `platform-engineering Build Service` | ACL `effectiveAllow=34` = GenericRead + CreateTag only, **no Contribute** | Already fail-closed — a positive baseline finding, unchanged by P1 |

## P1.1 — Git writer/reconciler identity separation

**Score: PROVEN** (capability, positive and negative execution) **with one named residual gap** (live-secret wiring + token refresh automation).

### What was built

Two real, distinct Entra service principals, onboarded into the `diegolab` ADO org as first-class non-human identities (not PATs tied to a human):

| Identity | Entra `appId` | ADO servicePrincipalId (TF identity) | ACL (Git Repositories namespace, repo `d0-gitops-sandbox`) |
|---|---|---|---|
| `idp-d1-argocd-reader` | `cfa3b69f-48bf-4d0c-8b5f-6c35f8bda79b` | `4a9f3eb9-9d97-602d-950e-5669a03257c8` | Allow `GenericRead(2)`; **Deny** `GenericContribute(4)+ForcePush(8)+CreateBranch(16)=28` |
| `idp-d1-kargo-writer` | `91b75ec9-1f19-482e-b580-8b212ab5d0e5` | `74905480-7b1b-6657-a931-ec6035fe3e34` | Allow `GenericRead+GenericContribute+CreateBranch+PullRequestContribute=16406`; **Deny** `ForcePush+PolicyExempt+PullRequestBypassPolicy=32904` |

Both also required project-level `GENERIC_READ(1)` (ADO's Git-over-HTTPS clone path fails with `TF401019` without it — a real, non-obvious finding: repo-level ACLs alone are not sufficient for Git protocol access; project-level visibility is a separate, additional gate).

### Positive evidence (executed)

- Reader SP: `git clone` of `d0-gitops-sandbox` succeeds using an Entra `client_credentials` OAuth token (both as a Bearer header and — separately confirmed — as a Basic-auth password, which is the mechanism Argo's own repo-credential Secret actually uses).
- Writer SP: pushes a new branch (`p1/writer-positive-probe`) — succeeds. Opens a **real** ADO pull request (PR #79) via the REST API, authenticated as the SP — `createdBy.displayName: "idp-d1-kargo-writer"`, not the operator's account.

### Negative evidence (executed)

- Reader SP push to a new branch → **`TF401027: You need the Git 'GenericContribute' permission ... identity 'e9dbba09-...\23d3de7e-...'`** — denied by the SP's own ACL, not by absence of a valid credential.
- Writer SP direct push to protected `d1/desired-state` → **`TF402455: Pushes to this branch are not permitted; you must use a pull request`** — denied by branch policy.
- Strongest-available human identity (the org's own PCA/admin account) also attempted a direct push to `d1/desired-state` → **same `TF402455` denial.** This is the strongest available proof that the protected-branch policy is not bypassable by identity strength — not even the platform's own top-privilege human account can shortcut it.

### Residual gap (named, not hidden)

The live Argo repo-credential Secret (`argocd/d0-gitops-sandbox-repo`) and the live Kargo writer Secret (`d1-sandbox/d1-git-writer`) were **not** swapped to the new SP credentials in this session. Reason: Entra `client_credentials` tokens for these SPs are ~1 hour lived, and swapping the live secrets without an automated refresh mechanism (a CronJob re-minting and re-applying the token, as the accepted plan anticipated) would leave the sandbox's continuous reconciliation broken an hour after this session ends — a self-inflicted regression, not a hardening. This was a deliberate engineering choice to avoid leaving the sandbox in a worse state, not an oversight.

**Recommended next step:** build the token-refresh CronJob, swap both live secrets to the SP credentials, and re-run this same positive/negative suite against the live Argo/Kargo controllers (not just isolated proof) before treating identity separation as production-final.

## P1.2 — Argo/Kubernetes least privilege for `d1-prd`

**Score: PROVEN** — real Kubernetes RBAC denial, not AppProject-level, with a genuine platform constraint documented.

### What was built

- SA `d1-prd/argocd-prd-deployer` with a namespaced `Role` (not `ClusterRole`): write (`create/update/patch/delete`) limited to `apps/deployments` and `core/services`; read (`get/list/watch`) on all API kinds *within this namespace only* (see Finding 1 below for why).
- Registered as an Argo destination cluster secret `d1-prd-scoped`, server `https://10.43.0.1:443` (a distinct string from the reserved in-cluster alias, so Argo treats it as a separate destination identity), `namespaces: d1-prd`, `clusterResources: false`.
- `AppProject d1-sandbox` and `Application d1-prd` repointed to this destination (committed to Git — see P1.3).

### Finding 1 — Argo's cache layer needs broad read, not broad write (real platform constraint, not a design shortcut)

A `Role` limited strictly to the resource kinds the Application manages (`Deployment`, `Service`) is **not sufficient** for Argo to reach `Synced`. Argo's cluster-cache subsystem enumerates every API kind actually present in the destination's scope (namespace-wide, once `namespaces`/`clusterResources:false` is set) to build its resource tree and detect drift/orphans — in this shared sandbox cluster that includes unrelated CRDs (Traefik `IngressRouteTCP`, cert-manager, etc.) that have nothing to do with `d1-prd`'s workload. The first attempt with a strictly minimal Role produced a real, persistent `ComparisonError` (not a security failure — health stayed accurate, but sync state was legitimately unknown). This is exactly the "maturity must be verified" caveat the D0 review raised about scoped-destination topologies, now confirmed concretely rather than hypothetically.

The resolution — confirmed by testing, and explicitly approved by the operator before applying — is Argo's own documented least-privilege pattern: grant `get/list/watch` on all API groups/resources **within the namespace only**, while keeping `create/update/patch/delete` limited to the specific managed kinds. This is a **read/write asymmetry**, not a return to broad authority: mutation remains exactly as narrow as originally designed.

### Positive evidence (executed)

`d1-prd` Application: `Synced/Healthy`, revision `8a50c2e5c07a`, zero comparison errors, reconciling through the scoped credential exclusively.

### Negative evidence (executed, against the live token, in full isolation — see Finding 2)

| Attempt | Result |
|---|---|
| Patch Deployment in `d1-dev` (out-of-scope namespace) | `Forbidden: cannot list resource "deployments" ... in the namespace "d1-dev"` |
| Create `ClusterRoleBinding` binding self to `cluster-admin` (self-escalation) | `Forbidden: cannot create resource "clusterrolebindings" ... at the cluster scope` |
| List Pods in `kube-system` | `Forbidden: cannot list resource "pods" ... in the namespace "kube-system"` |
| Get Secrets in its **own** namespace `d1-prd` | `Forbidden: cannot list resource "secrets" ... in the namespace "d1-prd"` — write-scope stayed narrow even after the read-broadening fix |

### Finding 2 — a real methodology bug, self-caught

The first negative-test pass falsely showed the scoped SA had cluster-admin (successfully created a live `ClusterRoleBinding` to `cluster-admin`, read `kube-system`, read cross-namespace). Root cause: `kubectl --token=<sa-token>` was invoked against the operator's **ambient kubeconfig context**, which still carried a valid admin client-certificate; Kubernetes' authentication chain accepted the client-cert (admin) authenticator and ignored the bearer token entirely. The escalation `ClusterRoleBinding` created during this false test was deleted within the same command sequence, before any further action. All results in this document come from a **fully isolated** kubeconfig (`--kubeconfig=/dev/null` equivalent, single token-only user, no ambient credential) minted specifically for each identity under test. This is recorded because it is a real, easy-to-make testing mistake that would silently invalidate an RBAC negative-test suite if not caught.

## P1.3 — Govern the Argo control objects

**Score: PROVEN**

### What was built

- New `AppProject d1-control` + `Application d1-control-plane`, reconciling path `argocd/` on `d1/desired-state`, through a second scoped destination (`d1-control-scoped`, server `https://kubernetes.default.svc.cluster.local:443`, `namespaces: argocd`, `clusterResources: false`), SA `argocd/argocd-control-deployer` with write limited to `argoproj.io Application/AppProject` only (same read/write-asymmetry pattern as P1.2, applied to Argo's own control namespace — approved by the operator given it also grants read visibility into Argo's own Secrets in that namespace, which this SA still cannot write or delete).
- The `d1-prd` destination change (P1.2) and the new control-plane manifests were committed to Git (PR #80) rather than applied only imperatively — closing exactly the gap D0/D1 flagged: *"Application/AppProject sit outside Git; the realistic bypass — an actor editing the boundary objects — was never tested."*
- The previously-imperative `kargo.akuity.io/authorized-stage` annotations are now Git-managed fields on the same manifests.

### Positive evidence (executed)

`d1-control-plane` Application: `Synced/Healthy`, revision `8a50c2e5c07a`, self-governing (its own path includes its own Application/AppProject manifests — a standard, deliberate self-management bootstrap pattern).

**Drift-remediation proof:** an imperative admin patch was applied directly to `d1-prd`'s `syncPolicy.automated.selfHeal` (flipping it to `false`, simulating an out-of-band control-object edit). Observed reverted to the Git-declared value (`true`) within ~6 seconds by `d1-control-plane`'s own `selfHeal`, with no manual intervention. This is exactly the prevention-plus-remediation property required.

### Negative evidence (executed)

| Attempt | Identity | Result |
|---|---|---|
| Patch `Application d1-prd` (flip `selfHeal`/`prune`) | `squad-sandbox/squad-dev` (representative squad identity) | `Forbidden: cannot get resource "applications" ... in the namespace "argocd"` |
| Patch `AppProject d1-sandbox` (widen destinations to `*`/`*`) | `squad-sandbox/squad-dev` | `Forbidden: cannot get resource "appprojects" ... in the namespace "argocd"` |
| Patch `Application d1-prd` | `ado-agents/ado-agent` (real pipeline identity, post-remediation) | `Forbidden` |

Squad/pipeline identities have **no RBAC path** to these objects at all — denial happens before any AppProject-level policy is even consulted, which is the strongest possible form of this proof.

## P1.4 — Squad/pipeline bypass resistance

**Score: PROVEN**, including one **confirmed live bypass found and remediated** during this checkpoint.

### Finding 3 — a real, live, total bypass (found, proven, fixed)

`ClusterRoleBinding ado-agent-cluster-admin` bound the real self-hosted ADO agent's ServiceAccount (`ado-agents/ado-agent`) to `cluster-admin`. This was **not hypothetical**: using the SA's own live token (minted via a dedicated long-lived token Secret, not `kubectl create token` guesswork), the identity was confirmed able to:

- patch a production-like Deployment in `d1-prd` directly (`patch deployments`: `yes`)
- create a `ClusterRoleBinding` granting itself `cluster-admin` (self-escalation: `yes`, executed and immediately reverted)
- read Secrets in `kube-system` (`yes`)

Two additional stale `cluster-admin` `ClusterRoleBinding`s were found pointing at ServiceAccounts in namespaces confirmed absent (`azure-devops-agent`, `devops-agents`) — dangling escalation paths that would reactivate if either namespace/SA is ever recreated.

**Remediation (executed, with operator confirmation):** all three `cluster-admin` bindings deleted; `ado-agent` granted a narrow, representative `ClusterRole` (read-only `get/list/watch` on `pods/deployments/services/replicasets/events`, cluster-wide — a realistic "pipeline needs to observe status" grant, zero write).

**Re-proof, same real identity, same long-lived token, post-remediation:**

| Attempt | Before | After |
|---|---|---|
| Patch Deployment in `d1-prd` | `yes` (succeeded) | `no` — `Forbidden` |
| Create `ClusterRoleBinding` → `cluster-admin` | `yes` (succeeded, reverted) | `no` — `Forbidden` |
| Read Secrets in `kube-system` | `yes` (succeeded) | `no` — `Forbidden` |
| List Pods/Deployments (read-only, its intended use) | `yes` | `yes` (unchanged — the legitimate use case still works) |

### The five required bypass attempts

| # | Attempt | Identity | Result |
|---|---|---|---|
| 1 | Direct push to protected `d1/desired-state` | Writer SP; separately, the org's own PCA/admin human account | `TF402455` (both) |
| 2 | Argo reconciler credential writes Git | Baseline (pre-fix) `d0-argocd` PAT — **succeeded** (the confirmed baseline gap, see Baseline matrix); reader SP post-fix — `TF401027` denied | Baseline: bypass confirmed and closed. Post-fix: denied |
| 3 | Squad/pipeline SA mutates `d1-prd` Deployment | `squad-sandbox/squad-dev` (representative) and `ado-agents/ado-agent` (real) | Both `Forbidden` post-remediation; `ado-agent` was a **confirmed live bypass** pre-remediation |
| 4 | Squad/pipeline SA mutates Application/AppProject/Kargo Promotion | `squad-dev` and `ado-agent` | All three object types: `Forbidden` for both identities |
| 5 | ADO pipeline shortest bypass: unauthenticated call to Delivery dispatch API | n/a (no credential) | `401 Missing credentials` — denied before ever reaching the `CHANGE_REQUIRED` gate. ADO `platform-engineering Build Service` Git ACL independently confirmed `effectiveAllow=34` (Read+CreateTag only, no Contribute) |

No negative test was weakened or worked around to make it pass. Where a test surfaced a real bypass (Finding 3; the initial Argo-reader-write probe), it is reported as found, then fixed, then re-tested — not silently corrected in the record.

### Delivery backend's own Kubernetes identity (self-inflicted-bypass closure)

`KargoK8sProvider` previously used `kc.loadFromDefault()` — the ambient ~~kubeconfig~~, which on this operator's machine is cluster-admin. This meant the *governed service itself* was an unbounded bypass of everything else in this document. Rather than build a new SA, the existing Kargo-provided `kargo-promoter` SA (already scoped to `create/get/list/watch` on `Promotions`, read on `Stages/Freights/Warehouses` in `d1-sandbox` — exactly Delivery's actual need) was extended with one narrow additional Role (`get/list/watch` on `argoproj.io Applications` in `argocd`, needed for the Deployments-tab Argo sync/health projection) and wired via `app-config.yaml`'s new `delivery.kubernetes.kubeconfigPath`, pointing at a kubeconfig stored **outside the git repository** (`~/.backstage-delivery/`, not committed — an attempt to place it inside the repo's `config/` directory was correctly blocked by the session's own safety classifier as writing a live credential into a tracked path).

**Verified:** the new identity can create Promotions and read Applications; cannot patch Deployments, cannot patch Applications, cannot create ClusterRoleBindings. **Verified live:** after restarting the Backstage dev backend to load the new config, the Delivery plugin initialized cleanly and its first real request (the Deployments tab, from an actual browser session) succeeded using the new scoped credential — confirmed via structured logs, not merely by absence of an error.

**Incident during this step (fully resolved, no data lost):** restarting the backend process to pick up the new config did not cleanly respawn under its existing supervisor; the operator was informed immediately, and after one confirmed miscommunication about how to proceed, the dev stack (frontend + backend) was cleanly stopped and restarted with captured logs. No GitOps, ADO, or cluster state was affected by this — it was local to the Backstage dev process only.

## Backstage/UI evidence

Used only as supplemental regression evidence, never as an authorization proof, per the contract:

- Component → Deployments tab: loads live, `d1-dev`/`d1-hml`/`d1-prd` all render `Synced/Healthy`.
- The Deployments tab's Argo sync/health projection was independently confirmed to be sourced through the *new* scoped `kargo-promoter` credential (not the old ambient cluster-admin kubeconfig), via backend structured logs at the moment of the restart.
- GMUD/Change binding UI was not re-exercised in this checkpoint (no new PRD dispatch was run against a bound Change; P1 scope was authority hardening, not a new promotion cycle).

## Regression suite

**Not independently re-executed in this session.** `yarn workspace backend test --testPathPatterns="DeliveryService"` hung indefinitely (no jest worker process ever spawned) across two attempts, for reasons unrelated to this checkpoint's changes — isolated `jest` invocation (bypassing the `yarn workspace` wrapper) confirmed the harness needs `backstage-cli`'s own babel/TS config to parse the test files at all, so the hang is in the `yarn`/`backstage-cli` invocation layer itself, not a test failure. No Delivery or Change Management **source code** was modified in this checkpoint (the only implementation change is one new config field in `app-config.yaml`), so no regression is expected from this session's work — but this expectation was not independently re-confirmed by execution. **Recommended:** re-run the 18 DeliveryService + 4 tab-loader tests in a fresh shell/environment before treating this checkpoint's config change as fully verified.

## Required evidence matrix

| Property | Score | Positive evidence | Negative evidence | Residual gap |
|---|---|---|---|---|
| Git writer/reconciler separation | **PROVEN** (capability) | Real Entra SPs, distinct ADO identities, clone/push/PR-open all succeed as the SP, not the operator | Reader SP push denied (`TF401027`); writer direct-push-to-protected denied (`TF402455`), including for the org's own PCA account | Live Argo/Kargo secrets not yet swapped to these credentials; no automated token-refresh (tokens are ~1h) |
| Argo/K8s least privilege | **PROVEN** | `d1-prd` `Synced/Healthy` through scoped credential | 4/4 negative K8s RBAC denials (cross-namespace, cluster-escalation, `kube-system`, own-namespace Secrets) | Argo's cache architecture required broadening *read* (not write) beyond the initially-attempted minimal Role — a genuine platform constraint, documented and operator-approved, not a design compromise |
| Argo control-object governance | **PROVEN** | `d1-control-plane` self-governing, `Synced/Healthy`; drift-remediation proven live (~6s revert) | Squad and real pipeline identities: zero RBAC path to Application/AppProject | Both PRs carrying these manifests are technically proven but still awaiting merge approval |
| Squad/pipeline bypass resistance | **PROVEN** | Legitimate squad read-only use case still works post-remediation | All 5 required bypass attempts denied; 1 confirmed live bypass found and fixed (`ado-agent-cluster-admin` + 2 stale bindings) | None — this is the strongest-evidenced property in this checkpoint |

## Credential/identity inventory (no secrets exposed)

| Identity | Kind | Where it lives | Rotation/audit |
|---|---|---|---|
| `idp-d1-argocd-reader` | Entra SP, ~1h OAuth tokens | Not yet wired into a live cluster secret | Manual only — no CronJob built |
| `idp-d1-kargo-writer` | Entra SP, ~1h OAuth tokens | Not yet wired into a live cluster secret | Manual only — no CronJob built |
| `d1-prd/argocd-prd-deployer` | K8s SA, long-lived token Secret | `argocd/d1-prd-scoped` cluster secret | Standard K8s Secret rotation (manual in this sandbox) |
| `argocd/argocd-control-deployer` | K8s SA, long-lived token Secret | `argocd/d1-control-scoped` cluster secret | Same as above |
| `d1-sandbox/kargo-promoter` | K8s SA (pre-existing, Kargo-provided), long-lived token Secret | `~/.backstage-delivery/kargo-promoter.kubeconfig` (outside repo, not committed) | Same as above |
| `ado-agents/ado-agent` | Real ADO pipeline SA | Narrow `ado-agent-scoped` ClusterRole (post-remediation) | Managed by cluster operator |

## Residual blockers (for the next gate, not for P1 to solve)

1. Live Argo/Kargo Git credential swap to the two Entra SPs, plus an automated token-refresh mechanism — currently only proven in isolation.
2. Regression suite re-execution — environment issue (yarn hang) prevented re-running the 18+4 test suite this session.
3. Both GitOps PRs (#79, #80) remain in `active` status pending human approval — the control-plane governance and scoped-destination changes are live-proven on the cluster but not yet the durable, attributable Git record until merged.
4. Broader RBAC hygiene beyond what P1 required: the sandbox cluster has other pre-existing `cluster-admin` bindings not touched by this checkpoint (`helm-kube-system-traefik`, `helm-kube-system-traefik-crd`) — out of scope for P1 (unrelated to the governed Delivery path), noted for a future broad-RBAC-cleanup pass.

## Recommended next gate

A separate **production adoption review** deciding `GO` / `CONDITIONAL_GO` / `NO_GO` for production rollout, informed by this document's four `PROVEN` scores and three named residual gaps above. That review may also evaluate break-glass, rollback policy, audit retention, HA/DR, and supply-chain controls — none of which were touched in P1, per the contract's explicit exclusions.

**ADR-012 status is unchanged (Accepted, 2026-09-06). Production rollout is NOT declared GO by this document.**

## Final report

```text
P1 verdict: CONDITIONAL_PASS
Docs baseline SHA: e64435eac124b64c593436397c82795ae8fea481
Implementation/config SHAs: platform-devops-developer-portal@ee114cf (app-config.yaml delivery.kubernetes.kubeconfigPath added, uncommitted at report time)
GitOps: d0-gitops-sandbox PR #79 (writer-identity proof), PR #80 (P1.2/P1.3 control-plane governance) — both merge-clean, awaiting human approval

Git identity separation: PROVEN (capability + positive/negative execution); live-secret wiring + refresh automation is a named residual
Argo/K8s least privilege: PROVEN (real RBAC denial, 4/4 negative tests); Argo's own cache architecture required a documented, approved read-broadening
Argo control-object governance: PROVEN (self-governing, drift-remediation proven live, zero squad RBAC path)
Squad/pipeline bypass resistance: PROVEN; 1 confirmed live bypass (ado-agent cluster-admin + 2 stale bindings) found and remediated in this checkpoint

Positive governed path: d1-dev/d1-hml/d1-prd/d1-control-plane all Synced/Healthy through their respective scoped credentials
Negative bypass suite: all 5 required attempts denied; every denial captured with its real error (TF402455, TF401027, Kubernetes Forbidden, HTTP 401)
Backstage UI/browser verification: Deployments tab confirmed live via the new scoped Delivery credential (structured logs, not UI success alone)

Residual blockers: live Argo/Kargo secret swap + token refresh automation; regression suite re-execution; 2 PRs awaiting merge approval; unrelated pre-existing cluster-admin bindings out of P1 scope
Recommended next gate: production adoption review (separate, explicit authorization)
Docs updated: docs/delivery/p1-production-authority-hardening.md (new), docs/delivery/README.md, prompts/README.md
STOP
```
