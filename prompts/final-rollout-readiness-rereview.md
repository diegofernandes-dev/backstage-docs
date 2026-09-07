# Final Rollout Readiness Re-review — Delivery / GitOps

## Role

Act as an **independent Principal Platform Architect / Staff+ reviewer** performing the final gate before the **first controlled production rollout** of the accepted Delivery / GitOps architecture.

This is a **review-only checkpoint**.

You are not an implementer, not a release operator, and not the owner of the evidence you are reviewing.

The accepted architecture remains the baseline:

> **Change Management authorizes. Delivery promotes. Git declares. Argo CD reconciles. Kubernetes executes. Backstage composes the experience.**

Your question is now deliberately narrow:

> **Have all pre-rollout conditions from the Production Adoption Review been objectively closed on the actual first-rollout target, with enough live evidence to authorize exactly one bounded production rollout?**

Do not optimize for approval.

Do not create new generic maturity work merely to avoid making a decision.

---

# 0. Hard execution rules

This checkpoint is **READ/REVIEW ONLY**.

## You MAY

- fetch and read canonical documentation;
- inspect committed implementation/configuration;
- inspect the actual first-rollout Backstage/Delivery runtime;
- inspect the actual first-rollout Kubernetes cluster/namespace;
- inspect the actual GitOps repository/branch/policies/ACLs;
- inspect current Entra non-human identities and credential metadata without exposing secret values;
- inspect Argo CD / Kargo / Kubernetes state using read-only commands;
- inspect PRs and human review evidence;
- re-run existing non-mutating tests from committed baselines;
- run read-only `kubectl auth can-i` / equivalent authority checks;
- inspect alert history and controller health;
- update canonical documentation with the final review result.

## You MUST NOT

- implement or fix any missing condition;
- create or mutate Kubernetes resources;
- rotate credentials;
- change Git ACLs or branch policies;
- create or merge PRs;
- approve your own or any pending PR;
- alter Backstage / Delivery code or configuration;
- trigger a real production promotion;
- create a GMUD for the rollout;
- select a production application on behalf of the operator;
- start the two-week observation period;
- reopen Deployments UX work;
- reopen ADR-012;
- expand into HA/DR, generalized monitoring, secrets-platform work, audit UX, Kargo Stage governance, global Delivery workbench, or post-rollout hardening.

If a required condition is not already closed, **do not close it during this review**.

---

# 1. Canonical baseline protocol

Before making any conclusion:

1. In `diegofernandes-dev/backstage-docs`:

   ```bash
   git fetch origin main
   ```

2. Record the exact `origin/main` SHA.

3. Read, at minimum, in this order:

   ```text
   docs/delivery/README.md
   docs/delivery/production-adoption-review.md
   docs/delivery/pre-rollout-condition-closure.md
   docs/delivery/rollback-emergency-runbook.md
   docs/delivery/p1-residual-closure.md
   docs/delivery/p1-production-authority-hardening.md
   docs/delivery/eligibility-window-toctou.md
   docs/delivery/adr-012-adoption-rereview.md
   docs/adr/ADR-012-delivery-management-gitops-promotion.md
   ```

4. Inspect the current implementation SHA recorded by canonical docs and verify it against the actual implementation repository.

5. Inspect the actual first-rollout target inventory. Do not infer it from the sandbox.

Canonical/current state always wins over values embedded in this prompt.

---

# 2. Mandatory prerequisite gate — run this before the review

The final review must **NOT** proceed unless all prerequisites below are objectively satisfied.

Verify each item independently.

## P1 — C1 durable token-refresh manifests

Required:

- the token-refresh / alerting manifests are merged into the governed version-controlled location;
- the merged revision is identified;
- `git ls-tree` / repository API confirms the files exist at that merged revision;
- the live mechanism still corresponds materially to the committed manifests;
- no literal secret material is stored in Git.

The previously open `d0-gitops-sandbox` PR #82 is only an orientation point. Do not assume its status; inspect current state.

## P2 — C3 reviewed rollback/emergency runbook

Required:

- the runbook is merged into canonical documentation;
- at least one human other than the author reviewed/approved it;
- rollback initiator and approver responsibilities are explicit;
- a real escalation/secondary contact exists; do not accept a placeholder such as `TBD`;
- the runbook distinguishes Git-revert recovery from Argo-side operational remediation.

