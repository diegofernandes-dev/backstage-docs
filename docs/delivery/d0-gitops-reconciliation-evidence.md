# D0 — Pure GitOps Reconciliation: Execution Evidence

- **Status:** EXECUTED — evidence recorded, awaiting architecture review
- **Verdict:** **CONDITIONAL PASS**
- **Date:** 2026-09-04
- **Checkpoint:** [`d0-gitops-reconciliation-checkpoint.md`](./d0-gitops-reconciliation-checkpoint.md)
- **Parent spike:** [`architecture-spike.md`](./architecture-spike.md)
- **Shared ADR:** [ADR-012](../adr/ADR-012-delivery-management-gitops-promotion.md) — **remains Proposed; not moved to Accepted by this document**

## Scope of this record

This records only what was executed and observed for D0. It authorizes nothing.
No Kargo, Change/GMUD integration, Delivery persistence, Backstage UI, Azure DevOps
Environment check, or production deployment was implemented or attempted.

## Baselines

| Item | Value |
| --- | --- |
| Canonical docs baseline | `backstage-docs@076dd03e6605580ac86b6e3f33179d2fc981a2dd` |
| GitOps repository | `diegolab/platform-engineering/d0-gitops-sandbox` (created for D0) |
| GitOps tracked branch | `d0/desired-state` |
| GitOps initial revision | `cd38398a229017a6668715914555e6f462385bb6` |
| GitOps final revision | `19dd327e976ed1ea695f74ce003302262a912cc1` |
| Argo CD | `v3.5.2` |
| Kubernetes | `v1.33.6+k3s1` (Rancher Desktop, arm64) |
| Argo Application / AppProject | `d0-sandbox` / `d0-sandbox` |
| Target | `https://kubernetes.default.svc`, namespace `d0-sandbox` |
| Reconciliation | Argo default 180s polling; **no webhook configured** |
| Implementation repo | none — `platform-devops-developer-portal` was deliberately **not** modified |

### Immutable artifacts

Both digests were resolved **before** any Git commit, so no promotion required a rebuild.
Multi-arch index digests, not per-platform digests.

| Ref | Tag equivalent | Digest |
| --- | --- | --- |
| A | `nginx:1.27-alpine` | `sha256:65645c7b…f2a10` |
| B | `nginx:1.29-alpine` | `sha256:56168782…830de` |

## Scenario results

| Scenario | Result | Git revision | Sync / Health | Observed runtime image |
| --- | --- | --- | --- | --- |
| A initial reconciliation | **PASS** | `cd38398a` | Synced / Healthy | `nginx@sha256:65645c7b…` (= A) |
| B artifact change A→B | **PASS** | `94f3ea1d` | Synced / Healthy | `nginx@sha256:56168782…` (= B) |
| C Git revert B→A | **PASS** | `604bcba8` | Synced / Healthy | `nginx@sha256:65645c7b…` (= A) |
| D drift / self-heal | **PASS** | `604bcba8` (unchanged) | Synced / Healthy | A, replicas restored 4→2 |
| E no webhook dependency | **PASS** | six changes | all converged | polling only |
| F failure visibility | **PASS** | `73b0f7ab` → `c328f6da` | **Synced / Degraded** → Synced / Healthy | `ImagePullBackOff` → A |
| G AppProject boundary | **PASS** | n/a | both probes `InvalidSpecError` | nothing deployed |
| H direct mutation posture | **PASS (documented)** | n/a | n/a | see findings |
| I prune / delete posture | **PASS (observed)** | `0c39eaf3` → `9aeadcab` | OutOfSync → Synced | Service retained, then pruned, then restored |

Raw command output for every scenario:
`d0-gitops-sandbox@19dd327e:evidence/D0-EVIDENCE.md`.

### Detail worth recording

**A/B/C — artifact identity correlates end to end.** For every scenario the Git-declared
digest, `Application.status.sync.revision`, and the pod `containerStatuses[].imageID`
agreed exactly. Runtime identity was deterministically traceable to a specific commit.

**D — self-heal is remediation, not release.** `kubectl scale --replicas=4` was reverted to
2 within the same second, `operationState.initiatedBy = {"automated": true}`. Git HEAD and
`sync.revision` were unchanged throughout.

> This scenario tests remediation back to an already-declared desired state. It does not
> represent a new release or a new business change.

**E — correctness is independent of notification.** Zero ADO hook subscriptions, no Argo
webhook secret, zero inbound `POST /api/webhook`. The only log mention of "webhook" is a
startup warning that none is configured. Observed commit→detection latency: 102s, 163s,
188s, 257s, 267s, 359s. Latency varied with the polling cycle; **correctness did not**.

**F — the central proof.** With a well-formed but unresolvable digest the Application was
simultaneously `Synced` (the commit applied cleanly) and `Degraded` (the workload could not
run), with `ImagePullBackOff … not found` and `ProgressDeadlineExceeded`. The prior
ReplicaSet kept serving 2/2 ready replicas. **A Git commit is demonstrably not a successful
deployment.** Recovery was by `git revert`, not by imperative rollback.

**I — prune posture, D0 observation only.** With `prune: false`, removing `app/service.yaml`
from Git left the Service in place, marked `requiresPruning: true`, and the Application
`OutOfSync` — the divergence was *visible but not acted on*. With `prune: true` the Service
was deleted; the Deployment was untouched. The Service was then restored through Git.

> This is D0 observed behaviour in a sandbox. **No production prune policy is derived from
> it.** Critical-resource deletion policy remains deferred.

## Architecture observations

**GitOps substrate.** Yes. Git → Argo → Kubernetes operated with no custom deployment state
machine, no orchestration service and no callback receiver. `argocd app sync` was never
invoked; every change was an ordinary commit.

