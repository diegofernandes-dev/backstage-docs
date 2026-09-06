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

- [`deployments-ux-v2-implementation.md`](./deployments-ux-v2-implementation.md) — implement the approved Component → Deployments UX v2 faithfully against the normative visual/product reference in [`../docs/delivery/deployments-ux-v2.md`](../docs/delivery/deployments-ux-v2.md) and [`../docs/delivery/assets/deployments-screen-v2.webp`](../docs/delivery/assets/deployments-screen-v2.webp). The agent is explicitly constrained from inventing a different dashboard: it must preserve the approved release panel, DEV/HML/PRD environment area, GMUD/eligibility side panel, promotion history, and recent-events layout. Browser navigation and final screenshot comparison are mandatory. Only the smallest provider-neutral Delivery read/query changes needed to make the screen truthful are authorized. Production rollout remains `NO-GO`; P1 residual hardening and the production-adoption review are out of scope for this UX checkpoint.

## Completed / historical prompts

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
Read `prompts/deployments-ux-v2-implementation.md` and treat it as the execution contract.
Open and inspect `docs/delivery/deployments-ux-v2.md` and `docs/delivery/assets/deployments-screen-v2.webp` before changing code; the approved visual hierarchy is normative and you are not authorized to invent a different Deployments dashboard.
Verify the current Backstage implementation baseline, then implement only the approved Deployments UX v2 plus the smallest provider-neutral Delivery read/query changes required to make it truthful.
Use the running Backstage browser extensively, validate the required states, compare the final screen against the approved reference, capture screenshots, update factual canonical evidence, and STOP at the prompt gate.
Do not redesign ADR-012 boundaries, do not continue P1 production hardening, do not run a production-adoption review, and do not expand into the global Delivery workbench or unrelated Backstage tabs.
```
