# D1 — Promotion Controller Fit Evaluation
## Narrow execution prompt — decide Kargo vs thin/direct Git path

You are acting as a **Senior Platform Engineer executing a tightly scoped architecture POC**.

Your job is to execute **D1 only**.

The goal is not to build the final Delivery platform. The goal is to answer one practical question quickly:

> **Does Kargo materially reduce the promotion/orchestration burden between immutable release material, stage progression, Git desired state, verification, and Argo CD — enough to justify its operational footprint — compared with a much thinner controlled-Git alternative?**

The required final decision is exactly one of:

```text
KARGO_FIT
KARGO_NO_FIT
```

Do not optimize for making Kargo succeed. A clean `KARGO_NO_FIT` is a valid and useful result.

The wider program now prioritizes a demonstrable vertical MVP. D1 must therefore be **short, evidence-driven, and decisive**. Avoid turning this checkpoint into a broad production-hardening exercise.

---

## 0. Canonical sources and mandatory starting protocol

Canonical documentation repository:

```text
diegofernandes-dev/backstage-docs
branch: main
```

Before doing anything:

```bash
git fetch origin main
```

Verify the exact current `origin/main` and read the current versions of at least:

```text
docs/adr/ADR-012-delivery-management-gitops-promotion.md
docs/delivery/architecture-spike.md
docs/delivery/branching-release-control.md
docs/delivery/d0-gitops-reconciliation-evidence.md
docs/delivery/d0-architecture-review.md
docs/delivery/README.md
```

Do not reconstruct architecture from memory.

The current gate must state, semantically:

```text
D0 = ACCEPT_CONDITIONAL_PASS
D1 = GO_D1_WITH_CARRIED_GAPS
ADR-012 = Proposed
```

If the current canonical docs materially differ, use the current canonical state and report the discrepancy before execution.

---

## 1. What D0 already proved — do not redo it

D0 already demonstrated that the basic execution substrate can be:

```text
Git
  -> Argo CD
  -> Kubernetes
```

without:

```text
custom deployment state machine
custom orchestration service
callback receiver
imperative argocd app sync as the normal deployment contract
```

D1 must not spend its time re-proving basic GitOps reconciliation, self-heal, Git revert, polling correctness, or ordinary Argo health behavior except where needed to validate Kargo interaction.

D0 also left carried production-hardening gaps, including:

```text
production-grade Argo/Kubernetes authority scoping not proven
Git writer vs reconciler identity separation not proven
AppProject/control-object defense in depth only partially proven
portable artifact-runtime correlation only partially proven
```

These remain real, but **they do not block D1**.

Do not fix them unless a very small temporary sandbox adjustment is necessary to execute D1. Do not claim they are solved.

---

## 2. D1 scope

Evaluate Kargo specifically as a **promotion controller** between immutable release material and Git/Argo reconciliation.

The target conceptual path is:

```text
immutable artifact
      ↓
Kargo Warehouse / Freight
      ↓
DEV Stage
      ↓
HML Stage
      ↓
verification
      ↓
Git desired-state mutation
      ↓
Argo CD
      ↓
Kubernetes
```

Production GMUD/change integration is **out of scope**.

D1 is not allowed to implement the final ReleaseCandidate/DeploymentRequest/ChangeBinding platform model.

---

## 3. Explicit non-goals

Do NOT implement:

```text
GMUD integration
ExecutionEligibility
ChangeBinding
Delivery database
Backstage Delivery UI
Backstage Kargo plugin as a product decision
Azure DevOps Environment checks
Teams/CAB
F3.1.1b
F3.1.2+
production clusters
production credentials
production rollout
custom generic workflow engine
custom generic state machine
full production Argo RBAC redesign
final organization-wide Git branching policy
```

Do not use D1 as an excuse to start D2/D3/D4.

---

## 4. Sandbox requirement

Use only sandbox/local/test infrastructure.

Reuse the D0 substrate if practical rather than rebuilding the world.

Prefer:

```text
existing sandbox Kubernetes
existing Argo CD
existing disposable GitOps repo/path
public or disposable OCI artifact source
```

No production credentials or namespaces.

---

## 5. Decision philosophy

