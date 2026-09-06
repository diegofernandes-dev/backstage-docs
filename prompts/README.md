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

- [`p1-production-authority-hardening.md`](./p1-production-authority-hardening.md) — first production-adoption hardening checkpoint after ADR-012 acceptance. It is deliberately limited to authority/security blockers: real Git writer vs reconciler identity separation, scoped Argo/Kubernetes reconciliation authority with negative RBAC proof, governed Argo Application/AppProject mutation authority, and active ordinary-squad/pipeline bypass attempts. It may use the running Backstage UI only as supplemental regression evidence. It must not declare production rollout GO or expand into HA/DR, break-glass, rollback automation, supply-chain expansion, or a new Delivery milestone.

## Completed / historical prompts

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
Read `prompts/p1-production-authority-hardening.md` and treat it as the execution contract.
Verify the current canonical docs, accepted ADR-012 state, implementation baseline, GitOps/Kargo/Argo topology, and current identity/authority graph before changing anything.
Execute only the P1 authority-hardening scope, prove both the legitimate governed path and the required negative bypass cases, use the running Backstage UI/browser only where it strengthens regression evidence, update canonical docs factually, and STOP at the prompt gate.
Do not redesign the accepted architecture, do not declare production rollout GO, and do not expand into HA/DR, break-glass, rollback automation, supply-chain expansion, or the next Delivery milestone.
```
