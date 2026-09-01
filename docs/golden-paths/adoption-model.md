# Brownfield adoption model

> **Status:** Reconciled with ADO evidence (Slice 0 complete)  
> **Principle:** Golden Path, not Golden Cage

## 1. Goal

Allow existing applications to obtain platform value incrementally while converging, over time, on the same Catalog, ownership, delivery and governance model used by greenfield applications.

Brownfield adoption is a first-class product path, not a migration appendix.

## 2. Adoption stages

Use explicit stages so teams can gain value before full modernization.

### Proposed stages (architecture)

| Stage | Meaning | Minimum outcome |
|---|---|---|
| 0 — Unregistered | Application exists outside the governed platform model | none |
| 1 — Registered | Application is discoverable and owned | Catalog identity, owner, system/component relations, repository link |
| 2 — Assessed | Current capabilities and gaps are known | conformance/adoption assessment with actionable gaps |
| 3 — Platform-connected | One or more central platform capabilities are adopted | explicit bindings such as supported pipeline contract, docs or governance controls |
| 4 — Conformant | Required baseline capabilities are satisfied or governed exceptions exist | policy/conformance result with no unknown critical gaps |
| 5 — Modernized | Application uses the target supported patterns for its applicable workload | reduced local delivery/platform debt |
| R — Retiring/Retired | Application is intentionally leaving service | explicit lifecycle and safe decommission path |

### Implemented stages (implementation ADR 0013, WIP)

| Stage ID | Meaning |
|---|---|
| `discovered` | Known to the org; not yet in IDP |
| `cataloged` | Component + owner + System + repo link (set by `register-existing-application`) |
| `documented` | TechDocs minimum + lifecycle defined |
| `governed` | Required metadata + ownership validated (set by `enrich-catalog-pr`) |
| `delivery-integrated` | Central CI + build validation + baseline scans |
| `platform-deployed` | Deploy contract + observability |
| `golden-path` | Aligned to current greenfield standard |

Stored in `metadata.annotations.idp.company/platform-adoption-stage`.

Stages describe platform adoption, not business criticality or software quality.

## 3. Registration

Registration should be low-friction and non-destructive.

The flow should capture or validate:

- repository identity;
- owner Group;
- System/product relationship;
- Component/workload identity;
- relevant lifecycle metadata;
- links/annotations required for discovery;
- known unsupported conditions that block later automation.

Registration must not require changing application source, pipeline implementation or runtime architecture unless those changes are necessary simply to express a valid Catalog entity.

## 4. Assessment

After registration, evaluate observable capabilities against a versioned platform profile.

Assessment categories may include:

- Catalog completeness and ownership;
- pipeline contract and delivery integration;
- repository governance;
- documentation/operational metadata;
- workload classification;
- deployment/platform integration;
- security/quality controls where observable;
- declared exceptions.

The assessment output should separate:

1. **facts** — what is currently observed;
2. **requirements** — what the applicable profile expects;
3. **gaps** — differences requiring action;
4. **exceptions** — intentional time-bounded deviations;
5. **unknowns** — items the platform cannot safely infer.

Unknown must not silently mean compliant or non-compliant.

## 5. Progressive adoption

A legacy application may adopt capabilities independently when safe.

Example progression:

```text
Register
  -> normalize ownership
  -> adopt central CI contract
  -> adopt deployment binding
  -> add required operational metadata
  -> remediate repository governance
  -> close remaining gaps / record exceptions
```

The ordering should be driven by dependencies and risk rather than by a requirement to make all changes in one large migration.

## 6. PR-based modernization

Where repository changes are needed, prefer reviewable pull-request based modernization.

A modernization action should:

- derive changes from current repository state, not from a stale assumption;
- make the smallest coherent change for one capability;
- avoid overwriting application-owned customization blindly;
- explain why files changed and which platform contract/version is being adopted;
- run validation before proposing the PR;
- leave merge authority with the repository's normal governance unless a separately approved automation model exists.

Examples implemented in WIP (`modernize-application` template):

- adding or normalizing `catalog-info.yaml` (`idp:enrich-catalog-pr`);
- replacing a duplicated pipeline implementation with a thin central binding (`idp:adopt-platform-ci-pr`);
- adding a required contract/version declaration via `idp.platform.yaml`;
- adding standard metadata or documentation bootstrap (`idp:adopt-techdocs-pr`);
- promoting to corporate catalog (`idp:promote-catalog-pr` — promote URL placeholder must be fixed before production use).

PR-based modernization is implemented in WIP but uncommitted. Production use requires commit, review, and trust-boundary validation.

## 7. Monorepos and multi-workload brownfield systems

Assessment and adoption operate on the correct subject boundary.

- repository-level facts apply once to the repository;
- Component/workload facts apply per independently deployable workload;
- System-level ownership/product relationships apply at the logical system boundary;
- one non-conformant workload must not force unrelated workloads in the same repository to be modeled as one Component.

The registration flow should therefore be capable of discovering or accepting multiple Components for one repository.

## 8. Exceptions

Exceptions allow adoption without pretending that a gap is resolved.

An exception should be:

- explicit;
- scoped to a subject and requirement;
- owned;
- justified;
- reviewable;
- time-bounded where possible;
- visible in conformance output.

An exception does not convert a legacy pattern into a supported Golden Path. It records a governed deviation.

## 9. Day-2 re-evaluation

Adoption is not a one-time import.

The platform should re-evaluate registered applications when:

- observable repository/catalog state changes;
- the applicable platform profile changes;
- an exception expires;
- a new required capability becomes applicable;
- modernization is completed.

Re-evaluation should be deterministic against a known profile/version and should not require re-running the creation template.

## 10. Retirement

Brownfield applications may enter retirement without first becoming fully conformant.

Retirement should prioritize safe ownership and dependency visibility over modernization work that has no remaining value.

A retiring application should retain enough Catalog/audit information to make dependencies and ownership understandable until decommissioning is complete.

## 11. Product experience

The Backstage experience should make the next useful action obvious rather than presenting a wall of failed controls.

Suggested experience:

```text
Existing Application
  -> Register
  -> Assessment summary
       Compliant: 6
       Gaps: 3
       Exceptions: 1
       Unknown: 2
  -> Recommended next actions
       [Adopt central pipeline]
       [Fix ownership metadata]
       [Review exception]
```

The score itself is secondary to actionable, explainable findings.

## 12. Success criteria

Brownfield adoption is successful when:

- teams can register without forced rewrite;
- ownership and discoverability improve immediately;
- gaps are explicit rather than hidden;
- modernization can occur in small reviewable steps;
- greenfield and brownfield eventually share the same durable platform contract;
- exceptions and retirement are modeled without template forks or undocumented side channels.
