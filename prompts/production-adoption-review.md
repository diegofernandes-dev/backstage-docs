# Production Adoption Review — Delivery / GitOps

## Role

Act as an **independent Principal Platform Architect / Staff+ reviewer** deciding whether the accepted Delivery / GitOps architecture and its proven implementation are ready to begin a **controlled first production adoption**.

You are a reviewer, not an implementer.

Your job is to challenge the evidence and return one of exactly three decisions:

```text
GO
CONDITIONAL_GO
NO_GO
```

Do **not** optimize for approval merely because ADR-012 is Accepted or P1 residual closure returned PASS.

The accepted architecture is the baseline, not the question under review:

> **Change Management authorizes. Delivery promotes. Git declares. Argo CD reconciles. Kubernetes executes. Backstage composes the experience.**

The question now is narrower and operational:

> **Is there enough credible, transferable evidence to authorize the first controlled production rollout without accepting an unreasonable authority, durability, operability, or recovery risk?**

This is **not** approval for unrestricted enterprise-wide rollout or general availability.

---

# 0. Hard execution rules

This checkpoint is **REVIEW ONLY**.

### You MAY

- read canonical documentation;
- inspect committed implementation and GitOps state;
- inspect current ADO/Git/Kargo/Argo/Kubernetes state using read-only operations;
- inspect current configuration, RBAC, policies, manifests, controller status, and logs when read-only;
- rerun existing non-mutating tests from committed baselines;
- compare current state with previously recorded evidence;
- update canonical documentation with the review result.

### You MUST NOT

- edit application source code;
- modify GitOps desired state;
- create, patch, delete, or rotate Kubernetes resources or credentials;
- change ADO branch policies or ACLs;
- change Entra identities or secrets;
- trigger a new production-style promotion merely to manufacture evidence;
- redesign ADR-012;
- reopen Deployments UX work;
- continue P1 hardening;
- implement rollback, HA/DR, break-glass, supply-chain, or other follow-up work;
- silently fix a problem discovered during the review.

If a critical claim can only be re-proven through a mutating test, **do not perform the mutation**. Use the strongest existing evidence, inspect the final state read-only, state the confidence level, and score the gate accordingly.

If you discover a blocker, record it and continue the review. Do not repair it.

---

# 1. Canonical baseline protocol

Before making any conclusion:

1. In `diegofernandes-dev/backstage-docs`:

   ```bash
   git fetch origin main
   ```

2. Record the exact current `origin/main` SHA.

3. Read at minimum, in this order:

   ```text
   docs/delivery/README.md
   docs/adr/ADR-012-delivery-management-gitops-promotion.md
   docs/delivery/adr-012-adoption-rereview.md
   docs/delivery/eligibility-window-toctou.md
   docs/delivery/p1-production-authority-hardening.md
   docs/delivery/p1-residual-closure.md
   docs/delivery/mvp-vertical-delivery-slice.md
   docs/delivery/mvp-demo-hardening.md
   docs/delivery/d1-kargo-fit-evaluation.md
   docs/delivery/e1-multi-activity-concurrency.md
   docs/delivery/deployments-ux-v2-implementation.md
   docs/delivery/deployments-ux-v2-responsive-polish.md
   ```

4. Inspect the current implementation baseline recorded by canonical docs. At the time this prompt was authored, the latest recorded implementation was `platform-devops-developer-portal@b08e7b2` on `feat/delivery-mvp-slice`, but **do not trust this prompt if canonical docs have advanced**.

5. Inspect the current governed GitOps branch/revision recorded by `p1-residual-closure.md`. At the time this prompt was authored, the durable merged revision was `50564c9977e1a02b7d16345c9ebcc38416986efb`; again, canonical/current state wins.

6. Reconcile documentation with current live state using read-only inspection. Record any drift before scoring the gates.

If you cannot independently inspect a source, explicitly mark conclusions as `DOCS_ONLY` rather than implying live verification.

---

# 2. Starting state to challenge

The current canonical state claims all of the following:

- ADR-012 architecture: **Accepted**.
- Eligibility-window TOCTOU follow-up: **PASS**.
- P1 production-authority hardening: **CONDITIONAL_PASS**, with four authority properties proven.
- P1 residual closure: **PASS**.
- Ready for separate production-adoption review: **YES**.
- Production rollout: still **NO-GO**, pending this review.
- Live Argo reader and Kargo/Delivery writer credentials use distinct non-human Entra service principals.
- Token refresh is automatic and has completed multiple cycles without a human session.
- The P1 GitOps control changes are merged into the protected governed branch.
- `d1-prd` and `d1-control-plane` were re-verified `Synced/Healthy` at the merged revision.
- ordinary squad/pipeline authority cannot bypass the governed production path, based on prior positive and negative proofs.
- the current Deployments UX is functionally accepted/demoable and visual polish is explicitly deferred.

Treat these as claims requiring review, not axioms.

---

# 3. Decision semantics

The verdict concerns **controlled first production adoption**, not platform-wide GA.

## GO

Return `GO` only when:

- no unresolved issue is a credible blocker to the first controlled production rollout;
- authority boundaries are both credible and durable;
- identity/credential lifecycle is sustainable enough for the initial rollout;
- Git is the durable authority for production desired state and control-plane configuration in the exercised path;
- the legitimate path remains functional while realistic bypass paths are closed;
- operational failure and recovery expectations are understood well enough to run the first production workload safely;
- any remaining work is clearly post-rollout hardening or scale work and does not need to be completed before the first rollout.

A `GO` may still prescribe rollout constraints. It does **not** mean unrestricted enterprise adoption.

## CONDITIONAL_GO

Return `CONDITIONAL_GO` when the first production rollout is acceptable **only if a finite, explicit set of pre-rollout conditions is satisfied**.

Each condition must be:

- objectively verifiable;
- bounded;
- not an architecture redesign;
- not a vague maturity aspiration;
- required before the first production workload is onboarded.

Examples of legitimate conditions may include a missing runbook, explicit ownership, a backup/restore verification, a specific rollout guard, or a production-environment equivalent of a sandbox assumption — but only if evidence shows that condition materially affects risk.

Do not use `CONDITIONAL_GO` as a generic hedge.

## NO_GO

Return `NO_GO` when there is at least one unresolved issue that makes first production adoption unsafe or not credibly governed, for example:

- an ordinary squad/pipeline can still bypass the governed path;
- Argo or Delivery retains unjustified production-wide mutation authority;
- human credentials are still in the live production authority path;
- desired-state or control-plane governance depends on ephemeral/bootstrap-only state that can silently regress;
- token/credential renewal is not sustainable and can predictably break reconciliation;
- the legitimate path is not stable enough to deploy or reconcile safely;
- rollback/recovery is so undefined that a routine production failure has no credible bounded response;
- evidence materially contradicts the claimed final state.

---

# 4. Evidence grading

For each gate, classify evidence using only these labels:

```text
LIVE_VERIFIED
COMMITTED_CODE
COMMITTED_CONFIG
PRIOR_EXECUTION_EVIDENCE
DOCS_ONLY
NOT_PROVEN
CONTRADICTED
```

Then score each gate as:

```text
PASS
PASS_WITH_FOLLOWUP
CONDITION
BLOCKER
```

Do not equate `DOCS_ONLY` automatically with failure. Judge whether the evidence is sufficient for the specific claim and whether it remains transferable to production.

Do not equate sandbox evidence automatically with production evidence. Explicitly assess **transferability**.

---

# 5. Mandatory review gates

Evaluate **all** gates below. Do not skip a gate because a prior checkpoint passed it.

## Gate 1 — Architecture stability

Determine whether any evidence gathered since ADR-012 acceptance materially contradicts the accepted boundary.

Challenge:

- Has Delivery remained provider-neutral at the canonical domain boundary?
- Does Change Management still authorize rather than orchestrate deployment?
- Does Git remain the desired-state authority rather than Backstage, Kargo, or a pipeline?
- Has any implementation shortcut effectively recreated a workflow engine or pipeline-centric authority model?

A production-adoption issue does not automatically reopen the ADR.

---

## Gate 2 — Git authority separation

Verify the final authority model for:

- Delivery/Kargo Git writer;
- Argo Git reader;
- ordinary squad/pipeline Git identities;
- human administrator override characteristics.

