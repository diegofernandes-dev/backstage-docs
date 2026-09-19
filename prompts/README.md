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

## Current authorized activity

- F3.1.2 plan is an **ACCEPTED IMPLEMENTATION CONTRACT**.
- F3.1.2a is **CLOSED / ACCEPTED IMPLEMENTED BASELINE** at ADO `ccee1e1676a2763e68880e5383ce1e5e48742843`.
- F3.1.1c implementation is published at ADO `3b302ab5c9caab38f96491b389b7ea9fe0b66c2f` and remains pending independent acceptance.
- [`f3-1-1c-architecture-implementation-acceptance.md`](./f3-1-1c-architecture-implementation-acceptance.md) — **current review-only checkpoint**. Independently inspect the exact ADO diff and return ACCEPT or REJECT. Do not repair code in the review.
- F3.1.2b remains **NO-GO** until F3.1.1c is accepted. If F3.1.1c is ACCEPTED, only F3.1.2b implementation-prompt authoring becomes GO; implementation still requires separate explicit launch.
- F3.2 CAB autonomy remains **NO-GO**.
## Production-rollout gate — deferred until a real target exists

- [`final-rollout-readiness-rereview.md`](./final-rollout-readiness-rereview.md) — prepared review-only final production gate. The most recent execution returned `NOT_READY` because no real first-rollout Delivery runtime / production cluster / namespace / GitOps repository was designated and the runbook review/escalation evidence was incomplete. This status **does not block continued Backstage/GMUD platform construction**. Re-run only after a real production target exists and the prerequisite evidence is intentionally closed. Do not manufacture production infrastructure merely to make this gate green.

## Completed / historical prompts

