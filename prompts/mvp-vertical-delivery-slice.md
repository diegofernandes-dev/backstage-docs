# MVP Vertical Delivery Slice — Demonstrable End-to-End Delivery

## Purpose

You are acting as a **Senior/Principal Platform Engineer implementing the smallest demonstrable vertical slice** that connects the platform capabilities already proven by D0 and D1.

This is **not another horizontal architecture spike**.

The objective is to produce a working, demoable flow that shows business value end to end while keeping production-hardening concerns explicitly deferred.

The target experience is:

```text
Backstage
  ↓
ReleaseCandidate
  ↓
Kargo DEV
  ↓
Kargo HML
  ↓
Request PRD
  ↓
CHANGE_REQUIRED
  ↓
GMUD creation / binding
  ↓
Authorization / ALLOW
  ↓
Kargo PRD promotion
  ↓
Git desired state
  ↓
Argo CD
  ↓
Kubernetes
  ↓
Deployment status visible in Backstage
```

The demo must prove that the **same immutable release** moves through the flow without rebuild between environments.

---

## 0. Authorization model

This file is the canonical long-form execution contract for the MVP slice.

**Reading this file does not by itself authorize execution.**
Execution is authorized only when the user explicitly launches the agent and tells it to execute this prompt.

Once explicitly launched, this prompt authorizes **sandbox/demo implementation only** for the MVP vertical slice defined here.

It does **not** authorize production rollout.

---

## 1. Mandatory canonical-source protocol

Canonical documentation repository:

```text
diegofernandes-dev/backstage-docs
branch: main
```

Before changing anything:

```bash
git fetch origin main
```

Verify the exact current `origin/main` SHA.

Read at minimum the current versions of:

```text
docs/adr/ADR-012-delivery-management-gitops-promotion.md
docs/delivery/architecture-spike.md
docs/delivery/branching-release-control.md
docs/delivery/d0-architecture-review.md
docs/delivery/d1-kargo-fit-evaluation.md
docs/backstage/current-state.md
docs/backstage/f3-1-1a-architecture-acceptance.md
prompts/README.md
```

Also inspect any newer Delivery/GMUD/Backstage documentation added after this prompt.

Do not reconstruct architecture from memory.

If current canonical docs materially contradict this prompt, **STOP** and report the contradiction instead of forcing the implementation.

Expected starting architecture direction:

```text
D0 = ACCEPT_CONDITIONAL_PASS
D1 = KARGO_FIT (qualified)
ADR-012 = Proposed
```

The two D0 authority gaps remain carried production-hardening concerns:

```text
- Argo/Kubernetes execution authority is not yet production-scoped.
- Git writer and reconciler identities are not yet production-separated.
```

Do not attempt to close those gaps unless they directly prevent the sandbox MVP from functioning.

---

## 2. Product outcome beats architecture completeness

The primary success criterion is a **working demonstration**, not architectural perfection.

Every proposed task must pass this test:

> Does this task materially help demonstrate the end-to-end flow?

If no, defer it.

Do not create new architecture workstreams merely because an unresolved production concern exists.

Do not split this implementation into D2, D3 and D4 as independent horizontal projects unless a genuine blocker makes that unavoidable.

The vertical slice itself should provide the evidence needed to refine those boundaries later.

---

## 3. Architectural boundaries that must remain intact

### Change Management

Change Management remains the authority for production business authorization.

It owns concepts such as:

```text
Change
AuthorizationRound
AuthorizationEvaluation
ExecutionEligibility
```

It must not become aware of Kargo-specific, Argo-specific or Azure DevOps-specific fields as canonical Change properties.

### Delivery Management

Delivery owns software release/promotion context and the correlation between a release, a deployment target and a Change.

For this MVP, keep the Delivery model as small as possible.

The smallest acceptable concepts are approximately:

```text
ReleaseCandidate
DeploymentTarget
DeploymentRequest
ChangeBinding
```

Names may differ only if current canonical docs already define better names.

Do not build a generic workflow engine.

### Kargo

Kargo is the selected promotion controller candidate for the MVP.

It owns promotion mechanics such as:

```text
Warehouse
Freight
Stage
Promotion
Verification
Git mutation
```

Kargo is **not** business approval authority.

Do not use Kargo approval semantics as a replacement for GMUD authorization.

### Git

Git remains desired-state authority.

### Argo CD

Argo remains Kubernetes reconciler.

### Backstage

Backstage composes the developer/operator experience.

Backstage must not become a deployment workflow engine.

---

## 4. MVP simplifications explicitly allowed

The following simplifications are acceptable for this demo slice:

```text
single application/component
single sandbox cluster
single GitOps repository
single DEV target
single HML target
single PRD-like sandbox target
one ReleaseCandidate at a time
one happy-path Change classification if needed
one approval path if existing Change policy supports it
manual human approval where already required
minimal durable Delivery persistence
polling/read-back instead of event-driven architecture
internal-only APIs
```

A PRD-like target in this MVP means a sandbox target exercising **production authorization semantics**. It does not mean a real production cluster.

Do not weaken the authorization semantics merely because the target is sandbox.

---

## 5. What must be demonstrable

The final MVP demonstration must show, in one coherent flow:

1. A known immutable artifact becomes a ReleaseCandidate.
2. The same artifact is promoted to DEV.
3. The same artifact is promoted to HML.
4. Backstage can display enough delivery state to identify the release and target states.
5. A user requests promotion to the PRD-like target.
6. Delivery detects that production-class promotion requires an authorized Change.
7. The user can create or associate a GMUD/Change from the delivery context.
8. The Change follows the existing authorization model.
9. Before authorization, PRD promotion is denied/not dispatchable.
10. After an `ALLOW` decision for the exact governed context, Delivery dispatches the Kargo promotion.
11. Kargo performs the desired-state Git mutation using the protected-path pattern proven in D1.
12. Argo reconciles the PRD-like target.
13. Kubernetes runs the same immutable artifact.
14. Backstage shows the resulting deployment state and the related Change.

This is the core deliverable.

---

## 6. ReleaseCandidate — minimum contract

Do not design a universal artifact platform.

For the MVP, ReleaseCandidate must carry only what is required to identify the exact promotable release and correlate it through the flow.

Minimum expected information:

```text
releaseCandidateId
componentRef
sourceRevision
artifactType
artifactFingerprint
createdAt
```

For OCI images, `artifactFingerprint` should identify the immutable logical artifact being promoted. For multi-platform OCI images, this will normally be the OCI index/manifest-list digest.

Do not put branch/environment routing semantics into ReleaseCandidate.

Do not rebuild between DEV, HML and PRD-like targets.

---

## 7. DeploymentTarget — minimum contract

Use first-class logical targets instead of treating branches as environments.

For the MVP, define only three targets:

```text
<component>/dev
<component>/hml
<component>/prd
```

A DeploymentTarget should minimally identify:

```text
deploymentTargetId
componentRef
environmentClass: dev | hml | production
provider mapping required by the Delivery adapter
```

Kargo/Argo/Git details may live in adapter/configuration data, but they must not leak into Change Management.

---

## 8. DeploymentRequest — minimum durable record

Create the smallest durable request/evidence record needed to support the demo and future audit.

At minimum persist or otherwise durably represent:

```text
deploymentRequestId
releaseCandidateId
deploymentTargetId
requestedBy
createdAt
changeBinding (optional until required)
provider correlation / Kargo Promotion reference
desired-state Git revision when produced
current/terminal outcome
```

Do not copy Kargo's full Promotion state machine into the platform.

Provider-native details such as:

```text
Promotion phase
Argo sync status
Argo health
AnalysisRun result
```

should remain provider projections, normalized only enough for Backstage UX.

---

## 9. ChangeBinding — exact purpose

The binding exists to answer:

> Which authorized business change permits this exact software promotion?

The PRD-like authorization context must bind at minimum:

```text
Change
+ optional activityId where applicable
+ exact ReleaseCandidate artifact fingerprint
+ DeploymentTarget
```

Do not add Kargo Promotion IDs, Argo Application names, ADO PR IDs or Git branch names into canonical Change.

Those are Delivery/provider evidence.

---

## 10. Production-class request behavior

The MVP should expose a clear behavior when a user requests PRD promotion without a Change.

Preferred result:

```text
CHANGE_REQUIRED
```

Backstage should offer an action such as:

```text
Create GMUD
```

with a deep-link or routed flow that pre-fills only safe contextual information, for example:

```text
component
release candidate
production target
related deployment request
```

The user must still provide governance information that cannot be safely derived, such as risk/window/rollback content required by the existing Change contract.

Do not create a persistent generic `ChangeDraft` domain unless the current architecture already requires it.

---

## 11. ExecutionEligibility is the gate

Before dispatching the PRD-like Kargo Promotion, Delivery must obtain a fresh server-authoritative eligibility decision from Change Management.

The MVP must prove:

```text
unauthorized / missing Change → no PRD promotion dispatch
AUTHORIZED + valid governed context → ALLOW
ALLOW → promotion may be dispatched
```

`ALLOW` does not itself mean deployment succeeded.

It means the execution is permitted to start.

The exact transport can be narrow and internal for this MVP.

