# P1 Residual Closure — Live Credential Cutover and Production-Review Readiness

## Role

Act as a senior Staff+/Principal Platform Engineer closing the **remaining bounded P1 production-authority residuals** after P1 returned `CONDITIONAL_PASS`.

This is a narrow execution-and-evidence checkpoint.

It is **not**:

- a new architecture review;
- a new Delivery milestone;
- a Backstage UX/design task;
- a general security-hardening sweep;
- the production-adoption review itself.

The accepted architecture is fixed:

> Change Management authorizes. Delivery promotes. Git declares. Argo CD reconciles. Kubernetes executes. Backstage composes the experience.

Core rule:

> **Close only the concrete P1 residuals needed to make the system ready for a separate production-adoption review. Do not reopen architecture or UI.**

---

## 0. Canonical baseline protocol

Before changing anything:

1. In `diegofernandes-dev/backstage-docs`:
   ```bash
   git fetch origin main
   ```
2. Record the current `origin/main` SHA.
3. Read, at minimum, in this order:
   - `docs/delivery/README.md`
   - `docs/delivery/p1-production-authority-hardening.md`
   - `docs/delivery/deployments-ux-v2-responsive-polish.md`
   - `docs/delivery/eligibility-window-toctou.md`
   - `docs/delivery/adr-012-adoption-rereview.md`
   - `docs/adr/ADR-012-delivery-management-gitops-promotion.md`
4. Inspect the actual current implementation branch/SHA in `platform-devops-developer-portal` before editing.
5. Inspect the actual live GitOps/Kargo/Argo/Kubernetes/ADO state before assuming any residual is still open.

Known last documented implementation references are only orientation points and must be re-verified:

- Deployments UX responsive polish: `platform-devops-developer-portal@b08e7b2`;
- P1 authority checkpoint: `CONDITIONAL_PASS`;
- ADR-012: `Accepted`;
- production rollout: `NO-GO`.

If canonical docs and live state disagree materially, STOP and report the conflict before modifying security-sensitive state.

---

## 1. Product/UX freeze — mandatory

The Component → Deployments UX has already consumed disproportionate iteration time and its current technical checkpoint is recorded as `PASS`.

For this checkpoint:

- **do not modify Deployments UX**;
- do not change cards, spacing, breakpoints, typography, icons, labels, masthead, tables, events, history, GMUD layout, or responsive behavior;
- do not create another visual-polish prompt;
- do not use browser review as an excuse to restart product design.

Treat the current Deployments screen as:

```text
functionally accepted / demoable
additional visual polish deferred
```

Browser use in this checkpoint is authorized only for a minimal smoke/regression check after security changes.

If a UI defect is discovered that does not block the legitimate delivery path, document it as deferred and continue. Do not fix it here.

---

# 2. Objective

Close the three concrete residual areas carried from P1:

1. **Live Git credential cutover**
   - Argo must use the non-human read-only reconciler identity live;
   - Kargo/Delivery promotion path must use the non-human writer identity live;
   - short-lived Entra tokens must have a sustainable refresh mechanism;
   - human PATs must no longer be the active credentials for these live GitOps controller paths.

2. **Durable GitOps steady state**
   - verify the P1 GitOps changes are merged/applied from the governed branch;
   - do not rely on bootstrap-applied branch content as the final steady state;
   - no branch-policy bypass is permitted to achieve closure.

3. **Regression + final authority proof on the actual live state**
   - rerun the relevant regression suites from a committed implementation baseline;
   - rerun the high-value positive and negative authority checks **after** the live credential cutover and durable GitOps state are in place.

The desired endpoint is not `production GO`.

The desired endpoint is:

```text
P1 residual closure: PASS | CONDITIONAL_PASS | FAIL
Ready for separate production-adoption review: YES | NO
Production rollout: still NO-GO
```

---

# 3. Phase A — Reconcile residual inventory before implementation

Before changing anything, inspect each previously documented residual and classify it:

```text
OPEN
ALREADY_CLOSED
CHANGED
BLOCKED_BY_HUMAN_ACTION
NO_LONGER_RELEVANT
```

At minimum inspect:

- current Argo repo credential identity;
- current Kargo Git writer credential identity;
- whether personal/human PATs remain active in either controller path;
- current token lifetime/refresh behavior;
- ADO PR #79 state;
- ADO PR #80 state;
- governed branch current revision;
- `d1-control-plane` Argo application state;
- `d1-prd` Argo application state;
- scoped destination/service-account state;
- absence/presence of the previously discovered `ado-agent` cluster-admin bypass and stale escalation bindings;
- current Delivery backend Kubernetes credential scope;
- implementation working tree cleanliness and latest committed SHA.

Do not blindly reimplement a solution because the old P1 document recommended it.

---

# 4. Phase B — Live non-human Git credential cutover

## 4.1 Required identities

The existing P1 separation model remains the target unless live evidence disproves it:

- **Argo reconciler reader**: non-human Entra service principal, Git read only;
- **Kargo/Delivery writer**: separate non-human Entra service principal, only the Git/PR capabilities required for governed desired-state mutation.

Do not merge these identities.
Do not reintroduce a personal PAT.
Do not expand ACLs merely to make a test pass.

## 4.2 Refresh mechanism

P1 proved the service principals but did not leave ~1-hour OAuth tokens installed live because they would expire without renewal.

Implement the **smallest maintainable sandbox mechanism** that keeps the live Argo and Kargo credentials current.

A small Kubernetes CronJob or equivalent narrow token-refresh mechanism is acceptable if it remains the smallest practical solution in the actual stack.

Do not build:

- a generic credentials platform;
- a custom secret manager;
- a controller/operator framework;
- a new platform-wide identity abstraction.

Security constraints:

- no client secret, OAuth token, PAT, private key, or refresh credential may be committed to Git;
- do not print credential values in logs, docs, terminal transcripts, screenshots, or evidence;
- use existing secret-management/bootstrap conventions available in the sandbox;
- document secret **names and ownership**, never secret values;
- scope the refresh mechanism so it can mutate only the credential Secrets it must maintain;
- if a more privileged bootstrap identity is required to provision the refresher, keep that bootstrap operation explicit and bounded.

## 4.3 Required refresh proof

Do not merely create a CronJob/YAML and call it done.

Prove the mechanism executes successfully.

At minimum:

1. trigger/observe a successful refresh run;
2. verify both live credential Secrets are updated by the intended mechanism;
3. prove a second refresh is safe/idempotent;
4. prove token renewal does not require a human session;
5. prove controller functionality after refresh.

You do **not** need to wait one hour for natural expiry if the same renewal path can be driven safely on demand.

Do not expose token values in the evidence. Use timestamps, Secret resourceVersions, safe fingerprints/metadata, job status, or controller behavior instead.

---

# 5. Phase C — Durable GitOps steady state

Verify P1 GitOps changes are part of the protected desired-state branch and not merely live bootstrap state.

Specifically inspect PR #79 and PR #80 (or their superseding PRs if the live history changed).

Required behavior:

- if already merged, record merge revision and continue;
- if superseded, identify the actual durable revision and prove equivalence;
- if awaiting a legitimate human approval, **do not bypass policy**;
- do not weaken reviewer policies;
- do not direct-push as an administrator;
- do not self-approve using an identity that violates the intended separation of duties.

If human approval is still required, complete all independent work first, then report exactly:

```text
HUMAN_ACTION_REQUIRED
PR: <id>
Required action: <specific legitimate approval/merge step>
Why automation must not perform it: <policy reason>
```

After merge, require Argo to converge from the governed branch revision. A bootstrap-applied object matching an unmerged branch is not sufficient final evidence.

Verify at minimum:

- `d1-control-plane`: Synced / Healthy;
- `d1-prd`: Synced / Healthy;
- current revision corresponds to governed Git state;
- AppProject/Application authority controls are still Git-managed;
- scoped destination configuration is still active.

---

# 6. Phase D — Prove the legitimate live path after cutover

Run a real, bounded positive path using the **live controller credentials after cutover**.

The proof must establish:

### Argo reader

- can authenticate to/read the GitOps repository through the live Argo configuration;
- Argo can compare/reconcile the governed desired state;
- application reaches or remains `Synced/Healthy`;
- the same reader identity cannot write Git (negative proof in Phase E).

### Kargo/Delivery writer

