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

- [`f3-1-2-plan-revision.md`](./f3-1-2-plan-revision.md) — **current planning/docs-only checkpoint** after the F3.1.2 plan architecture review returned `REJECT`. Revise `docs/backstage/f3-1-2-implementation-plan.md` so it embodies the review decisions exactly: keep repository explicit mode-mismatch `CONFLICT` and make stored-mode-wins a service-orchestration concern; mandate caller-owned `trx` for ledger/audit/index/idempotency/provider writes; lock `requirementId = requirementRole` with publication-time uniqueness; and make rollback to pre-F3.1.2 a mandatory runbook correctness gate while pending `LEDGER_REQUIRED` reservations exist.
- The revision must also correct the crash matrix, visibility authority, test matrix, expected source paths, stale rejected alternatives, and the construction-progress header.
- **F3.1.2a / F3.1.2b implementation remain NO-GO.** Implementation-prompt authoring remains NO-GO until the revised plan receives a fresh independent `ACCEPT`.
- If the revision reaches `READY_FOR_REREVIEW`, the next activity is a fresh independent architecture re-review of the revised plan.

## Production-rollout gate — deferred until a real target exists

- [`final-rollout-readiness-rereview.md`](./final-rollout-readiness-rereview.md) — prepared review-only final production gate. The most recent execution returned `NOT_READY` because no real first-rollout Delivery runtime / production cluster / namespace / GitOps repository was designated and the runbook review/escalation evidence was incomplete. This status **does not block continued Backstage/GMUD platform construction**. Re-run only after a real production target exists and the prerequisite evidence is intentionally closed. Do not manufacture production infrastructure merely to make this gate green.

## Completed / historical prompts

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

### Current launcher — F3.1.2 plan revision

```text
Fetch the latest `main` from `diegofernandes-dev/backstage-docs`.

Read `prompts/f3-1-2-plan-revision.md` and treat it as a strict planning/documentation-only revision contract.

Read the rejected review `docs/backstage/f3-1-2-plan-architecture-review.md` and revise `docs/backstage/f3-1-2-implementation-plan.md` so every failed gate is concretely corrected.

Mandatory decisions:
1. Keep KnexIdempotencyRepository explicit authorization-mode mismatch as CONFLICT. Make stored-mode-wins a service orchestration rule for existing reservations; newSubmissionAuthorizationMode applies only to a genuinely new reservation, including race-safe first-insert behavior.
2. Make one caller-owned Knex transaction mandatory for DevelopmentProvider + createRound + appendAuditEvent + index.finalize + idempotency.complete. External providers stay outside the platform transaction and converge through idempotent create/orphan retry.
3. Lock requirementId = requirementRole. Enforce per-rule requirementRole uniqueness at policy registration/publication; no hash fallback.
4. Make rollback to a pre-F3.1.2 binary a mandatory RUNBOOK CORRECTNESS GATE while any pending LEDGER_REQUIRED reservation exists; delete optional-hard-guard language.

Correct the crash matrix so Round/finalize/idempotency completion cannot appear as separately committed states when they are in one platform transaction. Correct visibility authority, tests, expected source paths, rejected alternatives, challenge answers, acceptance criteria and appendices. Remove every stale statement from the rejected design.

Do not modify ADO code, fix buildChange(), change repository semantics, add runtime transaction wiring/config/migrations/routes, wire POST /changes, create AuthorizationRounds, author an implementation prompt, or start F3.1.3/F3.1.4.

Update current-state, implementation-progress (including the stale top header), and prompts README. Commit documentation only.

Return READY_FOR_REREVIEW or BLOCKED. Keep all F3.1.2 implementation and implementation-prompt authoring NO-GO. STOP.
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