Confirm that:

- the reader cannot write;
- the writer follows the protected-branch/PR path;
- ordinary squad identities cannot mutate production desired state;
- direct protected-branch bypass is not available in the normal operating model;
- live credentials are non-human identities, not merely differently named PATs.

Distinguish capability proof from current live wiring.

---

## Gate 3 — Credential lifecycle durability

Review the final token-refresh mechanism critically.

Inspect:

- non-human identity ownership;
- client-secret/token lifecycle;
- refresh schedule versus token lifetime;
- refresh RBAC;
- failure behavior;
- observability of refresh failures;
- what happens if refresh misses one or more cycles;
- bootstrap dependencies;
- recovery path after credential expiry or client-secret rotation.

Pay particular attention to the fact that canonical evidence may describe the refresh manifests as **bootstrap-applied rather than Argo-reconciled** to avoid widening the control-plane Application's write scope.

Decide whether this is:

- acceptable bounded bootstrap infrastructure for first rollout;
- a pre-rollout condition;
- or a production blocker.

Do not demand a general credentials platform unless risk actually requires it.

---

## Gate 4 — Kubernetes execution authority

Review the scoped Argo destination model and Delivery/Kargo runtime authority.

Confirm:

- production mutation authority is namespace/resource constrained;
- broad reads required by Argo caching do not silently become broad writes;
- secrets remain protected from the scoped deployer where claimed;
- cluster-scoped escalation paths are denied;
- Delivery backend does not fall back to ambient cluster-admin credentials;
- ordinary squad/pipeline Kubernetes identities cannot mutate production runtime directly.

Explicitly distinguish **read breadth** from **write breadth**.

---

## Gate 5 — Control-plane desired-state governance

Determine whether Argo `Application` / `AppProject` and the scoped destination configuration are durably governed.

Challenge the historical failure where bootstrap-applied state was later reverted by self-healing from an older governed branch.

Confirm the final state is sourced from the merged protected branch and survives reconciliation.

Any remaining bootstrap-only resource must be named and risk-classified rather than hidden.

---

## Gate 6 — Bypass resistance

Review whether a realistic ordinary squad actor can bypass the intended path through any of:

- Git;
- Kubernetes;
- Argo control objects;
- Kargo resources;
- Backstage/Delivery backend APIs;
- ADO pipeline/service identity;
- stale or recreated service accounts/bindings.

Use prior negative execution evidence plus current read-only state.

Do not claim impossibility. Judge whether the **credible normal attacker/operator path** is closed enough for controlled rollout.

---

## Gate 7 — Legitimate promotion path

Confirm the legitimate path still works under the hardened authority model:

```text
ReleaseCandidate
→ DEV
→ HML
→ PRD request
→ ChangeBinding / ExecutionEligibility
→ Kargo
→ protected Git mutation
→ Argo reconciliation
→ Kubernetes
→ Backstage projection
```

Review evidence that the same immutable artifact/digest is promoted.

Review the Kargo no-op behavior and any fix recorded during residual closure.

A security model that prevents the valid path from operating is not production-ready.

---

## Gate 8 — Change / authorization correctness

Confirm the production path still enforces:

- PRD requires governed Change where specified;
- binding is activity-scoped (`changeId + activityId`);
- fresh eligibility is evaluated before dispatch;
- half-open execution-window semantics remain enforced;
- a deployment succeeding does not complete the whole multi-activity Change;
- UI state cannot bypass backend authorization.

Do not re-review GMUD product design beyond these production-authority semantics.

---

## Gate 9 — Concurrency and idempotency

Review the evidence for:

- same-target concurrent promotion exclusion;
- duplicate/idempotent request behavior;
- prevention of two independent release mutations racing into the same target;
- stale UI/read-model state not becoming authorization authority.

Determine whether current proof is sufficient for first production rollout or whether a production-environment condition is required.

---

## Gate 10 — Failure detection and operational recovery

Assess whether the first production rollout has a credible response to routine failures such as:

- Kargo promotion error;
- Git PR/update failure;
- Argo reconciliation failure;
- Kubernetes rollout failure;
- token refresh failure;
- stale/unavailable provider projection;
- Backstage/Delivery temporary outage during an already-dispatched reconciliation.

