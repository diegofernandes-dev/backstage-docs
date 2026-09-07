# Production Adoption Review — Delivery / GitOps

## 1. Review baseline

| Item | Value |
|---|---|
| Review contract | [`prompts/production-adoption-review.md`](../../prompts/production-adoption-review.md) |
| Canonical docs (`diegofernandes-dev/backstage-docs` `origin/main`) | `9649879574f52b530a5e436080f8e1d8b157b7ae` — re-fetched and re-verified unchanged at review close |
| Implementation baseline (`platform-devops-developer-portal`, `feat/delivery-mvp-slice`) | `b08e7b284f34e5d12c299b4ee6ab75697e21010f` — confirmed via `git rev-parse HEAD`, matches all canonical docs |
| Governed GitOps revision (`diegolab/platform-engineering/d0-gitops-sandbox`, branch `d1/desired-state`) | `50564c9977e1a02b7d16345c9ebcc38416986efb` — confirmed via ADO API directory listing (26 items) and via all four live Argo `Application.status.sync.revision` |
| Cluster | Rancher Desktop k3s v1.33.6+k3s1, context `rancher-desktop` (live-inspected) |
| Argo CD | v3.5.2 (live) | Kargo | v1.11.4 (live) |
| ADO org | `diegolab`, project `platform-engineering`, Entra tenant `e9dbba09-e7a3-42be-9a2c-f82470024e00` |

