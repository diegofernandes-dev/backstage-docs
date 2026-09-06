# P1 — Production authority hardening checkpoint

## Role

Act as an independent Principal Platform Architect / Staff+ Platform Engineer responsible for closing the **production-adoption authority blockers** that remain after ADR-012 was Accepted.

This is a narrow execution-and-evidence checkpoint. It is **not** a new architecture spike, not a Delivery feature milestone, and not a general security-hardening sweep.

The accepted architecture is the baseline:

> Change Management authorizes. Delivery promotes. Git declares. Argo CD reconciles. Kubernetes executes. Backstage composes the experience.

Do not redesign that boundary unless hard evidence shows it is unworkable.

---

## Canonical baseline protocol

Before changing anything:

1. `git fetch origin main` in `diegofernandes-dev/backstage-docs`.
2. Verify the current `origin/main` SHA.
3. Read at minimum:
   - `docs/adr/ADR-012-delivery-management-gitops-promotion.md`
   - `docs/delivery/README.md`
   - `docs/delivery/adr-012-adoption-rereview.md`
   - `docs/delivery/eligibility-window-toctou.md`
   - `docs/delivery/d0-architecture-review.md`
   - `docs/delivery/d1-kargo-fit-evaluation.md`
   - `docs/delivery/mvp-vertical-delivery-slice.md`
   - `docs/delivery/e1-multi-activity-concurrency.md`
4. Inspect the latest implementation SHA/branch recorded by canonical docs before making claims.
5. Inspect the actual current GitOps/Kargo/Argo sandbox state where available; do not reconstruct authority topology from memory.

If the implementation repository or sandbox cannot be independently inspected, state exactly which conclusions rely on canonical evidence only.

---

## Current known state to challenge, not blindly trust

At the prior accepted-ADR checkpoint:

- ADR-012 was **Accepted** as architecture.
- production rollout remained **NO-GO**.
- E1 closed activity binding, multi-activity completion semantics, and same-target concurrency.
- eligibility-window TOCTOU was subsequently closed and recorded as `PASS`.

The remaining production-adoption blockers were authority/security related, notably:

1. Argo CD / Kubernetes execution authority was broader than acceptable production scope in the sandbox topology.
2. Git writer authority and reconciler read authority were not separated into production-grade non-human identities.
3. resistance to ordinary squad / pipeline bypass of the governed promotion path was not proven.
4. Argo `Application` / `AppProject` governance remained partly imperative and therefore outside the same governed desired-state path.

These are the focus of P1.

---

# P1 objective

Prove, with the smallest credible production-like authority model, that an ordinary squad developer/pipeline **cannot bypass the governed Delivery path to mutate production desired state or production runtime**, while the legitimate platform path still works.

P1 should close the authority story with **positive and negative evidence**, not merely by showing YAML that looks restrictive.

---

# Scope — GO

P1 may implement only the minimum changes necessary to prove the four authority properties below.

## P1.1 — Separate Git identities

Create/prove separate non-human identities/credentials for:

- **Delivery/Kargo writer**: may perform only the required desired-state mutation workflow for the governed GitOps repository/path/branch.
- **Argo reconciler reader**: may read desired state but must not have write authority to the GitOps repository.

Requirements:

- do not use a personal PAT/user SSH key as the final proof;
- identify the actual Azure DevOps identity/service-principal/service-account model used;
- record repository/branch/path permissions factually;
- demonstrate a negative write attempt using the reconciler identity;
- demonstrate the legitimate writer path succeeds;
- do not broaden writer permissions merely to make the test pass.

If Azure DevOps cannot provide the exact intended read-only/write-separated identity shape, document the precise platform limitation and classify whether an alternative identity mechanism is required. Do not fake separation with two secrets tied to the same human authority.

## P1.2 — Argo/Kubernetes least privilege

Replace or constrain the production-like Argo reconciliation authority so it is no longer effectively cluster-admin-equivalent for the governed target.

The proof must demonstrate:

- Argo can reconcile the intended application resources in the intended namespace/target;
- Argo cannot mutate at least one explicitly out-of-scope namespace/resource;
- cluster-scoped mutation is denied unless specifically required and justified;
- `AppProject` source/destination/resource restrictions align with the Kubernetes credential/RBAC boundary rather than being the only line of defense.

Prefer the narrowest credible topology. If the local sandbox topology structurally prevents a production-like scoped destination identity, build the smallest separate target/cluster/credential shape necessary to prove it rather than weakening the requirement.

Do not call a YAML review sufficient evidence. Execute negative authorization checks.

## P1.3 — Govern Argo control objects

Prove a governed path for at least the production-like `Application` / `AppProject` objects used by this flow.

Goal:

- ordinary squad users/pipelines cannot imperatively alter the production Application/AppProject to escape source/destination/resource policy;
- the platform has an explicit authority for changing those control objects;
- the normal delivery path does not require ordinary Delivery code to mutate those control objects dynamically.

This may be Git-managed control-plane configuration or another narrowly documented platform-admin mechanism, but the authority boundary must be explicit and negatively tested.

Do not build a generic control-plane management framework.

## P1.4 — Squad/pipeline bypass resistance

Actively attempt bypasses using the strongest ordinary squad identity available in the sandbox/prod-like setup.

At minimum test:

1. direct push / mutation of protected production desired state;
2. use of the Argo reconciler credential to write Git;
3. direct Kubernetes mutation of the governed production namespace/target;
4. creation/alteration of Argo Application/AppProject or Kargo production promotion objects outside the governed Delivery authority, where applicable;
5. an Azure DevOps application pipeline attempting the shortest plausible bypass of Delivery/Change authorization.

The expected result is fail-closed denial for unauthorized paths.

