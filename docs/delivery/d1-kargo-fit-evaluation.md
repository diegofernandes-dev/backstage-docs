# D1 Promotion Controller Fit Result

## Verdict
KARGO_FIT

Conditional on the exact adoption scope in "If KARGO_FIT" below. Kargo materially removes real promotion machinery (artifact discovery/tracking, stage-to-stage promotion bookkeeping, protected-branch PR mutation with wait/resume semantics, verification orchestration, controller-restart recovery) and its operational footprint, while real, is justified by what it removes — provided the platform does not try to fully automate promotion onto a protected branch (Kargo's own behavior there is PR-gated, not push-gated, which is a governance positive, not a gap).

## Baselines
- Canonical docs SHA: `fc7f2488364111a05483bee2455c46754f28d4ca` (backstage-docs origin/main, fast-forwarded from `8c21b5c` at session start)
- Kargo version: v1.11.4 (Helm chart `oci://ghcr.io/akuity/kargo-charts/kargo:1.11.4`, digest `sha256:0a0cb3b7a4d6b35aa37bc0971857a6420ebc569bf73d4cae8728b7d06a8211de`)
- Argo CD version: v3.5.2 (reused D0 install, unchanged)
- Kubernetes version: v1.33.6+k3s1 (Rancher Desktop, node `lima-rancher-desktop`, dockerd container runtime — reused D0 sandbox cluster)
- GitOps repo/revision: `diegolab/platform-engineering/d0-gitops-sandbox`, new branch `d1/desired-state` (branched from `main`@`8c21b5c...` equivalent, base commit `90ef619ee26f1ed12dc743902ce664ee2432897c`), protected by ADO policy id 103 (Minimum number of reviewers, blocking, minimumApproverCount=1)
- Sandbox target: reused D0's single-cluster sandbox; new namespaces `d1-sandbox` (Kargo Project), `d1-dev`, `d1-hml` (Argo-managed workloads), plus `kargo`, `cert-manager`, `argo-rollouts` (new operator namespaces) and Kargo's own auto-created `kargo-shared-resources`/`kargo-system-resources`

## What Was Executed
1. Verified canonical docs state against `origin/main` before starting: gate read exactly `D0 = ACCEPT_CONDITIONAL_PASS`, `D1 = GO_D1_WITH_CARRIED_GAPS`, `ADR-012 = Proposed` — no discrepancy from the prompt's expectation.
2. Created a new `d1/desired-state` branch (Option A layout: one branch + `stages/dev`, `stages/hml` directories) and applied a real ADO branch policy to it only, leaving `main` and D0's branches untouched.
3. Verified the protection was real: a direct `git push` was hard-rejected (`TF402455`) before any Kargo object existed.
4. Installed Kargo v1.11.4 via Helm into the existing D0 sandbox cluster, discovering and resolving a hard dependency on cert-manager (installed) and, separately, on Argo Rollouts for verification (installed).
5. Created a Kargo `Project`, a `Warehouse` subscribed to `docker.io/library/nginx` (semver `1.27.x`), and `Stage` objects for `dev` and `hml`, each with a `promotionTemplate` doing `git-clone` → `yaml-update` → `git-commit` → `git-push` (new branch) → `git-open-pr` → `git-wait-for-pr` → `argocd-update`, plus one `AnalysisTemplate` running a real HTTP smoke-test Job.
6. Ran multiple real `Promotion` objects to completion, including two that required genuine human PR approval/merge in ADO (user-performed), one that survived a forced `kargo-controller` pod kill mid-wait, and one that produced a real, passing `AnalysisRun`.
7. Independently verified every claimed outcome against raw state: `git show`/`git fetch` on the actual branch content, `kubectl get application -o jsonpath` for Argo sync/health/revision, `kubectl get pods -o jsonpath` for running image digests, and ADO PR objects read back via MCP — not just Kargo's own status reporting.

## Artifact Discovery / Identity
Kargo's Warehouse discovered a live artifact from the public registry (`nginx`, semver `1.27.x` constraint) and created Freight `cba145d3d41877ef906137f733399ace2573240d` (alias `ugly-greyhound`), digest `sha256:05b8cb60c354a44ab824ea6e7dc69b46d50762cdbe728a347a5b656e6fb3d7c4`, tag `1.31.5`. This same Freight/digest was independently promoted into both `dev` and `hml`, and the promoted digest was confirmed present, unchanged, at every hop: Freight → Git commit content → Argo `status.sync.revision` → running pod `imageID`. No rebuild occurred between stages.

Runtime digest readback was `docker-pullable://nginx@sha256:...` — the same dockerd-backed shape D0 flagged as non-portable to CRI/containerd production nodes. This carried D0 follow-up remains unresolved; it was not re-testable on a different runtime in this sandbox.

**Answer to the framing question:** yes — Kargo removes real artifact-discovery/promotion-bookkeeping work. A thin alternative would need its own registry-polling or CI-supplied-digest mechanism, its own Freight-equivalent identity record, and its own "what's currently on each stage" tracking; Kargo provides all of this out of the box, correctly, with no rebuild-between-stages behavior observed.

## DEV -> HML Promotion
The same Freight/digest was promoted independently onto `dev` and `hml`, both succeeding through the full Git+PR+Argo chain, and Freight identity was preserved end to end in both. This proves the more architecturally important half of the framing question (promoted artifact identity is preserved, no rebuild).

**Deviation from the prompt's example topology:** the prompt's example topology (§6) has `hml` source verified Freight *from* `dev` (`sources.stages: [dev]`), i.e. an explicit stage-to-stage dependency gate. This was configured and accepted structurally by the Kargo API in this session, but not run to a completed promotion — `dev`'s own `currentFreight` status was delayed by an idempotency exploration (see Concurrency section) and, once resolved, time was prioritized toward the other mandatory scenarios. `hml` was instead configured to source directly from the Warehouse for the executed evidence. Stage-to-stage chaining (one stage gating on another stage's specifically-verified Freight, as opposed to both independently promoting from the source Warehouse) is therefore **not fully proven end-to-end**, though nothing observed suggests it would behave differently — it uses the same Freight/Promotion machinery already proven to work.

