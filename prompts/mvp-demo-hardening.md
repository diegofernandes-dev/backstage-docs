# MVP Demo Hardening — Execution Prompt

## Role

Act as a senior Staff+/Principal Platform Engineer responsible for stabilizing an already-proven Backstage Delivery MVP for a real stakeholder demonstration.

This is **not** an architecture redesign task.
This is **not** another spike.
This is **not** production hardening.

Your objective is to make the current vertical MVP **reliably demoable, repeatable, and easy to reset**, while preserving the accepted architecture and current product behavior.

The core rule is:

> **Make the current MVP reliably demoable. Do not redesign it.**

---

## Canonical source and baseline verification

Before changing anything:

1. `git fetch origin main` in `diegofernandes-dev/backstage-docs`.
2. Verify the latest canonical Delivery, GMUD, Golden Paths, and current-state documentation from `origin/main`.
3. Read at minimum:
   - `docs/delivery/mvp-vertical-delivery-slice.md`
   - `docs/delivery/d1-kargo-fit-evaluation.md`
   - `docs/delivery/d0-architecture-review.md`
   - `docs/delivery/README.md`
   - relevant ADRs, especially ADR-009 and ADR-012
   - the latest Golden Paths/current-state docs that cover the `Platform` tab and the recorded assessor 502 fix
4. Verify the current implementation branch and current implementation SHA before making edits.
5. If canonical docs have advanced and materially conflict with this prompt, STOP and report the conflict instead of guessing.

Treat canonical docs as the authority.

---

## Current accepted state

The current MVP has already demonstrated the following live end-to-end flow:

```text
ReleaseCandidate
  → DEV
  → HML
  → PRD request
  → CHANGE_REQUIRED
  → GMUD
  → ExecutionEligibility DENY / ALLOW
  → Kargo
  → Git
  → Argo CD
  → Kubernetes
  → Backstage Deployments projection
```

The same immutable artifact fingerprint was traceable across DEV, HML, and PRD.

The MVP result is `CONDITIONAL PASS`, with carried gaps such as:

- no dedicated automated test suite for the new Delivery module;
- Kargo PR-gated no-op promotion behavior can fail when no Git diff/PR exists;
- production-grade Argo/Kargo Kubernetes RBAC is not yet proven;
- Git writer vs reconciler credential separation remains open;
- some sandbox bootstrap/control-plane steps remain imperative;
- ADR-012 remains Proposed;
- production rollout is not authorized.

The recent `Platform` tab 502 regression has been separately fixed and documented. Do not regress it.

---

# GO — Authorized work

You are authorized to perform only the smallest changes needed to make the existing MVP reliable for demonstration.

The work has four goals.

## Goal 1 — Stabilize the demo path

Exercise the current flow from a known starting state and identify any issue that could break a live demo.

Fix only real demo blockers or obvious regressions.

Examples of in-scope issues:

- incorrect request state projection;
- broken Backstage actions or links;
- wrong Delivery/Change binding behavior;
- stale state that prevents retry;
- provider projection not refreshing correctly;
- an error that prevents a genuine DEV/HML/PRD happy path;
- a known no-op promotion bug that can predictably break the demo;
- regression of the existing `Platform` tab;
- inability to reset or seed the sandbox into a known state.

Do not refactor code merely because it could be cleaner.

### Kargo no-op requirement

The current MVP hit the documented Kargo PR-gated no-op limitation: when the target Git manifest already contains the desired digest, no Git diff is produced, no PR exists, and an unconditional `git-wait-for-pr` can fail.

For demo reliability, resolve this only as far as necessary to make the expected demo path deterministic.

Preferred order:

1. first determine whether the demo can be made deterministic by using an explicit reset/seed state that guarantees a real change;
2. if a narrow Kargo template fix can safely treat no-op as successful/idempotent without changing architecture, implement and verify it;
3. do not build a general workflow engine or custom state machine to solve this.

Document exactly which approach is chosen.

---

## Goal 2 — Add the minimum automated regression tests

The new Delivery module currently has a real test gap. Add a focused test suite that protects the most important MVP invariants.

At minimum prove:

1. PRD request without a bound Change cannot dispatch and produces `CHANGE_REQUIRED`.
2. PRD request with a Change that is not authorized cannot dispatch and produces `NOT_AUTHORIZED` / equivalent deny result.
3. An authorized eligible Change permits dispatch.
4. Denied dispatch creates no Kargo Promotion side effect.
5. Dispatch/retry is idempotent enough that the same logical DeploymentRequest does not create duplicate provider executions unexpectedly.
6. Provider projection correctly reflects at least the essential Kargo/Argo terminal outcomes used by the UI.
7. DEV/HML targets that do not require Change remain dispatchable under the existing MVP policy.

If practical with the existing test harness, add one focused regression check that protects the Component tabs so the recent `Platform` 502 fix and `Deployments` tab integration are not accidentally broken by backend registration/routing changes.

Do not create a massive test framework. Reuse the repository's established test patterns.

---

## Goal 3 — Create a deterministic demo baseline

Create a documented and reproducible sandbox/demo baseline.

The baseline must identify:

- the Backstage implementation SHA used for the demo;
- the GitOps repository/branch and baseline revision;
- the component used for the demo;
- the artifact repository;
- the immutable artifact fingerprint/digest;
- Kargo Project/Warehouse/Stage names involved;
- Argo Application names;
- Kubernetes namespaces;
- the expected initial desired-state digest for DEV/HML/PRD;
- whether any stage must intentionally start on an older/different digest to guarantee a real promotion;
- required sandbox services/controllers;
- any known local credentials/config prerequisites, without writing secrets into Git.

Provide a reset procedure that can restore the sandbox to the expected demo starting point.

The reset must prefer declarative/idempotent operations where already available, but may preserve documented sandbox-only imperative setup if replacing it would expand scope.

Do not turn the reset into a production bootstrap system.

---

## Goal 4 — Produce a 5–10 minute demo runbook

Create a concise demo runbook for a live stakeholder presentation.

It should be possible for another engineer to follow it without reconstructing architecture history.

The demo should visibly show:

```text
1. Component in Backstage
2. Current DEV/HML/PRD deployment state
3. ReleaseCandidate / immutable artifact identity
4. DEV promotion
5. HML promotion
6. PRD request blocked by CHANGE_REQUIRED
7. GMUD creation or association
8. DENY while authorization is pending
9. Approval / authorization decision
10. Fresh ALLOW result
11. PRD dispatch
12. Kargo/Git/PR path
13. Argo reconciliation
14. Kubernetes runtime state
15. Backstage Deployments tab showing the result
```

The runbook must call out which steps require a human action, such as ADO PR approval/merge or authorization decision.

Include a short fallback/recovery section for the most likely demo failures.

---

# Explicit non-goals / NO-GO

Do **not** do any of the following unless an actual demo-blocking defect proves it is unavoidable and you STOP for approval first:

- accept or rewrite ADR-012;
- redesign Delivery or Change Management boundaries;
- introduce a generic workflow/state-machine engine;
- implement production-grade multi-cluster topology;
- solve production RBAC comprehensively;
- solve Git writer/reconciler credential separation comprehensively;
- add HA/DR;
- implement production break-glass;
- implement automated production rollback;
- implement full supply-chain signing/provenance;
- redesign Golden Paths;
- redesign the `Platform` tab;
- turn Backstage into the runtime authority;
- move business authorization into Kargo;
- add provider-specific fields to canonical Change objects;
- continue into the next architecture checkpoint after this one.

The current architecture boundaries remain:

> Change Management authorizes. Delivery promotes. Git declares. Argo CD reconciles. Kubernetes executes. Backstage composes the experience.

Kargo is the accepted promotion-controller candidate for this MVP scope, not business approval authority.

---

# Required execution discipline

## Evidence first

For every meaningful claim, capture factual evidence from the real running sandbox and/or repository state.

Do not rely only on Kargo UI status, Backstage UI state, or an agent's assumption.

Where applicable, independently verify using:

- persisted Delivery/Change state;
- Kargo Promotion/Freight/Stage status;
- Git revision/content;
- ADO PR state;
- Argo Application sync/health/revision;
- Kubernetes runtime imageID/digest;
- Backstage frontend/backend behavior.

