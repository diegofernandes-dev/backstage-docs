# Pre-Rollout Condition Closure — Delivery / GitOps

## 1. Canonical baseline SHA

| Item | Value |
|---|---|
| Execution contract | [`prompts/pre-rollout-condition-closure.md`](../../prompts/pre-rollout-condition-closure.md) |
| Canonical docs (`diegofernandes-dev/backstage-docs`, `origin/main`) | `6c634d1be90a587f4c1ee5399c2109d58808ad25` — fetched fresh at the start of this checkpoint (2 commits ahead of the review baseline `f20eca1`: added this prompt) |

## 2. Implementation / GitOps / live baseline

| Item | Value |
|---|---|
| Implementation (`platform-devops-developer-portal`) | `b08e7b284f34e5d12c299b4ee6ab75697e21010f` on `feat/delivery-mvp-slice` at checkpoint start; **`170af45dc011e0e0f19ed6e7444e46794e32daaf` after** — one config-only commit made this checkpoint (`app-config.production.yaml`, see §5/C2), pushed directly per this repository's established direct-commit convention on this branch (no source code changed) |
| GitOps repo | `diegolab/platform-engineering/d0-gitops-sandbox`, governed branch `d1/desired-state` @ `50564c9977e1a02b7d16345c9ebcc38416986efb` at checkpoint start — unchanged; new content proposed via PR #82 (not yet merged) |
| Cluster | Rancher Desktop k3s v1.33.6+k3s1, context `rancher-desktop` (live-inspected, unchanged) |
| Argo CD | v3.5.2 (live, unchanged) | Kargo | v1.11.4 (live, unchanged) |
| ADO org | `diegolab`, project `platform-engineering`, Entra tenant `e9dbba09-e7a3-42be-9a2c-f82470024e00` (unchanged) |

**Canonical-vs-live conflict found before any change (per contract §1):** `p1-residual-closure.md:80` states the token-refresh manifests were "committed to `d0-gitops-sandbox` at `bootstrap/token-refresh/`". Independently re-verified via `git ls-tree -r` and `git log --all -- 'bootstrap/*'` against every ref of `d0-gitops-sandbox`: **zero hits**. This confirms the Production Adoption Review's own finding (§3.1 of that document) rather than contradicting it — the claim in `p1-residual-closure.md` was false when written and remains false as of this checkpoint's start. See §4 (C1) for the fix and §9 for the corrective note added to the historical document.

## 3. Actual first-rollout target inventory (Phase A)

Performed strictly per contract §3 before touching C2 or C5.

