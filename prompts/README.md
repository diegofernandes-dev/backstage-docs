# Agent Prompts

This directory stores the canonical long-form prompts used to execute tightly scoped architecture and implementation checkpoints.

The intent is to avoid repeatedly pasting large prompts into an agent session. A short launcher prompt should instruct the agent to fetch `backstage-docs@main`, read the relevant prompt in this directory, verify the current canonical architecture/docs baseline, execute only the authorized scope, update factual evidence/docs, and stop at the prompt gate.

## Rules

- Always `git fetch origin main` before using a prompt.
- The current canonical architecture/docs take precedence over assumptions embedded in an older prompt.
- Reading a prompt is not execution authorization unless the prompt says otherwise; the user must explicitly launch execution.
- A prompt authorizes only the work it explicitly marks as GO.
- STOP gates are mandatory.
- Agents must update canonical evidence/docs only with factual results.
- Do not expand scope to "helpfully" implement the next checkpoint.
- Prefer a demonstrable vertical product slice over additional horizontal hardening unless a real blocker requires it.

## Current execution prompt

- [`pre-rollout-condition-closure.md`](./pre-rollout-condition-closure.md) — close only the five explicit pre-rollout conditions from the independent Production Adoption Review `CONDITIONAL_GO`: (C1) make token-refresh manifests genuinely durable/version-controlled, (C2) replace the operator-laptop-bound Delivery Kubernetes credential with a production-appropriate non-human runtime identity, (C3) create and independently review the rollback/emergency runbook, (C4) add and safely prove a visible token-refresh failure alert without breaking live controller credentials, and (C5) replicate/re-verify the distinct reader/writer SP + protected-branch model on the actual first-rollout GitOps repository if it differs from the sandbox. The checkpoint must not invent a production target merely to pass, must not reopen ADR-012 or Deployments UX, and must not execute the production rollout. A successful result means only `Ready for final rollout-readiness re-review: YES`.

## Completed / historical prompts

- [`production-adoption-review.md`](./production-adoption-review.md) — completed with `CONDITIONAL_GO` for a narrow first production rollout only: one non-critical/medium-criticality component, one production namespace, platform-owner-attended, two-week observation before any second workload. The independent review evaluated all 14 mandatory gates against live Argo/Kargo/Kubernetes/ADO state and identified five objective pre-rollout conditions now owned by `pre-rollout-condition-closure.md`. ADR-012 remains Accepted; no rollout was executed by the review.
- [`p1-residual-closure-live-cutover.md`](./p1-residual-closure-live-cutover.md) — completed with `PASS`. Live Argo reader and Kargo/Delivery writer were moved from human PATs to the already-proven distinct non-human Entra service principals; automatic token refresh was implemented and re-verified across multiple unattended cycles; the governed P1 GitOps control diff was approved and merged through protected branch policy; `d1-prd`/`d1-control-plane` converged to the merged scoped-destination state; high-value positive/negative authority evidence and regressions were rechecked. Result: `Ready for separate production-adoption review: YES`.
- [`deployments-ux-v2-responsive-polish.md`](./deployments-ux-v2-responsive-polish.md) — completed with `PASS`; implementation recorded at `platform-devops-developer-portal@b08e7b2`. Responsive composition and laptop behavior were validated and documented. Further visual refinement is intentionally deferred; do not reopen this workstream during authority/rollout readiness work.
- [`deployments-ux-v2-implementation.md`](./deployments-ux-v2-implementation.md) — completed with `CONDITIONAL_PASS`; implementation recorded at `platform-devops-developer-portal@0163a49`. Implemented the approved release selector, environment views, GMUD/eligibility context, promotion history, and recent events with real browser/sandbox validation. Known truthful data gaps remain for Commit/Branch/Aprovador.
- [`p1-production-authority-hardening.md`](./p1-production-authority-hardening.md) — completed with `CONDITIONAL_PASS`. Proved Git writer/reconciler identity separation, Argo/Kubernetes least privilege for `d1-prd`, Argo control-object governance under Git with live drift-remediation, and squad/pipeline bypass resistance — including finding and remediating one confirmed live bypass (`ado-agent` bound to `cluster-admin`). Named residual gaps were subsequently closed by `p1-residual-closure-live-cutover.md`.
- [`e1-commit-and-adr012-rereview.md`](./e1-commit-and-adr012-rereview.md) — completed. Converted E1 into the reproducible implementation baseline `platform-devops-developer-portal@c2feb8a`, re-ran the E1 regressions, and independently re-reviewed ADR-012. Result: ADR-012 `ACCEPT`; production rollout remained separate. The bounded eligibility-window TOCTOU follow-up was subsequently closed at `ee114cf`.
- [`e1-multi-activity-concurrency.md`](./e1-multi-activity-concurrency.md) — completed with `PASS`. Proved activity-scoped Delivery binding (`changeId + activityId + release + target`), proved one successful deployment does not complete a multi-activity Change, and added deterministic same-target dispatch exclusion.
- [`adr-012-adoption-gate.md`](./adr-012-adoption-gate.md) — completed with `REMAIN_PROPOSED`; production rollout remained `NO-GO`. Independent review identified E1 as the smallest next architecture-evidence checkpoint.
- [`mvp-demo-hardening.md`](./mvp-demo-hardening.md) — completed with `CONDITIONAL PASS`. Stabilized the proven MVP for demonstration, closed the Kargo no-op PR failure in the exercised template, added minimum Delivery regression coverage, fixed stale deployment projection behavior, and made the demo checkout/build reproducible.
- [`mvp-vertical-delivery-slice.md`](./mvp-vertical-delivery-slice.md) — completed with `CONDITIONAL PASS`. Proved the end-to-end ReleaseCandidate → DEV → HML → PRD → GMUD/ExecutionEligibility → Kargo → Git → Argo → Kubernetes → Backstage flow in the sandbox.
- [`d1-kargo-fit-evaluation.md`](./d1-kargo-fit-evaluation.md) — completed. D1 concluded `KARGO_FIT` with qualified adoption scope. Retained as historical execution-contract/evidence context.

## Launcher pattern

Use a short launcher instead of pasting the long prompt into the agent session. Current launcher:

```text
Fetch the latest `main` from `diegofernandes-dev/backstage-docs`.

Read `prompts/pre-rollout-condition-closure.md` and treat it as the strict execution-and-evidence contract.

Reconcile the latest canonical docs with the current implementation, ADO/GitOps, Entra, Argo/Kargo/Kubernetes, and intended first-production-rollout environment before changing anything.

Close only the five conditions recorded by `docs/delivery/production-adoption-review.md`: durable token-refresh manifests; production-appropriate non-human Delivery runtime credential; reviewed rollback/emergency runbook; proven visible token-refresh failure alert; and real production Git reader/writer SP + branch-policy equivalence when applicable.

Do not invent a production target, repo, cluster, tenant, or runtime merely to make the checkpoint pass. If the actual first-rollout target is not identified, leave the dependent conditions BLOCKED.

Do not reopen ADR-012, P1, Deployments UX, global Delivery workbench, Kargo Stage governance, HA/DR, generalized monitoring, secrets-platform work, audit UX, or other post-rollout hardening.

Never expose secrets. Obey existing branch policies and human-review requirements; do not self-approve or bypass them.

After changes, rerun the high-value Delivery/eligibility/concurrency regressions and the final positive/negative authority checks against the committed/live state.

Write factual evidence to `docs/delivery/pre-rollout-condition-closure.md`, update `docs/delivery/README.md`, and STOP. Do not execute the production rollout.
```
