# Branching and Release Control — Delivery Spike Refinement

- **Status:** Spike contract / refinement to ADR-012 (ADR-012 remains Proposed)
- **Date:** 2026-09-03
- **Related:** ADR-012 Delivery Management / GitOps promotion

## Purpose

Refine the source-control assumptions used by the Delivery spike without turning a branching brand (Git Flow, GitHub Flow, trunk-based) into a canonical Delivery-domain concept.

The architecture must support teams with different maturity levels while preserving deterministic provenance for the exact software revision and artifact deployed in production.

## Core invariants

1. **Branch is not environment.** No canonical mapping such as `develop -> DEV`, `release -> HML`, `main -> PRD` is allowed in Delivery or Change Management.
2. **`main` means integrated code.** It is not synonymous with DEV, HML or PRD.
3. **Build once, promote many.** A `ReleaseCandidate` freezes the promotable source revision and immutable artifact fingerprint. DEV/HML/PRD promote that candidate; they do not rebuild it per environment.
4. **Production is traceable to exact source + artifact identity.** At minimum: source commit/revision, release identity/tag when applicable, and immutable artifact fingerprint/digest.
5. **Hotfix starts from what is actually in production.** Never blindly from current `main` when `main` contains changes not present in PRD.
6. **Hotfix returns to the integration line.** A production correction must be reconciled back into `main` (and any still-active stabilization branch that also needs it) so the next normal release does not reintroduce the defect.
7. **Release branches are optional stabilization tools, not environment branches or permanent lifecycle branches.**
8. **Tags are immutable release identities, not environment selectors.** A tag identifies a releasable source revision; Delivery records where that release is deployed.

## Greenfield source flow

Preferred default for new Golden Path repositories:

```text
feature/* / bugfix/*
       |
       v
      PR
       |
       v
protected main
       |
       v
trusted CI
       |
       v
ReleaseCandidate(source revision + immutable artifact)
```

A feature enters `main` when it is ready for integration under repository policy (review, CI, tests, quality/security checks). Merge to `main` does **not** mean production deployment or production authorization.

## When a ReleaseCandidate is created

A ReleaseCandidate is created from an **exact trusted source revision** selected for promotion.

Example:

```text
main: A -- B -- C -- D -- E -- F
                    ^
                    |
                  RC-42
```

`RC-42` freezes the revision at `D` plus its immutable artifact fingerprint. `main` may continue to E/F/G while HML still validates RC-42.

This is why a release branch is not automatically required merely to stop later `main` commits reaching PRD: promotion uses RC-42, not the mutable HEAD of `main`.

## When a release branch is created

Create a short-lived `release/*` branch only when the selected release needs stabilization independent of the advancing `main` line.

Typical trigger:

```text
RC-42 from commit D
  -> HML finds defect
  -> main has already advanced to E/F/G
  -> create release/1.8 from D (or the exact RC source revision)
  -> apply only stabilization fixes
```

Allowed work on the release branch:

- defect fixes discovered during validation;
- release-specific compatibility/configuration corrections that are source-controlled;
- no opportunistic new features.

If the ReleaseCandidate passes validation without source corrections, a release branch may never exist.

## When a release tag is created

A version tag identifies the exact source revision accepted as a releasable version.

Preferred timing for the spike:

```text
exact candidate revision
  -> HML accepted / release accepted
  -> create immutable version tag (e.g. v1.8.0)
  -> promote the already-built artifact associated with that revision to PRD when governance allows
```

The tag does not trigger a rebuild for PRD. The artifact promoted to PRD must be the same immutable material that was validated.

A tag also does not mean "currently in production" forever; deployment history belongs to Delivery.

## Hotfix flow

Assume:

```text
PRD = v1.8.0 / source D / artifact digest AAA
main = D + E + F + G
```

A production incident must not create the hotfix from HEAD(main), because that would implicitly include E/F/G.

Required source lineage:

```text
v1.8.0 (or exact deployed source revision D)
   |
   +--> hotfix/INC-123
           |
           +--> fix H
           |
           +--> trusted CI -> immutable hotfix candidate
           |
           +--> required validation / governance
           |
           +--> v1.8.1 (new immutable tag when released)
           |
           +--> PRD promotion of that exact artifact

hotfix H
   +--> reconcile/PR back to main
```

Never move/reuse the previous tag. A released correction receives a new immutable release identity (for example `v1.8.1`).

## Brownfield compatibility

The organization currently has no single branch standard. That is not a D0/D1 blocker.

The spike may model a narrow `ReleaseEligibilityPolicy` boundary that validates trusted source provenance per repository while migration occurs, for example:

```text
new repo          -> protected main / approved exact revision
legacy repo A     -> main | release/* under explicit policy
legacy repo B     -> current hotfix convention under explicit policy
```

This boundary is **not** a generic policy DSL and branch names must not enter Change Management semantics. It only answers whether a source revision is eligible to become a trusted immutable ReleaseCandidate.

## Golden Path direction

For new repositories:

- protected `main`;
- no permanent `develop` branch;
- short-lived PR branches;
- release branch only when stabilization requires an isolated line;
- production hotfix branch automatically/manual-created from the exact deployed source revision, preferably through platform tooling;
- immutable tag/version identity for released revisions;
- mandatory back-merge/reconciliation of stabilization/hotfix fixes into `main`.

The Backstage/Delivery experience should eventually make the safe operation easier than manual Git archaeology. A future action such as **Create Hotfix** should resolve the source revision from the production deployment record rather than asking the developer to guess a base branch.

## D0/D1 questions introduced by this refinement

The Delivery spike must now prove or decide:

1. how the exact production source revision and artifact fingerprint are recorded;
2. whether version tag creation belongs to CI/release tooling or Delivery, without coupling tag identity to environment;
3. how ReleaseCandidate creation is triggered from a trusted source revision;
4. how optional release/stabilization branches are represented as provenance only;
5. how a future safe hotfix action can resolve the production base revision deterministically;
6. how branch policy exceptions for brownfield are kept outside Change Management and environment routing.

## Gate

This refinement replaces the simplistic interpretation "trunk-based means every hotfix starts from current main". It does **not** select classic Git Flow.

The target is a pragmatic invariant-based model: protected integration trunk, immutable release candidates, optional short-lived stabilization branches, and production-based hotfix lineage.