The previously open `backstage-docs` PR #1 is only an orientation point. Inspect current state.

## P3 — actual first-rollout target identified

All of the following must be concrete, not inferred:

```text
first workload/component
criticality
Backstage/Delivery backend runtime
Kubernetes cluster/account
production namespace
Argo production destination / ServiceAccount
GitOps repository
protected GitOps branch
ADO org/project/repository if applicable
Entra tenant / identity authority
platform owner
rollback approver/escalation contact
```

The workload must fit the already-authorized envelope: **one non-critical or medium-criticality component only**.

## P4 — C2 production Delivery identity closed

Required on the **actual runtime** from P3:

- production configuration no longer resolves to an operator laptop/home-directory kubeconfig;
- a non-human production credential/workload identity is actually mounted/provisioned to the runtime;
- that identity can perform the exact provider action Delivery requires;
- that identity cannot patch production Deployments directly;
- cannot mutate Argo `Application`/`AppProject`;
- cannot read arbitrary Kubernetes Secrets;
- cannot create cluster-scoped escalation objects;
- credentials are not copied from a human PAT/session.

A config placeholder or environment-variable name alone does not close C2.

## P5 — C5 production Git authority closed

Required on the **actual GitOps repository and protected branch** from P3:

- reconciler reader and Delivery/Kargo writer are distinct non-human identities;
- reader can read but cannot contribute/push;
- writer can create the required branch/PR workflow;
- writer cannot directly push to the protected desired-state branch;
- writer cannot force-push;
- writer cannot bypass branch policy;
- branch requires independent human review according to the actual production policy;
- creator self-approval does not satisfy the gate;
- no human PAT is in the controller authority path.

If the real production GitOps repository is the same repository used for the sandbox, prove that explicitly rather than assuming equivalence.

## P6 — C4 alerting still active

Required:

- token refresh remains healthy;
- the failure alert mechanism is enabled on the current live mechanism;
- recent scheduler/job state shows the mechanism has not silently stopped;
- the alert route still has permission to deliver its signal;
- no secret value appears in logs or alert payloads.

A historical test alone is not sufficient if the mechanism has since drifted.

---

# 3. Prerequisite outcome

If **ANY** prerequisite in §2 is missing, incomplete, blocked, only proposed, only present in an open PR, or tied to an unidentified target, return:

```text
Final rollout-readiness re-review: NOT_READY
Production rollout authorized: NO
```

List the exact unmet prerequisite(s), with evidence.

Then **STOP**.

Do not run the full review.

Do not implement the missing work.

This is intentional. `NOT_READY` is not a new architectural verdict and does not reopen ADR-012.

---

# 4. Full final review — only if all prerequisites pass

If every prerequisite in §2 passes, perform the following final review using the **actual first-rollout target**, not the sandbox as a substitute.

For each gate, classify evidence as one of:

```text
LIVE_VERIFIED
COMMITTED_CODE
COMMITTED_CONFIG
COMMITTED_DOC
PRIOR_EXECUTION_EVIDENCE
NOT_PROVEN
CONTRADICTED
```

Score each gate exactly as:

```text
PASS
PASS_WITH_FOLLOWUP
BLOCKER
```

Do not introduce a generic `CONDITION` score at this stage. The conditional phase already happened.

A newly discovered issue is either:

- a non-blocking follow-up; or
- a blocker to the first rollout.

---

# 5. Mandatory final gates

## Gate 1 — All five pre-rollout conditions are truly closed

Reconcile the final state of C1–C5 from `production-adoption-review.md` and `pre-rollout-condition-closure.md`.

Required:

```text
C1 PASS
C2 PASS
C3 PASS
C4 PASS
C5 PASS
```

No `IN_PROGRESS`, `BLOCKED`, `PARTIAL`, or open-PR dependency is acceptable.

## Gate 2 — First-rollout target is real and bounded

Verify the target matches exactly the authorized envelope:

- one component only;
- non-critical or medium criticality;
- one production namespace;
- one cluster/account for this wave;
- namespace-scoped Argo destination / execution identity;
- platform owner explicitly assigned;
- no second workload or second target included.

If the scope has expanded since the Production Adoption Review, score `BLOCKER` unless a new explicit architecture/adoption review authorized the wider scope.

## Gate 3 — Delivery runtime authority

Against the actual production runtime identity, prove/read-only verify:

- intended Kargo/Delivery action: allowed;
- direct production Deployment mutation: denied;
- Argo control-object mutation: denied;
- arbitrary Secret read: denied;
- cluster-admin/self-escalation: denied.

The runtime must not depend on the operator laptop.

## Gate 4 — Argo production execution authority

Verify the production `Application` uses the scoped destination and namespace-limited ServiceAccount/Role pattern.

Required:

- no accidental fallback to default in-cluster broad credentials;
- AppProject destination restriction matches the actual namespace;
- cluster resources remain disallowed for the application path unless explicitly justified by prior accepted architecture;
- representative cross-namespace and escalation checks remain denied.

Do not treat broad controller RBAC elsewhere as an automatic blocker if the actual production Application demonstrably executes through the scoped destination; record it as post-rollout hardening unless the scoped boundary is bypassable.

## Gate 5 — Git authority and branch governance

Verify live on the actual first-rollout GitOps repo:

- Argo reader is read-only;
- Delivery/Kargo writer follows PR-gated mutation;
- protected branch rejects writer direct push;
- policy-bypass permissions remain denied;
- independent reviewer is required;
- the desired-state branch is the authority for production;
- production config is not dependent on a bootstrap-only branch or local workspace.

## Gate 6 — Credential lifecycle and alerting

Verify:

- token refresh is scheduled and healthy;
- committed manifests represent the mechanism;
- watchdog/failure alert is active;
- alert route is functional;
- non-human identity markers remain in live controller credentials;
- SP credential expiry dates are known and do not fall inside the planned first-rollout/observation window.

Future long-term rotation maturity may remain follow-up if it cannot break the bounded first rollout.

## Gate 7 — Change authorization / GMUD boundary

Using committed implementation and prior proof, verify no regression in:

- PRD requires Change binding;
- binding pins `changeId + activityId + release + target`;
- fresh eligibility is checked at dispatch;
- half-open execution window remains enforced;
- Delivery cannot turn a successful deployment into completion of the whole Change;
- Change Management authorizes; Delivery executes promotion.

Do not create a production GMUD in this review.

## Gate 8 — Concurrency and idempotency

Re-run existing committed regression tests and verify:

- same-target dispatch exclusion remains green;
- idempotent retry remains green;
- provider call happens at most once for the exercised concurrency contract;
- no config changes introduced since the last checkpoint invalidate these assumptions.

Do not manufacture a live production race.

## Gate 9 — Legitimate promotion path readiness

Read-only inspect the actual production-equivalent wiring and ensure the intended path is complete:

```text
trusted CI / immutable ReleaseCandidate
-> Delivery request
-> Change binding / eligibility for PRD
-> Kargo/provider dispatch
-> protected Git PR
-> independent human review
-> merge
-> Argo reconciliation
-> Kubernetes target
-> Backstage projection
```

This gate does **not** require executing the first real production promotion during the review.

It requires proving that no known missing wiring would make the first attempt experimental infrastructure assembly.

## Gate 10 — Rollback / emergency readiness

Verify the merged runbook against the actual first-rollout repo/branch/cluster.

Required:

- known-good revision can be identified;
- revert PR path exists;
- approver/escalation contact is available;
- Argo self-heal/prune behavior matches the runbook assumptions;
- normal rollback does not require force-push, branch-policy bypass, or imperative production mutation;
- manual Argo intervention is clearly classified as an exception, not the default rollback path.

## Gate 11 — Failure visibility

Verify there is a credible way for the platform owner to detect at minimum:

- token refresh failure/staleness;
- Argo `OutOfSync` / reconciliation error;
- Kargo promotion failure;
- production application health degradation through the currently available surfaces.

Do not require a generalized observability platform unless the actual first rollout cannot be operated safely without it.

## Gate 12 — Audit/evidence chain

Verify the first rollout can be reconstructed from durable records:

- release/artifact digest;
- target;
- requester;
- Change/activity binding;
- eligibility decision;
- Git PR/commit;
- human reviewer/approval;
- reconciliation revision/status;
- timestamps sufficient to establish sequence.

Known UI gaps such as Commit/Branch/Aprovador being unavailable may remain follow-up if the underlying authority evidence is reconstructable from durable systems.

## Gate 13 — No ordinary bypass regression

Re-run the final high-value negative checks using current identities/configuration:

- ordinary squad identity cannot patch production Deployment;
- ordinary pipeline identity cannot patch production Deployment;
- neither can mutate Argo `Application`/`AppProject`;
- neither can create Kargo Promotion;
- prior `ado-agent-cluster-admin` bypass remains absent;
- Delivery runtime identity cannot escalate beyond its intended provider action.

## Gate 14 — No hidden scope expansion

Explicitly confirm this review does **not** authorize:

- a second workload;
- a second production namespace;
- a second cluster/account;
- enterprise-wide GA;
- removal of human review from protected production desired state;
- global Delivery workbench;
- further Deployments UX redesign;
- broad Argo controller RBAC cleanup;
- HA/DR completion;
- automated rollback;
- generalized secrets platform;
- full Kargo control-plane Git governance.

Those remain separate follow-ups unless already completed independently.

---

# 6. Mandatory regression suite

Re-run the current committed equivalents of the high-value suites documented by the pre-rollout checkpoint.

At minimum, if paths remain valid:

```bash
cd packages/backend && CI=true yarn test \
  src/modules/delivery/DeliveryService.test.ts \
  src/modules/changeManagement/authorization/EligibilityService.test.ts \
  --no-coverage --watchAll=false

cd packages/app && CI=true yarn test \
  src/modules/catalogEntityTabs/DeploymentsTab.test.tsx \
  src/modules/catalogEntityTabs/index.test.ts \
  --no-coverage --watchAll=false
```

If the repository structure changed, use the current equivalent suites and document the substitution.

Do not use browser aesthetics as a review gate.

---

# 7. Decision semantics

After all gates are evaluated, return **exactly one** of these final outcomes:

## GO

Use only when:

- every prerequisite in §2 passed;
- no mandatory final gate is `BLOCKER`;
- the actual target is identified and bounded;
- authority separation is proven on the actual target;
- rollback and alerting are ready;
- the legitimate path is credibly wired;
- remaining issues are genuinely post-rollout follow-ups.

`GO` means:

> The platform may execute **one** controlled first production rollout inside the exact envelope below.

It does **not** mean general availability.

## NO_GO

Use when:

- all prerequisites were nominally present, but final live review discovers a blocker or contradiction that makes the first rollout unsafe or ungoverned.

Examples:

- production Delivery identity is broader than documented;
- writer can bypass protected branch;
- reader can write;
- production Argo path silently falls back to broad cluster credentials;
- rollback mechanism assumptions are false on the real target;
- alerting is not actually active;
- actual scope is more critical/broader than authorized;
- ordinary squad/pipeline mutation is possible.

Do not return `CONDITIONAL_GO` here. If a new blocker is discovered, return `NO_GO` and name the smallest corrective checkpoint separately.

---

# 8. Exact authorized rollout envelope if GO

If and only if the verdict is `GO`, authorize exactly:

```text
Workloads:           1
Criticality:         non-critical or medium-criticality only
Production targets:  1 namespace
Clusters/accounts:   1
Platform owner:      named and reachable
Human Git review:    mandatory
Observation period:  2 weeks
Expansion rule:      no second workload until observation closes
Success requirement: at least one successful real promotion cycle + stable Synced/Healthy state
```

The first rollout itself remains a **separate execution activity**.

This review does not perform it.

---

# 9. Required output

If prerequisite gate fails, record only the bounded result needed to explain `NOT_READY` and STOP.

If the full final review runs, write:

```text
docs/delivery/final-rollout-readiness-review.md
```

The document must contain:

1. canonical docs SHA;
2. implementation SHA;
3. actual first-rollout target inventory;
4. exact GitOps repo/branch/revision;
5. prerequisite matrix P1–P6;
6. 14-gate final evidence matrix;
7. regression results;
8. final negative-authority results;
9. rollback/runbook verification;
10. credential/alert health verification;
11. exact residual follow-ups;
12. final verdict;
13. if GO, the exact authorized rollout envelope;
14. explicit statement that the rollout was **not executed by this review**.

Update `docs/delivery/README.md` factually with the new gate state.

Commit documentation only.

---

# 10. Final STOP gate

After recording the result:

```text
Final rollout-readiness re-review: GO | NO_GO | NOT_READY
Production rollout authorized: YES | NO
Production rollout executed by this review: NO
```

Then **STOP**.

Do not execute the production rollout.

Do not create the production GMUD.

Do not select or onboard another workload.

Do not begin the observation window.

Do not automatically create the next milestone.