**Artifact identity.** Yes. Git, Argo and runtime were correlated to the same immutable
digest in every scenario. Recording *index* digests rather than per-platform digests
mattered on arm64 and should be an explicit rule in any future promotion contract.

**Failure semantics.** Yes, and without inventing a lifecycle. Argo's existing two axes were
sufficient: `sync` answers "is the declared state applied?", `health` answers "does it
work?". Scenario F is the case that matters — `Synced` + `Degraded` — and it was legible
without custom status modelling.

**Recovery.** Yes. `git revert` restored both the declared state and the runtime in
scenarios C, F and I. Git never disagreed with runtime after recovery, because the revert
commit itself became the target revision.

**Drift.** Yes. Self-heal restored the declared value without any Git change.

**Security boundary.** Partly — this is the reason for CONDITIONAL PASS. See below.

## Security / authority findings

1. **The AppProject boundary works.** Both negative probes were rejected before reaching the
   cluster: a forbidden destination (`kube-system`) and a forbidden source repository
   (`platform-gitops`), each with an explicit `InvalidSpecError`. `clusterResourceWhitelist`
   was empty and the namespaced whitelist allowed only `apps/Deployment` and `core/Service`.
   Repository isolation, destination isolation and basic blast-radius limitation are all
   realistic for the future production model.

2. **Kubernetes RBAC provided no blast-radius limitation.** The stock Argo CD install binds
   `argocd-application-controller` to a ClusterRole of `*/*/*`. It could
   `delete deployments -n kube-system`. In D0 the AppProject was the *only* effective
   constraint. **Defence in depth is therefore not yet demonstrated** — production needs a
   scoped controller role or per-destination service accounts.

3. **Git write authority and Argo read authority were not separable.** SSH Git access is
   broken org-wide for this account (auth succeeds, `git-upload-pack`/`git-receive-pack`
   fail on every repo), so both the operator and Argo CD used the same broadly-scoped
   `AZURE_DEVOPS_PAT`. Production requires a narrowly-scoped, ideally read-only credential
   for the reconciler, distinct from human write credentials. **Not proven by D0.**

4. **Human access to the sandbox namespace is cluster-admin.** The local kubeconfig answers
   `yes` to `auth can-i '*' '*' --all-namespaces`. Acceptable for a local POC, recorded as a
   privilege that must not carry into production. No CI identity had cluster access.

5. **Temporary POC privilege / cluster repair.** Nine orphaned Kyverno admission webhooks
   (`failurePolicy: Fail`, backing service deleted) were blocking all Pod and Deployment
   creation cluster-wide. They were backed up and deleted with operator authorization. This
   was pre-existing breakage unrelated to D0.

Intended production direction:

```text
normal desired-state mutation:  Git
runtime reconciliation:         Argo CD
direct cluster mutation:        restricted operational exception
```

## Deviations

1. **SSH Git unusable org-wide** → all Git traffic over HTTPS + PAT; authority separation
   (finding 3) not demonstrated.
2. **`refs/heads/main` is PR-protected project-wide** (blocking required-reviewers policy on
   every repo). D0 tracked the unprotected sandbox branch `d0/desired-state`, permitted by
   checkpoint section 14. **The PR-gated promotion path was not exercised.** An attempt to
   drive PRs programmatically was blocked by local tooling policy and was not worked around.
3. **Artifacts are public upstream images**, not platform-built. D0 proves digest identity
   propagation, not provenance.
4. **Orphaned Kyverno webhooks deleted** (see finding 5).
5. **Argo CD installed with `--server-side`** — the stock manifest exceeds the client-side
   apply annotation limit on the `applicationsets` CRD.
6. **Scenario A convergence (~6s) reflects initial Application creation**, which syncs
   immediately. Polling evidence comes from B, C, F and I.

## Why CONDITIONAL PASS rather than PASS

Every mandatory scenario A–H passed and I was exercised and documented. No imperative Argo
operation was required as the normal deployment mechanism, and no custom orchestration was
needed. The reconciliation model itself is **not** in question.

Two authority concerns remain open, neither of which invalidates the desired-state model:

- the reconciler holds cluster-admin-equivalent Kubernetes RBAC (finding 2);
- Git write and Argo read authority could not be separated in this environment (finding 3).

Both are narrow production-hardening items, which matches the checkpoint's CONDITIONAL PASS
definition rather than PASS.

## Production gaps

- Scoped RBAC for the Argo controller; per-destination service accounts.
- A dedicated least-privilege, ideally read-only, repository credential for the reconciler.
- Promotion through a protected branch with review — available in this org, not exercised.
- Platform-owned CI → registry → provenance/signature path.
- Prune and critical-resource deletion policy, with admission-level guardrails.
- Multi-cluster destinations; D0 used a single local cluster only.

## Open questions for D1

1. Which GitOps layout — one protected branch with stage directories, or stage branches —
   given that `main` is PR-protected org-wide and PR-gated promotion is therefore the
   realistic production mechanism?
2. Should the reconciler's Git credential be read-only, and what issues it?
3. What scoped Kubernetes RBAC replaces the default `*/*/*` controller role?
4. Does the promotion contract mandate index digests over per-platform digests?
5. `Synced + Degraded` is legible to an operator — what is the minimum durable Delivery
   evidence that must outlive Argo's own history?
6. Does a promotion controller (Kargo) reduce custom orchestration relative to the plain
   Git→Argo path D0 just demonstrated, which already required none?

## Gate

**D0 executed. Evidence recorded. STOP for architecture review.**

- ADR-012 remains **Proposed**.
- **NO-GO** for D1 until this evidence is reviewed.
- **NO-GO** for Kargo, Delivery implementation, Backstage Delivery UI, GMUD integration,
  F3.1.1b and F3.1.2+ — unchanged.