**Live-state inspection performed (read-only):** `kubectl get/auth can-i` against Argo Applications, AppProjects, ClusterRoles/RoleBindings, CronJob/Job history, Kargo Stages/Promotions/Warehouse, Secrets (identity fields only, no secret values read or printed); ADO branch-policy and repository-permission queries via `az devops security permission show` and the ADO MCP tools (PR #81 detail, branch list, repository tree at the governed revision, org-wide code search); `az ad app credential list` for SP client-secret expiry (dates only); local + remote Git history scan of `d0-gitops-sandbox` across every ref; two backend/frontend regression suites re-executed from the current working tree.

**Not independently re-executed (relied on prior evidence):** the live dual-Freight PRD Git-conflict drill (E1/D1 evidence only); a fresh negative-authority run against the ADO Build Service Git ACL (unchanged since P1, not independently re-queried this checkpoint — same carried-forward gap p1-residual-closure.md already named). No mutating action was taken to manufacture evidence for either.

**Unavailable sources:** none. Every canonical doc in the mandated reading list was read; the GitOps repo, ADO org, and cluster were all reachable.

---

## 2. Executive verdict

```text
Production adoption review: CONDITIONAL_GO
ADR-012 architecture status: ACCEPTED
Authorized rollout scope: One non-critical/medium-criticality component, one production namespace on the existing scoped d1-prd destination, platform-owner-attended, 2-week observation window before any second workload is onboarded.
```

---

## 3. Why this verdict

Ordered by residual risk:

1. **The token-refresh manifests were never committed to Git**, despite `p1-residual-closure.md` stating they were. Independently confirmed via ADO API tree listing, full local+remote ref/history scan, and org-wide code search (all three: zero hits). The only copies are live cluster objects plus untracked files on a since-deleted local branch. This is exactly the "bootstrap state can silently regress" failure the contract calls out, applied to the credential-refresh mechanism itself — condition, not blocker, because the *live* mechanism is proven working and running unattended (verified: three consecutive unattended cycles during this review, 02:00/02:30/03:00 UTC).
2. **Refresh failure is completely silent.** No monitoring stack exists (confirmed: no Prometheus/Grafana/Alertmanager/Loki); Argo's own `argocd-notifications-controller` runs but is inert (empty ConfigMap, zero Application subscriptions). Combined with `backoffLimit: 1` and a public-CDN `apk add` dependency on every run, two consecutive failures (60 min) exceed the ~1h token lifetime with no alert firing.
3. **Both SP client secrets expire on the same calendar date, 2027-09-07**, with no reminder mechanism. Outside the first-rollout window but a real, dateable future blocker if not tracked.
4. **Kargo Stage/Promotion CRs (including the no-op-guard fix) are imperative, ungoverned cluster-only state** — not in Git, no backup, no drift detection.
5. **No written rollback/break-glass procedure exists**, though the underlying mechanism is sound (`d1-prd` runs `selfHeal:true, prune:true`, so a reverted+merged Git commit converges automatically without further action).
6. Architecture boundary held under live inspection: Change Management authorizes via HTTP eligibility check, Delivery's `claimDispatch` enforces atomic same-target exclusivity before any provider call, Git remains desired-state authority, Kargo's PR-gated write path cannot bypass ADO branch policy.
7. Authority separation is real and independently re-verified, not merely re-read from docs: ADO ACLs pulled live via `aadsp.` descriptors show the writer SP has `ForcePush` and both policy-bypass bits explicitly `Deny`; the reader SP is read-only (`Contribute: Deny`). PR #81 was confirmed via the ADO API to be created by the non-human SP identity and approved by an independent human reviewer under live branch policy 103.
8. Squad/pipeline bypass resistance re-confirmed live: `ado-agent-cluster-admin` binding is `NotFound`; representative squad and real pipeline identities are denied on Deployment patch, Application/AppProject mutation, and Promotion creation.
9. Argo's application-controller ServiceAccount retains literal cluster-admin RBAC (`*/*/*`), but only `d1-dev`/`d1-hml`/other non-production apps use that identity's default destination; `d1-prd` runs through the scoped, namespace-limited destination. The `d1-sandbox` AppProject additionally constrains all its apps to `Deployment`+`Service` only with `clusterResourceWhitelist: []` — a real, verified second layer of containment.
10. Audit gaps (Commit/Branch/Aprovador hardcoded `não disponível`; Git revision only in an overwritten, "never authoritative" projection blob) are genuine but do not prevent reconstructing a production deployment from the durable schema (digest, target, requester, eligibility decision, binding, timestamps) plus the governed Git commit history — classified as post-rollout hardening, not a blocker.

---

## 4. Evidence matrix

| Gate | Score | Evidence class | Key evidence | Residual risk | Classification |
|---|---|---|---|---|---|
| 1. Architecture stability | PASS | COMMITTED_CODE + LIVE_VERIFIED | `DeliveryService.dispatch`: HTTP eligibility check, no orchestration of Change; `stages/*` in Git hold app manifests only, Kargo owns promotion mechanics | None material | No regression since ADR-012 acceptance |
| 2. Git authority separation | PASS | LIVE_VERIFIED | Writer SP ACL live: `allow=16406 deny=32904` (ForcePush/BypassPolicy Deny); reader SP: `allow=2 deny=28` (read-only); branch policy 103 live (`minimumApproverCount:1, creatorVoteCounts:false`); PR #81 confirmed creator=SP, approver=independent human via ADO API | Build Service ACL not independently re-queried this checkpoint (unchanged, carried) | PASS |
| 3. Credential lifecycle durability | CONDITION | LIVE_VERIFIED + CONTRADICTED (doc claim) | CronJob ran 3 unattended cycles during this review; refresh script fails closed (`set -eu`, null-checks) — but manifests are NOT in Git (contradicts p1-residual-closure.md:80); no alerting on failure; SP secrets expire 2027-09-07 | Silent multi-cycle failure before token expiry goes undetected; single calendar date failure for both controllers | CONDITION |
| 4. Kubernetes execution authority | PASS | LIVE_VERIFIED | `d1-prd/argocd-prd-deployer`: write limited to Deployment/Service in-namespace; 4/4 negative RBAC checks re-run live and denied (cross-namespace, self-escalation, kube-system, own-namespace secrets); Delivery backend identity `kargo-promoter` verified scoped (`create promotions: yes`, `patch deployments: no`, `patch applications: no`) | Argo controller itself retains cluster-admin RBAC (see Gate 5/13) | PASS for the production execution path specifically |
| 5. Control-plane desired-state governance | CONDITION | LIVE_VERIFIED + CONTRADICTED (doc claim) | `d1-control-plane`/`d1-prd`/`d1-project`/`d1-sandbox` Application/AppProject objects sourced from Git at `50564c99`, confirmed drift-remediation (P1 doc, ~6s self-heal revert, not re-driven live this checkpoint); BUT Kargo Stage/Promotion CRs (the actual promotion templates, incl. the no-op guard fix) are cluster-only, never committed | Any manual edit to a Stage CR (e.g. the nil-safe guard) has zero durability if the cluster is rebuilt | CONDITION |
| 6. Bypass resistance | PASS | LIVE_VERIFIED | `ado-agent-cluster-admin` confirmed `NotFound` live; `ado-agents/ado-agent` and `squad-sandbox/squad-dev` both denied on Deployment patch, Application/AppProject get, Promotion create (re-run live, not just re-read) | Delivery's own `kargo-promoter` identity *can* create Promotions directly (bypassing Delivery's eligibility gate) if its kubeconfig leaks — gate is enforced in application code, not RBAC | PASS with a named code-vs-RBAC layering note |
| 7. Legitimate promotion path | PASS | LIVE_VERIFIED | Live `Stage.status.lastPromotion`: dev/hml/prd all `Succeeded`, same freight digest `cba145d3d41877ef906137f733399ace2573240d` across all three stages (build-once/promote-immutable holds); nil-safe guard (`outputs['open-pr']?.pr?.id ?? ''`) confirmed present on live `hml`/`prd` Stage templates | Guard fix itself is ungoverned (Gate 5) | PASS |
| 8. Change/authorization correctness | PASS | COMMITTED_CODE | `DeliveryService.dispatch`: binding required before eligibility fetch; `eligibility.decision !== 'ALLOW'` throws `NOT_AUTHORIZED`; half-open window enforced in `EligibilityService` (TOCTOU proven, `eligibility-window-toctou.md`, regression re-run 32/32 green this session) | None material | PASS |
| 9. Concurrency and idempotency | PASS | COMMITTED_CODE + PRIOR_EXECUTION_EVIDENCE | `claimDispatch`: target-busy check + CAS before provider call; E1 evidence: concurrent distinct RCs → exactly one winner + `CONFLICT` for the loser; regression re-run 32/32 green | Live dual-Freight PRD race not independently re-driven this checkpoint (architecturally closed at the Delivery gate, per E1) | PASS |
| 10. Failure detection and operational recovery | CONDITION | LIVE_VERIFIED | No monitoring stack exists (confirmed absent); `argocd-notifications-controller` running but inert; CronJob `backoffLimit:1`, `restartPolicy:Never`; regression suites are the only automated verification | Silent refresh failure could break reconciliation ~1h later with no alert; no on-call/observability story for this specific mechanism | CONDITION |
| 11. Rollback and break-glass readiness | CONDITION | DOCS_ONLY + LIVE_VERIFIED (mechanism only) | Git revert mechanism verified sound: `d1-prd` Application has `selfHeal:true, prune:true`, so a merged revert PR converges automatically; but no written procedure/owner/escalation path exists in either repo's runbooks (confirmed: only unrelated `ado-governance` runbook present) | An operator under incident pressure has no documented steps, only a bare mechanism they'd have to reconstruct in the moment | CONDITION |
| 12. Auditability and evidence chain | PASS_WITH_FOLLOWUP | COMMITTED_CODE + LIVE_VERIFIED | `delivery_requests`/`delivery_change_bindings` schema durably stores `artifact_digest`, target, `requested_by`, `eligibility_decision_id`, `change_id`+`activity_id`, timestamps; PR #81 fully reconstructable via ADO API (creator, approver, merge commit) | `argoRevision` only in overwritten `projection_json`; Commit/Branch/Aprovador hardcoded `não disponível` (absent from domain model, not merely un-rendered) — genuine UX/data gap, not a blocker on the fields that matter for authority reconstruction | Follow-up |
| 13. Production transferability | CONDITION | See matrix below | — | — | See §5 |
| 14. First-rollout blast radius | CONDITION (defines envelope) | — | — | — | See §11 |

