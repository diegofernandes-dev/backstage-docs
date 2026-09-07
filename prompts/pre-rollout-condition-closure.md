# Pre-Rollout Condition Closure — Delivery / GitOps

## Role

Act as a senior Staff+/Principal Platform Engineer executing the **smallest bounded implementation-and-evidence checkpoint required to close the five pre-rollout conditions from the Production Adoption Review**.

This is not a new architecture milestone. It is not a rollout. It is not a general production-hardening program.

The accepted architecture remains fixed:

> **Change Management authorizes. Delivery promotes. Git declares. Argo CD reconciles. Kubernetes executes. Backstage composes the experience.**

The Production Adoption Review returned:

```text
Production adoption review: CONDITIONAL_GO
ADR-012 architecture status: ACCEPTED
Authorized rollout scope: one non-critical/medium-criticality component, one production namespace, platform-owner-attended, 2-week observation before any second workload
```

Your objective is only:

> **Close the finite pre-rollout conditions that stand between `CONDITIONAL_GO` and a final rollout-readiness decision for the first narrow production workload.**

Do not optimize for closing every item. If a condition cannot be truthfully closed because the real production target/repository/runtime has not been identified or is unavailable, leave it OPEN and report exactly what is missing.

---

# 0. Hard scope rules

## You MAY

- inspect canonical docs, implementation, ADO, GitOps repos, Kubernetes, Argo, Kargo, Entra and current runtime configuration;
- make the minimum source/config/GitOps/Kubernetes/ADO changes required to close the five explicit conditions below;
- create branches and pull requests required by existing branch policies;
- create or update narrowly scoped Kubernetes manifests required by the conditions;
- create a small operational runbook;
- add the smallest failure-alert mechanism necessary for the token-refresh CronJob;
- provision or rebind the Delivery runtime credential for the actual first-rollout environment;
- provision equivalent non-human identities / ACLs / branch policy on the actual production GitOps repository only if that repository is already identified as the first-rollout source of truth;
- rerun existing regression and authority checks after the changes;
- update canonical evidence docs.

## You MUST NOT

- start the production rollout;
- deploy a real application workload to production;
- onboard a second workload, namespace, cluster, or account;
- redesign ADR-012;
- reopen Deployments UX work;
- build the global Delivery workbench;
- expand into HA/DR, generalized secrets management, generalized observability, supply-chain expansion, audit UX, or automated rollback;
- move Kargo Stage/Promotion/Warehouse governance into Git unless it is strictly necessary to close one of the five conditions below — the adoption review explicitly classified that as post-rollout hardening;
- reduce the Argo application-controller cluster-wide RBAC in this checkpoint — also post-rollout hardening;
- add ReleaseCandidate Commit/Branch/Aprovador fields;
- create a new monitoring platform merely to alert on one CronJob;
- weaken any existing ACL, branch policy, namespace isolation, scoped destination, or negative-authority control;
- bypass human approval requirements to make a PR merge;
- print, echo, log, commit, or include any secret/token/client-secret value in evidence.

The current Deployments UX remains frozen as functionally accepted/demoable. No visual changes are authorized.

---

# 1. Canonical baseline protocol

Before changing anything:

1. In `diegofernandes-dev/backstage-docs`:

   ```bash
   git fetch origin main
   ```

2. Record the exact current `origin/main` SHA.

3. Read, in this order:

   ```text
   docs/delivery/README.md
   docs/delivery/production-adoption-review.md
   docs/delivery/p1-residual-closure.md
   docs/delivery/p1-production-authority-hardening.md
   docs/delivery/eligibility-window-toctou.md
   docs/delivery/adr-012-adoption-rereview.md
   docs/adr/ADR-012-delivery-management-gitops-promotion.md
   ```

4. Inspect the actual current implementation SHA/branch for `platform-devops-developer-portal`.

5. Inspect the actual governed GitOps repo/branch/revision and current live Argo/Kargo/Kubernetes state.

6. Reconcile current state against the Production Adoption Review before any mutation.