## Git Mutation
**This is the mandatory scenario and the D0-carried acceptance condition; it is fully proven end to end against a genuinely protected branch.**

Full chain (Promotion `dev.01m1qbpr5da0pt0eh6qyfh8qvf.cba145d`, later repeated for `hml` and again for `dev`):
1. `git-clone` checked out `d1/desired-state` — Succeeded.
2. `yaml-update` edited `stages/{dev,hml}/deployment.yaml`'s image field to the Freight's digest, using expr-lang `imageFrom("...").Digest` (note: capital `D` — Kargo's expression engine binds Go struct field names, not JSON tags; the lowercase `.digest` from Kargo's own status JSON fails with `type v1alpha1.Image has no field digest`) — Succeeded.
3. `git-commit` committed with a message encoding the promoted digest — Succeeded.
4. `git-push` pushed to a **new** branch (`kargo/promote-dev-N`) — Succeeded. (A first attempt using `git-push` with `targetBranch: d1/desired-state` directly was correctly, loudly rejected: `TF402455: Pushes to this branch are not permitted; you must use a pull request to update this branch.` Kargo did not weaken or bypass the policy.)
5. `git-open-pr` opened a **real** ADO pull request from the new branch into `d1/desired-state`, with a title auto-encoding the promoted digest and a description auto-appending a "View in Kargo UI" deep link — Succeeded. (PRs #71, #72, #74 were all real, independently verified via ADO MCP.)
6. `git-wait-for-pr` correctly blocked (`Promotion.status.phase = Running`) until a human reviewer approved and completed the PR in ADO — in one case for ~17.5 real minutes, in another surviving a full `kargo-controller` pod kill during the wait.
7. `argocd-update` triggered Argo CD refresh of the target Application — Succeeded (after a required authorization-annotation fix, see RBAC below).

The actual Git mutation was independently confirmed via `git fetch && git show` on the merged branch content (not Kargo's own status): the promoted digest was genuinely present in `stages/dev/deployment.yaml` at the merge commit. Argo CD then converged the running workload to that exact revision through its own ordinary automated sync — independent confirmation that the mutation was real, not merely reported.

**Answer to the framing question:** Kargo's Git mutation is materially safer against a protected branch than a naive thin writer only if that thin writer is *also* built with PR-based mutation from day one — Kargo does not provide any capability here a competent custom writer couldn't also implement (open a PR, wait for merge), but it does provide it *out of the box*, correctly refusing to bypass policy, with no additional code. A **notable weakening**: on a protected branch, "Promotion succeeded" no longer means "fully automated" — it now means "a human closed the loop," and Kargo does not distinguish this from a directly-pushed success in its terminal phase semantics. Any Delivery UI consuming `Promotion.status.phase == Succeeded` must know separately whether the target Stage's template used a PR-gated or direct-push path to correctly report "automated" vs. "pending human action" during the wait window.

## Verification
Behaviorally proven with a real, passing result — not merely structurally accepted. `AnalysisRun dev.01m1sev27t0cm77zfkmhq10grw.46e1a8c`, phase `Successful`, ran a real Kubernetes Job (`http-ok`) that curled the promoted Service and exited 0 (`runSummary: {count:1, successful:1}`, ~6s).

**Hard dependency finding:** Kargo's verification mechanism is not its own — `AnalysisTemplate` is Argo Rollouts' CRD, reused by Kargo. This cluster had no Argo Rollouts installed initially; Kargo's controller silently downgraded ("Argo Rollouts integration was enabled, but no Argo Rollouts CRDs were found. Proceeding without Argo Rollouts integration.") rather than erroring loudly. Installing Argo Rollouts (2 pods, 5 new CRDs) was required, plus a `kargo-controller` restart to pick up the new CRDs.

**UX/observability gap:** `Stage.status.conditions` shows the identical message ("Freight has been verified", `Ready=True reason=Verified`) whether an `AnalysisTemplate` actually ran and passed, or whether no verification was configured at all (Kargo trivially verifies in the latter case). This was discovered directly: an earlier `hml` Stage revision silently dropped its `verification` block on a `kubectl apply`, and its status was indistinguishable from a real pass until `kubectl get analysisrun` came back empty. A Delivery UI cannot rely on `Stage.status.conditions` alone to know whether verification meaningfully ran; it must separately check for `AnalysisRun` existence.

**Answer to the framing question:** yes, real orchestration value beyond Argo's `Synced/Healthy` — a distinct, scoped-per-promotion check with its own pass/fail gate on promotion progression — but it comes bundled with a second operator (Argo Rollouts) as a mandatory dependency, not an optional add-on.

## Failure / Recovery
Two distinct, clean failure scenarios were observed (not manufactured — both arose from genuine execution attempts):
1. **Invalid Git credential**: an AAD OAuth token used as the Git-writer password caused `git-clone` to fail with `Authentication failed`; `Promotion.status.phase = Errored`, subsequent steps correctly `Skipped` (no partial mutation attempted).
2. **Protected-branch push rejection**: `git-push` to `d1/desired-state` directly failed with the exact same `TF402455` server rejection independently reproduced by a manual `git push` — full raw Git stderr surfaced in `Promotion.status.message` without needing Kargo's UI or CLI.

In both cases, `kubectl get promotion -o yaml` alone gave a complete, correctly-attributed root cause. This is good failure visibility: a future Delivery UI could project these states without inventing a second workflow engine, by simply surfacing `Promotion.status.phase` and `.message`.

## Concurrency / Idempotency
Two Promotion objects were created near-simultaneously for the same Stage and same (already-current) Freight. Kargo does not reject duplicates at admission time (no dedup on (stage, freight) pairs), but both independently reached the same real idempotency result: `yaml-update` detected no content diff, so `git-commit`/`git-open-pr` were correctly `Skipped` rather than creating redundant commits or duplicate PRs.

**Gap found:** this idempotency short-circuit is not safely composed with a `git-wait-for-pr` step that unconditionally expects a PR to exist — both no-op promotions terminated in `phase=Errored` rather than a clean success, because the naive template had no conditional gating. An attempt to add `if:` conditionals to skip the wait step when no PR was opened also failed: Kargo's expr-lang is not nil-safe against a fully-skipped upstream step's output (`cannot fetch id from <nil>`), so this failure mode surfaces even inside the *guard* condition meant to prevent it. Authoring a genuinely no-op-safe, protected-branch `promotionTemplate` requires defensive expression handling that is non-obvious from Kargo's schema or default examples, and was not caught until runtime.

No true concurrent-*mutation-conflict* scenario (two different Freight racing for the same Stage, or an actual Git push conflict) was exercised — only one Freight was ever discovered in this session's Warehouse, and time was prioritized toward the mandatory scenarios that were reachable with available artifacts.

**Answer to the framing question:** partially — Kargo gives good idempotency detection out of the box (avoiding redundant commits/PRs), but the specific PR-gated template pattern this D0 acceptance condition requires does not compose safely with that idempotency without extra, non-obvious authoring care that Kargo's tooling does not surface proactively.

## RBAC / Promotion Authority
Two distinct findings, one concerning and one reassuring:

- **Concerning:** the `kargo-controller-argocd` ClusterRole grants `get/list/patch/watch` on **all** `argoproj.io Applications`, cluster-wide — not scoped to any specific Project, Stage, or namespace. This is a broad capability grant at the Kubernetes RBAC layer.
- **Reassuring:** despite that broad *capability*, Kargo's own controller code enforces a separate, additional authorization gate: it refused to mutate the `d1-dev` Argo Application ("skipping unauthorized Application ... does not permit mutation by Kargo Stage dev in namespace d1-sandbox") until the Application carried an explicit `kargo.akuity.io/authorized-stage: <project>:<stage>` annotation, added imperatively in this session. This is a genuine defense-in-depth layer *in addition to* Argo's own AppProject boundary — but it constrains Kargo's *behavior* (code-level check), not a compromised or buggy controller's *capability* (the broad ClusterRole still technically permits it).

The architecture's stated invariant — "Kargo must never become a second business approval authority" — was not violated by anything observed: Kargo's PR-wait step defers entirely to ADO's own PR/branch-policy mechanism for human authorization, and nothing in this session's Kargo configuration expressed or required its own approval semantics beyond that.

## Operational Footprint
| Component | Namespaces | Pods | New CRDs | Notes |
|---|---|---|---|---|
| Kargo itself | `kargo` (+ auto-created `kargo-shared-resources`, `kargo-system-resources`) | 5 (api, controller, external-webhooks-server, management-controller, webhooks-server) | 9 | ~1-2m CPU / 25-47Mi memory per pod at idle |
| cert-manager (**mandatory** dependency) | `cert-manager` | 3 | included in CRD count via jetstack chart | Kargo's Helm install hard-fails without it: `no matches for kind Certificate` |
| Argo Rollouts (**mandatory** for verification) | `argo-rollouts` | 2 | 5 (rollouts, analysisruns, analysistemplates, clusteranalysistemplates, experiments) | Required to exercise §7.4 at all; Kargo silently degrades without it |
| **Total new CRDs cluster-wide** | | | 57 (post) − 37 (pre) = **20** | |
| **Kargo-related RBAC objects** | | | | **19** ClusterRoles + **16** ClusterRoleBindings |

Kargo auto-creates its own project namespace (`d1-sandbox`) and two cluster-wide "system resource" namespaces on first `Project` creation — this is convention, not something the platform configures.

**Answer to the framing question:** the footprint is not small — three operators, ~20 CRDs, 35 RBAC objects, plus non-trivial authoring friction (two distinct expr-lang field-naming gotchas, one nil-safety gotcha, one dropped-config UX trap) discovered only by hitting them at runtime. Whether this is "justified by the custom code it removes" depends heavily on how much of §7.1–7.7's capability the platform would otherwise have had to build itself (see comparison table). At idle, raw compute footprint is genuinely trivial (~10 pods, low tens of MB each); the real cost is operational/cognitive surface area (CRDs, RBAC, authoring correctness) rather than resources.

## Kargo vs Thin Alternative
| Concern | Kargo | Thin/direct Git alternative | Assessment |
|---|---|---|---|
| Artifact discovery | Warehouse polls registry, creates immutable Freight records automatically | CI supplies digest to a platform API, or platform polls registry itself | **Kargo wins** — this is real, working, out-of-box machinery a thin path must otherwise build |
| Stage promotion / Freight tracking | Native `Stage.status.currentFreight`, full Freight history, alias naming | Platform's own promotion-state table (target → current release fingerprint) | **Kargo wins** — but the thin version is a straightforward CRUD table, not exotic |
| Git mutation (direct push) | Loudly refuses on a protected branch (correct, but means it cannot fully automate there) | Same limitation applies to any writer against the same policy | **Tie** — neither can bypass ADO branch policy, nor should either try |
| Git mutation (PR-gated) | `git-open-pr` + `git-wait-for-pr` work correctly, but required real authoring fixes (field names, nil-safety) not visible from docs/schema | A thin writer implementing "open PR, poll for merge" is ~50-100 lines against the ADO REST API in a general-purpose language with ordinary error handling | **Close** — Kargo's version exists out of the box; the thin version is not exotic to build and arguably easier to debug (normal stack traces vs. expr-lang error messages) |
| Verification | Real Job-backed AnalysisRun, pass/fail gate on promotion — but requires installing and operating Argo Rollouts as a second controller | Argo health projection + one explicit smoke-test Job triggered by the platform's own writer, no second controller | **Thin wins on footprint**; Kargo wins on having a structured pass/fail *record* (AnalysisRun) rather than an ad hoc Job the platform must track itself |
| Conflict/retry (idempotency) | Genuine no-op detection (skips redundant commits/PRs) but does not compose safely with a PR-wait step without non-obvious defensive authoring | Platform writer checks "is target already at this digest" before mutating — an ordinary, debuggable if-statement | **Thin wins on simplicity/predictability** for this specific interaction; Kargo's underlying detection is real and correct, but its composition with a protected-branch PR flow needs care Kargo does not surface |
| History / audit | Freight + Promotion objects persist in-cluster as the history | Platform's own minimal audit record (target, fingerprint, Git revision, outcome, timestamps) — exactly what D0's review already said Delivery needs regardless | **Tie-leaning-thin** — Kargo's history exists, but D0 already established Delivery needs its own durable audit record *anyway* (provider retention is not guaranteed), so this is not incremental value unique to Kargo |
| Operational footprint | 3 operators, 20 new CRDs, 35 RBAC objects | 0 new operators; platform writer is one more service alongside existing infra | **Thin wins decisively** on footprint; Kargo's footprint is the single largest cost line in this comparison |
| Restart/recovery | Fully durable via Kubernetes CRs; genuinely proven to survive a hard controller kill mid-promotion | A stateless writer service reading "did I already push this commit" from Git/its own DB on restart is an equally standard pattern | **Tie** — both are achievable; Kargo's version is proven and free, the thin version requires the platform to build its own idempotent-restart logic correctly |

## GitOps Layout Observations
Option A (one protected branch + stage directories) was used throughout and worked cleanly with Kargo: a single `d1/desired-state` branch, protected once, with `stages/dev` and `stages/hml` subdirectories referenced by distinct Argo `Application.spec.source.path` values. Kargo's `git-clone`/`yaml-update`/`git-commit` steps operate naturally on a path-scoped subtree of a single branch; no friction was observed from having both stages' desired state coexist on one branch.

Option B (stage-specific branches: `stage/dev`, `stage/hml`, `stage/prd`) was not exercised, but based on Option A's behavior, it would require Kargo's `checkout`/`targetBranch` config to point at different branches per Stage template rather than different paths — mechanically equivalent effort, but would also require **branch policy be applied and maintained per stage branch** rather than once, and each stage's PR flow would then target a different, separately-protected branch. Option A concentrates policy management in one place; Option B would spread it across as many branches as stages, at what appears to be no compensating benefit for this narrow evaluation. This observation is not a recommendation to finalize Option A company-wide — only that it introduced no friction with Kargo specifically.

## Carried D0 Gaps
All three remain open and unresolved, exactly as required — none were solved by D1, and none should be read as solved:

- **Argo/Kubernetes authority** (Concern A): unchanged. The Argo `argocd-application-controller` ClusterRole remains cluster-admin-equivalent in this sandbox; D1 additionally observed that Kargo's own controller ClusterRole grants broad, un-scoped Application patch rights cluster-wide. Neither was narrowed. Required future proof (a supported least-privilege Argo topology with a negative Kubernetes RBAC test) remains outstanding.
- **Git writer vs. reconciler identity separation** (Concern B): **partially addressed, not resolved.** Kargo was given a genuinely separate PAT (`d1-kargo-writer`, scoped Code Read&Write) distinct from Argo's read credential — a real improvement over D0's single shared PAT. However, this PAT is still tied to the same human ADO account (`diego.fernandes@outlook.com`) rather than a true non-human service identity; an attempt to provision one via ADO's PAT REST API failed with `VS403363` (the session's authenticated identity lacks work/school-account access to that surface), and this session's environment offered no other mechanically simple path to a non-human credential. The open question from D0 — "what non-human identity issues the reconciler/writer credential, and can Azure DevOps scope it to read-only/read-write for a non-human identity" — remains genuinely unanswered, now with first-hand evidence that it is not a mechanically trivial question for this ADO organization's account model.
- **AppProject/control-object governance**: unchanged, and now with more surface. Both D0's and D1's `AppProject`/`Application` objects remain outside Git management, applied imperatively via `kubectl apply`. D1 additionally introduced Kargo's own `authorized-stage` annotation as an imperative, out-of-Git control mechanism layered on top. Who may write `Application`/`AppProject` CRs (and now, who may set Kargo's authorization annotations) remains an open question, to be evaluated together with the eventual GitOps layout decision as D0's review specified.