---

## 5. Production transferability matrix

| Assumption / proof | Sandbox-specific? | Transfers directly? | Production equivalent required before rollout? | Risk if different |
|---|---|---|---|---|
| Kubernetes distribution (k3s / Rancher Desktop, single node) | Yes | No | Yes — confirm target production cluster's RBAC/API-server behavior matches (standard Kubernetes RBAC semantics; low risk if it's a conformant distro, but must be checked, not assumed) | Low-medium: RBAC semantics are standardized, but admission controllers/PSA policies differ by distro |
| Argo CD v3.5.2 / Kargo v1.11.4, single shared cluster | Partially | Mostly | Yes — confirm production Argo/Kargo versions match or the nil-safe guard / destination-scoping behavior is re-verified on the actual versions | Medium: the no-op-guard bug (Gate 7) was itself a version-specific behavior (`v1.11.4` template-expansion ordering) |
| ADO org `diegolab`, project `platform-engineering`, branch policy 103 | Yes (this exact org/project) | No — must be replicated | Yes — the exact branch-policy configuration (min reviewers, `creatorVoteCounts:false`, no self-approve) must be applied to whatever repo hosts real production GitOps state | High if skipped: without an equivalent policy, the writer SP's PR-gated model has no enforcement backing it |
| Entra tenant `e9dbba09-…`, two dedicated SPs with 1-year client secrets | Yes (this tenant/these app registrations) | No — new SPs needed per real deployment | Yes — provision equivalent non-human SPs in the real tenant hosting production ADO/Entra, with the same read/write ACL split | High if skipped: falls back to human PATs, reopening the exact gap P1 closed |
| Network reachability (repo-server → ADO over public internet, no private link) | Possibly | Untested | Yes — confirm production network path (firewalls, private endpoints, proxy) to the real ADO org from the real Argo/Kargo controllers | Medium: an unreachable Git host silently blocks all reconciliation the same way a Secret-swap could |
| Git repo layout: single protected branch (`d1/desired-state`) + `stages/{dev,hml,prd}` dirs | No (this is the accepted Option A layout) | Yes | No — Option A is accepted; a new production repo simply needs the same layout, not new design | Low |
| Namespace isolation (`d1-prd` scoped Argo destination, namespace-limited Role) | No — pattern is portable | Yes, pattern-wise | Yes — the specific destination secret / scoped SA must be created per real production namespace | Low if the same pattern is followed; Medium if skipped and the default in-cluster alias (cluster-admin path) is used instead, as happens today for non-prod stages |
| Secret storage/rotation (CronJob-based, bootstrap-applied) | Yes — built for this sandbox's constraints | Partially | Yes — either the CronJob mechanism is committed to the real GitOps repo (closing Gate 3's C1) or replaced with a managed secret-rotation service before scaling past the first rollout | High: this is the single biggest transferability gap found in this review — the mechanism works, but nothing about its current form is production-hardened for durability |
| Backstage/Delivery runtime identity: local kubeconfig file (`~/.backstage-delivery/`, operator's laptop) | Yes — entirely dev-machine specific | No | Yes — production Delivery backend must run with a properly provisioned, non-laptop-bound credential (e.g. a mounted Secret or workload identity) | High: the current identity cannot exist in a real production deployment of the Backstage backend; this must change before any real rollout, not just be "equivalent" |
| Production target naming/account/cluster boundaries (`d1-prd` namespace on the same cluster as `d1-dev`/`d1-hml`) | Yes — sandbox does not use separate clusters/accounts | Not directly | Recommended (not mandatory) — the first real production rollout should target a namespace/account with clear separation from non-prod, consistent with the destination-scoping pattern already proven | Medium: shared-cluster blast radius is larger than a separate cluster/account, though AppProject scoping partially mitigates |
| Persistence: dev uses `better-sqlite3`; production config specifies Postgres | Yes | Not verified | Yes — the Delivery schema/migrations must be exercised against Postgres before treating durable evidence claims as production-verified | Medium: migrations are standard Knex SQL, low technical risk, but zero live proof exists today |

**Assessment:** Several of these are not "sandbox passed, production untested" hand-waves — they are concrete, nameable gaps (SP provisioning, branch policy replication, the local-laptop kubeconfig, the uncommitted refresh manifests) that a controlled first rollout can require as explicit preflight conditions rather than open-ended risk. None of them indicate a materially different authority topology that invalidates the architecture; they indicate operational setup work.

---

## 6. Authority model at review time

| Role | Identity | Verified scope |
|---|---|---|
| Git writer (Delivery/Kargo) | Entra SP `idp-d1-kargo-writer` (non-human) | ADO ACL live: `Contribute` allowed, `ForcePush`/`BypassPolicy` (push and PR-completion) explicitly `Deny`; must go through PR + branch policy 103 |
| Git reader (Argo) | Entra SP `idp-d1-argocd-reader` (non-human) | ADO ACL live: `Read` only, `Contribute`/`ForcePush`/`CreateBranch` explicitly `Deny` |
| Reconciler | Argo CD `argocd-application-controller` | Cluster-admin-equivalent ClusterRole (`*/*/*`), but constrained per-Application by AppProject (`d1-sandbox`: `Deployment`+`Service` only, `clusterResourceWhitelist:[]`, single sourceRepo); production (`d1-prd`) additionally uses a namespace-scoped destination identity |
| Runtime deployer (production) | K8s SA `d1-prd/argocd-prd-deployer` | Namespaced Role: write (`create/update/patch/delete`) limited to Deployment+Service in `d1-prd`; broad read (`get/list/watch`) in-namespace only for Argo's cache; verified live: cross-namespace, cluster-escalation, `kube-system`, and own-namespace-Secrets all denied |
| Delivery backend's own K8s identity | K8s SA `d1-sandbox/kargo-promoter` | Verified live: `create promotions: yes`; `get applications: yes`; `patch deployments: no`; `patch applications: no`; `create clusterrolebindings: no`; `get secrets: no` |
| Ordinary squad/pipeline | Representative `squad-sandbox/squad-dev`; real `ado-agents/ado-agent` | Verified live: denied on Deployment patch, Application/AppProject get, Promotion create; `ado-agent-cluster-admin` binding confirmed `NotFound` |
| Human/platform admin | Operator ADO account (`diego.fernandes@outlook.com`) | Direct push to `d1/desired-state` denied (`TF402455`) even for this account per P1 evidence; acts only as an independent PR-approving reviewer in the live-verified PR #81 flow |

---

## 7. Failure / recovery assessment

- **Git (ADO):** Reachable over the public internet from this sandbox; no private-link tested. If Git becomes unreachable, both readers and writers fail closed (no fallback to direct cluster mutation exists in the architecture) — the legitimate path simply stalls, which is the correct failure mode, but no alert surfaces it.
- **Argo:** `d1-prd` confirmed `Synced/Healthy` at governed revision throughout this review, including after a fresh `git fetch`. If Argo cannot read Git, reconciliation stalls at last-known-good state (fail-static, not fail-open) — acceptable for a first rollout, no alerting exists to notice it.
- **Kargo:** Promotion CRs are cluster-only (Gate 5); a Kargo/etcd loss would require manually re-creating Stage/Warehouse objects from the last-known configuration, which exists only as ad hoc knowledge, not a committed artifact.
- **Kubernetes:** Standard node/pod failure handling applies; no delivery-specific gap identified beyond the general absence of monitoring.
- **Token refresh:** Verified live and functioning (3 unattended cycles during this review), but a failure would be silent for up to ~90 minutes (two missed 30-minute cycles) before the ~1h token expires and reconciliation breaks — with no alert at any point in that window.
- **Backstage/Delivery availability:** `DeliveryService.dispatch` commits the `dispatched` state via `claimDispatch` before calling the provider, and the provider's own `dispatch()` is documented idempotent (looks up by deployment-request-id annotation before creating) — a crash between these steps is recoverable by retry, not a double-promotion risk. A Backstage/Delivery outage after dispatch does not affect Argo's already-running reconciliation (Argo is independent of Backstage uptime).

---

## 8. Rollback / emergency assessment

**What exists today:** A sound mechanical rollback path — revert the Git commit on `d1/desired-state`, get it approved and merged under the same branch-policy-gated PR flow already proven, and `d1-prd`'s `selfHeal:true`/`prune:true` automated sync policy converges the cluster without further manual action.

**What is sufficient:** The mechanism itself. It uses the exact same governed path already proven for the forward case, so no new authority or tooling gap exists.

**What is not sufficient:** There is no written procedure naming who initiates a rollback, who approves the revert PR under incident time pressure, what the escalation path is if the normal approver is unavailable, or how an operator distinguishes "needs Git revert" from "needs Argo-side manual intervention." This is a genuine, bounded gap — a real runbook, not a redesign — appropriate as a pre-rollout condition.

---

## 9. Mandatory challenge answers

1. **Strongest ordinary-squad bypass path:** None found with a live RBAC path. All tested identities (`squad-sandbox/squad-dev`, `ado-agents/ado-agent`) are denied on every mutation surface (Deployment patch, Application/AppProject, Promotion creation) and the ADO Build Service Git ACL is read+CreateTag only. The only remaining latent path is code-level: the Delivery backend's own `kargo-promoter` identity can create Promotions directly, bypassing the eligibility HTTP check, if an attacker obtained that specific kubeconfig file — but that is an insider/credential-theft scenario, not an ordinary-squad path.
2. **Strongest platform-admin accidental-bypass path:** Direct push to `d1/desired-state` is denied even for the org's own top-privilege human account (`TF402455`, per P1 evidence, still enforced by live branch policy 103 verified this checkpoint). The realistic accidental path is an admin manually editing a Kargo Stage CR (imperative, no branch-policy gate applies there at all) — not the Git desired-state path.
3. **Which control still depends on bootstrap/manual state:** The entire token-refresh mechanism (Secret, CronJob, RBAC, script) and all Kargo Stage/Promotion/Warehouse CRs. Neither is sourced from Git.
4. **Token-refresh CronJob fails for more than one token lifetime:** Both live Git credential Secrets go stale; Argo read and Kargo/Delivery write both start failing with auth errors. No automated remediation exists; an operator must notice (no alert fires) and manually re-trigger the refresh Job or investigate root cause.
5. **SP client secret expires or is revoked:** Refresh script's `mint_token` call fails, the script's `FATAL: … token mint failed` check triggers `exit 1`, the Job fails visibly in Job history (but with no alert). Recovery requires an `az ad app credential reset` and reloading the bootstrap Secret — a manual, undocumented-as-runbook procedure today.
6. **Recovery if Argo can no longer read Git:** Argo fails static at last-reconciled state (verified: this is Argo's documented behavior, not special to this setup). No automatic recovery; requires diagnosing and fixing the credential/network issue, then a manual or scheduled re-sync.
7. **Recovery if Kargo can no longer write Git:** Promotions error at the `git-push`/`git-open-pr` step; no partial/corrupt state is left in Git (the write is all-or-nothing per Kargo's step model). Recovery requires fixing the writer credential, then re-triggering promotion.
8. **Recovery if Git is correct but Argo reconciliation fails:** Argo Application shows `OutOfSync` or a comparison/health error; live cluster state simply does not converge until the underlying issue (e.g., admission-controller rejection, resource conflict) is fixed and a manual or next-scheduled sync succeeds.
9. **What protects production if Backstage is unavailable:** Nothing in the reconciliation path depends on Backstage uptime — Argo continues reconciling independently, and any already-dispatched promotion continues through Kargo/Argo without Backstage. Only new deployment *requests* are blocked, which is the correct fail-safe direction.
10. **What protects production if Delivery backend is unavailable after dispatch:** `claimDispatch` already committed the `dispatched` state before the provider call, and the provider's `dispatch()` is idempotent by deployment-request-id — a backend restart/outage does not risk double-promotion; the in-flight Kargo Promotion and Argo reconciliation proceed independently of Backstage.
11. **Sufficiency of the evidence chain:** Sufficient for the properties that matter for authority reconstruction — digest, target, requester, eligibility decision, Change/activity binding, and (via ADO API) the exact PR/commit/approver — but not sufficient for source commit/branch provenance display in the UI, which is a genuine, named, non-blocking gap (Gate 12).
12. **Sandbox assumptions needing production proof:** SP provisioning in the real tenant; branch-policy replication on the real GitOps repo; a non-laptop-bound Delivery backend credential; the token-refresh mechanism actually committed to Git; Postgres-backed Delivery persistence exercised at least once.
13. **Architecture flaw vs. condition vs. post-rollout hardening, for each open item:** Nothing found is an architecture flaw (see §3 point 6, boundary review). The token-refresh commit gap, the missing rollback runbook, and the laptop-bound Delivery kubeconfig are pre-rollout conditions. The Argo controller's broad RBAC, the audit-field UX gaps, and general observability/HA are post-rollout hardening.
14. **What would downgrade this verdict:** See §13.

---

## 10. Conditions before rollout

1. **Commit the token-refresh manifests to the governed GitOps repo** (or an equivalent version-controlled location the platform team owns), replacing the current untracked/dead-branch state. Owner: platform team. Evidence required: a merged PR in `d0-gitops-sandbox` (or the real production GitOps repo) adding `bootstrap/token-refresh/` (or equivalent), verified present via `git ls-tree` at the merged revision.
2. **Replace the operator-laptop-bound Delivery backend kubeconfig with a properly provisioned credential** (mounted Secret, workload identity, or equivalent) for the environment that will run the first production rollout. Owner: platform team. Evidence required: `app-config` for that environment shows a `kubeconfigPath`/equivalent that does not resolve to a home-directory path on any individual's machine.
3. **Write and store a short rollback/emergency runbook** naming: who initiates a Git revert, who approves it, the escalation contact if the approver is unavailable, and how to distinguish a Git-revert scenario from an Argo-side manual-intervention scenario. Owner: platform team + designated on-call. Evidence required: the runbook document, reviewed by at least one person other than its author.
4. **Wire a basic alert on token-refresh Job failure** (Argo's own `argocd-notifications-controller` is already deployed and can be configured, or an equivalent minimal mechanism). Owner: platform team. Evidence required: a deliberate failure test (e.g., temporarily wrong credential in a non-production test of the mechanism) demonstrably produces a visible alert, not just a failed Job silently sitting in history.
5. **Provision the two non-human Entra SPs (or equivalent) and the matching ADO branch policy in the actual repository that will hold real production GitOps desired state**, if different from `d0-gitops-sandbox`. Owner: platform team. Evidence required: live ACL verification (same method used in this review) showing the same read-only/write-with-no-bypass split on the real repo.

Each condition above is independently verifiable, bounded, and does not require any architecture redesign. None require completing HA/DR, full audit UX, or a generalized secrets platform.

---

## 11. Authorized first-rollout envelope

- **Workload criticality:** one non-critical or medium-criticality component only — not a business-critical or customer-facing system for the first rollout.
- **Applications/targets:** exactly one component, one production namespace, using the already-proven scoped `d1-prd`-pattern destination (namespace-limited Role, `clusterResourceWhitelist:[]`, `Deployment`+`Service` only).
- **Cluster/account:** the cluster/account hosting that first real production target; do not simultaneously onboard a second cluster/account in the same wave.
- **Operator ownership:** an explicit, named platform owner present and reachable for the full observation period — not an unattended rollout.
- **Observation period:** two weeks of stable `Synced/Healthy` state and at least one successful real promotion cycle before authorizing a second workload or target.
- **Explicit exclusions:** no simultaneous migration of multiple workloads; no expansion to a second production namespace/cluster/account until the observation period closes; no removal or weakening of any currently-proven RBAC/branch-policy control to accommodate the rollout; no skipping any condition in §10 "to save time" for this first workload.

---

## 12. Post-rollout hardening backlog

- Reduce the Argo application-controller's cluster-wide RBAC where feasible, or document why the AppProject-level containment is accepted as the durable boundary long-term.
- Add first-class Commit/Branch/source-provenance fields to the `ReleaseCandidate` domain model, replacing the hardcoded `não disponível` UI strings.
- Promote `argoRevision`/Git desired-state revision from the overwritten `projection_json` blob into a durable, first-class audit column.
- Exercise Delivery's Postgres-backed persistence path at least once outside of `better-sqlite3` dev usage.
- General HA/DR, automated rollback, supply-chain provenance expansion, global Delivery workbench, and further Deployments UX polish — all remain explicitly out of scope for the first rollout per the review contract's non-blocker list, and none were found during this review to be a concrete first-rollout risk that would override that list.

---

## 13. Verdict downgrade triggers

- The token-refresh CronJob fails silently for more than one cycle in production without being caught by whatever alerting Condition 4 establishes.
- A real production PR attempting to use the writer SP's branch-policy-gated path is completed via any bypass (self-approval succeeding, or a policy-exempt path found) not present in this review's live-verified ACL.
- The Delivery backend's production credential is found to still resolve to an individual's local machine at the time of rollout (Condition 2 not actually met).
- Any negative-authority check re-run at rollout time (squad/pipeline mutation attempts) returns anything other than `Forbidden`/denied.
- The rollback runbook (Condition 3) is found to be missing, untested, or unowned at the moment an actual incident requires it.

---

## 14. STOP

This review authorized nothing beyond the narrow first-rollout envelope in §11, contingent on the conditions in §10. No implementation, rollout, new milestone, UI work, GitOps desired-state change, Kubernetes/credential/ACL/branch-policy mutation, or P1 continuation was authorized or executed by this review. All inspection in this checkpoint was read-only.
