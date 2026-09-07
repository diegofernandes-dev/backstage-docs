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

- [`p1-residual-closure-live-cutover.md`](./p1-residual-closure-live-cutover.md) — close only the bounded P1 residuals before a separate production-adoption review: move the live Argo Git reader and Kargo/Delivery Git writer onto the already-proven distinct non-human Entra service principals, prove sustainable non-human token refresh, establish the P1 GitOps control changes as durable protected-branch steady state, then rerun the high-value positive/negative authority proofs and regressions against the final live state. The current Deployments UX is explicitly frozen as functionally accepted/demoable; further visual polish is deferred and no UI redesign is authorized. This checkpoint may return readiness for a later production-adoption review, but production rollout remains `NO-GO` and the adoption review itself is out of scope.

## Completed / historical prompts

- [`deployments-ux-v2-responsive-polish.md`](./deployments-ux-v2-responsive-polish.md) — completed with `PASS`; implementation recorded at `platform-devops-developer-portal@b08e7b2`. Responsive composition and laptop behavior were validated and documented. Further visual refinement is intentionally deferred; do not reopen this workstream during authority closure.
- [`deployments-ux-v2-implementation.md`](./deployments-ux-v2-implementation.md) — completed with `CONDITIONAL_PASS`; implementation recorded at `platform-devops-developer-portal@0163a49`. Implemented the approved release selector, environment views, GMUD/eligibility context, promotion history, and recent events with real browser/sandbox validation. Known truthful data gaps remain for Commit/Branch/Aprovador.
- [`p1-production-authority-hardening.md`](./p1-production-authority-hardening.md) — completed with `CONDITIONAL_PASS`. Proved Git writer/reconciler identity separation (two real Entra service principals with ACL-enforced positive/negative Git proofs), Argo/Kubernetes least privilege for `d1-prd` (real RBAC denial, not AppProject-level), Argo control-object governance under Git with live drift-remediation, and squad/pipeline bypass resistance — including finding and remediating one confirmed live bypass (`ado-agent` bound to `cluster-admin`). Named residual gaps: live Argo/Kargo credential swap + token-refresh automation, regression suite re-execution, two GitOps PRs awaiting merge approval. Production rollout remains `NO-GO`.
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
Read `prompts/p1-residual-closure-live-cutover.md` and treat it as the execution contract.
Verify the current canonical docs, implementation SHA, ADO/GitOps state, live Argo/Kargo credentials, scoped Kubernetes identities, and P1 residual inventory before changing anything.
Freeze the current Deployments UX exactly as-is; no further visual polish or UI work is authorized.
Close only the bounded P1 residuals: live non-human Git credential cutover with sustainable token refresh, durable governed GitOps steady state, final positive/negative authority proof on the live state, and regression re-execution from a committed baseline.
Do not bypass branch policies or human approval requirements, do not expose secrets, do not declare production rollout GO, and do not run the production-adoption review.
Update canonical evidence factually and STOP at the prompt gate.
```