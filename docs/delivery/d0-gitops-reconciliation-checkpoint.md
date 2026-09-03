# D0 — Pure GitOps Reconciliation Checkpoint

- **Status:** GO — POC authorized; no production implementation authorized
- **Date:** 2026-09-03
- **Parent spike:** `docs/delivery/architecture-spike.md`
- **Branch/release refinement:** `docs/delivery/branching-release-control.md`
- **Change integration:** explicitly out of scope

## Objective

Prove the minimal Kubernetes GitOps execution substrate independently of GMUD, Kargo, Backstage Delivery UI, and Azure DevOps Environment approvals.

D0 answers only:

> Can a known immutable artifact be declared in Git, reconciled by Argo CD into a sandbox Kubernetes target, observed as converged, drift-remediated safely, and reverted through Git without imperative deployment orchestration?

## Required flow

```text
known immutable image digest
      |
      v
controlled Git desired-state change
      |
      v
Argo CD Application
      |
      v
sandbox Kubernetes target
      |
      +--> Synced
      +--> Healthy
      |
      v
Git revert / desired-state correction
      |
      v
Argo reconciles back
```

## Explicit non-goals

D0 must not introduce:

- Change / GMUD integration;
- `ExecutionEligibility`;
- Kargo;
- a custom Delivery controller or workflow/state machine;
- DeploymentRequest persistence;
- Backstage UI/plugin implementation;
- production clusters;
- production credentials;
- automatic production rollback;
- Azure DevOps Environment checks/approvals;
- public inbound API exposure;
- final organization-wide branching decision.

## Test substrate

Use one disposable/sandbox application and one sandbox Kubernetes target. The application image must be referenced by immutable digest, not only by a mutable tag.

The GitOps repository/layout may use the simplest temporary D0 structure. Do not treat that choice as the final single-branch-vs-stage-branches decision; D1/D2 will compare the real promotion layouts.

## Acceptance scenarios

### A. Initial reconcile

1. Commit desired state referencing image digest A.
2. Argo detects the Git revision without an imperative `argocd app sync` dependency in the normal path.
3. Application reaches `Synced` + `Healthy`.
4. Record Git revision, Argo Application revision/status, namespace/target, and image digest actually running.

### B. Immutable release change

1. Change Git desired state from digest A to digest B.
2. Argo reconciles to B.
3. Kubernetes runtime evidence confirms B is running.
4. No environment-specific rebuild occurs.

### C. Git revert

1. Revert desired state B back to A through Git.
2. Argo reconciles back to A.
3. Runtime converges to A.
4. Git and runtime are not left disagreeing after recovery.

### D. Drift/self-heal

1. After Git declares A and runtime is Healthy, create a controlled out-of-band mutation in the sandbox cluster.
2. With the selected self-heal posture enabled, Argo restores the state declared by Git.
3. Confirm this is remediation to an existing desired state, not a new Git release promotion.

### E. Missed notification / polling recovery

1. Do not rely on a webhook callback as the only signal.
2. Demonstrate that Argo's reconciliation/read-back reaches the correct state even when no external Delivery notification is consumed.

### F. Failed desired state

1. Introduce a safe sandbox-only desired state that cannot become Healthy (for example invalid image digest/reference or intentionally failing readiness behavior).
2. Confirm failure is visible and does not get misreported as successful merely because the Git commit exists.
3. Recover by correcting/reverting Git.

### G. AppProject / authority boundary

Prove a dedicated sandbox AppProject (or equivalent Argo project boundary) restricts at least:

- allowed source repository;
- allowed destination cluster/namespace;
- privileged/undesired destination scope as appropriate for the POC.

Do not use unrestricted default-project permissions as evidence of production viability.

### H. Direct-cluster mutation posture

Document which identities can mutate the sandbox namespace directly and what the intended future production posture is. Production direction should minimize ordinary direct mutation so Git remains the normal desired-state authority.

### I. Prune/delete posture

Exercise deletion only in the sandbox. Record:

- whether automated prune is enabled for D0;
- what happens when a manifest is removed from Git;
- which additional safeguards would be required before production.

No conclusion that "all production prune is safe" may be derived from D0 alone.

## Evidence to capture

For every scenario capture concise reproducible evidence:

```text
scenario
Git commit SHA
image digest
Argo Application name
AppProject
sync status
health status
target namespace/cluster
observed runtime image digest
result
notes/deviation
```

Screenshots may supplement the evidence but are not the canonical proof.

## Security constraints

- no secrets committed to Git;
- no production credentials;
- no broad cluster-admin credential used as the intended final pattern;
- Git write authority and Argo runtime authority should be distinguishable;
- record any temporary POC privilege that would be unacceptable in production.

## Decision questions at D0 close

D0 must answer:

1. Does declarative Git -> Argo -> Kubernetes reconciliation work reliably enough to be the execution substrate?
2. Can immutable digest promotion be observed end to end?
3. Does self-heal behave as expected for drift remediation?
4. Can failed sync/health be detected without a custom state machine?
5. Does Git revert provide a coherent recovery path?
6. Are the basic AppProject/RBAC boundaries compatible with a future production design?
7. What concrete gaps remain before testing promotion semantics in D1?

## Exit criteria

### PASS

All mandatory scenarios A-H pass with documented evidence; I is explicitly documented and safely exercised or deferred with a defensible reason. No imperative pipeline/Argo sync operation is required as the normal deployment mechanism.

### CONDITIONAL PASS

The core Git/Argo reconciliation works, but one non-core production-hardening concern needs a narrow follow-up before D1. The unresolved item must not invalidate the desired-state/reconciliation model.

### FAIL

Examples:

- normal deployment still depends on imperative orchestration;
- Git revision cannot be deterministically correlated to runtime state;
- Argo cannot recover or expose failure coherently;
- required authority boundaries cannot be established even in principle;
- the POC needs a custom workflow engine merely to reconcile Git to Kubernetes.

## Gate

**GO for D0 POC only.**

On completion: record evidence and STOP for architecture review.

**NO-GO for D1** until D0 is reviewed.

**NO-GO for F3.1.1b/F3.1.2+** remains unchanged.
