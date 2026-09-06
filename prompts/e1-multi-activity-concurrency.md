# E1 — Multi-activity binding and same-target concurrency

## Role

You are acting as an independent Staff+/Principal Platform Architect and implementation reviewer.

Your job is to execute only the narrow E1 evidence checkpoint identified by the ADR-012 adoption review.

Do not optimize for accepting ADR-012. Do not redesign the platform. Do not expand into production hardening.

## Canonical source protocol

Before doing anything:

1. `git fetch origin main` in `diegofernandes-dev/backstage-docs`.
2. Record the exact latest `origin/main` SHA.
3. Read the current canonical sources, at minimum:
   - `docs/delivery/adr-012-adoption-review.md`
   - `docs/adr/ADR-012-delivery-management-gitops-promotion.md`
   - `docs/adr/ADR-008-multi-activity-change-execution-plan.md`
   - `docs/adr/ADR-009-change-authorization-model.md`
   - `docs/delivery/mvp-vertical-delivery-slice.md`
   - `docs/delivery/mvp-demo-hardening.md`
   - `docs/delivery/README.md`
4. Inspect the implementation repository/branch actually used by the current Delivery MVP and record its exact SHA before changing code.
5. If any remembered assumption conflicts with current canonical docs or source, current evidence wins.

## Why E1 exists

The ADR-012 adoption review concluded:

```text
ADR-012: REMAIN_PROPOSED
Production rollout: NO-GO
```

The smallest next architecture-evidence checkpoint is E1. E1 exists to close architecture-decision uncertainty around:

- binding one software deployment to a specific `ExecutionActivity` without turning activities into workflow tasks;
- proving that one successful deployment does not falsely complete a multi-activity Change;
- deterministic behavior when two distinct releases attempt to promote to the same target concurrently.

This is not a broad Delivery milestone.

## GO scope

E1 authorizes only the smallest implementation/evidence necessary to prove the following three mandatory claims.

### E1.1 — Activity-scoped Delivery binding

Create a real Change containing at least two `ExecutionActivity` items.

Delivery must be able to bind a governed production-oriented `DeploymentRequest` to:

```text
changeId
+ activityId
+ release fingerprint / ReleaseCandidate
+ DeploymentTarget
```

The binding must remain owned by Delivery. Do not add Kargo, Argo, Git, pipeline, deployment, stage, or provider identifiers to canonical Change.

`ExecutionActivity` must remain a planned work unit as defined by ADR-008. Do not add per-activity workflow lifecycle, approval state, execution engine semantics, DAG behavior, or task orchestration.

If the current Delivery schema lacks `activityId`, implement only the narrowest provider-neutral contract/storage/API change required to add that correlation.

Required negative checks:

- an `activityId` that does not belong to the bound Change must fail;
- an activity belonging to another Change must not be accepted;
- changing release material or target must not silently reuse the prior governed binding;
- provider-specific IDs must not leak into canonical Change.

### E1.2 — Multi-activity completion boundary

Using the same Change with at least two activities, successfully execute or faithfully exercise one software-delivery activity.

Prove with state/evidence that success of that deployment:

- may be correlated as execution evidence for the bound activity;
- does **not** mark the whole Change completed merely because the deployment succeeded;
- does **not** imply that the remaining activity is complete;
- does **not** create hidden per-activity workflow/lifecycle semantics inconsistent with ADR-008.

Inspect the Change record, authorization/evidence surfaces, and any Delivery projection before and after the deployment.

Safe-by-omission is not enough. The evidence must explicitly demonstrate the boundary.

### E1.3 — Same-target concurrency

Create two distinct immutable ReleaseCandidates/Freights targeting the same production-oriented `DeploymentTarget`.

Trigger the two promotion/dispatch attempts concurrently or sufficiently overlapped to exercise the real same-target mutation path.

The outcome must be deterministic and safe. Acceptable evidence includes one of:

```text
exactly one winner + second request safely blocked/queued/rejected
```

or

```text
one succeeds + one fails visibly/retryably due to a real Git/provider conflict
```

provided that all of the following are true:

- no silent double mutation;
- no force-push;
- no ambiguous final desired state;
- no duplicate logical promotion hidden behind different provider objects;
- final Git desired state is explainable;
- Argo/Kubernetes final state is explainable if the live path reaches reconciliation.

Do not invent a large lock manager, workflow engine, queueing subsystem, or distributed coordinator just to pass this checkpoint. If the current architecture cannot satisfy the test without such a redesign, stop and report that as architecture evidence.

## Optional adjacent evidence — only if cheap

If it fits naturally inside the same harness without expanding scope, exercise one window TOCTOU case:

```text
eligibility ALLOW while window is valid
→ window expires
→ dispatch performs fresh eligibility
→ dispatch DENY
→ no provider promotion created
```

Do not delay E1 completion to build a full time/window framework if the current code cannot support this narrowly.

## Backstage UI/browser inspection

You have the ability to navigate the running Backstage UI. Use it where it materially improves the evidence.

Do not treat browser inspection as decoration. Use it to validate the composed product experience and to catch projection/UX defects that source-only review may miss.

At minimum, if the running UI is available:

