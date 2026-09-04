# D0 Architecture Review

## Review Baseline
- Canonical docs SHA: `47c978dbbb4fb5b408800de5535e860bb6d0460d`
- D0 evidence: `docs/delivery/d0-gitops-reconciliation-evidence.md` blob `6fc332105ca05c03dd2ab5d8867cfff9d25b0b38`; execution record references docs baseline `076dd03e6605580ac86b6e3f33179d2fc981a2dd`
- GitOps evidence repo/revision: `diegolab/platform-engineering/d0-gitops-sandbox@19dd327e976ed1ea695f74ce003302262a912cc1`, tracked branch `d0/desired-state`; raw evidence referenced as `evidence/D0-EVIDENCE.md`

The review used the exact `origin/main` canonical versions of ADR-012, the Delivery architecture spike, branching/release refinement, D0 checkpoint, D0 execution evidence, and F3.1.1a architecture acceptance. The current canonical SHA matched the state described when the review was requested: D0 `CONDITIONAL PASS`, D1 `NO-GO` pending architecture review.

The raw Azure DevOps GitOps evidence repository referenced by the canonical record was not independently fetchable during the first review pass. Therefore command-level observations were initially challenged against the canonical execution evidence rather than re-executed. This limitation does not convert recorded POC observations into production guarantees.

**Supplementary verification pass.** That limitation was subsequently lifted: `d0-gitops-sandbox@19dd327e` was fetched and its raw `evidence/D0-EVIDENCE.md` (400 lines) and all six manifests were read, and the sandbox cluster was re-read read-only (Argo CD `v3.5.2`; Application `d0-sandbox` `Synced / Healthy` @ `19dd327e`). No gate decision in this document changed as a result. The verification confirmed both authority findings, corrected the attributed cause of Concern B, downgraded two properties from `PROVEN` to `PARTIALLY_PROVEN`, and surfaced three observations not present in the canonical execution record. Those are marked **[verified]** below.

## D0 Verdict
ACCEPT_CONDITIONAL_PASS

## What D0 Proved
D0 sufficiently proves its narrowly defined objective: a known immutable artifact can be declared in Git, reconciled by Argo CD into a sandbox Kubernetes target, observed through independent sync and health axes, remediated after runtime drift, and reverted through Git without requiring a custom deployment workflow engine, callback receiver, or imperative `argocd app sync` as the normal mechanism.

The A-I evidence is internally coherent with that objective. In particular, the evidence records A→B promotion, B→A Git revert, self-heal with unchanged Git revision, polling-only convergence, `Synced + Degraded` failure visibility, AppProject negative source/destination probes, and both prune-disabled and prune-enabled behavior. The immutable artifact reference was correlated across desired state, Argo revision, and runtime for the exercised sandbox path.

D0 therefore establishes Git → Argo → Kubernetes as a viable execution substrate for the next architecture experiment. It does not establish production readiness.

## What D0 Did Not Prove
D0 did not prove production-grade Kubernetes execution authority, separation of Git writer and reconciler identities, production repository mutation through a protected path, platform-owned artifact provenance/signature, multi-cluster isolation, critical-resource deletion policy, or end-to-end Delivery/Change governance.

The two authority findings are not cosmetic. Both are mandatory production security properties. They remain open because the POC topology did not prove them, not because the architecture permits ignoring them.

**[verified]** Three further gaps surfaced by the raw evidence, none of which change a gate decision:

- **Defence in depth of the AppProject boundary itself was not tested.** The Argo `Application` and `AppProject` CRs are not under Git management — the Application tracks path `app/` only, and `argocd/*.yaml` in the sandbox repo is documentation, not reconciled state. Scenario I's own output shows the control object mutated imperatively (`application.argoproj.io/d0-sandbox configured`) to flip `prune`. The negative probes were therefore non-adversarial: no probe asked what happens when an actor edits the AppProject or the Application, which is the realistic bypass.
- **Portability of the runtime correlation method.** The digest readback was `docker-pullable://nginx@sha256:…`, a dockerd-backed Rancher Desktop shape. On CRI/containerd production nodes the readback shape differs and may not equal the multi-arch index digest.
- **Scenario H is scored `PASS (documented)`, but what H documented is Concern A.** Scoring the discovery of a negative finding as a passing scenario overstates the A-I result. This does not change the `CONDITIONAL PASS` verdict; it is one more reason not to upgrade it.

## Concern A — Argo Kubernetes Authority
Classification:
PRODUCTION_HARDENING_GAP