- the live promotion path can authenticate with the writer identity;
- can create/push the required working branch when a real mutation is needed;
- can create a PR through the governed process;
- the PR/audit identity is the non-human writer, not the human operator;
- protected desired-state branch remains PR-only;
- no-op behavior remains safe/idempotent.

Prefer a controlled sandbox promotion that makes a reversible, non-destructive desired-state change using the existing immutable-artifact path.

Do not rebuild application artifacts solely for this proof.

If a real Git mutation would create unnecessary state churn, use the smallest existing sandbox-safe proof that still exercises the **live controller credential**, not an isolated credential test disconnected from the controller.

---

# 7. Phase E — Repeat the high-value negative authority suite

The security claim must be re-proven against the **final live state**, not inherited from pre-cutover P1 evidence.

At minimum execute and record these negative checks with isolated credentials/contexts:

## Git

1. Argo reader attempts Git write/new branch → **DENIED** by ACL.
2. Writer attempts direct push to protected `d1/desired-state` → **DENIED** by branch policy.
3. Ordinary squad/build-service identity attempts governed Git write → **DENIED** unless explicitly intended by current policy.

## Kubernetes / Argo

4. PRD scoped Argo destination identity attempts mutation outside `d1-prd` → **DENIED**.
5. PRD scoped Argo identity attempts cluster-scoped privilege escalation (`ClusterRoleBinding` or equivalent) → **DENIED**.
6. PRD scoped Argo identity attempts access to unrelated privileged namespace resources → **DENIED**.

## Squad/pipeline bypass

7. Verify the previously discovered `ado-agent-cluster-admin` binding remains absent/remediated.
8. Verify stale cluster-admin bindings that could become active by namespace/service-account recreation remain removed or otherwise fail closed.
9. Ordinary pipeline identity attempts direct mutation of the protected PRD workload → **DENIED**.

## Delivery backend Kubernetes authority

10. Verify the Delivery backend does not fall back to ambient cluster-admin kubeconfig/credentials.
11. Prove its active Kubernetes identity is bounded to only the provider operations it requires.

Important methodology rule:

> Never run Kubernetes negative tests with an ambient admin client certificate still present in the kubeconfig.

Use fully isolated kubeconfigs/contexts for every identity under test so the earlier P1 false-positive authentication mistake cannot recur.

Do not create a persistent escalation object as a test artifact. Clean up any safe probe resources immediately.

---

# 8. Phase F — Regression suite from committed state

Before final verdict, establish a reproducible implementation baseline.

Requirements:

1. identify and record implementation SHA before this checkpoint;
2. commit only the narrow authorized residual-closure changes;
3. leave unrelated/brownfield WIP untouched;
4. run the required regression tests from the final committed SHA.

At minimum execute the existing suites covering:

- Delivery service / dispatch / concurrency / provider projection;
- Change execution eligibility / authorization integration relevant to PRD dispatch;
- Deployments tab loader/smoke guard;
- provider/security scripts or tests introduced by P1 where they exist.

If the repository's standard full regression command is practical and stable, run it as well and record passed/skipped/pre-existing failures separately.

Do not fix unrelated test failures in this checkpoint.

The Deployments browser validation here is **smoke only**:

- page loads;
- release/environment/governance data still renders;
- no material regression from credential cutover.

Do not perform visual polish.

---

# 9. No-human-credential final-state assertion

Before declaring PASS, explicitly prove the final live controller path is no longer dependent on a human Git credential.

The final evidence must answer:

```text
Argo Git credential owner: <non-human identity>
Argo Git write capability: DENIED
Kargo/Delivery Git credential owner: <non-human identity>
Kargo/Delivery protected-branch direct push: DENIED
Token refresh: AUTOMATED / NON-HUMAN
Human PAT still active in these controller paths: NO
```

Do not reveal credentials while proving this.

If a human PAT remains active as the controller fallback, PASS is not allowed.

---

# 10. Explicit NO-GO scope

Do **not** do any of the following:

- production rollout decision;
- production-adoption review;
- change ADR-012 status;
- additional Deployments UX work;
- new Backstage screens;
- global Delivery workbench;
- HA/DR;
- automated rollback program;
- break-glass architecture;
- supply-chain/signing expansion;
- generic secret-management platform;
- generalized identity broker;
- multi-cluster production rollout;
- organization-wide RBAC redesign;
- unrelated pipeline-template work;
- unrelated Golden Paths work.

If one of these is genuinely required to close a P1 residual, STOP and request explicit authorization rather than expanding scope.

---

# 11. Verdict rules

## PASS

Return `PASS` only if all of these are true:

- live Argo uses the non-human read-only identity;
- live Kargo/Delivery writer path uses the distinct non-human writer identity;
- automated/non-human token renewal is proven and repeatable;
- human PATs are no longer active in those controller paths;
- governed GitOps changes are durably merged/applied from the protected branch;
- positive controller path succeeds after cutover;
- required negative Git/Kubernetes/pipeline checks deny as expected;
- no cluster-admin squad/pipeline bypass remains;
- Delivery backend does not use ambient cluster-admin authority;
- regression suites required by this prompt are green except explicitly pre-existing unrelated skips/failures;
- final state is documented from committed SHAs.

Set:

```text
Ready for separate production-adoption review: YES
Production rollout: NO-GO (unchanged; review not executed here)
```

## CONDITIONAL_PASS

Use only for a narrow external/human-action dependency that does not invalidate the technical closure already proven, for example a legitimate protected-branch approval still awaiting a separate human approver.

Do not use `CONDITIONAL_PASS` to hide:

- a human PAT still active;
- unproven token refresh;
- an authority bypass;
- missing negative tests;
- broken live reconciliation;
- failed regressions caused by this checkpoint.

`Ready for separate production-adoption review` may be `NO` if the remaining condition materially affects the final steady state.

## FAIL

Return `FAIL` if any of these occur:

- Argo still has Git write authority;
- Kargo/Delivery writer can bypass protected-branch policy;
- a human credential remains required in the live controller path;
- credential refresh is manual-only;
- ordinary squad/pipeline can mutate PRD runtime or desired state directly;
- Argo destination identity can escalate or mutate outside intended scope;
- durable GitOps control state is not attributable to the governed branch;
- the legitimate delivery/reconciliation path no longer functions after cutover;
- the checkpoint requires redesigning accepted architecture.

---

# 12. Documentation requirements

Preferred evidence document:

```text
docs/delivery/p1-residual-closure.md
```

Update `docs/delivery/README.md` only to reflect factual checkpoint status.

Do not rewrite historical evidence documents.
Do not mark production rollout `GO`.
Do not change ADR-012 from `Accepted`.

Record:

- docs baseline SHA;
- implementation before/after SHA;
- GitOps governed revision;
- PR #79/#80 or superseding PR states;
- live credential owners by identity name only;
- refresh mechanism resource names and execution evidence without secret material;
- positive proof;
- negative proof matrix;
- regression results;
- human action required, if any;
- final verdict;
- `Ready for separate production-adoption review: YES|NO`.

Also record the UX freeze decision explicitly:

> Deployments UX is functionally accepted/demoable. Additional visual polish is deferred and was not part of this checkpoint.

---

# 13. Final report format

Use exactly these sections:

```markdown
# P1 Residual Closure Result

## 1. Baseline
## 2. Verdict
## 3. Residual Inventory Reconciliation
## 4. Live Credential Cutover
## 5. Token Refresh Proof
## 6. Durable GitOps Steady State
## 7. Legitimate Path Proof
## 8. Negative Authority Proofs
## 9. Regression Tests
## 10. Human Credential Elimination
## 11. Human Action Required
## 12. Remaining Gaps
## 13. Production-Review Readiness
## 14. Documentation Updated
## 15. UX Freeze Confirmation
## 16. STOP
```

Section 13 must contain exactly:

```text
P1 residual closure: PASS | CONDITIONAL_PASS | FAIL
Ready for separate production-adoption review: YES | NO
Production rollout: NO-GO
```

---

# 14. STOP

After completing this checkpoint:

**STOP.**

Do not automatically run the production-adoption review.
Do not return to Deployments visual polish.
Do not start another Delivery milestone.

The next step, if and only if this checkpoint is sufficiently closed and the user explicitly authorizes it, is a separate **Production Adoption Review**.