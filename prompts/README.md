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

- **F3.1.2 plan revision — concurrent Round 1 convergence only.** The fresh revised-plan re-review returned `REJECT` with 18/20 gates PASS; the four original architecture blockers are closed. See [`docs/backstage/f3-1-2-revised-plan-architecture-rereview.md`](../docs/backstage/f3-1-2-revised-plan-architecture-rereview.md).
- Revise [`docs/backstage/f3-1-2-implementation-plan.md`](../docs/backstage/f3-1-2-implementation-plan.md) so a healthy concurrent same-key/same-payload Round-1 loser has one deterministic contract: roll back its losing transaction, re-read the winner's completed reservation + finalized matching index + Round 1, and return the same logical success; only true committed inconsistency fails closed.
- Add the required end-to-end concurrency proof to the plan: two concurrent identical create calls converge to the same `changeId`/result with exactly one finalized index, one Round 1, one requirement set, one canonical submission-audit set, one completed reservation, and no Round 2. Add a different-payload concurrent negative case.
- **F3.1.2a / F3.1.2b implementation and implementation-prompt authoring remain NO-GO.** After the narrow revision, perform a focused fresh re-review.
## Production-rollout gate — deferred until a real target exists

- [`final-rollout-readiness-rereview.md`](./final-rollout-readiness-rereview.md) — prepared review-only final production gate. The most recent execution returned `NOT_READY` because no real first-rollout Delivery runtime / production cluster / namespace / GitOps repository was designated and the runbook review/escalation evidence was incomplete. This status **does not block continued Backstage/GMUD platform construction**. Re-run only after a real production target exists and the prerequisite evidence is intentionally closed. Do not manufacture production infrastructure merely to make this gate green.

## Completed / historical prompts

- [`f3-1-2-plan-revision.md`](./f3-1-2-plan-revision.md) — completed by documentation revision. Revised plan is `READY_FOR_REREVIEW` at docs `6284195`; ADO implementation unchanged; all implementation remains NO-GO pending fresh independent ACCEPT.
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

### Current launcher — F3.1.2 narrow concurrency plan revision

```text
Fetch the latest `main` from `diegofernandes-dev/backstage-docs`.

Read `docs/backstage/f3-1-2-revised-plan-architecture-rereview.md` and revise only the remaining concurrency ambiguity in `docs/backstage/f3-1-2-implementation-plan.md`.

For two concurrent same actor + Idempotency-Key + payload submissions that both reach Round 1 finalization, define one deterministic loser contract: the expected Round-1 uniqueness loser rolls back its transaction, re-reads the winner's durable completed reservation/finalized matching index/Round 1, and returns the same logical success when coherent. Transient DB lock/serialization remains retryable; committed fact mismatch is an invariant error. Never create Round 2.

Add an end-to-end concurrency test requirement proving both identical callers converge to the same changeId/result and exactly one finalized index, Round 1, requirement set, canonical submission-audit set, and completed reservation. Add the different-payload concurrent conflict case.

Do not reopen the four closed review decisions, do not modify ADO code, and do not create an implementation prompt.

Update canonical docs, mark the revised plan READY_FOR_REREVIEW, commit documentation only, and STOP.
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
If every prerequisite passes, inspect the actual target read-only, evaluate the mandatory final gates, and return exactly GO or NO_GO.
Do not trigger the real production rollout from the review.
```
