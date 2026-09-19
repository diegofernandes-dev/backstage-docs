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

- [`f3-1-1b-architecture-implementation-acceptance.md`](./f3-1-1b-architecture-implementation-acceptance.md) — independent review-only acceptance gate for the already implemented/published F3.1.1b candidate at ADO `188d8e9cc43423f3644b3cacfb9849257838a583`. Verify lineage/scope, ADR-009 selector/principal semantics, Catalog service-credential resolution, fail-closed behavior, active-pair startup validation, append-only selector-bundle publication, separation-of-duty layer boundaries, and complete isolation from `POST /changes` / AuthorizationRound creation. Return only `ACCEPT` or `REJECT`; do not fix code. `ACCEPT` closes F3.1.1b and authorizes F3.1.2 planning only; F3.1.2 implementation remains NO-GO pending a separate reviewed plan and explicit authorization.

## Production-rollout gate — deferred until a real target exists

- [`final-rollout-readiness-rereview.md`](./final-rollout-readiness-rereview.md) — prepared review-only final production gate. The most recent execution returned `NOT_READY` because no real first-rollout Delivery runtime / production cluster / namespace / GitOps repository was designated and the runbook review/escalation evidence was incomplete. This status **does not block continued Backstage/GMUD platform construction**. Re-run only after a real production target exists and the prerequisite evidence is intentionally closed. Do not manufacture production infrastructure merely to make this gate green.

## Completed / historical prompts

- [`f3-1-1b-selector-resolution-implementation.md`](./f3-1-1b-selector-resolution-implementation.md) — completed with implementation `PASS`, published to ADO `platform-devops-developer-portal@188d8e9cc43423f3644b3cacfb9849257838a583`. Canonical evidence is `docs/backstage/f3-1-1b-implementation-evidence.md`. Architecture implementation acceptance is intentionally separate and is the current checkpoint.
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

Use a short launcher instead of pasting the long prompt into the agent session.

### Current launcher — F3.1.1b architecture implementation acceptance

```text
Fetch the latest `main` from `diegofernandes-dev/backstage-docs`.

Read `prompts/f3-1-1b-architecture-implementation-acceptance.md` and treat it as a strict independent, review-only architecture acceptance contract.

Review the exact ADO implementation candidate `platform-devops-developer-portal@188d8e9cc43423f3644b3cacfb9849257838a583` on `feat/ado-repo-governance`, with expected parent `d3c0751a15b908cec8f5595c97e52f41226344ed`. Independently inspect source and rerun high-value evidence when access permits; if ADO source is inaccessible, state that limitation explicitly and rely only on canonical evidence without inventing independent verification.

Evaluate every mandatory G1-G17 gate in the prompt: ADR-009 authority boundaries; generic selector semantics; User/Group typing; no authority-member expansion; backend service credentials; fail-closed Catalog resolution; active-pair-only startup validation; selector-bundle digest/immutability; normal append-only publication with no genesis reuse; separation-of-duty layer boundaries; complete submission-path isolation; cross-cutover idempotency preservation; semantic architecture guards; environment/production separation; resolver provenance; no hidden F3.1.2+ behavior in plugin wiring; and test/evidence credibility.

Do not modify implementation, manifest/config, routes, migrations, permissions, frontend, Delivery, or production infrastructure. Do not wire `POST /changes`, create AuthorizationRounds, or begin F3.1.2.

Return exactly `ACCEPT` or `REJECT`. No generic conditional acceptance.

Write the factual result to `docs/backstage/f3-1-1b-architecture-acceptance.md`, update `docs/backstage/current-state.md`, `docs/backstage/implementation-progress.md`, and `prompts/README.md` only as required by the verdict, commit documentation only, and STOP.

If ACCEPT: F3.1.1b becomes CLOSED / ACCEPTED IMPLEMENTED BASELINE; F3.1.2 planning becomes GO; F3.1.2 implementation remains NO-GO pending a separate reviewed plan and explicit authorization.
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