Do not weaken policies after a negative test fails simply to preserve the current design. If ordinary squad authority can bypass the model, classify that honestly as a blocker.

---

# Backstage / UI evidence

The agent may navigate the running Backstage UI/browser where useful.

Use UI evidence only to verify that hardening did not damage the composed product experience, for example:

- Component -> Deployments still loads;
- GMUD/Change binding remains visible;
- a legitimate governed PRD request can still progress to the point allowed by the sandbox;
- security denials surface as understandable errors rather than misleading success states.

Do **not** use Backstage UI behavior as proof of infrastructure authorization. The authority proof must come from Git, Azure DevOps permissions, Kubernetes RBAC, Kargo/Argo objects, and negative execution evidence.

If browser screenshots are available, capture useful before/after evidence. If they are not, record exactly what was verified and through which observable source.

---

# Out of scope — NO-GO

Do not implement any of the following in P1 unless a minimal change is strictly required to preserve the authority proof:

- new Delivery features or global Delivery workbench;
- multi-cluster architecture expansion beyond the smallest proof topology;
- HA/DR for Backstage, Kargo, Argo, or Change Management;
- break-glass implementation;
- automatic rollback;
- supply-chain signing/admission expansion;
- organization-wide branching migration;
- generalized policy engine / workflow engine;
- redesign of Change Management / Delivery boundaries;
- Teams/CAB work;
- broad RBAC cleanup unrelated to the governed production path.

Do not declare production rollout GO merely because P1 passes. Production rollout needs a separate adoption review after P1 evidence is recorded.

---

# Required execution discipline

## Before implementation

Produce a short factual baseline matrix:

| Surface | Current identity | Current authority | Risk | P1 target |
|---|---|---|---|---|
| Delivery/Kargo Git writer | ... | ... | ... | ... |
| Argo Git reader | ... | ... | ... | ... |
| Argo Kubernetes identity | ... | ... | ... | ... |
| Kargo production promotion authority | ... | ... | ... | ... |
| App/Application/AppProject mutation authority | ... | ... | ... | ... |
| ordinary squad pipeline identity | ... | ... | ... | ... |

Do not change anything until the current authority graph is understood.

## During implementation

- make the narrowest changes possible;
- keep configuration declarative where practical;
- preserve existing successful Delivery tests;
- do not mix unrelated brownfield WIP into P1 commits;
- use dedicated commits for authority changes where repository boundaries allow it;
- record exact implementation/config SHAs.

## After implementation

Re-run:

- relevant Delivery/Change regression suites;
- legitimate governed promotion path at least far enough to prove the intended authority works;
- all required negative bypass tests.

A positive happy path without negative-denial evidence is not a PASS.

---

# Required evidence matrix

Score each property as one of:

- `PROVEN`
- `PARTIALLY_PROVEN`
- `NOT_PROVEN`
- `CONTRADICTED`

| Property | Score | Positive evidence | Negative evidence | Residual gap |
|---|---|---|---|---|
| Git writer/reconciler separation | | | | |
| Argo/K8s least privilege | | | | |
| Argo control-object governance | | | | |
| ordinary squad/pipeline bypass resistance | | | | |

Also record every credential/identity claim without exposing secrets.

---

# P1 verdict

Return exactly one:

```text
P1 verdict: PASS
```

Use only if all four authority properties are PROVEN or any remaining partial item is demonstrably non-blocking for production-adoption review.

or

```text
P1 verdict: CONDITIONAL_PASS
```

Use only if the core bypass model is proven but one narrowly bounded authority item still needs a final production-environment confirmation.

or

```text
P1 verdict: FAIL
```

Use if an ordinary squad/pipeline can still bypass the governed path, if identity separation is cosmetic rather than real, or if Argo/Kubernetes authority remains effectively unconstrained.

Do not modify ADR-012 status. It is already Accepted unless new evidence shows architecture rework is required.

Do not declare production rollout GO in this checkpoint.

---

# Canonical documentation update

At completion, update canonical docs factually.

Create:

`docs/delivery/p1-production-authority-hardening.md`

Update at minimum:

- `docs/delivery/README.md`
- any current-state pointer that would otherwise be factually stale
- `prompts/README.md` only if the normal prompt lifecycle requires marking this checkpoint complete

Do not rewrite historical evidence.

The P1 evidence record must include:

- docs baseline SHA;
- implementation/config repos, branches, and SHAs;
- identity/authority matrix;
- exact positive and negative tests executed;
- commands/actions at a reproducible level without secrets;
- Backstage/UI observations if used;
- residual risks;
- final P1 verdict;
- recommended next gate.

---

# Recommended next gate after P1

If P1 is PASS/CONDITIONAL_PASS, recommend a separate **production adoption review** that decides whether production rollout may become `GO`, `CONDITIONAL_GO`, or remain `NO_GO`.

That later review may evaluate remaining operational items such as break-glass, rollback policy, audit retention, HA/DR, and supply-chain controls according to actual rollout requirements.

Do not start that review or implement those items inside P1 unless separately authorized.

---

# Final report format

Return a compact report with:

```text
P1 verdict: <PASS | CONDITIONAL_PASS | FAIL>
Docs baseline SHA: <sha>
Implementation/config SHAs: <repo@sha ...>

Git identity separation: <PROVEN | PARTIALLY_PROVEN | NOT_PROVEN | CONTRADICTED>
Argo/K8s least privilege: <...>
Argo control-object governance: <...>
Squad/pipeline bypass resistance: <...>

Positive governed path: <result>
Negative bypass suite: <result>
Backstage UI/browser verification: <result or not used>

Residual blockers: <none or concise list>
Recommended next gate: <production adoption review or blocker-specific checkpoint>
Docs updated: <paths>
STOP
```

STOP after recording P1 evidence. Do not continue into the production-adoption decision in the same run.