- [`f3-1-1c-cab-safe-policy-implementation.md`](./f3-1-1c-cab-safe-policy-implementation.md) — completed with implementation `PASS`, published to ADO `platform-devops-developer-portal@3b302ab5c9caab38f96491b389b7ea9fe0b66c2f`. Canonical evidence: `docs/backstage/f3-1-1c-implementation-evidence.md`. Acceptance pending separate review.
- [`f3-1-2a-architecture-implementation-acceptance.md`](./f3-1-2a-architecture-implementation-acceptance.md) — completed with `ACCEPT`. Closed F3.1.2a as the accepted implemented baseline at ADO `ccee1e1`. F3.1.1c was subsequently implemented; F3.1.2b remains NO-GO until F3.1.1c acceptance.
- F3.1.2a implementation evidence checkpoint — implementation PASS and published to ADO `ccee1e1`; independent acceptance completed separately (`ACCEPT`).
- [`f3-1-2a-canonical-change-implementation.md`](./f3-1-2a-canonical-change-implementation.md) — completed with implementation `PASS`, published to ADO `platform-devops-developer-portal@ccee1e1676a2763e68880e5383ce1e5e48742843`. Canonical evidence: `docs/backstage/f3-1-2a-implementation-evidence.md`. Acceptance completed separately (`ACCEPT`).
- Prompt-authoring checkpoint — F3.1.2a canonical-Change implementation prompt and F3.1.1c CAB-safe policy implementation prompt authored after final F3.1.2 plan ACCEPT. F3.1.2a later executed and accepted; F3.1.1c still awaiting explicit launch.
- [`f3-1-2-final-architecture-rereview.md`](./f3-1-2-final-architecture-rereview.md) — completed with `ACCEPT`. Canonical review: [`docs/backstage/f3-1-2-final-architecture-rereview.md`](../docs/backstage/f3-1-2-final-architecture-rereview.md). ADO tip independently verified `188d8e9`. Prior blockers 4/4 closed; concurrency + ADR-013 + F3.1.1c/2a/2b gates PASS. Plan is ACCEPTED IMPLEMENTATION CONTRACT. No ADO code modified; no implementation prompt authored inside the review.
- ADR-013 CAB-safe plan realignment — documentation completed in canonical ADR/plan. Normal-low is now primary + CAB by default; F3.1.1c is required before F3.1.2b; bounded autonomy is deferred to F3.2.
- [`f3-1-2-concurrency-plan-revision.md`](./f3-1-2-concurrency-plan-revision.md) — completed by documentation revision. Concurrency-corrected plan is `READY_FOR_REREVIEW`. Healthy Round-1 loser convergence is deterministic; C1–C4 concurrency proofs specified. ADO implementation unchanged; all F3.1.2 implementation remains NO-GO pending fresh independent ACCEPT.
- [`f3-1-2-plan-revision.md`](./f3-1-2-plan-revision.md) — completed by documentation revision. Revised plan was `READY_FOR_REREVIEW` at docs `6284195`; subsequent focused re-review REJECTED on concurrency only (18/20). ADO implementation unchanged.
- [`f3-1-2-plan-architecture-review.md`](./f3-1-2-plan-architecture-review.md) — completed with `REJECT`. Canonical review: [`docs/backstage/f3-1-2-plan-architecture-review.md`](../docs/backstage/f3-1-2-plan-architecture-review.md). ADO baseline verified `188d8e9`. Gates 12/20 PASS. Critical decisions resolved by review (not yet embodied in plan): keep repository mode-mismatch CONFLICT + service orchestration; mandate caller-owned `trx` (API already exists); Option A `requirementId = requirementRole` + uniqueness validation. Next: plan revision only.
- [`f3-1-2-planning.md`](./f3-1-2-planning.md) — completed with `READY_FOR_REVIEW`. Produced [`docs/backstage/f3-1-2-implementation-plan.md`](../docs/backstage/f3-1-2-implementation-plan.md) against ADO `188d8e9` (20/20 gates, 15/15 challenges). Subsequent architecture review rejected the plan as implementation contract; see review document.
- [`f3-1-1b-architecture-implementation-acceptance.md`](./f3-1-1b-architecture-implementation-acceptance.md) — completed with `ACCEPT`. Closed F3.1.1b as the accepted implemented baseline at ADO `188d8e9`. Authorized F3.1.2 planning only; F3.1.2 implementation remains NO-GO.
- [`f3-1-1b-selector-resolution-implementation.md`](./f3-1-1b-selector-resolution-implementation.md) — completed with implementation `PASS`, published to ADO `platform-devops-developer-portal@188d8e9cc43423f3644b3cacfb9849257838a583`. Canonical evidence is `docs/backstage/f3-1-1b-implementation-evidence.md`. Architecture acceptance completed separately (`ACCEPT`).
- [`pre-rollout-condition-closure.md`](./pre-rollout-condition-closure.md) — completed with `CONDITIONAL_PASS`. C1 token-refresh durability subsequently closed through governed PR #82; C4 alerting was proven. C2/C5 remain target-dependent and are deferred until a real production rollout target exists. C3 is an operational/runbook readiness item and is not a blocker to unrelated platform implementation slices.
- [`production-adoption-review.md`](./production-adoption-review.md) — completed with `CONDITIONAL_GO` for a narrow first production rollout only: one non-critical/medium-criticality component, one production namespace, platform-owner-attended, two-week observation before any second workload. ADR-012 remains Accepted; no rollout was executed by the review.
- [`p1-residual-closure-live-cutover.md`](./p1-residual-closure-live-cutover.md) — completed with `PASS`. Live Argo reader and Kargo/Delivery writer were moved from human PATs to distinct non-human Entra service principals; automatic token refresh was implemented; governed P1 GitOps control changes were approved/merged; negative authority evidence and regressions were rechecked.
- [`deployments-ux-v2-responsive-polish.md`](./deployments-ux-v2-responsive-polish.md) — completed with `PASS`; implementation recorded at `platform-devops-developer-portal@b08e7b2`. Further visual refinement is intentionally deferred.
- [`deployments-ux-v2-implementation.md`](./deployments-ux-v2-implementation.md) — completed with `CONDITIONAL_PASS`; implementation recorded at `platform-devops-developer-portal@0163a49`. Implemented the release selector, environment views, GMUD/eligibility context, promotion history, and recent events.
- [`p1-production-authority-hardening.md`](./p1-production-authority-hardening.md) — completed with `CONDITIONAL_PASS`. Proved Git writer/reconciler identity separation, Argo/Kubernetes least privilege for `d1-prd`, Argo control-object governance under Git with live drift-remediation, and squad/pipeline bypass resistance.
- [`e1-commit-and-adr012-rereview.md`](./e1-commit-and-adr012-rereview.md) — completed. Converted E1 into reproducible baseline `platform-devops-developer-portal@c2feb8a`, re-ran regressions, and independently re-reviewed ADR-012. Result: ADR-012 `ACCEPT`.
- [`e1-multi-activity-concurrency.md`](./e1-multi-activity-concurrency.md) — completed with `PASS`. Proved activity-scoped Delivery binding, multi-activity non-completion semantics, and same-target concurrency exclusion.
- [`adr-012-adoption-gate.md`](./adr-012-adoption-gate.md) — completed with `REMAIN_PROPOSED`; independent review identified E1 as the smallest next architecture-evidence checkpoint.
- [`mvp-demo-hardening.md`](./mvp-demo-hardening.md) — completed with `CONDITIONAL PASS`. Stabilized the proven MVP for demonstration, closed the Kargo no-op-PR failure, added minimum Delivery regression coverage, and fixed stale deployment projection behavior.
- [`mvp-vertical-delivery-slice.md`](./mvp-vertical-delivery-slice.md) — completed with `CONDITIONAL PASS`. Proved the end-to-end ReleaseCandidate → DEV → HML → PRD → GMUD/ExecutionEligibility → Kargo → Git → Argo → Kubernetes → Backstage flow in the sandbox.
- [`d1-kargo-fit-evaluation.md`](./d1-kargo-fit-evaluation.md) — completed. D1 concluded `KARGO_FIT` with qualified adoption scope.

