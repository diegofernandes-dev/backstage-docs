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

- [`mvp-vertical-delivery-slice.md`](./mvp-vertical-delivery-slice.md) — implement the smallest sandbox/demo vertical slice connecting ReleaseCandidate → DEV → HML → PRD request → Change/GMUD → ExecutionEligibility ALLOW → Kargo → Git → Argo → Kubernetes → Backstage status. This is the current path toward a demonstrable deliverable. It explicitly defers non-blocking production hardening and stops after MVP evidence/documentation.

## Completed / historical prompts

- [`d1-kargo-fit-evaluation.md`](./d1-kargo-fit-evaluation.md) — completed. D1 concluded `KARGO_FIT` with qualified adoption scope. Retained as the historical execution contract/evidence context; do not re-run it unless a later architecture decision explicitly requires a new Kargo evaluation.

## Launcher pattern

Use a short launcher instead of pasting the long prompt into the agent session. Example:

```text
Fetch the latest `main` from `diegofernandes-dev/backstage-docs`.
Read `prompts/mvp-vertical-delivery-slice.md` and treat it as the execution contract.
Verify the current canonical docs/gates first, execute only the authorized sandbox MVP scope, collect factual evidence, update the required canonical docs, and STOP at the prompt gate.
Do not expand into production hardening or the next checkpoint.
```
