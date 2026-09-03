# F3.1.1a — Architecture Implementation Acceptance

- **Status:** ACCEPTED IMPLEMENTED BASELINE (architecture acceptance)
- **Date:** 2026-09-03
- **Implementation reference:** Azure DevOps `platform-devops-developer-portal@d3c0751a15b908cec8f5595c97e52f41226344ed`, branch `feat/ado-repo-governance`
- **Documentation review baseline:** `backstage-docs@275a387ffeb18c51181875b388e5a81bfd2a1ee7`
- **Authority:** ADR-009 + F3.1.1-R2 implementation plan and implementation-progress checkpoint

## Review scope and evidence limitation

This acceptance is an **architecture review of the canonical implementation evidence/handoff** already recorded in `backstage-docs`. The current review environment does not expose the Azure DevOps implementation repository directly, so the source tree at `d3c0751` was **not independently re-fetched or re-executed in this checkpoint**. Claims about changed files, test counts, PostgreSQL execution, lint/build results, TypeScript-baseline equivalence and remote publication therefore remain grounded in the previously recorded ADO-verified implementation checkpoint rather than a new independent verification.

No new ADO implementation claim is introduced here.

## Decision

F3.1.1a is **ACCEPTED** as the implemented baseline for the published authorization-policy foundation.

The slice remains deliberately pure and unwired from submission/runtime authorization. It establishes:

- serializable `PublishedAuthorizationPolicy` data;
- explicit `policyModelVersion: 1` semantics;
- a shared evaluator with fail-closed model-version dispatch;
- closed six-row classification/risk totality;
- canonical artifact hashing over `{ policyModelVersion, rules }`;
- append-only publication-manifest validation with one-time genesis bound to the trusted pre-manifest baseline;
- deep runtime immutability over the canonical serializable value domain;
- runtime registry behavior consistent with Retention Model B;
- repository command capability for publication-history validation.

It does **not** authorize selector resolution, first-round creation, POST `/changes` wiring, approval commands, permissions, Teams/CAB UI, or execution enforcement.

## Architecture review findings

### 1. Genesis dual-code resolution — ACCEPTED

The R2 plan contained a real documentation contradiction for an unauthorized genesis attempt against a baseline where the manifest is absent: one clause expected `GENESIS_NOT_AUTHORIZED`, while another expected `BASELINE_MANIFEST_MISSING`.

The shipped behavior records:

- `BASELINE_MANIFEST_MISSING` whenever the trusted baseline lacks the manifest and genesis is not fully authorized; and
- `GENESIS_NOT_AUTHORIZED` as an additional violation when a genesis flag was supplied but a required authorization condition failed.

This is accepted because the behavior is deterministic, strictly fail-closed, preserves both diagnostic meanings, and does not widen the genesis path. The implementation behavior is now the accepted interpretation for F3.1.1a. Future maintenance of the planning document should describe the same dual-code behavior rather than treating the two codes as mutually exclusive.

### 2. Emergency CAB retrospective SLA = 432000 seconds — ACCEPTED AS POLICY CONTENT

The first published policy artifact contains `432000` seconds (5 calendar days elapsed time) for the emergency CAB retrospective SLA.

Architecture accepts this value **only as immutable content of that published policy version**, not as a domain constant, organization-wide permanent rule, or business-calendar semantic.

Consequences:

- the published artifact must never be edited in place;
- changing 5 calendar days requires a new policy version + publication-manifest entry;
- “5 business days” would be a different semantic model involving calendar/timezone/holiday behavior and is not implied by the current duration-based contract.

### 3. Retention Model B — ACCEPTED

The publication manifest preserves publication identity/digest history; the authorization ledger preserves the immutable round evidence needed for audit/reconstruction; the deployed runtime need only ship active and deliberately supported rollback policy versions.

No requirement is introduced to retain executable historical evaluators forever.

### 4. No CI enforcement claim — ACCEPTED

The validator is a publication-integrity capability and command. The evidence explicitly states that the implementation repository had no pipeline enforcing this command at the reviewed baseline. This acceptance does not upgrade the capability into a mandatory CI gate by documentation fiat.

## Carried-forward blockers / constraints

Unchanged:

- `ChangeManagementService` still calls `buildChange()` twice; this remains **MUST FIX BEFORE F3.1.2**.
- Existing `LEGACY_PRE_F3` idempotency reservations must resume the legacy path; retries must never be reinterpreted into ledger-required mode.
- Missing RBAC CSV / conditional-policy files remain an F3.1.4 prerequisite.

Strategic constraint added by ADR-012:

- F3.1.2 must not hardwire an Azure DevOps pipeline/environment-specific execution context before the Delivery architecture spike resolves the provider-neutral governed execution context (`ReleaseCandidate` / `DeploymentTarget` / Change binding).

## Gate

**F3.1.1a: CLOSED / ACCEPTED IMPLEMENTED BASELINE at ADO `d3c0751` according to the canonical evidence checkpoint.**

**F3.1.1b: NO-GO.** Not implemented and not automatically authorized by this acceptance.

**F3.1.2+: NO-GO.** The next authorized work is the Delivery/GitOps architecture spike, beginning with D0, not execution integration into Change Management.