1. Open the relevant Catalog Component.
2. Inspect the `Deployments` tab before and after the E1 binding/execution.
3. Open the related GMUD/Change detail/workbench surface.
4. Verify whether the bound `activityId` / activity identity is visible or traceable in a way that helps a human understand which planned activity the deployment belongs to.
5. Verify the UI does not imply that the entire Change is complete after only one deployment activity succeeds.
6. During the concurrency scenario, inspect whether the UI presents both requests and their outcomes clearly enough to avoid a false impression of two simultaneous successful mutations.
7. Capture screenshots or equivalent factual UI evidence if the agent environment permits.

If the UI exposes an architectural or product defect directly related to E1 — for example, it cannot distinguish activity binding, shows stale deployment state, or falsely implies Change completion — fix only the narrowest E1-blocking issue and add regression coverage where appropriate.

Do not redesign the Backstage information architecture, add a global Delivery Workbench, or polish unrelated UI.

Browser/UI evidence is supplemental. The core E1 claims must still be proven through backend/domain/provider/Git state; the UI must not be treated as runtime authority.

## Evidence requirements

Use real evidence wherever the sandbox supports it.

For each mandatory E1 claim, collect:

- exact implementation SHA;
- exact docs baseline SHA;
- request/Change/ReleaseCandidate/target/activity identifiers;
- relevant API responses or database/state readback;
- relevant automated test output;
- provider/Kargo object names where useful as evidence only;
- Git desired-state commit/revision where applicable;
- Argo sync/health and Kubernetes runtime evidence if the live path executes;
- UI/browser observations where available.

Distinguish clearly between:

```text
[code]
[live-system]
[git]
[provider]
[ui]
[docs/prior-run]
```

Do not claim live proof from source inspection alone.

## Minimum regression coverage

Add only tests directly protecting the E1 invariants introduced or clarified by this checkpoint.

At minimum, cover:

- valid `changeId + activityId` binding;
- reject activity from another Change / nonexistent activity;
- activity-scoped binding persists with release + target correlation;
- successful bound deployment does not complete the whole multi-activity Change;
- deterministic same-target concurrent-dispatch behavior at the narrowest testable layer;
- idempotent retry behavior remains intact.

Preserve existing Delivery, GMUD, Platform-tab, and demo-hardening tests.

## Explicit non-goals / NO-GO

Do not implement:

- production Argo/Kubernetes RBAC redesign;
- Git writer/reconciler credential redesign;
- production cluster topology changes;
- break-glass implementation;
- automatic production rollback;
- HA/DR platform work;
- multi-cluster rollout;
- global Delivery Workbench;
- Teams/CAB expansion;
- generic workflow/state-machine infrastructure;
- organization-wide branching enforcement;
- supply-chain/signature/admission work;
- ADR-012 acceptance as a side effect of implementation.

Do not change ADR-012 status during E1.

## Decision gate

At the end, classify E1 as exactly one of:

```text
PASS
CONDITIONAL_PASS
FAIL
ARCHITECTURE_REWORK_SIGNAL
```

Use:

- `PASS` — all three mandatory E1 claims are strongly proven with no material E1-specific gap;
- `CONDITIONAL_PASS` — the architecture claims are proven but a narrow non-architectural limitation remains;
- `FAIL` — required evidence could not be produced or a mandatory invariant failed;
- `ARCHITECTURE_REWORK_SIGNAL` — satisfying E1 appears to require breaking ADR-008/009/012 boundaries, adding workflow-engine semantics, or introducing disproportionate orchestration machinery.

Do not translate an E1 PASS into ADR-012 Accepted. E1 only authorizes a subsequent independent adoption-gate review.

## Required documentation update

Update canonical docs with factual results only.

Create or update an E1 evidence record under `docs/delivery/`, for example:

```text
docs/delivery/e1-multi-activity-concurrency.md
```

Update `docs/delivery/README.md` and any current-state pointer necessary to make the E1 result discoverable.

Do not rewrite historical D0/D1/MVP evidence.
Do not change ADR-012 status.
Do not add a new ADR unless the execution reveals a genuinely new architecture decision that cannot be represented as evidence against ADR-012; if that occurs, STOP and report instead of inventing the ADR automatically.

Record both:

```text
implementation SHA
canonical docs SHA
```

in the final handoff.

## Final report format

Return a concise but factual report with:

```text
E1 verdict
Docs baseline SHA
Implementation SHA

E1.1 activity-scoped binding
- evidence
- negative checks
- result

E1.2 multi-activity completion boundary
- evidence
- Change state before/after
- result

E1.3 same-target concurrency
- concurrent inputs
- observed provider/Git behavior
- final desired/runtime state
- result

Backstage UI/browser verification
- screens visited
- observations
- any narrow fix performed

Optional window TOCTOU evidence
- performed / not performed
- result or reason omitted

Regression tests
Residual E1 gaps
Architecture implications
Recommended next gate
Docs updated
STOP
```

## STOP

Stop immediately after E1 evidence, tests, and documentation are complete.

Do not:

- accept ADR-012;
- start production hardening;
- start the next Delivery milestone;
- implement E2 or any broader roadmap item.

The next step after a successful E1 is a separate independent ADR-012 adoption review, explicitly authorized by the user.