At the time this prompt was authored, the review baseline was:

```text
backstage-docs@f20eca1
platform-devops-developer-portal@b08e7b2
GitOps d1/desired-state@50564c9
Production Adoption Review: CONDITIONAL_GO
```

These are orientation points only. **Current canonical/live state wins.**

If docs and live state disagree materially, record the conflict before changing anything security-sensitive.

---

# 2. The only five conditions authorized

The source of truth is `docs/delivery/production-adoption-review.md`, section **Conditions before rollout**.

Do not invent a sixth pre-rollout condition.

The five conditions are:

```text
C1. Commit/version-control the token-refresh manifests.
C2. Replace the operator-laptop-bound Delivery backend Kubernetes credential.
C3. Create and review the rollback/emergency runbook.
C4. Add and prove a visible alert for token-refresh failure.
C5. Replicate/prove non-human Git identities and branch policy on the actual production GitOps repo, if different from the sandbox.
```

Treat each condition independently as `OPEN`, `IN_PROGRESS`, `PASS`, `NOT_APPLICABLE`, or `BLOCKED`.

A condition is `PASS` only when the exact evidence below exists.

---

# 3. Phase A — Reconcile the actual first-rollout target

Before implementing C2 or C5, determine what the **actual first-rollout environment** is intended to be.

Identify, from canonical docs and current infrastructure only:

- which Backstage/Delivery backend runtime will issue the first real production promotion;
- which Kubernetes cluster/account hosts the first production target;
- which production namespace will be used;
- which GitOps repository and protected branch are the source of truth for that target;
- whether the real production GitOps repo is the same `d0-gitops-sandbox` repository or a different repository;
- which Entra tenant / ADO organization owns the real production Git repo.

### Critical anti-fabrication rule

Do **not** invent a production cluster, namespace, GitOps repo, tenant, or runtime merely to make this checkpoint pass.

If the first real production target has not been identified by existing architecture/configuration or explicit operator input, then:

- C1, C3 and C4 may still be closed where valid;
- C2 and/or C5 must remain `BLOCKED` if they depend on an unidentified target;
- the final checkpoint verdict cannot be `PASS`.

Do not silently reinterpret the sandbox as production.

---

# 4. Condition C1 — Durable token-refresh manifests

## Objective

Remove the contradiction found by the Production Adoption Review: the live token-refresh mechanism exists and runs, but its non-secret manifests are not actually present in version control.

## Required scope

Version-control **only non-secret declarative material** needed to reproduce the mechanism, such as:

- Namespace, if platform-owned and appropriate;
- ServiceAccount;
- Roles / RoleBindings;
- ConfigMap/script;
- CronJob;
- README/bootstrap instructions if needed.

Do not commit:

- client secret values;
- OAuth tokens;
- Kubernetes Secret data;
- copied kubeconfigs;
- secret hashes that reveal secret material;
- transient Job objects.

## Location

Use the platform-owned GitOps/config repository already identified by the current architecture. The Production Adoption Review suggested `bootstrap/token-refresh/` or equivalent.

Do not broaden `d1-control-plane` write scope solely to reconcile this directory if doing so would weaken the P1 authority boundary. A version-controlled bootstrap mechanism is acceptable for this condition if that is the current bounded architecture.

## Required Git workflow

- use a short-lived branch;
- open a PR;
- obey the existing branch policy;
- do not self-approve if policy forbids creator approval;
- do not bypass the policy;
- do not call this condition closed while the files exist only on the source branch.

## PASS evidence

C1 = `PASS` only after all are true:

1. PR merged/completed;
2. the governed/default intended branch contains the manifests;
3. `git ls-tree` / ADO tree API confirms the expected path at the merged revision;
4. no secret values are present in the committed diff;
5. the live CronJob remains functional after the commit;
6. docs explicitly correct the earlier false claim that these manifests were already committed during P1 residual closure.

---

# 5. Condition C2 — Production-appropriate Delivery Kubernetes identity

## Objective