## Narrow fixes

When you find a defect:

1. reproduce it;
2. identify exact root cause;
3. make the smallest safe fix;
4. add a regression test when appropriate;
5. re-run the affected happy path;
6. check unrelated MVP behavior was not regressed.

Do not perform opportunistic cleanup.

## Secrets

Never commit credentials, PATs, tokens, kubeconfigs, or secret values to docs, source, GitOps manifests, or evidence.

---

# Acceptance criteria

This checkpoint may return `PASS`, `CONDITIONAL_PASS`, or `FAIL`.

## PASS

All of the following are true:

- the demo flow is repeatable from the documented baseline;
- the critical Delivery/Change gate invariants have automated regression coverage;
- the expected demo path does not depend on an undocumented manual repair;
- the Kargo no-op behavior is either safely handled or deterministically avoided by the documented reset/baseline;
- `Platform` and `Deployments` tabs both work;
- PRD remains fail-closed before authorization;
- no unauthorized Kargo Promotion is created;
- the same immutable artifact remains traceable across stages;
- a 5–10 minute demo runbook exists;
- a reset procedure exists;
- evidence and SHAs are recorded.

## CONDITIONAL_PASS

The demo is reliable enough to present, but one or more non-blocking sandbox/demo limitations remain and are explicitly documented with a practical workaround.

Do not use `CONDITIONAL_PASS` to hide a flaky or non-repeatable critical path.

## FAIL

Use `FAIL` if any of the following remains true:

- PRD can dispatch before authorization;
- a denied dispatch can create a provider execution side effect;
- the demo cannot be reset/repeated predictably;
- critical Backstage UI path is broken;
- the artifact promoted to PRD cannot be correlated to the intended ReleaseCandidate;
- a fix would require redesigning architecture beyond this prompt's authority.

---

# Documentation updates

Update canonical docs with factual results.

Preferred outputs:

- `docs/delivery/mvp-demo-hardening.md` — execution result, evidence, fixes, remaining gaps, final verdict;
- update `docs/delivery/README.md` only if the current gate/status needs to change;
- update relevant Golden Paths/current-state docs only if a real behavior/state changed;
- preserve historical D0/D1/MVP result docs as historical evidence; do not rewrite history.

Record:

- canonical docs baseline SHA used at execution start;
- implementation branch and before/after SHA;
- relevant GitOps SHA(s);
- exact tests run and results;
- demo reset procedure;
- demo runbook;
- known remaining demo limitations;
- final verdict.

ADR-012 must remain unchanged unless a separate explicit architecture-acceptance prompt authorizes changing its status.

---

# Final report format

Return exactly these sections:

## 1. Baseline
Canonical docs SHA, implementation branch/SHA, GitOps baseline, Kargo/Argo/Kubernetes versions used.

## 2. Verdict
`PASS`, `CONDITIONAL_PASS`, or `FAIL`.

## 3. Demo Blockers Found
Only actual reproduced issues.

## 4. Fixes Applied
Smallest fixes, with implementation SHA(s).

## 5. Regression Tests Added
Tests and results.

## 6. Kargo No-op Handling
Exactly how the known no-op behavior is handled for the demo.

## 7. Demo Baseline / Reset
Exact reproducible starting state and reset procedure.

## 8. Demo Runbook
The final 5–10 minute sequence.

## 9. End-to-End Verification
ReleaseCandidate → DEV → HML → PRD → Change/ALLOW → Kargo → Git → Argo → Kubernetes → Backstage evidence.

## 10. Remaining Gaps
Only real remaining gaps, clearly separated into demo risk vs production hardening.

## 11. Documentation Updated
Paths and docs commit SHA.

## 12. STOP
State explicitly that no architecture redesign, production hardening, ADR-012 acceptance, or subsequent checkpoint was executed.

---

# Final STOP gate

After the demo-hardening evidence and documentation are complete:

**STOP.**

Do not start production hardening.
Do not accept ADR-012.
Do not start the next Delivery milestone.
Do not redesign the MVP.

Wait for an explicit next GO.