Assessment:
The observed stock `argocd-application-controller` authority equivalent to `*/*/*` is unacceptable as the sole production execution posture. The D0 AppProject proved useful application-level policy: allowed source, destination and resource kinds can reject invalid Applications before deployment. That is a meaningful guardrail, but it is not sufficient as the only blast-radius boundary when the controller credential itself can mutate resources outside the intended target if the project boundary is misconfigured, bypassed through a privileged administrative path, or changed by an overly privileged identity.

This does not invalidate the Git → Argo → Kubernetes architecture. Broad controller authority is not inherent to that direction: Argo supports namespace-scoped installation patterns and target-cluster credentials/service accounts that can be restricted to selected namespaces. Therefore the D0 result exposed an unproven defense-in-depth property, not an architectural impossibility.

**[verified] The gap is a topology artefact, and the evidence does not draw this distinction.** Direct inspection confirmed the controller ClusterRole is literally `[{apiGroups:["*"],resources:["*"],verbs:["*"]},{nonResourceURLs:["*"],verbs:["*"]}]` — the `nonResourceURLs: *` rule is not mentioned in the execution record. More importantly, **no cluster secrets are registered** (`argocd.argoproj.io/secret-type=cluster` returns nothing), so D0 ran the single worst case: control plane and workload in one cluster, with the controller reconciling through its own service account. The most mature mitigation — remote destinations registered as cluster secrets carrying namespace-scoped service-account tokens, where the authority granted is exactly the token's RBAC — was *structurally excluded by the POC topology*, not evaluated and rejected. Separately, per-destination impersonation exists in the installed v3.5.2 but was confirmed off: `argocd-cmd-params-cm` carries no data and the AppProject declares no `destinationServiceAccounts`. Its maturity and upgrade posture must be verified before it is relied on; it is not accepted here.

Production architecture should treat AppProject and Kubernetes RBAC as complementary layers. AppProject remains the logical application guardrail; Kubernetes authorization must bound the actual execution identity so compromise or policy error does not automatically imply cluster-wide mutation.

Required future proof:
Before any production rollout, demonstrate at least one supported Argo topology in which the reconciler for a representative production target can reconcile the required namespaced workload/resources while Kubernetes authorization denies mutation outside the authorized target scope. The proof must include a negative `can-i`/equivalent check and an attempted out-of-scope reconciliation or mutation that is denied by Kubernetes RBAC, not only by AppProject. The selected topology may use destination-specific service accounts, restricted cluster credentials, namespace-scoped authority, restricted controller roles, or another supported least-privilege pattern. D1 does not need to choose the final production topology unless Kargo evaluation depends on it.

**[verified]** Add one further required answer: **who may write `Application`/`AppProject` CRs**, since that authority subsumes the AppProject boundary entirely. Bringing the Argo control objects under a governed Git path is the obvious candidate and should be evaluated together with the GitOps layout decision, because those objects would inherit whatever protection model that decision selects.

## Concern B — Git Credential Authority Separation
Classification:
PRODUCTION_HARDENING_GAP

Assessment:
Read/write identity separation is an architectural security invariant for the target model. The reconciler needs authority to read desired state; the identity that mutates desired state has materially greater authority and must be separately attributable and constrained. Sharing one broad PAT collapses those trust domains and would be unacceptable in production.

HTTPS + PAT is not itself the problem. Protocol choice is orthogonal to the security property. A properly scoped read-only HTTPS credential for Argo and a distinct writer identity can satisfy the architecture just as an SSH-based implementation could. SSH is not a requirement.

**[verified] The execution record misattributes its own cause, and the correction matters.** Deviation 1 records the cause as *"SSH Git is broken org-wide, therefore both used the same PAT."* That reasoning does not hold: an Azure DevOps SSH key is registered against a **user** and carries that user's repository permissions, so had SSH worked, Argo would have been handed a key belonging to a writer and the separation would have been *equally* absent. The SSH breakage is a real POC obstacle but it is not why separation failed. The question D0 did not ask is the one that actually matters: **what non-human identity issues the reconciler credential, and can Azure DevOps scope that identity to read-only on a single repository?** That is a provisioning and identity-model question — service principal / workload identity / dedicated service account with repo-scoped Read — and it is genuinely open, because ADO PAT scopes are account- and scope-based rather than natively per-repository. It belongs in D1/D2 planning and is more consequential than the SSH breakage the record foregrounds.

The shared POC PAT does not invalidate D0 because D0's objective was reconciliation correctness in a sandbox and did not depend on proving production repository authorization. It does mean no production security conclusion may be drawn from the repository credential setup.