**Newly surfaced, D1-specific gap:** ADO's own branch-policy evaluation service stalled once during this session (a PR's "Minimum number of reviewers" policy sat in `status: queued` indefinitely despite a real satisfying vote, and a manual re-queue via `az repos pr policy queue` did not unstick it; the PR was ultimately completed successfully via the ADO web UI, which evidently resolves state the API read was not reflecting). This is an ADO/tenant reliability finding, not a Kargo behavior, but it is directly relevant to any PR-gated promotion path's practical latency: a stalled policy-evaluation service can silently block an otherwise-correct promotion indefinitely, and the only way this was detected was by directly querying policy-evaluation state (`az repos pr policy list`) — not from the PR object's own top-level fields.

## Decision Rationale
Kargo cleared the D0-raised bar: it does not merely reprove GitOps reconciliation (D0 already did that), and it does materially remove real promotion-specific machinery — immutable-artifact tracking, protected-branch PR-based mutation with correct wait/resume semantics (proven durable across a real controller restart), and Job-backed verification with a structured pass/fail record. None of §11's STOP conditions were triggered: Kargo did not need to become a second approval authority (it correctly deferred to ADO's own PR/policy mechanism), did not force provider-specific fields into canonical Change Management, did not require broad production credentials as an inherent model (the sandbox credential gap is a provisioning question, not an architectural requirement), did not need a generic workflow/rules language the MVP would have to own, its Git mutation *did* work with a realistic protected repository path (after switching to the PR-gated step, which is the correct behavior, not a workaround), and neither Backstage nor Change Management needed any modification to make the D1 POC function.