Kargo must justify itself against an already-simple baseline.

The comparison baseline is not a giant custom controller. It is intentionally thin:

```text
controlled platform action/service
      ↓
select exact immutable artifact
      ↓
mutate protected Git desired state
      ↓
Argo reconciles
```

Therefore Kargo is valuable only if it materially removes difficult promotion concerns such as:

```text
artifact discovery
stage progression
verification orchestration
promotion history
Git mutation
conflict/retry handling
concurrency
stage locking/serialization
promotion policy expression
operational visibility
recovery after controller restart/event loss
```

Do not count functionality already provided by Argo CD as a Kargo advantage.

---

## 6. Minimal D1 topology

Use the smallest topology that can prove Kargo's incremental value.

Example:

```text
OCI artifact source
      ↓
Warehouse
      ↓
Freight
      ↓
DEV Stage
      ↓
HML Stage
      ↓
Promotion
      ↓
Git desired state
      ↓
Argo CD
      ↓
Kubernetes sandbox
```

PRD may be represented only as a **sandbox stage concept** if needed to evaluate authorization boundaries, but no real production semantics or GMUD integration may be added.

Do not add more stages or clusters than necessary.

---

## 7. Required evaluation areas

D1 must answer the following with executed evidence where practical.

### 7.1 Artifact discovery and immutable identity

Prove or disprove that Kargo can discover/promote immutable OCI material in a way that aligns with the future `ReleaseCandidate` concept.

Capture:

```text
source artifact/tag if any
immutable digest/fingerprint
Warehouse observation
Freight identity
Stage-selected material
Git desired-state material
runtime material where observable
```

Do not build a final platform-owned ReleaseCandidate model.

Question:

> Does Kargo reduce our need to build artifact discovery/promotion bookkeeping ourselves?

---

### 7.2 DEV → HML promotion semantics

Exercise at least one real promotion from DEV to HML.

Expected conceptual behavior:

```text
artifact A
  ↓
DEV
  ↓
verification / readiness condition
  ↓
HML
```

The same immutable artifact identity must move forward. No rebuild between stages.

Determine whether Stage/Freight/Promotion semantics are understandable enough to map cleanly behind a provider-neutral Delivery boundary.

---

### 7.3 Git mutation

This is mandatory.

Prove how Kargo changes desired state in Git.

Record:

```text
which identity writes Git
which branch/path is changed
commit shape/message
how artifact identity is encoded
what happens on concurrent/conflicting changes
how retry/recovery works
```

Prefer exercising a **representative protected Git path** if practical, because the D0 review explicitly identified this as a D1 acceptance concern.

Do not weaken branch protection merely to produce a green result without documenting the deviation.

Question:

> Is Kargo's Git mutation materially safer/simpler than writing a thin controlled Git mutation service ourselves?

---

### 7.4 Verification

Exercise one minimal verification mechanism after promotion.

Do not build a broad testing framework.

The goal is to determine whether Kargo's verification model provides useful orchestration value beyond Argo's `Synced/Healthy` projection.

Capture:

```text
what verification ran
where its result lives
how failure affects promotion
how retry behaves
what minimum state Delivery would need to project
```

---

### 7.5 Failure visibility

Introduce one safe promotion/verification failure.

Examples:

```text
invalid artifact reference
failed verification
Git mutation conflict
intentionally failing target state
```

Observe whether Kargo exposes the failure clearly enough that a future Delivery UI could project it without inventing a second workflow engine.

Do not copy Kargo's full internal lifecycle into platform semantics.

---

### 7.6 Concurrency and idempotency

Exercise at least one meaningful concurrency/idempotency scenario.

Examples:

```text
two promotions toward the same Stage
duplicate promotion request
newer Freight arriving while older promotion is active
Git conflict from concurrent mutation
```

The exact scenario should target the hardest behavior Kargo claims to remove from custom code.

Answer:

> Does Kargo give deterministic enough behavior that we avoid implementing same-target locking, retry and deduplication ourselves?

---

### 7.7 Restart / reconciliation recovery

At minimum inspect and, if cheap, exercise controller restart/reconciliation behavior.

The key question is:

> If an event/callback is missed or the controller restarts, can Kargo reconstruct/reconcile current promotion state from durable Kubernetes/Git facts without our platform becoming the recovery engine?

Do not build an external reconciler.

---

### 7.8 RBAC / promotion authority

Do not solve production security, but inspect whether Kargo can distinguish ordinary read/observe rights from promotion authority, especially for a future production Stage.

Prove or document whether ordinary developer credentials can be prevented from directly promoting the restricted Stage.

The architecture invariant is:

> Kargo must never become a second business approval authority. It may execute a promotion only after the future Delivery/Change layer permits it.

Do not add GMUD to prove this.

---

### 7.9 Operational footprint

Capture enough operational facts to decide whether Kargo is worth owning.

At minimum:

```text
components/controllers installed
CRDs introduced
resource footprint at rough order of magnitude
required credentials/secrets
upgrade/API maturity observations
failure surfaces
observability/logging quality
operator complexity
```

Do not perform a full production capacity study.

The question is relative:

> Is this operational footprint lower than the platform code and operational burden it removes?

---

## 8. Compare against the thin alternative

For every major Kargo capability proven, write the likely thin alternative.

Example format:

| Concern | Kargo | Thin/direct Git alternative | Which is simpler? |
|---|---|---|---|
| Artifact discovery | ... | CI supplies digest / registry lookup | ... |
| Stage promotion | ... | platform updates target path | ... |
| Git mutation | ... | narrow Git writer | ... |
| Conflict/retry | ... | custom retry/saga | ... |
| Verification | ... | Argo health + explicit smoke job | ... |
| History | ... | minimal Delivery audit record | ... |
| Concurrency | ... | target lock in service/DB | ... |

Do not compare Kargo against a deliberately bloated custom implementation.

---

## 9. GitOps layout observation

D1 should gather evidence for, but not necessarily finalize, the repository layout choice:

```text
Option A
one protected branch
+ target directories

Option B
stage-specific branches
stage/dev
stage/hml
stage/prd
```

Observe which model fits Kargo naturally and which introduces less:

```text
branch protection complexity
conflict/retry complexity
permission complexity
promotion friction
Argo targetRevision complexity
```

Do not turn the result into a company-wide source-code branching mandate.

---

## 10. Carried D0 gaps

D1 may proceed with these unresolved, but the final report must keep them visible:

### Argo/Kubernetes authority

Do not claim production-safe least privilege merely because the sandbox works.

### Git writer vs reconciler identity

Do not normalize the shared broad PAT as acceptable architecture.

If Kargo introduces a new Git writer identity, explicitly record that this identity is a **writer** and must remain separate from Argo's future read-only identity.

### AppProject/control-object governance

If D1 touches Application/AppProject objects imperatively, record it. Do not claim their governance is solved.

---

## 11. STOP conditions

STOP D1 early and classify `KARGO_NO_FIT` if any of the following becomes clear and cannot be narrowly justified:

```text
Kargo requires becoming a second business approval authority
Kargo forces provider-specific fields into canonical Change Management
Kargo requires broad production credentials as an inherent operating model
Kargo adds a generic workflow/rules language that the MVP would have to own deeply
Kargo's Git mutation cannot work with a realistic protected repository path
promotion/retry/concurrency semantics are less predictable than a thin controlled-Git path
operational footprint is clearly greater than the custom code it removes
Backstage or Change Management must be modified just to make the D1 POC function
```

Do not keep expanding the experiment to rescue Kargo.

---

## 12. KARGO_FIT criteria

Choose `KARGO_FIT` only if evidence shows that Kargo materially removes difficult promotion machinery while remaining narrow enough behind a provider boundary.

A credible `KARGO_FIT` should demonstrate most of:

```text
immutable artifact discovery/selection
clear DEV -> HML promotion semantics
safe/recoverable Git mutation
useful verification orchestration
credible same-stage concurrency/idempotency behavior
recoverable controller semantics
authority model compatible with future external Change eligibility
operational footprint justified by removed custom code
```

It does not need to be production hardened in D1.

---

## 13. KARGO_NO_FIT criteria

Choose `KARGO_NO_FIT` if the evidence shows that the platform could reach the MVP faster and more safely with a thin path such as:

```text
Backstage / Delivery API
      ↓
controlled writer
      ↓
Git desired-state mutation
      ↓
Argo
      ↓
Kubernetes
```

where the platform would own only the minimum missing concerns:

```text
release/target correlation
authorization check
idempotency
same-target serialization
Git mutation/retry
minimal audit evidence
```

Do not design that service fully in D1. Only bound its likely scope precisely enough to compare.

---

## 14. Keep the MVP deadline in view

This program needs a demonstrable deliverable soon.

Do not spend disproportionate time on secondary questions once the Kargo fit decision is evidence-backed.

The next intended move after D1 is **not another long horizontal spike**.

The intended next move is to begin a narrow end-to-end MVP that can eventually demonstrate:

```text
Backstage
  ↓
select/request exact release
  ↓
DEV / HML promotion
  ↓
request PRD
  ↓
CHANGE_REQUIRED
  ↓
create/select GMUD
  ↓
authorization
  ↓
fresh ALLOW
  ↓
Delivery promotion
  ↓
Git
  ↓
Argo
  ↓
Kubernetes
  ↓
status visible in Backstage
```

D1 should therefore end as soon as it can support the Kargo-vs-thin decision.

---

## 15. Evidence to capture

Create concise reproducible evidence for each executed test.

At minimum record:

```text
scenario
Kargo object(s)
artifact digest/fingerprint
source Stage
target Stage
Git commit/revision
Argo Application/revision
sync/health/verification outcome
commands/API actions used
result
notes/deviation
```

Screenshots are supplementary only.

Do not replace textual evidence with screenshots.

---

## 16. Documentation update

After execution, update the canonical docs with factual evidence only.

Preferred new checkpoint/evidence files under:

```text
docs/delivery/
```

Record at minimum:

```text
canonical docs SHA used
sandbox environment
Kargo version
Argo version
Kubernetes version
GitOps repo/path/branch
artifact source
scenarios executed
results
Kargo vs thin comparison
carried D0 gaps
final KARGO_FIT or KARGO_NO_FIT decision
```

Do not mark ADR-012 Accepted solely because D1 succeeds.

Do not authorize production.

Do not authorize GMUD/Change integration automatically.

---

## 17. Required final report

Use this structure:

```markdown
# D1 Promotion Controller Fit Result

## Verdict
KARGO_FIT | KARGO_NO_FIT

## Baselines
- Canonical docs SHA:
- Kargo version:
- Argo CD version:
- Kubernetes version:
- GitOps repo/revision:
- Sandbox target:

## What Was Executed
...

## Artifact Discovery / Identity
...

## DEV -> HML Promotion
...

## Git Mutation
...

## Verification
...

## Failure / Recovery
...

## Concurrency / Idempotency
...

## RBAC / Promotion Authority
...

## Operational Footprint
...

## Kargo vs Thin Alternative
| Concern | Kargo | Thin alternative | Assessment |
|---|---|---|---|
| ... | ... | ... | ... |

## GitOps Layout Observations
...

## Carried D0 Gaps
...

## Decision Rationale
...

## If KARGO_FIT
- Exact capabilities we will use in the MVP:
- Capabilities explicitly not adopted:

## If KARGO_NO_FIT
- Precise thin-provider responsibilities:
- What must NOT be built:

## Documentation Updated
...

## Gate
STOP
```

---

## 18. Final gate

Even if Kargo is an excellent fit:

```text
STOP
```

Do NOT continue directly into:

```text
full Delivery implementation
Backstage Delivery UI
GMUD integration
F3.1.2
production rollout
```

The architecture reviewer will use the D1 evidence to authorize the next **vertical MVP slice**.

---

## Authorization summary

```text
D1 sandbox evaluation: GO

Kargo production adoption: NO-GO until D1 verdict reviewed
Delivery implementation: NO-GO
GMUD integration: NO-GO
Backstage Delivery UI implementation: NO-GO
F3.1.1b: unchanged / separate
F3.1.2+: NO-GO
Production rollout: NO-GO
```

**Execute D1 only, make the Kargo-vs-thin decision quickly, document the evidence, and STOP.**