Required future proof:
Before production, demonstrate a dedicated reconciler credential with repository read-only access, a separate human/promotion writer identity, repository permissions that deny write from the reconciler, and no shared broad credential spanning both roles. The test should prove both the positive path (Argo can still reconcile) and the negative path (the Argo identity cannot mutate the desired-state repository). **[verified]** The proof must also record *what issues, rotates and audits* both identities — including whether ADO can express repo-scoped read-only for a non-human identity, and the fallback if it cannot.

## Property Assessment
| Property | Result | Reason |
|---|---|---|
| Git desired-state authority | PROVEN | All exercised normal state transitions originated in Git and Argo reconciled them; no imperative sync was required. |
| Argo reconcile without custom workflow engine | PROVEN | A-I were exercised without a custom deployment state machine, orchestration service, or callback receiver. |
| Immutable artifact identity correlated end to end | **PARTIALLY_PROVEN** *(downgraded on verification)* | Correlation held exactly in every scenario, but on one dockerd-backed node: the readback was `docker-pullable://nginx@sha256:…`, whose shape differs on CRI/containerd and may not equal the multi-arch index digest. The correlation *method* is therefore not proven portable to the production runtime. Artifacts were public upstream images, so provenance is untested. |
| Git revert is a coherent recovery path | PROVEN | B→A and failure/prune recovery returned desired and runtime state through Git history. |
| Self-heal remediates runtime drift | PROVEN | Runtime replica drift was restored while Git revision remained unchanged. |
| Sync and health are sufficiently distinct | PROVEN | `Synced + Degraded` demonstrated that desired state can be applied while workload health fails. |
| Webhooks are not required for correctness | PROVEN | Polling-only reconciliation converged repeatedly; webhooks remain a latency accelerator. |
| AppProject enforces source/destination policy | **PARTIALLY_PROVEN** *(downgraded on verification)* | Forbidden source and destination probes were rejected pre-cluster with `InvalidSpecError`, and nothing reached `kube-system`. But the probes were non-adversarial, and the `Application`/`AppProject` CRs sit outside Git and were imperatively patched during the run — the realistic bypass (an actor editing the boundary objects) was never tested. This does not prove Kubernetes least privilege. |
| Production Kubernetes authority safely scoped | NOT_PROVEN | Controller authority was cluster-admin-equivalent and Kubernetes RBAC did not bound sandbox blast radius. |
| Git writer and reconciler identities separated | NOT_PROVEN | Human/operator and Argo used the same broad PAT. |

## Provider State / Delivery State
The `Synced + Degraded` observation supports the architectural conclusion that a Git commit is not equivalent to a successful deployment. D1 does not need a new canonical deployment lifecycle merely to distinguish those states.

Operational sync, health, resource conditions, reconciliation revision, and provider-native verification details should remain provider projections owned by Argo/Kargo/Kubernetes or other execution providers. Delivery may normalize those observations for UX, but should not copy every provider transition into an independently authoritative platform state machine.

Delivery will eventually need a small durable evidence record that survives provider retention, such as deployment request/correlation identity, target, release fingerprint, desired-state Git revision produced or selected, provider execution correlation, start/finish timestamps, terminal outcome, and verification/evidence references where applicable. This is audit evidence, not a second reconciliation engine.

Do not copy Argo's full sync state machine, Kubernetes rollout conditions, every Kargo Promotion phase, retry internals, resource trees, or transient controller events into canonical Delivery lifecycle semantics.

## Artifact Identity
The future `ReleaseCandidate` contract should state a provider-neutral invariant: the artifact fingerprint identifies the immutable logical artifact that is promoted between targets.

For OCI multi-platform images, that will normally be the OCI image index / manifest-list digest because it identifies the complete promotable multi-platform artifact; per-platform manifest digests identify one platform-specific child. The contract should still allow an explicitly platform-specific release when that is the actual promotable unit. This rule belongs in the `ReleaseCandidate` artifact identity contract, not in branching or Change Management, and must not be generalized to non-OCI artifact types without an artifact-specific identity rule.

## Rollback / Revert
D0 proves a technical recovery mechanism only: a desired-state Git revert can move the runtime from B back to A through ordinary reconciliation while preserving Git as authority.

It does not prove that a production rollback is automatically authorized, that every rollback may bypass Change policy, or that rollback should be triggered automatically. Technical ability and governance authorization remain separate decisions. ADR-012's deferral of automatic production rollback remains appropriate.

## Prune
D0 proves that prune behavior is understood and observable in the sandbox: with prune disabled, deletion intent remains visible as drift requiring pruning; with prune enabled, Argo deletes the removed managed resource.