The verdict is qualified `KARGO_FIT` rather than unqualified because: (a) the operational footprint (3 operators, 20 CRDs, 35 RBAC objects) is real and non-trivial, and its justification depends on the MVP actually using enough of what Kargo provides to be worth that cost rather than adopting it for reconciliation alone (which D0 already proved doesn't need it); (b) authoring correct, no-op-safe `PromotionTemplate`s for a protected-branch flow required non-obvious defensive expression-language handling not caught until runtime, meaning template authoring is a real, ongoing engineering cost, not a one-time setup; and (c) two of the carried D0 gaps (Git identity separation, control-object governance) were only partially advanced, not closed, and D1 surfaced that closing them may be less mechanically simple in this ADO organization than assumed.

## If KARGO_FIT
- **Exact capabilities we will use in the MVP:** Warehouse-based artifact discovery and Freight identity tracking; Stage-based promotion with PR-gated Git mutation against protected branches (using `git-open-pr`/`git-wait-for-pr`, never direct `git-push` against any policy-protected target); Job-backed `AnalysisTemplate` verification for at least the production-request gate; Kargo's per-Application `authorized-stage` annotation as an additional authorization layer on top of Argo's AppProject.
- **Capabilities explicitly not adopted:** Kargo's own approval/authorization semantics as a substitute for GMUD/Change Management (Delivery/Change retains sole authorization authority; Kargo only executes after a fresh ALLOW, exactly as ADR-012 already states); Kargo's Promotion history as the durable audit record (Delivery still needs and will build its own minimal audit record per D0's review, independent of Kargo's in-cluster history); any stage-specific-branch (Option B) GitOps layout without a separate, explicit decision; Kargo's admin UI/API as an end-user-facing surface (Backstage composes the UX, per the existing architectural invariant).

## Documentation Updated
- `docs/delivery/d1-kargo-fit-evaluation.md` (new) — full evidence record, this report's content
- `docs/delivery/README.md` — gate line updated to record the D1 verdict (see below); ADR-012 status left as `Proposed`

No production authorization, GMUD/Change integration authorization, or ADR-012 Accepted status was recorded, per the prompt's explicit constraints.

## Gate
STOP