## Launcher pattern

Use a short launcher instead of pasting the long prompt into an agent session.

### Historical launcher — final F3.1.2 re-review after ADR-013 (completed — ACCEPT)

```text
Fetch the latest main from diegofernandes-dev/backstage-docs.

Read prompts/f3-1-2-final-architecture-rereview.md and treat it as a strict independent review-only contract.

Verify the actual current ADO platform-devops-developer-portal/feat/ado-repo-governance source, then re-review docs/backstage/f3-1-2-implementation-plan.md against ADR-009 as partially superseded by ADR-013.

Regress the four original blockers and the corrected Round-1 concurrency contract. Verify F3.1.1c is a narrow new immutable policy publication prerequisite, F3.1.2a remains canonical-Change-only, and F3.1.2b contains no CAB autonomy/bypass while normal-low materializes primary + CAB.

Return exactly ACCEPT or REJECT. Do not modify ADO code and do not author implementation prompts from inside the review.

Write docs/backstage/f3-1-2-final-architecture-rereview.md, update canonical state/progress/prompts, commit documentation only, and STOP.
```

### Historical launcher — concurrency-corrected plan re-review (superseded by ADR-013 alignment)

```text
Fetch the latest main from diegofernandes-dev/backstage-docs.
Independently re-review the concurrency-corrected F3.1.2 plan in docs/backstage/f3-1-2-implementation-plan.md.
```
### Historical launcher — F3.1.2 narrow concurrency plan revision (completed)

```text
Fetch the latest main from diegofernandes-dev/backstage-docs.

Read prompts/f3-1-2-concurrency-plan-revision.md and treat it as a strict planning/documentation-only contract.

Revise docs/backstage/f3-1-2-implementation-plan.md only for the remaining concurrency ambiguity from docs/backstage/f3-1-2-revised-plan-architecture-rereview.md.

Do not reopen the four already-closed architecture decisions. Do not modify ADO code and do not create an implementation prompt.

Update canonical docs, return READY_FOR_REREVIEW or BLOCKED, keep all F3.1.2 implementation NO-GO, commit documentation only, and STOP.
```

### Historical launcher — F3.1.2 plan architecture review (completed — REJECT)

```text
Fetch the latest `main` from `diegofernandes-dev/backstage-docs`.
Read `prompts/f3-1-2-plan-architecture-review.md` and produce the review document only.
```

### Historical launcher — F3.1.2 planning (completed)

```text
Fetch the latest `main` from `diegofernandes-dev/backstage-docs`.
Read `prompts/f3-1-2-planning.md` and produce `docs/backstage/f3-1-2-implementation-plan.md` only.
```

### Gated launcher — final rollout-readiness re-review

Use only after a real production target exists and the canonical prerequisite evidence is deliberately closed.

```text
Fetch the latest `main` from `diegofernandes-dev/backstage-docs`.
Read `prompts/final-rollout-readiness-rereview.md` and treat it as a strict independent, review-only final gate.
First execute its prerequisite gate. If any prerequisite is missing, return NOT_READY and STOP without implementing or repairing anything.
If every prerequisite passes, inspect the actual target read-only, evaluate the mandatory final gates, and return exactly GO or NO-GO.
Do not trigger the real production rollout from the review.
```
