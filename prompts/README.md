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

- [`deployments-ux-v2-responsive-polish.md`](./deployments-ux-v2-responsive-polish.md) — deterministic responsive/layout polish for the already-implemented Component → Deployments UX v2. This is explicitly **not** a design task: section order, labels, field ordering, component structure, and breakpoint behavior are fixed. At `xl` and above the approved richer desktop composition is used (DEV/HML/PRD in three columns, GMUD beside them, History + Events side by side). Below `xl` — including the user's 13-inch MacBook class of viewport — DEV/HML/PRD must stack one per row, GMUD must move below PRD, and History/Events must stack full-width. No new visual language, icon/font library, charts, backend/domain concepts, or creative reinterpretation is authorized. Browser validation and screenshots at both wide and laptop viewports are mandatory.

## Completed / historical prompts

- [`deployments-ux-v2-implementation.md`](./deployments-ux-v2-implementation.md) — completed with `CONDITIONAL_PASS`; implementation recorded at `platform-devops-developer-portal@0163a49`. Implemented the approved release selector, environment views, GMUD/eligibility context, promotion history, and recent events with real browser/sandbox validation. Known truthful data gaps remain for Commit/Branch/Aprovador. The original committed visual asset was found truncated/corrupted; the textual contract and user-supplied working-session reference were used instead.
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
Read `prompts/deployments-ux-v2-responsive-polish.md` and treat it as a strict execution contract, not a design brief.
Verify the latest canonical docs and the current `platform-devops-developer-portal` implementation baseline before editing.
Implement exactly the specified responsive composition: `xl` and above uses the rich desktop layout; below `xl`, DEV/HML/PRD stack one per row, GMUD moves below PRD, and History/Events stack full-width.
Preserve the exact section order, labels, field order, existing Backstage/MUI visual language, actions, data truthfulness, and business behavior. Do not creatively reinterpret the page or add new icons/fonts/design libraries/domain concepts.
Use the running Backstage browser, validate both wide and MacBook/laptop viewport classes, capture screenshots, update factual canonical evidence, and STOP at the prompt gate.
```
