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

- [`production-adoption-review.md`](./production-adoption-review.md) — independent, review-only production-adoption gate after P1 residual closure `PASS`. The reviewer must challenge all prior evidence and return exactly `GO`, `CONDITIONAL_GO`, or `NO_GO` for a **controlled first production adoption**, while keeping ADR acceptance separate from rollout approval. The checkpoint is read-only: no source/config/infrastructure changes, no P1 continuation, no rollout execution, and no Deployments UX work. It explicitly evaluates authority separation, credential lifecycle, scoped Kubernetes authority, durable control-plane Git governance, bypass resistance, legitimate promotion, Change/eligibility semantics, concurrency/idempotency, failure recovery, rollback/break-glass readiness, auditability, sandbox→production transferability, and first-rollout blast radius. A positive verdict authorizes only a narrow rollout envelope, not enterprise-wide GA.

## Completed / historical prompts

- [`p1-residual-closure-live-cutover.md`](./p1-residual-closure-live-cutover.md) — completed with `PASS`. Live Argo reader and Kargo/Delivery writer were moved from human PATs to the already-proven distinct non-human Entra service principals; automatic token refresh was implemented and re-verified across multiple unattended cycles; the governed P1 GitOps control diff was approved and merged through protected branch policy; `d1-prd`/`d1-control-plane` converged to the merged scoped-destination state; high-value positive/negative authority evidence and regressions were rechecked. Result: `Ready for separate production-adoption review: YES`; production rollout remained `NO-GO` pending that review.
- [`deployments-ux-v2-responsive-polish.md`](./deployments-ux-v2-responsive-polish.md) — completed with `PASS`; implementation recorded at `platform-devops-developer-portal@b08e7b2`. Responsive composition and laptop behavior were validated and documented. Further visual refinement is intentionally deferred; do not reopen this workstream during authority closure.
- [`deployments-ux-v2-implementation.md`](./deployments-ux-v2-implementation.md) — completed with `CONDITIONAL_PASS`; implementation recorded at `platform-devops-developer-portal@0163a49`. Implemented the approved release selector, environment views, GMUD/eligibility context, promotion history, and recent events with real browser/sandbox validation. Known truthful data gaps remain for Commit/Branch/Aprovador.
- [`p1-production-authority-hardening.md`](./p1-production-authority-hardening.md) — completed with `CONDITIONAL_PASS`. Proved Git writer/reconciler identity separation (two real Entra service principals with ACL-enforced positive/negative Git proofs), Argo/Kubernetes least privilege for `d1-prd` (real RBAC denial, not AppProject-level), Argo control-object governance under Git with live drift-remediation, and squad/pipeline bypass resistance — including finding and remediating one confirmed live bypass (`ado-agent` bound to `cluster-admin`). Named residual gaps were subsequently closed by `p1-residual-closure-live-cutover.md`.
- [`e1-commit-and-adr012-rereview.md`](./e1-commit-and-adr012-rereview.md) — completed. Converted E1 into the reproducible implementation baseline `platform-devops-developer-portal@c2feb8a`, re-ran the E1 regressions, and independently re-reviewed ADR-012. Result: ADR-012 `ACCEPT`; production rollout remained `NO-GO`. The bounded eligibility-window TOCTOU follow-up was subsequently closed at `ee114cf`.
- [`e1-multi-activity-concurrency.md`](./e1-multi-activity-concurrency.md) — completed with `PASS`. Proved activity-scoped Delivery binding (`changeId + activityId + release + target`), proved one successful deployment does not complete a multi-activity Change, and added deterministic same-target dispatch exclusion.
- [`adr-012-adoption-gate.md`](./adr-012-adoption-gate.md) — completed with `REMAIN_PROPOSED`; production rollout remained `NO-GO`. Independent review identified E1 as the smallest next architecture-evidence checkpoint.
- [`mvp-demo-hardening.md`](./mvp-demo-hardening.md) — completed with `CONDITIONAL PASS`. Stabilized the proven MVP for demonstration, closed the Kargo no-op PR failure in the exercised template, added minimum Delivery regression coverage, fixed stale deployment projection behavior, and made the demo checkout/build reproducible.
- [`mvp-vertical-delivery-slice.md`](./mvp-vertical-delivery-slice.md) — completed with `CONDITIONAL PASS`. Proved the end-to-end ReleaseCandidate → DEV → HML → PRD → GMUD/ExecutionEligibility → Kargo → Git → Argo → Kubernetes → Backstage flow in the sandbox.
- [`d1-kargo-fit-evaluation.md`](./d1-kargo-fit-evaluation.md) — completed. D1 concluded `KARGO_FIT` with qualified adoption scope. Retained as the historical execution contract/evidence context; do not re-run it unless a later architecture decision explicitly requires a new Kargo evaluation.

## Launcher pattern

Use a short launcher instead of pasting the long prompt into the agent session. Current launcher:

```text
Fetch the latest `main` from `diegofernandes-dev/backstage-docs`.

Read `prompts/production-adoption-review.md` and treat it as a strict independent REVIEW contract.

Verify the latest canonical docs, committed implementation baseline, governed GitOps revision, and current live Argo/Kargo/Kubernetes/ADO state using read-only inspection only.

Do not implement or fix anything. Do not modify source code, GitOps desired state, Kubernetes resources, credentials, ACLs, branch policies, or Backstage UI.

Challenge the existing ADR-012 Accepted state, P1 authority evidence, and P1 residual-closure PASS without optimizing for approval.

Evaluate every mandatory gate in the prompt, including sandbox-to-production transferability and the bounded first-rollout blast radius.

Return exactly one production-adoption verdict: GO, CONDITIONAL_GO, or NO_GO. Keep ADR acceptance separate from rollout approval.

If GO or CONDITIONAL_GO, authorize only the narrow first-production rollout envelope supported by evidence — not enterprise-wide GA.

Write the factual result to `docs/delivery/production-adoption-review.md`, update `docs/delivery/README.md`, commit the documentation, and STOP. Do not execute the rollout or implement any resulting condition.
```