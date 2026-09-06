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

- [`e1-commit-and-adr012-rereview.md`](./e1-commit-and-adr012-rereview.md) — first converts the already-passed E1 working-tree delta into a committed, reproducible implementation baseline and re-runs the relevant regression checks; then performs an independent ADR-012 adoption re-review against all 12 gates. It explicitly challenges the remaining window TOCTOU, production-security, and GitOps-layout questions without implementing production hardening or a new eligibility framework. The running Backstage UI/browser should be used where it materially strengthens the evidence, but UI is supplemental to backend/domain/provider proof.

## Completed / historical prompts

- [`e1-multi-activity-concurrency.md`](./e1-multi-activity-concurrency.md) — completed with `PASS`. Proved activity-scoped Delivery binding (`changeId + activityId + release + target`), proved one successful deployment does not complete a multi-activity Change, and added deterministic same-target dispatch exclusion. The E1 evidence recorded the implementation delta as uncommitted; the current prompt closes that reproducibility gap before re-reviewing ADR-012.
- [`adr-012-adoption-gate.md`](./adr-012-adoption-gate.md) — completed with `REMAIN_PROPOSED`; production rollout remained `NO-GO`. Independent review scored the 12 ADR-012 gates 3 PROVEN / 7 PARTIALLY_PROVEN / 2 NOT_PROVEN and identified E1 as the smallest next architecture-evidence checkpoint.
- [`mvp-demo-hardening.md`](./mvp-demo-hardening.md) — completed with `CONDITIONAL PASS`. Stabilized the proven MVP for demonstration, closed the Kargo no-op PR failure in the exercised template, added minimum Delivery regression coverage, fixed stale deployment projection behavior, and made the demo checkout/build reproducible.
- [`mvp-vertical-delivery-slice.md`](./mvp-vertical-delivery-slice.md) — completed with `CONDITIONAL PASS`. Proved the end-to-end ReleaseCandidate → DEV → HML → PRD → GMUD/ExecutionEligibility → Kargo → Git → Argo → Kubernetes → Backstage flow in the sandbox.
- [`d1-kargo-fit-evaluation.md`](./d1-kargo-fit-evaluation.md) — completed. D1 concluded `KARGO_FIT` with qualified adoption scope. Retained as the historical execution contract/evidence context; do not re-run it unless a later architecture decision explicitly requires a new Kargo evaluation.

## Launcher pattern

Use a short launcher instead of pasting the long prompt into the agent session. Current launcher:

```text
Fetch the latest `main` from `diegofernandes-dev/backstage-docs`.
Read `prompts/e1-commit-and-adr012-rereview.md` and treat it as the execution/review contract.
First make the E1 implementation reproducible by isolating and committing only the verified E1 delta, then re-run the required regression checks from the committed SHA.
Only after that baseline is trusted, independently re-score all 12 ADR-012 gates, use the running Backstage UI/browser where it materially strengthens evidence, update canonical docs with factual conclusions, and STOP at the prompt gate.
Do not optimize for accepting ADR-012, do not implement window/TOCTOU or production hardening, and do not start the next Delivery milestone.
```