**[verified] Two observations the execution record understates.** First, prune was flipped by imperatively patching the Application — an object outside the GitOps loop — so the prune *policy* was not itself under desired-state control. Second, **restore was not identity-preserving**: the Service ClusterIP moved `10.43.147.50` → `10.43.218.94` across prune and Git restore. For a Service this is nearly harmless; for a `LoadBalancer` (external IP), a `PersistentVolumeClaim` (data), or anything with external DNS or firewall dependencies, "delete and recreate from Git" is not a round trip. This is concrete evidence that critical-resource deletion needs its own controls rather than a single global prune flag.

It does not accept automatic production prune. Critical deletion still requires a later policy decision with appropriate layered safeguards such as protected Git review, admission/policy controls, resource protection mechanisms, or explicit deletion policy. The implementation choice remains open.

## GitOps Layout
D0's temporary `d0/desired-state` branch must not decide the final repository topology. Both one protected branch + environment directories and stage-specific desired-state branches remain valid D1/D2 candidates.

A production-like desired-state mutation path will almost certainly be protected and attributable. D1 should explicitly exercise a representative protected Git path before declaring `KARGO_FIT`, because promotion-controller value includes Git mutation, conflict/retry behavior, concurrency, authorization and operational ergonomics. This is a D1 acceptance condition, not a reason to rerun D0.

D1 may use a sandbox repository/namespace, but its Git mutation experiment should represent the intended protection model closely enough to expose whether Kargo can work with the organization's review/ruleset constraints. The test must not silently normalize an unprotected branch as the production design.

## Kargo Hypothesis After D0
D0 materially raises the bar for Kargo. Plain Git → Argo → Kubernetes already demonstrated reconciliation, failure visibility, drift repair and Git-based recovery without custom orchestration. Therefore Kargo cannot justify itself by claiming to provide GitOps reconciliation.

D1 must identify incremental value in the promotion problem itself: artifact discovery, stage promotion, verification orchestration, Git mutation with safe conflict/retry handling, concurrency control, promotion policy, history, stage abstraction and operational visibility. It must compare that value against a simpler controlled Git mutation path or a narrowly scoped provider/service.

D1's decision remains binary at the candidate level: `KARGO_FIT` only if the product removes enough real promotion machinery and authority/concurrency burden to justify its operational footprint; otherwise `KARGO_NO_FIT`, with a precise thin-provider/direct-controlled-Git alternative. The experiment must not become “how to make Kargo fit.”

## D1 Gate
GO_D1_WITH_CARRIED_GAPS

The two authority gaps do not materially invalidate the D0 execution-substrate result and do not require production hardening before the promotion-controller comparison can run in a sandbox. They are mandatory carried gaps, and D1 must not convert the POC's broad RBAC or shared PAT into accepted architecture.

## Conditions / Follow-ups
- D1 remains sandbox-only: no production cluster, namespace, credential, rollout, GMUD integration, Delivery implementation, or Backstage implementation.
- No production inference may be drawn from the D0 controller's `*/*/*` authority.
- Shared broad Git credentials are explicitly non-acceptable for production; read/write identity separation remains mandatory proof before production.
- D1 must exercise a representative protected Git desired-state mutation path before `KARGO_FIT` can be concluded.
- D1 must compare Kargo against direct controlled Git mutation / a thin Delivery provider, including conflict/retry, concurrency, verification, audit/history and operational footprint.
- **[verified]** D1 must record the runtime digest readback method on its target container runtime, alongside the declared index digest, so the artifact correlation method is known rather than assumed.
- **[verified]** The production-readiness checkpoint must also answer who may write `Application`/`AppProject` CRs, evaluated together with the GitOps layout decision.
- A production-readiness checkpoint later must prove Kubernetes least privilege for the Argo execution identity and repository read/write identity separation with negative tests.
- AppProject remains one defense-in-depth layer, not the only production security boundary.
- ADR-012 remains `Proposed`.
- F3.1.1b remains separate and `NO-GO`.
- F3.1.2+ remains `NO-GO`.
- GMUD implementation remains unchanged.
- Backstage Delivery implementation remains `NO-GO`.
- Production rollout remains `NO-GO`.

## Documentation Updated
Created this checkpoint, then amended it in place with a supplementary raw-evidence verification pass (entries marked **[verified]**). No gate decision changed; two property classifications were downgraded from `PROVEN` to `PARTIALLY_PROVEN` and the attributed cause of Concern B was corrected.

Historical D0 execution evidence and observed facts were not rewritten. The D0 execution evidence document's only edit is its Status line, pointing here. ADR-012 was not moved to Accepted.

## Final Gate
STOP