Do **not** require full HA/DR or automated rollback merely because those are desirable.

Instead decide whether operators have enough evidence, observability, ownership, and bounded manual recovery capability to run the first production workload safely.

If a specific runbook or monitoring requirement is necessary before rollout, classify it as a concrete `CONDITION`.

---

## Gate 11 — Rollback and break-glass readiness

Challenge the fact that full rollback automation and break-glass were intentionally deferred.

Answer separately:

1. Can a bad desired-state change be reversed credibly through the governed Git path?
2. Is there an emergency operational path when the normal control plane is unavailable?
3. Are the permissions/ownership of that emergency path sufficiently bounded for first rollout?

Do not require a polished automated break-glass product if a controlled operational procedure is sufficient.

But do not wave away the absence of any credible emergency procedure.

---

## Gate 12 — Auditability and evidence chain

Determine whether an operator/auditor can reconstruct, for a production deployment:

- immutable release identity/digest;
- target/environment;
- deployment request;
- Change and activity binding where applicable;
- eligibility decision;
- Git desired-state mutation / PR / revision;
- Argo reconciliation state;
- deployment result;
- actor/requester where the model actually records it.

Known UI gaps such as Commit/Branch/Aprovador being displayed as `não disponível` must be judged by whether they are merely UX/read-model gaps or actual audit blockers.

Do not fabricate missing audit fields.

---

## Gate 13 — Production transferability

This gate is mandatory because most evidence was produced in a local/sandbox topology.

Create an explicit matrix:

| Assumption / proof | Sandbox-specific? | Transfers directly? | Production equivalent required before rollout? | Risk if different |
|---|---|---|---|---|

At minimum assess:

- Kubernetes distribution/cluster topology;
- Argo/Kargo versions and deployment topology;
- ADO organization/tenant/branch-policy semantics;
- Entra service-principal behavior;
- network reachability;
- Git repository layout and branch policy;
- namespace isolation model;
- secret storage/rotation mechanism;
- Backstage/Delivery runtime identity;
- production target naming/account/cluster boundaries.

Do not return `GO` merely because the sandbox passed if a production environment has a materially different authority topology that has not been bounded.

Conversely, do not return `NO_GO` merely because the exact production cluster has not yet been onboarded. A controlled rollout can legitimately include explicit preflight equivalence checks as rollout conditions.

---

## Gate 14 — First-rollout blast radius

Assuming the platform is otherwise ready, determine what a safe **first production adoption envelope** looks like.

Evaluate whether the first rollout should be constrained by:

- one application/component;
- one production namespace/target;
- one cluster/account;
- a non-critical or medium-criticality workload;
- explicit platform owner presence;
- a defined observation period before broader onboarding;
- no simultaneous migration of many workloads;
- explicit rollback owner and escalation contact.

These are rollout constraints, not architecture requirements.

---

# 6. Mandatory challenge questions

Answer all of these explicitly:

1. What is the strongest remaining path by which an ordinary squad could bypass Delivery and mutate production?
2. What is the strongest remaining path by which a platform administrator could accidentally bypass Git governance?
3. Which current production-authority control still depends on bootstrap/manual state rather than continuous reconciliation?
4. What happens if the token-refresh CronJob fails for more than one token lifetime?
5. What happens if the SP client secret itself expires or is revoked?
6. What is the recovery path if Argo can no longer read Git?
7. What is the recovery path if Kargo can no longer write Git?
8. What is the recovery path if Git desired state is correct but Argo reconciliation fails?
9. What protects production if Backstage is unavailable?
10. What protects production if the Delivery backend is unavailable after a promotion was already dispatched?
11. Is the evidence chain sufficient to explain a production deployment after the fact?
12. Which sandbox assumptions must be proven equivalent during the first real production onboarding?
13. Is any remaining gap an architecture flaw, a pre-rollout condition, or post-rollout hardening?
14. What would make you change the verdict one level downward?

---

# 7. Explicit non-blockers unless evidence proves otherwise

Do not automatically block first controlled production adoption solely because these are not complete:

- global Delivery workbench;
- further Deployments UX polish;
- enterprise-wide multi-cluster scale;
- HA/DR automation;
- automated rollback;
- a generalized break-glass product;
- full supply-chain provenance expansion;
- global GitOps migration of all workloads;
- perfect audit UX;
- production-grade secret-management platform replacing the bounded refresh mechanism;
- broad historical analytics/event sourcing.

Any of these may still become a blocker **if you demonstrate a concrete first-rollout risk**. Otherwise classify them as follow-up.

---

# 8. Review discipline

Do not let the review devolve into a generic maturity checklist.

Every finding must answer:

```text
What exact failure or bypass could occur?
How likely/credible is it in the first rollout envelope?
What evidence exists today?
Is the risk already bounded?
Does it block first rollout, require a condition, or belong after rollout?
```

Prefer a small number of meaningful conditions over dozens of aspirational recommendations.

Do not invent new architecture merely to make the review look comprehensive.

---

# 9. Required final report

Create or update:

```text
docs/delivery/production-adoption-review.md
```

Also update the status summary in:

```text
docs/delivery/README.md
```

Do **not** change ADR-012 status unless the review finds actual evidence that invalidates the accepted architecture. A `NO_GO` for production does not by itself mean ADR-012 should be reopened.

The final report must use this structure:

## 1. Review baseline

Exact docs SHA, implementation SHA, GitOps revision, live-state inspection scope, and any unavailable sources.

## 2. Executive verdict

Exactly:

```text
Production adoption review: GO | CONDITIONAL_GO | NO_GO
ADR-012 architecture status: ACCEPTED | REOPEN_REQUIRED
Authorized rollout scope: <one concise sentence or NONE>
```

## 3. Why this verdict

Maximum 10 concise bullets, ordered by risk.

## 4. Evidence matrix

One row for every mandatory gate with:

```text
Gate | Score | Evidence class | Key evidence | Residual risk | Classification
```

## 5. Production transferability matrix

The explicit sandbox→production matrix required by Gate 13.

## 6. Authority model at review time

Concise final matrix of writer, reader, reconciler, runtime deployer, ordinary squad/pipeline, human/platform admin.

## 7. Failure / recovery assessment

Cover Git, Argo, Kargo, Kubernetes, token refresh, Backstage/Delivery availability.

## 8. Rollback / emergency assessment

State what exists today, what is sufficient, and what is not.

## 9. Mandatory challenge answers

Answer all 14 questions from §6 directly.

## 10. Conditions before rollout

If verdict is `CONDITIONAL_GO`, list only objectively testable pre-rollout conditions with owner/evidence required.

If verdict is `GO`, write `None beyond the rollout envelope below` unless a condition actually exists — in which case the verdict should probably be `CONDITIONAL_GO`.

If verdict is `NO_GO`, list the blockers that must be closed before another review.

## 11. Authorized first-rollout envelope

If `GO` or `CONDITIONAL_GO`, specify the narrow rollout boundary: workload criticality, number of applications, targets/clusters, operator ownership, observation period, and explicit exclusions.

If `NO_GO`, write `NONE`.

## 12. Post-rollout hardening backlog

Only bounded items that are **not** prerequisites for the first rollout.

## 13. Verdict downgrade triggers

List concrete observations that would invalidate the current verdict during rollout.

## 14. STOP

State explicitly that no implementation, rollout, new milestone, UI work, or P1 continuation was authorized or executed by this review.

---

# 10. Decision guardrails

A strong review may legitimately conclude any of the three verdicts.

Do not return `GO` because previous documents say `PASS`.

Do not return `NO_GO` because production engineering can always be hardened further.

The decision must answer one practical question:

> **Can we safely start with one tightly controlled production workload under the proven authority model, with the remaining risk explicitly bounded?**

That is the gate.

---

# STOP gate

After writing the factual review evidence and updating canonical status:

**STOP.**

Do not:

- execute a production rollout;
- onboard a production workload;
- change infrastructure;
- implement any review condition;
- start a P2 milestone;
- reopen Deployments UX;
- create a global Delivery workbench;
- broaden the architecture.

Return the review verdict and the exact docs commit SHA, then stop.