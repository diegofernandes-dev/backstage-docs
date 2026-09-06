# Parallel CI Foundation workstream

## Status

**Active parallel workstream — not a Delivery milestone authorization.**

A new Azure DevOps CI foundation is being developed in parallel in:

```text
diegofernandes-dev/pipeline-template
```

The work is intentionally scoped to trusted CI / release-material creation and does **not** authorize or replace any Delivery, Kargo, GitOps, Argo CD, GMUD, or production-adoption milestone governed by this documentation set.

## Why this is being done in parallel

ADR-012 already defines a boundary in which trusted CI produces immutable releasable material and Delivery owns promotion after that point.

The existing pipeline implementation historically mixed CI and imperative Kubernetes deployment. The parallel workstream is removing that coupling while modernizing the .NET build contract, so the eventual integration surface aligns with:

```text
trusted CI
  -> immutable release material
  -> Delivery ReleaseCandidate
  -> promotion / target policy
  -> Kargo provider
  -> Git desired state
  -> Argo CD
  -> Kubernetes
```

The CI pipeline is therefore being treated as an **artifact/release-material producer**, not as the Delivery control plane.

## Backstage .NET software templates

`diegofernandes-dev/pipeline-template` is the **target CI pipeline for Backstage .NET software templates**.

```text
Backstage .NET Software Template
  → scaffolds application repo + thin azure-pipelines.yml
  → consumes diegofernandes-dev/pipeline-template
  → extends pipeline/templates/dotnet-ci.yml @ immutable tag
  → produces immutable release material (image + digest + CI metadata)
  → END OF CI
```

Current platform surface (lab):

| Concern | Repository / path |
|---------|-------------------|
| CI implementation | `diegofernandes-dev/pipeline-template` |
| Consumer-facing template | `pipeline/templates/dotnet-ci.yml` |
| Current immutable ref | `refs/tags/v0.2.0` |
| Sample consumer | `diegofernandes-dev/dotnet-templates` |

This replaces the historical direction of using `platform-pipeline-templates` / `dotnet-ci-1.0.0` as the long-term .NET CI contract for new Backstage-scaffolded apps.

**Scope note:** documenting this target does **not** by itself rewire the Backstage portal scaffolder/provisioner. Portal bootstrap generation and golden-path cutover remain a follow-up implementation step. Until that cutover lands, treat `pipeline-template` / `dotnet-ci.yml` as the accepted CI contract for .NET templates going forward.

## Current CI direction

The `pipeline-template` workstream is converging on these invariants:

1. **CI-only responsibility.** Build, test, publish, containerize, and publish immutable image material. Environment promotion is not the target responsibility of Azure Pipelines.
2. **Build once.** Restore once, compile once, run tests without rebuilding, publish without rebuilding, then package that publish output into the OCI image.
3. **Source-owned .NET toolchain.** Application repositories declare the SDK through `global.json`; the pipeline does not hardcode .NET 10 as the universal build SDK.
4. **Deterministic onboarding contract.** New/onboarded applications are expected to declare `global.json` and NuGet lock files rather than relying on an implicit compatibility mode.
5. **Separate build and deployable scopes.** `buildPath` represents the restore/build/test unit; `projectPath` identifies the workload project to publish.
6. **Runtime-only container build.** The Dockerfile no longer restores or recompiles .NET source. It packages previously published output.
7. **Immutable image identity.** Application source commit remains a human-readable image tag; the workstream is expected to capture the pushed OCI digest as the stronger artifact fingerprint.
8. **Cross-repository platform ownership.** Pipeline YAML, build scripts, Dockerfile, and contract tests live in the pipeline-template repository and are consumed by application/template repositories through an immutable repository ref.
9. **No normal imperative Kubernetes CD.** The target architecture is not to retain Helm deployment stages, Azure DevOps environment waits, or private deployment agents as the normal GitOps path.

## Intended Delivery integration boundary

This workstream does **not** define the final `ReleaseCandidate` HTTP/schema contract.

It is expected to make trustworthy CI facts available for a future Delivery adapter, including at least the following concepts:

```text
component / application identity
source repository
source commit
CI run/build identity
image repository
image tag
image digest
platform-template version/commit
created-at / provenance facts
```

The authoritative ReleaseCandidate should not blindly trust arbitrary caller-supplied branch/build/artifact claims. Where Azure DevOps is the CI authority, the Delivery integration should resolve or verify authoritative run/source/artifact metadata server-side as already required by ADR-012.

Conceptually:

```text
Azure DevOps trusted CI
  -> ECR image + digest
  -> CI/build identity
  -> Delivery integration resolves/verifies authoritative metadata
  -> immutable ReleaseCandidate
```

Whether this is triggered by a pipeline callback, event, polling adapter, or another thin integration mechanism remains a Delivery integration decision and is not fixed by the CI workstream.

## Relationship to Kargo / Argo CD

The CI workstream does not call Kargo to perform environment promotion and does not call `argocd app sync` as the normal deployment path.

After ReleaseCandidate creation, the existing Delivery boundary remains authoritative:

```text
Delivery
  -> promotion provider (Kargo in the accepted Kubernetes direction)
  -> Git desired state
  -> Argo CD reconciliation
  -> Kubernetes
```

Kargo remains a provider under Delivery semantics, not the business authorization authority. Change Management remains the production business authorization authority where policy requires a Change.

## Helm ownership impact

The current pipeline repository still contains a generic Helm chart because it originated from the imperative-CD implementation.

Under the accepted GitOps boundary, the long-term ownership of workload deployment manifests/chart must be moved or referenced from the GitOps/workload-delivery side rather than remaining a runtime dependency of CI.

That relocation is a follow-up design step. The CI foundation should not deepen coupling to the current chart location while this boundary is being finalized.

## Parallel-workstream guardrails

While CI work proceeds in parallel:

- do not interpret CI completion as authorization to advance Delivery production rollout;
- do not add Delivery/Kargo/Argo identifiers to canonical Change objects;
- do not move PRD approval back into Azure DevOps environment waits;
- do not make source branch an environment-routing mechanism;
- do not rebuild images per environment;
- do not make Kargo approval a duplicate business approval layer;
- do not bypass the existing production authority hardening / adoption gates in this documentation set.

## Near-term convergence checkpoint

Before connecting the new CI to Delivery, prove the CI foundation independently with at least:

```text
cross-repository immutable template consumption
solution + real test project via buildPath
mandatory global.json
mandatory locked NuGet restore
.NET 8 and .NET 10 consumers
multi-SDK/global.json scoping
compile-once / publish-no-build semantics
runtime-only non-root linux/amd64 image
ECR OIDC publication
captured image digest
```

After that proof, define and review the narrow CI -> Delivery integration contract against ADR-012 before implementing it.

## Architectural summary

The two parallel streams should converge at one explicit boundary:

```text
CI workstream
  source -> trusted immutable release material
                    |
                    v
Delivery workstream
  ReleaseCandidate -> target promotion -> Git -> Argo CD -> Kubernetes
```

This preserves the accepted rule:

> **Change Management authorizes. Delivery promotes. Git declares. Argo CD reconciles. Kubernetes executes. Backstage composes the experience.**