Eliminate the Delivery backend dependency on an individual operator's laptop/home-directory kubeconfig for the environment that will execute the first real production rollout.

## First inspect the runtime model

Determine how the intended Backstage backend is actually hosted for the first rollout.

Use the smallest native identity pattern supported by that runtime, in this preference order where applicable:

1. workload identity / pod ServiceAccount / cloud-native identity;
2. mounted Kubernetes Secret containing a scoped kubeconfig/token where workload identity is not available;
3. another existing platform-managed non-human credential mechanism already used by the environment.

Do not build a new generic credential platform.

## Authority invariants

The replacement identity must preserve the already-proven narrow Delivery authority:

- can create/read the required Kargo Promotion resources for the authorized provider path;
- cannot patch production Deployments directly;
- cannot mutate Argo `Application` / `AppProject` objects unless explicitly required by the accepted provider path — it should not be;
- cannot create ClusterRoleBindings or escalate itself;
- cannot read unrelated Secrets;
- is not tied to a human workstation or user session.

## Configuration rule

Application configuration must not point to paths such as:

```text
/home/<person>/...
~/.backstage-delivery/...
/Users/<person>/...
```

for the production rollout runtime.

A local developer override may remain for local development if clearly separated from production configuration.

## PASS evidence

C2 = `PASS` only when:

1. the actual first-rollout backend runtime has a non-human credential provisioned;
2. runtime configuration references that credential without depending on an individual's filesystem;
3. positive authority test succeeds for the legitimate Kargo action;
4. negative tests prove the identity cannot directly mutate production Deployment, cannot mutate Argo control objects, cannot create cluster-scope escalation bindings, and cannot read unrelated Secrets;
5. the backend can start/use the configured provider path with this identity;
6. no credential material is committed to Git or printed in evidence.

If the actual production backend runtime does not yet exist or is not identified, C2 = `BLOCKED`; do not fake proof using the operator laptop.

---

# 6. Condition C3 — Rollback / emergency runbook

## Objective

Create the minimum usable operational procedure required by the review. This is documentation, not rollback automation.

## Required content

Create a concise platform-owned runbook covering at minimum:

### Trigger classification

How to decide between:

- **Git revert** — desired state itself is wrong;
- **Argo reconciliation issue** — Git desired state is correct but convergence fails;
- **Kargo/Delivery credential or promotion issue** — desired-state change was never successfully created;
- **application-runtime incident** where rollback is not the appropriate platform action.

### Git rollback path

Document the bounded normal path:

```text
identify known-good Git revision
→ create revert branch/commit
→ open PR to protected desired-state branch
→ obtain required independent approval
→ merge
→ observe Argo reconciliation
→ verify health and deployment revision
```

Do not document a normal happy-path direct `kubectl patch`, direct `argocd app sync` override, force push, or policy bypass.

### Roles / ownership

Name the operational roles, not vague "someone":

- rollback initiator;
- required approver role/person/team;
- platform owner;
- escalation contact/path if the normal approver is unavailable;
- who decides whether break-glass is necessary.

### Break-glass boundary

The runbook may describe when break-glass would be invoked, but this checkpoint does not need to build a new break-glass automation system.

Explicitly state that emergency direct mutation must be exceptional, attributable, and followed by restoration of Git authority/drift reconciliation.

## Review requirement

At least one human other than the document author must review/approve the runbook.

Do not have the same agent create and "self-approve" its own evidence.

## PASS evidence

C3 = `PASS` only when:

1. runbook exists in a durable platform-owned repository;
2. it contains all required sections above;
3. another human has reviewed/approved it, with attributable evidence (PR review, approval, or equivalent);
4. the final document is merged/published;
5. the runbook does not instruct operators to bypass normal Git authority for routine rollback.

If human review is pending, C3 remains `IN_PROGRESS` and the checkpoint must not claim PASS.

---

# 7. Condition C4 — Token-refresh failure alert

## Objective

Ensure token-refresh failure cannot remain silent for longer than the credential lifetime.