| Question | Finding |
|---|---|
| Which Backstage/Delivery backend runtime will issue the first real production promotion? | **None identified.** The backend runs via `yarn dev` on the operator's laptop (`http://localhost:7007`, confirmed live: `200` on `/`, `401` on the authenticated `/api/notifications` route as expected without a session). No Dockerfile, Helm chart, Kubernetes `Deployment`/`Pod` spec, or any deployment artifact for the Backstage backend exists anywhere in the repository (`find . -iname '*helm*' -o -iname '*deploy*' -o -iname 'Dockerfile*'` — none found) or in the cluster (no workload in any namespace resembles a Backstage backend Pod). |
| Which Kubernetes cluster/account hosts the first production target? | **None identified.** Only the single Rancher Desktop k3s cluster exists, the same one hosting `d1-dev`/`d1-hml`/`d1-prd` sandbox namespaces. `docs/delivery/d0-architecture-review.md:129` states explicitly: "D1 remains sandbox-only: no production cluster, namespace, credential, rollout, GMUD integration, Delivery implementation, or Backstage implementation." |
| Which production namespace? | **None identified.** `d1-prd` is the sandbox's scoped-destination-pattern namespace, not a designated production namespace on a separate account/cluster as required by the authorized envelope (`production-adoption-review.md` §11: "the cluster/account hosting that first real production target; do not simultaneously onboard a second cluster/account"). |
| Which GitOps repository/branch is the source of truth for that target? | **None identified.** Live ADO org inventory (`az repos list --project platform-engineering`): `platform-helm-charts`, `platform-engineering` (itself), `d0-gitops-sandbox`, `platform-gitops`, `platform-terraform`, `platform-pipeline-templates`. `platform-gitops` was investigated specifically (its name suggested a candidate): it is the repository backing the **negative-control** Argo Application `d0-negative-repo` (confirmed via `kubectl -n argocd get applications -o jsonpath` showing `d0-negative-repo`'s `repoURL` = `platform-gitops`) — i.e. it exists specifically to prove Argo correctly refuses an unauthorized source repo, not as a production candidate. |
| Is the real production GitOps repo the same `d0-gitops-sandbox` or different? | **Not designated either way.** No canonical doc or live config names `d0-gitops-sandbox` as the intended first-rollout source of truth; it is documented everywhere as the sandbox proving ground. |
| Which Entra tenant / ADO org owns the real production Git repo? | **Not determinable** — follows from the above; no production repo means no production tenant/org binding to verify. |

### Anti-fabrication conclusion

Per contract §3: no production cluster, namespace, GitOps repo, tenant, or Delivery-backend runtime is invented in this checkpoint. **C2 and C5 are `BLOCKED`** as the contract requires when the target is unidentified. C1, C3, and C4 do not depend on this target and are pursued in full below.

## 4. C1 — Durable token-refresh manifests

**Status: `IN_PROGRESS`** (implementation complete and live-verified; merge pending independent human review, required by branch policy 103 which forbids creator self-approval).

**What was done:** The exact live objects — `Namespace`, `ServiceAccount`, three narrowly-scoped `Role`/`RoleBinding` pairs, the `refresh.sh` `ConfigMap`, and the `*/30 * * * *` `CronJob` — were reconstructed as non-secret YAML in `bootstrap/token-refresh/` and committed to a short-lived branch, `c1-c4/token-refresh-manifests-and-alert`, off `d1/desired-state`.

**Secret-free verification:**
- `diff` between the live `ConfigMap/idp-token-refresh-script`'s `refresh.sh` and the committed version: **byte-identical** (confirmed before any edit).
- `git diff --cached | grep -iE 'client-secret|password|token|Bearer'` on the full commit: every match is a variable name, a documented placeholder in the bootstrap README (`<reader-sp-client-secret>` etc.), or a live-pattern reference to the pod's own `ServiceAccount` token file path (`/var/run/secrets/kubernetes.io/serviceaccount/token`) — the same non-secret pattern already proven safe in the live script. No literal secret value is present.
- The one Secret the mechanism depends on (`idp-token-refresh/sp-credentials`) is referenced by key name only in the README, never by value.

**Live re-application and functional proof (before opening the PR):**
- `kubectl apply -f` against all six manifest files: clean apply, no drift from live state (`unchanged`/`configured` for pre-existing objects).
- Manual job triggered from the (unmodified-by-this-change) `CronJob/idp-token-refresh`: `Complete`, log `refresh complete: reader + writer tokens rotated at 2026-09-07T03:44:19Z`.
- A second manual trigger after all C4 changes (below) also landed: `Complete`, `refresh complete: reader + writer tokens rotated at 2026-09-07T03:46:21Z`.

**PR:** [`d0-gitops-sandbox` PR #82](https://dev.azure.com/diegolab/platform-engineering/_git/d0-gitops-sandbox/pullrequest/82), `c1-c4/token-refresh-manifests-and-alert` → `d1/desired-state`, created by the human operator's own ADO session (`createdBy: Diego Fernandes`). **Not self-approved.** Per branch policy 103 (`minimumApproverCount:1`, `creatorVoteCounts:false`), this PR requires an independent reviewer's approval before merge — this checkpoint does not and cannot supply that approval itself.

**PASS evidence still outstanding:** PR #82 merged; `git ls-tree` at the merged revision confirms `bootstrap/token-refresh/*` present on `d1/desired-state`. Until merge, C1 = `IN_PROGRESS`, not `PASS`.

**Corrective note added:** see §9.

## 5. C2 — Production-appropriate Delivery Kubernetes identity

**Status: `BLOCKED`** — per contract §5: "If the actual production backend runtime does not yet exist or is not identified, C2 = BLOCKED; do not fake proof using the operator laptop." No such runtime exists (§3). No positive/negative authority test was run against a fabricated target, and none is claimed.

**Bounded partial remediation performed (does not close C2):** `app-config.production.yaml` previously had no `delivery:` block at all, meaning production configuration silently inherited `app-config.yaml:252`'s laptop path (`/Users/diegofernandes/.backstage-delivery/kargo-promoter.kubeconfig`) by virtue of Backstage's config-layering. Added an explicit `delivery.kubernetes.kubeconfigPath: ${DELIVERY_KUBERNETES_KUBECONFIG_PATH}` to `app-config.production.yaml` (env-var-driven, no default) so that:
- production configuration no longer silently resolves to any individual's filesystem;
- production startup fails closed (unset env var) rather than falling back to the laptop path;
- the local developer override in `app-config.yaml` remains clearly separated and unchanged.

This is recorded as **necessary groundwork, not evidence of C2 closure**. No non-human credential is provisioned, and contract §5's required positive/negative authority tests cannot be run against a runtime that does not exist.

**File changed:** `app-config.production.yaml` (this repository, `platform-devops-developer-portal`).

## 6. C3 — Rollback / emergency runbook

**Status: `IN_PROGRESS`** (document written and PR opened; human review pending — the contract requires review by someone other than the author before `PASS`).

**What was done:** `docs/delivery/rollback-emergency-runbook.md` added, covering:
- trigger classification (Git revert vs. Argo reconciliation issue vs. Kargo/Delivery credential/promotion issue vs. application-runtime incident), with a concrete `kubectl` discriminator;
- the bounded Git rollback path (identify known-good revision → revert branch → PR → independent approval under policy 103 → merge → observe `selfHeal`/`prune` convergence → verify), explicitly excluding `kubectl patch`, `argocd app sync` overrides, force-push, or policy bypass as routine steps;
- named roles: rollback initiator (any platform-team member), required approver (the platform owner, same identity proven independently approving PR #81), platform owner;
- **one gap named explicitly rather than fabricated:** no second approver/escalation contact exists today beyond the single named platform owner — recorded as a pending action, not invented;
- the break-glass boundary (exceptional, attributable, followed by restoration of Git authority) described but not built as new automation, consistent with contract §6's explicit statement that this checkpoint does not need to build break-glass tooling.

**PR:** [`backstage-docs` PR #1](https://github.com/diegofernandes-dev/backstage-docs/pull/1), branch `docs/rollback-emergency-runbook` → `main`. Opened specifically as a review request; not merged by this checkpoint pending that review.

**PASS evidence still outstanding:** independent human approval/review, then merge.

## 7. C4 — Token-refresh failure alert

**Status: `PASS`.**

**Design:** No monitoring stack exists; `argocd-notifications-controller` is running but is `Application`-scoped and structurally cannot watch a `CronJob` (confirmed: empty `ConfigMap`, zero `Application` subscriptions, unchanged this checkpoint). Rather than deploy a new monitoring platform (explicitly forbidden by contract §0), the alert destination is an ADO `Task` work item in `diegolab/platform-engineering`, raised via an Entra token minted from credentials the mechanism already holds.

**ADO permission change (narrow, additive, verified non-overlapping with Git authority):**
- Granted the writer SP (`idp-d1-kargo-writer`) a `WORK_ITEM_READ|WRITE` ACE (bit `48`) on the ADO `CSS` security namespace, token `vstfs:///Classification/Node/199945ea-c8c6-47d2-87af-b0ee1f4ff04d` (the project's root Area Classification Node). Before this grant, the SP had no explicit permission there (`allow=0, deny=0`, inherited only).
- **No ADO group membership was granted** (would have widened `Contribute` beyond what's needed).
- **Git ACL bit-identity proven, before and after:** captured `allow=16406 deny=32904` on the `Git Repositories` namespace / `d0-gitops-sandbox` token immediately before the CSS grant, and again immediately after — `diff` of the two JSON captures: **identical**. The proven P1/production-review ACL split is unweakened.

**Mechanism (both committed alongside C1, in `bootstrap/token-refresh/`):**
1. **In-script trap** — `refresh.sh`'s existing `FATAL: reader/writer token mint failed` branches now additionally source `alert-lib.sh` and call `post_ado_alert` before `exit 1`.
2. **Independent watchdog `CronJob`** (`idp-token-refresh-watchdog`, `*/15 * * * *`) — reads `cronjob/idp-token-refresh`'s own `status.lastSuccessfulTime` via a Role scoped to `get` on that one named `CronJob`, and alerts if the last success exceeds a staleness threshold (5400s / 90 min — two missed 30-min cycles, the exact window the Production Adoption Review named as when both credentials go stale before the ~1h token expiry). This is what catches failures the in-script trap structurally cannot: a pod that never starts, an image-pull failure, or a failure during the `apk add --no-cache curl jq` step the review flagged as a public-CDN dependency — none of which ever reach `refresh.sh`'s own code.
3. **Shared `alert-lib.sh`** (`ConfigMap`) — mints its own short-lived Entra token from the writer SP's existing bootstrap credentials and files one ADO `Task`. No token or secret value is ever echoed/logged; only success/failure of the mint and post is logged.

**Live proof (all performed without touching the live SP credentials or letting live tokens expire, per contract §7):**

| Test | Result |
|---|---|
| Normal successful refresh does not alert | Ran the real, unmodified `CronJob` manually before any test-only artifact existed: `Complete`, no ADO work item created (`az boards query` for Title-contains-`token-refresh` Tasks: empty result) |
| Deliberate in-script failure fires the alert | Created an isolated test-only `ConfigMap`/`Job` (`idp-token-refresh-script-failtest` / `idp-token-refresh-failtest`) with a hardcoded empty reader token — **did not modify or touch** the live `d0-gitops-sandbox-repo`/`d1-git-writer` Secrets. Job: `Failed` (expected — `backoffLimit: 0`, by design for the test). Log: `FATAL: reader token mint failed` → `alert raised: TEST-ONLY token-refresh: reader mint failed`. Confirmed landed: ADO work item **#1**, title `TEST-ONLY token-refresh: reader mint failed`, created `2026-09-07T03:45:03.997Z`, description contains only a timestamp and "Not a real incident" — no secret material |
| Watchdog reports healthy against real state, no alert | Manually triggered `idp-token-refresh-watchdog`: `Complete`, log `WATCHDOG: healthy — last success 75s ago`, no new work item |
| Watchdog alerts on staleness | Ran an isolated test-only `Job` using the real watchdog script with only `STALE_AFTER_SECONDS` forced to `0` (via `sed`, applied at container-command time, not committed) against the real, unmodified `lastSuccessfulTime` — did not alter the live `CronJob` or its schedule. Log: `WATCHDOG: last successful refresh was 93s ago (threshold 0s)` → `alert raised: token-refresh: stale — no success in 1 minutes` |
| Real CronJob still completes after all changes | Final manual trigger of the real, unmodified `idp-token-refresh` `CronJob` post-cleanup: `Complete`, `refresh complete: reader + writer tokens rotated at 2026-09-07T03:46:21Z` |
| No secret material in any alert | Both fired alerts' bodies inspected via `az boards work-item show --id 1`: title/description contain only identity names, timestamps, and a runbook cross-reference — no token, no client secret |
| Alert path does not require an interactive human session | Both alert-raising `Job`s ran as in-cluster `ServiceAccount`s (`idp-token-refresh`, `idp-token-refresh-watchdog`) with no human credential in the path, mirroring the refresh mechanism itself |

**Cleanup:** all test-only `Job`/`ConfigMap` objects (`idp-token-refresh-manual-test1`, `idp-token-refresh-failtest`, `idp-token-refresh-script-failtest`, `idp-token-refresh-watchdog-manual-test1`, `idp-token-refresh-watchdog-staletest`, `idp-token-refresh-postcleanup-check`) deleted after use. ADO work item #1 (the one real test alert) was closed (state `Closed`) rather than deleted, preserving an auditable record that the test occurred without leaving an open/actionable item behind.

**Named limitation (documented in the bootstrap README and the runbook):** both SP client secrets expire on the same date, `2027-09-07`. If both are simultaneously the failure cause, the alert path — which mints its own token from the same credential pair — cannot authenticate either, and this would need to be diagnosed manually. This is outside the first-rollout window but is recorded, not hidden.

## 8. C5 — Real production Git reader/writer SP + branch-policy equivalence

**Status: `BLOCKED`** — per contract §8: "If no actual production GitOps repository has been identified... C5 = BLOCKED. Do not create a random 'prod' repository merely to make this checkpoint green." Per §3 above, no production GitOps repository is designated. `platform-gitops` was investigated and ruled out (it is the negative-control repo, not a candidate).

**Carried evidence only (explicitly not claimed as C5 `PASS`, since it is evidence against a repo not confirmed to be the production target):** the existing model on `d0-gitops-sandbox` was re-verified live this checkpoint as part of Phase B (§10):
- reader SP push denied (`TF401027`, live rotated token);
- writer SP direct push to `d1/desired-state` denied (`TF402455`, live rotated token);
- writer SP branch push + PR creation succeeded (PR #82, this checkpoint);
- branch policy 103 confirmed live: `minimumApproverCount:1`, `creatorVoteCounts:false`, blocking, enabled;
- writer SP's Git ACL bits (`allow=16406 deny=32904`) confirmed unchanged by the unrelated C4 CSS grant (§7).

This is the same authority model P1 and the Production Adoption Review already proved on this repository. It is recorded here as currency, not as C5 closure, because the repository it was proven against has not been designated as the first-rollout source of truth.

## 9. Secrets-handling statement

- No Kubernetes `Secret` data, client secret value, OAuth token, or kubeconfig was printed, logged, committed, or pasted into any PR/commit/doc during this checkpoint.
- Every `kubectl get secret` invocation used `-o jsonpath` targeting a single named field, immediately piped to `base64 -d` and consumed locally (e.g. to construct a throwaway test kubeconfig using a freshly-minted `ServiceAccount` token, or to diff a ConfigMap's script content) — never displayed as a full object dump.
- The committed diff (`bootstrap/token-refresh/*`) was explicitly grepped for secret-shaped strings before commit; all matches were variable names, ADO/Kubernetes-standard token-file paths, or documented placeholders (§4).
- The two ADO ACL captures (`/tmp/writer_git_acl_before.json`, `/tmp/writer_git_acl_after.json`) contain only permission bitmasks and identity descriptors, never credential values; they were not committed anywhere durable and existed only as local scratch files for this session.
- No secret was accidentally displayed during this checkpoint (unlike the one incident recorded and remediated in `p1-residual-closure.md`, which remains accurately documented there, unchanged).

## 10. Authority regression matrix (Phase B)

All checks re-run against final live state, each with a freshly minted, single-purpose, token-only kubeconfig (never the ambient admin context).

### Kubernetes

| # | Identity | Attempt | Result |
|---|---|---|---|
| 1 | `d1-prd/argocd-prd-deployer` | list Deployments in `d1-dev` (cross-namespace) | **DENIED** |
| 2 | `d1-prd/argocd-prd-deployer` | create `ClusterRoleBinding` (self-escalation) | **DENIED** |
| 3 | `d1-prd/argocd-prd-deployer` | list Secrets in `kube-system` | **DENIED** |
| 4 | `d1-sandbox/kargo-promoter` | create Promotion in `d1-sandbox` | **ALLOWED** (intended scope) |
| 5 | `d1-sandbox/kargo-promoter` | patch Deployment in `d1-prd` | **DENIED** |
| 6 | `d1-sandbox/kargo-promoter` | patch Application in `argocd` | **DENIED** |
| 7 | `d1-sandbox/kargo-promoter` | create `ClusterRoleBinding` | **DENIED** |
| 8 | `d1-sandbox/kargo-promoter` | get Secrets in `d1-sandbox` | **DENIED** |
| 9 | `squad-sandbox/squad-dev` | patch Deployment in `d1-prd` | **DENIED** |
| 10 | `squad-sandbox/squad-dev` | get Application in `argocd` | **DENIED** |
| 11 | `squad-sandbox/squad-dev` | create Promotion in `d1-sandbox` | **DENIED** |
| 12 | `ado-agents/ado-agent` (real pipeline identity) | patch Deployment in `d1-prd` | **DENIED** |
| 13 | `ado-agents/ado-agent` | get AppProject in `argocd` | **DENIED** |
| — | `ado-agent-cluster-admin` `ClusterRoleBinding` | presence check | **Confirmed absent** (`NotFound`) |
| — | `azure-devops-agent`, `devops-agents` namespaces | presence check | **Confirmed absent** (`NotFound`, both) |

### Git (ADO), live rotated SP tokens

| # | Identity | Attempt | Result |
|---|---|---|---|
| 14 | `idp-d1-argocd-reader` | push new branch to `d0-gitops-sandbox` | **DENIED** — `TF401027: You need the Git 'GenericContribute' permission…`, identity confirmed = reader SP's descriptor |
| 15 | `idp-d1-kargo-writer` | direct push to `d1/desired-state` | **DENIED** — `TF402455: Pushes to this branch are not permitted; you must use a pull request…` |
| 16 | `idp-d1-kargo-writer` | branch push + PR creation | **ALLOWED** (PR #82, this checkpoint's C1/C4 change) |

### CSS (work-item) ACE isolation check

| # | Check | Result |
|---|---|---|
| 17 | Writer SP Git ACL, `Git Repositories` namespace, `d0-gitops-sandbox` token — captured before and after the C4 `WORK_ITEM_READ\|WRITE` grant | **Bit-identical**: `allow=16406 deny=32904` both times |

No test created a durable escalation object; the one `ClusterRoleBinding` self-escalation attempt (#2/#7) was rejected server-side, never created. No isolated probe branch was left pushed to any remote (the two Git-side denial tests failed before any ref was created remotely).

## 11. Functional regression results

Implementation SHA before this checkpoint: `b08e7b284f34e5d12c299b4ee6ab75697e21010f`; after: `170af45dc011e0e0f19ed6e7444e46794e32daaf` (one config-only commit, `app-config.production.yaml`, §5; no source code changed). Regressions below were re-run and confirmed green after this commit landed.

```text
cd packages/backend && CI=true yarn test \
  src/modules/delivery/DeliveryService.test.ts \
  src/modules/changeManagement/authorization/EligibilityService.test.ts \
  --no-coverage --watchAll=false
→ 2 suites, 32 passed, 32 total (0.17s)

cd packages/app && CI=true yarn test \
  src/modules/catalogEntityTabs/DeploymentsTab.test.tsx \
  src/modules/catalogEntityTabs/index.test.ts \
  --no-coverage --watchAll=false
→ 2 suites, 8 passed, 8 total (5.2s)
```

Both suites green, both re-run after the `app-config.production.yaml` change, using the documented direct invocation (the `yarn workspace backend test` wrapper is not used, per the known hang recorded in `p1-residual-closure.md`).

## 12. Live controller health after changes

| Check | Result |
|---|---|
| Token refresh succeeding | Real `idp-token-refresh` `CronJob` completed successfully both before and after all C1/C4 changes (`refresh complete: reader + writer tokens rotated at 2026-09-07T03:44:19Z` and again `…03:46:21Z`) |
| Argo applications | `d0-sandbox`, `d1-control-plane`, `d1-dev`, `d1-hml`, `d1-prd` all `Synced`/`Healthy` at revision `50564c99…`, unchanged throughout this checkpoint (the negative-control `d0-negative-namespace`/`d0-negative-repo` remain `Unknown`/`Unknown` by design, unchanged) |
| Kargo stages | `dev`, `hml`, `prd` all `Healthy`, `lastPromotion.status.phase: Succeeded` |
| Watchdog CronJob | `idp-token-refresh-watchdog` present, `*/15 * * * *`, healthy scheduled runs observed |
| No human PAT re-entered the live Git controller path | Both live Secrets' `username` fields remain the non-human SP identity markers (`idp-d1-argocd-reader`, `idp-d1-kargo-writer`); nothing in this checkpoint touched either Secret's identity, only rotated the token value via the unmodified refresh mechanism |
| No broad cluster-admin binding reintroduced | `ado-agent-cluster-admin`: confirmed `NotFound` (unchanged) |

## 13. Files / repos / resources changed

| Repo | Change |
|---|---|
| `d0-gitops-sandbox` | Branch `c1-c4/token-refresh-manifests-and-alert` (7 new files under `bootstrap/token-refresh/`), PR #82 opened against `d1/desired-state` — **not yet merged** |
| Live cluster (`rancher-desktop`) | Applied (via `kubectl apply`, matching the committed manifests): `ConfigMap/idp-token-refresh-alert-lib` (new), updated `ConfigMap/idp-token-refresh-script` (added alert trap), updated `CronJob/idp-token-refresh` (projected volume for the alert-lib mount), new `ServiceAccount`/`Role`/`RoleBinding`×2/`ConfigMap`/`CronJob` for `idp-token-refresh-watchdog` |
| ADO (`diegolab`) | Granted `idp-d1-kargo-writer` a `WORK_ITEM_READ\|WRITE` ACE (bit 48) on the `CSS` namespace, project root Area node — verified not to affect any Git ACL |
| ADO work items | Created work item #1 during the C4 failure test (title `TEST-ONLY token-refresh: reader mint failed`); closed after verification, not deleted |
| `platform-devops-developer-portal` | `app-config.production.yaml` — added explicit `delivery.kubernetes.kubeconfigPath` (env-var-driven, no default), §5. Committed directly to `feat/delivery-mvp-slice` at `170af45` (this repository's established convention — no branch-policy PR gate exists on this branch, unlike the ADO GitOps repo) |
| `backstage-docs` | `docs/delivery/rollback-emergency-runbook.md` added on branch `docs/rollback-emergency-runbook`, PR #1 opened against `main` — **not yet merged**; this document and the `p1-residual-closure.md` corrective note (below) added directly to `main`, consistent with this repo's established pattern (no branch protection exists on `backstage-docs`, and every prior evidence checkpoint in this workstream was committed the same way) |

## 14. Human approvals still pending

1. **PR #82** (`d0-gitops-sandbox`, C1/C4 manifests) — needs one independent ADO reviewer's approval (branch policy 103: creator self-approval does not count). Owner: platform team / the operator's next available reviewer.
2. **PR #1** (`backstage-docs`, C3 runbook) — needs review/approval by at least one human other than the author. Owner: platform team.
3. Naming a second rollback approver/escalation contact (`rollback-emergency-runbook.md` §3) — a real, currently-unfilled gap, owner: platform owner, not scheduled to a date.

## 15. Deviations / limitations

- **C2 and C5 are `BLOCKED`, not `PASS` or `CONDITIONAL_PASS` individually** — no first-rollout target exists to close them against. This is the expected, contract-mandated outcome (§3/§5/§8), not a deviation from the plan.
- The C4 alert mechanism shares its credential pair with the refresh mechanism it monitors — a named, accepted limitation (§7), not a defect: building a fully independent credential path for the alert channel alone would be the "new credential platform" contract §0 explicitly forbids.
- The watchdog's staleness test used a `sed`-modified in-memory copy of the real script (threshold forced to 0) rather than waiting 90 real minutes for a live threshold breach; this proves the alert-firing code path correctly, while the *unmodified* threshold value (`5400`) is what is actually committed and applied live.

## 16. Final checkpoint verdict

```text
Pre-rollout condition closure: CONDITIONAL_PASS
```

C1: `IN_PROGRESS` (implementation complete, live-verified, PR #82 open, human merge approval pending).
C2: `BLOCKED` (no first-rollout Delivery backend runtime identified; partial config separation done as groundwork only).
C3: `IN_PROGRESS` (runbook written, PR #1 open, human review pending).
C4: `PASS` (implemented, applied live, both positive and negative alert paths proven with isolated test artifacts, no live credential touched, secrets handling clean).
C5: `BLOCKED` (no first-rollout GitOps repository identified; existing sandbox model re-verified live as carried evidence only).

```text
Ready for final rollout-readiness re-review: NO
Production rollout executed: NO
```

Two of five conditions remain gated on human PR review/approval (a bounded external dependency, exactly the kind contract §10 names as legitimate for `CONDITIONAL_PASS`), and two remain `BLOCKED` on an undesignated first-rollout target — not fabricated to force a pass. No architecture boundary was weakened; every negative-authority check that passed before this checkpoint still passes identically after it.

## 17. Production rollout executed: NO

## 18. STOP

This checkpoint stops here. The next authorized step — a separate final rollout-readiness re-review — requires: (a) PR #82 and PR #1 merged with genuine independent human approval, (b) the first-rollout target (backend runtime, cluster/namespace, GitOps repo) explicitly identified by the operator so C2 and C5 can be attempted for real, and (c) separate, explicit user authorization to begin that re-review. No production rollout, application choice, GMUD creation, real promotion, observation period, or post-rollout hardening was started or is authorized by this document.