Do not introduce a public callback architecture unless unavoidable.

---

## 12. Start evidence / window semantics

Do not redesign ADR-009 during this MVP.

Use the current architecture hypothesis that the Change window governs accepted execution start/dispatch, not the entire reconciliation duration, unless newer canonical docs say otherwise.

For this slice, record at least:

```text
eligibility evaluated at
promotion dispatch accepted at
provider correlation created
```

If dispatch fails after `ALLOW`, do not record false start evidence.

If this cannot be represented cleanly without changing canonical Change semantics, STOP and document the blocker.

---

## 13. Kargo implementation constraints inherited from D1

Reuse what D1 proved instead of re-inventing it.

The MVP should use Kargo for:

```text
artifact discovery/tracking
stage promotion
Git mutation
PR creation/wait against protected Git path where applicable
verification
promotion/provider status
```

Carry the D1 findings explicitly:

- PR-gated mutation is preferred where the Git protection model requires it.
- Do not bypass branch policy with direct push.
- A no-op/idempotent promotion must not fail merely because a PR-producing step was skipped; harden the promotion template enough for the MVP's actual path.
- Kargo verification requires Argo Rollouts in the currently proven configuration.
- Backstage must not infer meaningful verification only from a generic Stage `Ready/Verified` condition; use actual verification evidence such as AnalysisRun when relevant.
- Kargo controller RBAC breadth remains a production-hardening gap.

Do not spend the MVP solving Kargo's entire operational model.

---

## 14. DEV and HML scope

Keep DEV/HML simple.

The same ReleaseCandidate must be promoted without rebuild.

A practical flow is:

```text
Warehouse / ReleaseCandidate
  → DEV
  → verification
  → HML
  → verification
```

Prefer actual stage-to-stage promotion if the D1 gap can be closed cheaply while implementing the MVP.

If doing so causes disproportionate delay, using the same Freight/ReleaseCandidate independently for DEV and HML is acceptable for the first demo **only if the exact immutable identity is visibly preserved and the deviation is documented**.

Do not let this become a new spike.

---

## 15. PRD-like target scope

The PRD-like target must differ from HML primarily through governance semantics:

```text
DEV/HML:
Change not required for MVP

PRD-like:
Change required
fresh ExecutionEligibility ALLOW required before dispatch
```

The target remains sandbox infrastructure.

No real production credentials or namespace are authorized.

---

## 16. Backstage UX — minimum demonstrable surface

Do not build the final Delivery Workbench.

Implement the smallest Backstage-native UI that can demonstrate the flow.

At minimum, from a Component context, the user should be able to see:

```text
release candidate identity
DEV status
HML status
PRD-like status
related Change/GMUD when present
```

And perform the key actions:

```text
promote/request DEV or HML as appropriate
request PRD
create/open GMUD when CHANGE_REQUIRED
retry/request promotion after authorization
```

A single Component-level `Deployments` tab/page is sufficient for the MVP.

A global Delivery Workbench may be deferred.

Use a shared detail route only if it is cheap and naturally follows the existing frontend architecture.

Do not build decorative UI or filters that do not help the demo.

---

## 17. Backstage/backend implementation discipline

Before coding, inspect the actual `platform-devops-developer-portal` implementation state and follow its existing backend/frontend architecture.

Do not assume old handoff notes still match the code.

Use the repository's current patterns for:

```text
backend plugin/module structure
frontend extensions/routes
permission integration
identity
catalog references
database access
```

Do not introduce a second framework inside Backstage.

If Azure DevOps implementation access is unavailable in the agent environment, STOP before making source-code claims and report that limitation.

---

## 18. Fix only blockers in existing Change implementation

Known historical prerequisites may include issues such as the old `buildChange()` double-build defect or missing RBAC config files.

Do not automatically perform all historical F3 work.

If an existing defect directly blocks the vertical MVP, fix the **smallest necessary defect** and document why it was required.

Do not resume the entire F3.1.x horizontal roadmap.

Any fix must preserve existing accepted ADR semantics.

---

## 19. Explicitly deferred production hardening

Unless one of these directly blocks the sandbox demo, defer:

```text
production-grade Argo controller RBAC
production Git credential separation
multi-cluster topology
full provenance/signature/admission chain
production prune/deletion policy
HA/DR
break-glass automation
automatic production rollback
advanced retention
organization-wide branch migration
public ingress/callback architecture
Teams approval integration polishing
full CAB UX
multi-provider Delivery support
MuleSoft/VM/non-Kubernetes delivery
```

Record them as carried follow-ups, not MVP failures.

---

## 20. Hard STOP conditions

STOP and report instead of expanding scope if any of the following becomes necessary:

```text
Change Management must gain Kargo/Argo/ADO-specific canonical fields
Kargo must become business approval authority
a generic DAG/workflow engine is required
ReleaseCandidate cannot be bound strongly to immutable artifact + target
PRD promotion cannot be prevented before eligibility ALLOW
provider state cannot be correlated back to the DeploymentRequest
Backstage must hold long-running workflow state to make the flow work
normal deployment requires imperative argocd app sync
production credentials are required to prove the slice
```

A STOP condition is useful architecture evidence. Do not hide it with extra machinery.

---

## 21. Evidence required for the MVP

Capture reproducible evidence for at least:

```text
ReleaseCandidate identity
DEV promotion
DEV runtime artifact
HML promotion
HML runtime artifact
PRD request before Change → blocked / CHANGE_REQUIRED
Change creation/binding
authorization state
ExecutionEligibility DENY before authorization if exercised
ExecutionEligibility ALLOW after authorization
Kargo PRD-like Promotion reference
Git desired-state revision
Argo sync + health
PRD-like runtime artifact
Backstage UI state before and after
```

The same immutable artifact fingerprint must be traceable across DEV, HML and PRD-like targets.

Screenshots are useful for demo evidence but must not replace textual identifiers and revisions.

---

## 22. MVP acceptance criteria

The vertical slice passes only if all of these are true:

```text
one immutable release is identifiable
same release reaches DEV
same release reaches HML
PRD-like request requires Change
Change can be created/associated from the Delivery experience
PRD-like promotion cannot dispatch before ALLOW
fresh ALLOW permits dispatch
Kargo performs promotion mechanics
Git records desired-state mutation
Argo reconciles
Kubernetes runs the expected immutable artifact
Backstage displays the resulting state and Change relationship
no custom workflow engine was introduced
```

The MVP may still have production-hardening gaps.

---

## 23. Failure semantics

Do not report success merely because:

```text
Git changed
or
Kargo Promotion = Succeeded
or
Argo = Synced
```

For the demo, distinguish enough state to show:

```text
request blocked by governance
promotion running/waiting
Git desired state updated
Argo Synced
Argo Healthy/Degraded
verification result where configured
```

Keep these as projections; do not create a second authoritative state machine.

---

## 24. Documentation update

After successful implementation/evidence collection, update canonical docs on `backstage-docs@main`.

Create or update a focused document under:

```text
docs/delivery/
```

Suggested name:

```text
mvp-vertical-delivery-slice.md
```

Record at minimum:

```text
canonical docs baseline
Backstage implementation repo/branch/SHA
GitOps repo/branch/SHA
Kargo/Argo/Kubernetes versions
implemented flow
evidence identifiers
known simplifications
carried production gaps
MVP verdict
next recommended hardening/product step
```

Do not mark ADR-012 Accepted solely because the MVP works.

A later architecture acceptance decision may do that.

---

## 25. Repository hygiene

For every repository touched:

```bash
git status --porcelain
git fetch
```

Understand the current branch and unrelated local changes before modifying files.

Do not overwrite unrelated work.

Before each commit:

```bash
git diff --stat
git diff
git status --short
```

Keep commits narrow and traceable.

---

## 26. Final report format

Return exactly this structure:

```markdown
# MVP Vertical Delivery Slice Result

## Verdict
PASS | CONDITIONAL PASS | FAIL | BLOCKED

## Baselines
- Canonical docs SHA:
- Backstage repo/branch/SHA:
- GitOps repo/branch/SHA:
- Kargo version:
- Argo CD version:
- Kubernetes version:

## Demo Flow Achieved
...

## ReleaseCandidate
...

## DEV
...

## HML
...

## PRD / Change Gate
...

## ExecutionEligibility
...

## Kargo / Git / Argo Evidence
...

## Backstage UX
...

## End-to-End Artifact Correlation
...

## Simplifications Used
...

## Carried Production Gaps
...

## Deviations / Blockers
...

## Files Changed
...

## Commits
...

## Documentation Updated
...

## Recommended Next Step
...

## Gate
STOP
```

---

## 27. Final gate

Even after a successful demo:

```text
STOP
```

Do not automatically proceed to:

```text
production hardening
production rollout
organization-wide adoption
ADR-012 acceptance
multi-provider Delivery
full Delivery Workbench
Teams/CAB expansion
F3 horizontal roadmap
```

The user must explicitly choose the next step after seeing the working MVP.

---

## Execution principle

**Favor a demonstrable vertical product slice over additional architecture sophistication.**

If the architecture can support the end-to-end flow safely in sandbox, implement the smallest coherent version, prove it, document it, and stop.