The goal is **one reliable visible alert**, not a monitoring platform.

## Design constraint

Prefer the smallest existing notification/monitoring mechanism already available in the environment.

Examples may include an existing notification controller, existing alert service, existing webhook sink, or another already-operated platform channel.

Do not deploy Prometheus/Grafana/Alertmanager/Loki solely for this condition.

Do not depend on someone manually checking `kubectl get jobs`.

## Required alert semantics

The mechanism must produce a visible signal when a scheduled refresh execution fails.

At minimum the alert should communicate:

- token-refresh job/cronjob identity;
- failure state;
- timestamp;
- environment/cluster context;
- operator action/runbook reference.

Do not include any token or secret value.

## Failure test safety rule

The Production Adoption Review requires a deliberate failure proof. Perform this **without breaking the live controller credentials**.

Preferred pattern:

- create a temporary isolated test execution/path using deliberately invalid test input or a test-only copy of the Job/CronJob;
- prove the alert fires;
- remove or disable only the temporary test artifact afterward if explicitly safe and within scope.

Do **not** revoke the live SP, corrupt the live credential Secret, or let the actual Argo/Kargo tokens expire merely to test the alert.

If the selected alert integration necessarily requires impacting live credentials, STOP and choose a safer mechanism or leave C4 open.

## PASS evidence

C4 = `PASS` only after:

1. alert configuration is durable/version-controlled where appropriate;
2. a controlled failure test causes a visible alert at the intended destination;
3. the alert contains no secret material;
4. a normal successful refresh does not emit the failure alert;
5. the real CronJob still completes successfully afterward;
6. the alert path does not require an interactive human session to function.

---

# 8. Condition C5 — Real production Git authority replication

## Objective

Ensure the actual repository that will hold the first real production desired state has the same non-human authority split proven in P1.

## First determine applicability

### If the first production GitOps repository is exactly the already-proven repository

Do not create duplicate service principals or policies.

Re-verify the existing repository/branch and classify C5 as `PASS` only if it is explicitly confirmed to be the repository intended for the first real production rollout.

### If a different production GitOps repository is already identified

Provision/prove the equivalent model there.

### If no actual production GitOps repository has been identified

C5 = `BLOCKED`.

Do not create a random "prod" repository merely to make this checkpoint green.

## Required authority model

### Reconciler reader

- non-human identity;
- Git read allowed;
- Git contribute/write denied;
- branch creation/write denied where appropriate.

### Promotion writer

- distinct non-human identity;
- may create the branch/PR workflow required for desired-state mutation;
- cannot force-push;
- cannot bypass branch policy;
- cannot directly push the protected production branch.

### Protected branch

At minimum preserve the proven P1 invariants:

- PR required;
- at least one independent approval;
- creator self-approval does not satisfy the policy;
- writer identity cannot bypass policy;
- strongest relevant human/admin direct-push path is denied or equivalently governed.

## PASS evidence

C5 = `PASS` only with live evidence against the actual first-rollout repository:

1. reader clone/read succeeds;
2. reader write attempt is denied;
3. writer short-lived branch push succeeds;
4. writer PR creation succeeds;
5. writer direct push to protected branch is denied;
6. writer force/policy-bypass rights are denied;
7. branch policy requires independent review;
8. no personal PAT/user SSH key is the final controller credential.

Use isolated/throwaway proof branches where required. Do not mutate production desired-state content merely to prove the ACL.

---

# 9. Phase B — Regression and authority re-verification

After implementing any conditions, rerun the highest-value regression checks against the final committed/live state.

At minimum:

## Delivery / Change regressions

- existing DeliveryService regression suite;
- eligibility-window / TOCTOU tests;
- same-target concurrency/idempotency tests;
- activity-binding/non-completion tests.

Use the repository's documented working test invocation rather than the previously known hanging workspace wrapper.

## Authority checks

Re-run, with isolated credentials where applicable:

- ordinary squad identity cannot patch production Deployment;
- ordinary pipeline identity cannot patch production Deployment;
- those identities cannot mutate Argo Application/AppProject;
- those identities cannot create Kargo Promotion;
- Delivery runtime identity retains only the intended Kargo authority;
- scoped Argo destination still cannot cross namespaces or self-escalate;
- Git reader still cannot write;
- Git writer still cannot bypass the protected branch.

## Live controller health

Confirm after all changes:

- token refresh is succeeding;
- Argo applications relevant to the sandbox/proven path remain `Synced/Healthy`;
- Kargo stages remain healthy enough that the legitimate path has not been broken;
- no human PAT has re-entered the live Git controller path;
- no broad cluster-admin binding was reintroduced.

Do not execute the real production rollout in this phase.

---

# 10. Decision for this checkpoint

Return exactly one checkpoint verdict:

```text
Pre-rollout condition closure: PASS
Pre-rollout condition closure: CONDITIONAL_PASS
Pre-rollout condition closure: FAIL
```

And separately:

```text
Ready for final rollout-readiness re-review: YES | NO
Production rollout executed: NO
```

## PASS

Only if C1–C5 are all `PASS` or a condition is legitimately `NOT_APPLICABLE` with explicit evidence.

## CONDITIONAL_PASS

Use only when substantial conditions are closed but one or more remain objectively pending due to a bounded external dependency such as:

- human PR/runbook approval;
- first production runtime not yet provisioned;
- actual production GitOps repo not yet designated;
- unavailable existing notification destination.

List the exact remaining action and owner.

## FAIL

Use when:

- implementation weakens an accepted authority boundary;
- a condition cannot be closed without architecture redesign;
- a required negative-authority test now succeeds unexpectedly;
- secrets are placed in Git;
- a human credential becomes the final production authority path again;
- the legitimate controller path is left broken;
- evidence contradicts the claimed final state materially.

A `PASS` does **not** itself authorize the rollout. It only means the five conditions are closed and the system is ready for the final narrow rollout-readiness re-review.

---

# 11. Documentation contract

Create/update:

```text
docs/delivery/pre-rollout-condition-closure.md
```

The evidence document must include, in this order:

1. canonical baseline SHA;
2. implementation/GitOps/live baseline;
3. actual first-rollout target inventory;
4. C1 status + evidence;
5. C2 status + evidence;
6. C3 status + evidence;
7. C4 status + evidence;
8. C5 status + evidence;
9. secrets-handling statement;
10. authority regression matrix;
11. functional regression results;
12. live controller health after changes;
13. files/repos/resources changed;
14. human approvals still pending, if any;
15. deviations/limitations;
16. final checkpoint verdict;
17. explicit statement: `Production rollout executed: NO`;
18. STOP.

Update `docs/delivery/README.md` with the factual result.

Update `prompts/README.md` only after the checkpoint is complete, moving this prompt to historical/completed status and pointing to the next authorized prompt only if the user separately authorizes one.

Do not rewrite historical P1 evidence to hide contradictions. Add a corrective note/link where necessary.

---

# 12. Security handling rules

- Never print Secret `.data` values.
- Never use `kubectl get secret -o yaml/json` in a way that exposes encoded secret material to logs.
- Never commit bootstrap credentials.
- Never paste OAuth tokens/client secrets into docs or PR descriptions.
- Use identity names, app IDs, resourceVersions, expiry dates, and success/deny results as evidence instead of credential values.
- If any secret is accidentally displayed, rotate it immediately, record the incident factually without reproducing the secret, and do not continue as if nothing happened.

---

# 13. STOP gate

When the five-condition execution/evidence checkpoint is complete:

**STOP.**

Do not:

- execute the first production rollout;
- choose the first application;
- create the production GMUD for that application;
- perform a real production promotion;
- start the two-week observation period;
- declare enterprise GA;
- start post-rollout hardening;
- return to Deployments UX polish.

The only authorized next step after a successful closure is a **separate final rollout-readiness re-review**, and even that requires explicit user authorization.
