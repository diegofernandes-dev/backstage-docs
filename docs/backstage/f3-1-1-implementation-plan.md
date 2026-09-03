# GMUD F3.1.1 — Published Policy & Selector Resolution implementation plan

- Status: **F3.1.1a IMPLEMENTED / PUBLISHED — pending architecture implementation review.**
  Implementation SHA: ADO `d3c0751` (full SHA `d3c0751a15b908cec8f5595c97e52f41226344ed`),
  direct child of `4bad41d`, on `feat/ado-repo-governance`. F3.1.1b remains
  **NOT IMPLEMENTED, NOT AUTHORIZED**; F3.1.2+ remain **NOT AUTHORIZED**. See
  `implementation-progress.md` for the full evidence checkpoint.
  **Documentation defect found and resolved during implementation:** §6's
  normative table and §6a case C assign `GENESIS_NOT_AUTHORIZED` to a
  candidate whose `--allow-genesis-from` resolves to a baseline other than
  `AUTHORIZED_GENESIS_BASELINE_SHA`, while §6a row D and test case J5 assign
  `BASELINE_MANIFEST_MISSING` to the same input. The shipped implementation
  resolves this by emitting **both** codes whenever the baseline is absent
  and genesis is not fully authorized — `BASELINE_MANIFEST_MISSING` always,
  plus `GENESIS_NOT_AUTHORIZED` (with a `flag-mismatch` or
  `baseline-ref-mismatch` reason) whenever a genesis flag was also supplied
  but failed a condition. This satisfies every §6/§6a/§26.J case as written
  and remains fail-closed; it is flagged here for architecture review to
  reconcile the plan text itself.
- Prior status: **PLAN CORRECTED (R2) / UNDER FINAL REVIEW**
- Date: 2026-09-03 (F3.1.1-R2); prior revision 2026-09-02 (F3.1.1-R)
- ADO implementation baseline (verified): `platform-devops-developer-portal@4bad41d`
  (full SHA `4bad41d058edf5c5314d17275e0c8bdb5abf690f`), branch `feat/ado-repo-governance`.
  Re-verified for R2 **against the Azure DevOps REST API** (`refs/heads/feat/ado-repo-governance`
  → `objectId 4bad41d058edf5c5314d17275e0c8bdb5abf690f`), not merely against a local
  `origin/*` ref; ADO has not moved since F3.1.0-H closure.
- Documentation baseline used to produce this R2 correction: `backstage-docs@d5fc3ff93a8916932860f0c18ad5f3863f9f163c`
  (the F3.1.1-R checkpoint this revision corrects). The prior R revision was produced
  from `75abd9d60facc6ff1dfc54af4a46192f4c287c9e`.
- Architecture authority: [ADR-009](../adr/ADR-009-change-authorization-model.md) (unchanged by this checkpoint — see [§0a](#0a-what-changed-in-r2-and-why))
- Prior checkpoint: [F3.1.0 implementation plan](./f3-1-implementation-plan.md) — **F3.1.0 CLOSED**

## Correction notice (F3.1.1-R, superseded by R2 below)

This document was first a **corrective revision** of the F3.1.1 plan published at
`75abd9d`. Architecture review found three defects in that version, described in
[§0](#0-what-changed-and-why) below. The plan is corrected in place — sections
are renumbered where the correction reshapes the model; the F3.1.1 scope,
decomposition, and non-goals are otherwise preserved. **F3.1.1a implementation
remains unauthorized**; nothing in ADO was touched to produce this correction.

## Correction notice (F3.1.1-R2 — current)

Architecture review of the R revision (`d5fc3ff`) found **three further contract
gaps**, corrected here and described in [§0a](#0a-what-changed-in-r2-and-why):
undefined first-publication (genesis) semantics, unversioned evaluator
semantics, and shallow runtime immutability. **The accepted R direction is
preserved and not redesigned** — policy versions remain serializable data, one
generic evaluator is shared, behavior lives in typed `rules`, input remains
`{classification, risk}`, output remains `RequirementDefinition[]`, the manifest
remains append-only, and the F3.1.1a/F3.1.1b split is unchanged. **F3.1.1a
implementation remains unauthorized**; no ADO source, test, config, script,
migration, pipeline, or runtime behavior was modified to produce this revision.

## Objective

F3.1.0 created the durable, append-only authorization ledger
(`AuthorizationRound`, `ApprovalRequirement`, `ApprovalDecision`,
`AuthorizationAuditEvent`) but populates nothing — every `POST /changes` still
marks the Change `LEGACY_PRE_F3` and creates no round.

F3.1.1 designs the deterministic mechanism that answers, for an immutable
Change snapshot:

1. which published authorization policy version applies;
2. which `ApprovalRequirement`s that policy produces;
3. which generic selectors those requirements need;
4. which platform principal each selector resolves to;
5. what immutable provenance must be captured for a future `AuthorizationRound`;
6. how a published policy or selector-bundle identity is proven **append-only**
   across repository history, not merely internally self-consistent (new in
   this correction — see §0, §5–§6).

F3.1.1 does **not** create an `AuthorizationRound` during `POST /changes` — that
is F3.1.2. This document is architecture/implementation **planning only**; no
ADO source, migration, configuration, test, or runtime behavior was modified to
produce it or this correction.

## Method

This plan was produced by reading the actual F3.1.0 implementation at ADO
`4bad41d`, not by re-deriving requirements from ADR-009 alone. This correction
re-inspected the same baseline (unchanged since the prior checkpoint) plus:

- [`authorization/types.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/authorization/types.ts) — the exact persisted shape of `AuthorizationRound`/`ApprovalRequirement`/`ApprovalDecision`/`AuthorizationAuditEvent`
- [`authorization/canonical.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/authorization/canonical.ts) — the existing canonical-JSON/SHA-256 primitives; confirmed `canonicalJson` throws on `undefined` properties, which bounds how optional rule fields may be hashed
- [`authorization/evaluators.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/authorization/evaluators.ts) — the pure `AuthorizationEvaluation`/`GovernanceEvaluation` derivations
- [`architecture.test.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/architecture.test.ts) — the guards F3.1.1 code must keep passing; confirmed the existing `sources`/`FORBIDDEN_FIELDS` idiom the new semantic guards extend
- [`changeManagement/types.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/types.ts) / [`validation.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/validation.ts) — confirmed neither `ChangeClassification` nor `ChangeRiskLevel` has a runtime enumeration outside `z.enum` literals; F3.1.1a's totality check must supply its own
- root `package.json` / `packages/backend/package.json` / `@backstage/cli` base `tsconfig.json` — confirmed `resolveJsonModule: true`, `allowImportingTsExtensions: true`, node engine `22 || 24`; **confirmed the repository has no CI pipeline definition at `4bad41d`** (no `.github/`, no root `azure-pipelines.yml`, no `.azuredevops/` — the only `azure-pipelines.yml` files present belong to `templates/` scaffolding output, not this repo's own build)
- [`executionPlanValidator.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/executionPlanValidator.ts) — the existing Catalog-ref validation pattern to reuse for selector resolution (F3.1.1b, unchanged by this correction)
- [`idpProvisionerConfig.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/idpProvisioner/idpProvisionerConfig.ts) — the existing `read*Config(config: Config)` pattern to reuse for selector bundle configuration (F3.1.1b, unchanged)

## 0. What changed and why

Three defects found in the `75abd9d` plan, and the corrections applied:

**Defect A — a self-declared digest next to a mutable artifact is not
immutability.** The prior plan checked a policy digest table and a selector
`contentDigest` against the *current* artifact only. This detects "content
changed, digest forgotten" but not "content changed **and** the digest was
updated in the same commit, under the same `key@version`" — i.e. historical
identity reuse, exactly what ADR-009's "published versions are never edited or
reused for materially different content" forbids. **Correction:** an
append-only **publication manifest**, checked by a deterministic validator
against a **previous trusted repository state** (a git ref), not just against
itself — §5, §6.

**Defect B — the hashed artifact did not determine behavior.** The prior
`AuthorizationPolicyVersion` carried `evaluate: (input) => …` as executable
code, while `policyArtifactSha256` covered only the matrix data feeding it — so
`evaluate()`'s implementation could change with the digest unchanged.
**Correction:** the published policy version becomes pure serializable data
(`rules`); one generic, shared `evaluatePolicy(policy, input)` function is
application code, not part of any published version — §1, §2, §12.

**Defect C — fragile job-title source-text scanning.** The prior guard grepped
source files for tokens including `manager`, an ordinary software word.
**Correction:** replaced with guards over domain shape and published-artifact
*content* — forbidden canonical field names, ref-kind checks, no e-mail-shaped
values — never English source tokens. A variable named `manager` must pass —
§17, §26.N.

Two further corrections fall out of A–C:

- **Historical vs. active validation.** An inactive historical policy must not
  fail application startup merely because the *active* selector bundle no
  longer contains a selector only that inactive policy references — §11, §16.
- **Three separated failure domains** — CI/publication, runtime startup,
  submission — previously blended into one failure table — §16.

## 0a. What changed in R2, and why

Three further gaps found in the `d5fc3ff` (R) plan, and the corrections applied.
Each correction is narrow; none reopens the R direction.

**Gap A — first-publication (genesis) semantics were undefined.** R's §6
requires the baseline manifest to be read via
`git show <baseline-ref>:<manifest-path>`, but the trusted implementation
baseline `4bad41d` **predates the manifest**, so that path does not exist at
that ref. The plan never said what happens then. The obvious repair — "file
missing means empty baseline" — would be a **permanent append-only bypass**:
anyone could delete the manifest and re-publish an existing identity against a
synthesized empty baseline. **Correction:** an explicit, narrow, self-closing
**genesis** rule, bound to one compiled-in approved baseline SHA, with generic
missing-file-means-empty behavior forbidden outright — §5a, §6.

**Gap B — shared evaluator semantics can drift, silently reinterpreting history.**
R correctly replaced per-version `evaluate()` closures with published `rules` +
a shared `evaluatePolicy()`. But `evaluatePolicy()` is described as shared,
**unversioned** application code. So an old policy artifact can stay
byte-identical, with an identical `policyArtifactSha256`, while a future change
to the evaluator changes what those same rules *mean*:

```text
2026: policy v1 rules = R, digest = AAA, evaluator semantics = S1
2027: policy v1 rules = R, digest = AAA, evaluator semantics = S2   ← same identity, different meaning
```

**Correction:** an explicit `policyModelVersion` interpretation contract,
included in the artifact digest, dispatched explicitly, and **append-only in its
semantics** — §1, §2a, §12, §13a.

**Gap C — runtime immutability was shallow.** R's registry freezes with
`Object.freeze(policy)`, which leaves every nested structure mutable:
`policy.rules[0].requirements.push(…)` still succeeds. TypeScript `readonly` is
compile-time only and disappears at runtime. **Correction:** a small
`deepFreezeSerializable` utility applied after digest computation, with explicit
nested-mutation tests — §17a, §17b, §26.L.

Two dependent corrections fall out of A–C:

- **Historical retention was over-promised.** R's §1/§8 claim the registry
  "never evicts" a version and that every published version is "always available
  for historical-round replay." That would make application binaries an
  ever-growing archive, and — verified against ADR-009 and the shipped F3.1.0
  schema — **no ADR-009 clause requires executable replay**. Withdrawn and
  replaced with an explicit three-layer retention model — §29, §30.
- **Publication *capability* is not publication *enforcement*.** R already
  deferred CI wiring, but its failure table was headed "CI / publication
  validation," implying a gate that does not exist. Relabelled, with the
  distinction stated explicitly — §16, §21a.

### ADR-009 reconciliation (required before correcting §29)

ADR-009 is **not modified**, and this checkpoint proposes no amendment. The
relevant clauses were read in full and are satisfied by the corrected model:

| ADR-009 clause | What it requires | How the corrected model satisfies it |
|---|---|---|
| *Canonical persisted facts* — persists "the minimum immutable evidence required to **reproduce** authorization, governance, and execution-eligibility outcomes", explicitly including "**immutable effective requirements and their source**" | Evidence sufficient to reproduce outcomes | The F3.1.0 ledger already persists it — verified in ADO `authorization/types.ts`, see §29 |
| *Audit requirements* — "An auditor must be able to **reconstruct**, in order: submission; policy version and input; round; requirements; selector resolution; …" | Reconstruction **from the ledger** | Ledger is self-contained; no re-execution of a historical artifact is implied |
| *Authorization policy and publication* — "Existing rounds remain bound to their recorded versions; a **mutable 'current policy' is never consulted to reinterpret history**" | **Forbids** reinterpretation | Model B never reinterprets: a round's recorded identity/digest/requirements are authoritative. `policyModelVersion` (§13a) additionally forbids reinterpretation by evaluator drift |

**"Reproduce" and "reconstruct" are satisfied by retained evidence, not by
retained executable code.** ADR-009 nowhere requires the runtime to hold every
historical policy artifact forever. R's "never evicts … forever" was therefore a
self-imposed constraint, not an ADR obligation, and is withdrawn in §29.

## 1. Policy model

`PublishedAuthorizationPolicy` is a published, immutable, **serializable data**
artifact — reviewed application code's *data*, not a rules engine, DSL,
BPM/workflow engine, mutable database row, or executable function.

```ts
type PolicyInput = {
  classification: ChangeClassification; // 'normal' | 'emergency'
  risk: ChangeRiskLevel;                // 'low' | 'medium' | 'high'
};

type RequirementDefinition = {
  requirementRole: string;               // stable key, e.g. 'normal-primary'
  kind: ApprovalRequirementKind;         // 'individual' | 'authority' | 'cab'
  phase: ApprovalRequirementPhase;       // 'pre_execution' | 'post_execution'
  mandatory: true;                       // literal — no optional policy requirement in F3 MVP
  selectorKey: string;
  requiredPrincipalType: PrincipalType;  // 'user' | 'authority'
  separationOfDutyKey?: string;
  sla?: {
    durationSeconds: number;
    anchor: 'execution_completion';
  };
};

// One matrix row: a closed (classification, risk) pair to its requirements.
// `match` is a literal pair, never a predicate/expression — see §13.
type PolicyRule = {
  ruleId: string;                        // e.g. 'normal.medium' — becomes matchedRuleProvenance
  match: { classification: ChangeClassification; risk: ChangeRiskLevel };
  requirements: readonly RequirementDefinition[];
};

// The published, hashed artifact. Identity fields are NOT part of the hash
// (see §12); `{ policyModelVersion, rules }` IS the hash (see §12).
type PublishedAuthorizationPolicy = {
  policyModelVersion: 1;                 // interpretation contract — see §2a, §13a
  policyKey: string;                     // identity metadata
  version: string;                       // identity metadata, e.g. '2026-09-02.1'
  provenance: string;                    // publication metadata (reviewing PR/commit)
  rules: readonly PolicyRule[];          // the hashed behavioral artifact
};

type PolicyEvaluationResult = {
  policyKey: string;
  policyVersion: string;
  policyArtifactSha256: string;
  policyProvenance: string;
  matchedRuleProvenance: string;         // = the matched rule's ruleId
  input: PolicyInput;
  policyInputSha256: string;
  requirements: RequirementDefinition[];
};
```

- **policyModelVersion**: the **technical interpretation contract** for the
  artifact's shape and the meaning of its `rules` — distinct from, and never
  conflated with, the business `version` below. See §2a (dispatch), §12
  (hashed), §13a (append-only semantics). **`1` is the only model version in
  F3 MVP.**
- **policyKey**: a stable identifier, e.g. `default-change-authorization`.
- **policyVersion**: an opaque, monotonically-meaningless string, e.g.
  `2026-09-02.1`. No syntax is prescribed beyond "unique per key, never reused
  for different content" — enforced by the publication manifest, not by
  self-declaration (§5, §6).
- **Publication identity**: `policyKey@policyVersion`, unique both in the
  in-memory registry (§7) and in the publication manifest (§5).
- **Provenance**: `policyProvenance` names the reviewing PR/commit that
  published the version; `matchedRuleProvenance` names which `ruleId` of the
  matrix produced the output for a given input (audit trail down to the
  specific rule, not just the version). **Provenance is publication metadata,
  not hashed behavior** — see §12–§13: changing a provenance string must never
  read as a behavior change.
- **Deterministic input**: `PolicyInput` only — see §9.
- **Output**: `RequirementDefinition[]`, pure — see §10.
- **Active/current version selection**: an explicit config pin, never
  "whatever is latest" — see §8.
- **No function is ever part of a published version.** Behavior lives entirely
  in `rules`; the shared `evaluatePolicy` function is ordinary reviewed
  application code, identical for every version — see §2.
- **Validation**: startup-time totality and shape checks — see §16.
- **Startup/load failure behavior**: fail application startup on any
  structural defect — see §16.
- **Version immutability**: enforced mechanically by the append-only
  publication manifest, checked against a previous trusted repository state —
  a genuine correction from the prior "self-declared digest" model — see §5,
  §6.
- **New versions**: added as a new file exporting a new
  `PublishedAuthorizationPolicy`, registered alongside old ones, and appended
  to the publication manifest — never editing an existing version's file or
  manifest entry.
- **Old versions remain interpretable — but only conditionally.** **Corrected
  in R2** (R claimed the registry "never evicts a registered version" and that
  `registry.get` is "always available … for the lifetime of the deployed
  application"; that over-promise is withdrawn — §0a, §29). A historical policy
  is *executably* interpretable only while **all** of:
  1. its policy artifact is still retained/shipped in the build;
  2. its `policyModelVersion` is still supported by the evaluator (§2a);
  3. that model version's semantics are unchanged (§13a).

  Historical *audit* does **not** depend on any of the three: the ledger
  already snapshots the round's policy identity, artifact digest, input,
  matched-rule provenance, effective requirements, and resolved principals
  (§29). Audit is always available; executable re-evaluation is not promised.

### Business version vs. model version

These two axes are independent and must never be conflated:

| Concept | Meaning | Changes when |
|---|---|---|
| `version` (**policy version**) | **business** policy publication identity | the authorization *policy* changes (different requirements for a pair) |
| `policyModelVersion` | **technical** interpretation semantics of the artifact shape and rule meaning | the *contract for interpreting rules* changes |

```text
default-change-authorization@2026-09-02.1   policyModelVersion = 1   ← MVP
default-change-authorization@2026-10-01.1   policyModelVersion = 1   ← business policy change
default-change-authorization@2028-01-01.1   policyModelVersion = 2   ← future semantic-model change
```

A business policy change bumps `version` and keeps `policyModelVersion: 1`. Only
a change to *what the rule shape means* introduces `policyModelVersion: 2`, with
a new rule shape and/or a new evaluator (§13a).

## 2. The generic evaluator

```ts
function evaluatePolicy(
  policy: PublishedAuthorizationPolicy,
  input: PolicyInput,
): PolicyEvaluationResult;
```

One function, shared by every published version. It dispatches on the
interpretation contract (§2a), then, for model version 1:

1. finds the single `PolicyRule` in `policy.rules` whose `match` equals
   `input` (totality guaranteed at load time — §16, §26.C — so this lookup
   never falls through);
2. computes `policyInputSha256 = sha256Canonical(input)`;
3. returns `PolicyEvaluationResult` with `matchedRuleProvenance = rule.ruleId`
   and `requirements = rule.requirements`.

This directly answers §12's original concern: because `rules` — not an
`evaluate` closure — is what a policy version *is*, `policyArtifactSha256`
covering `{ policyModelVersion, rules }` now covers **all** policy-specific
behavior *and* the contract under which it is interpreted. `evaluatePolicy`
itself is identical, reviewed, tested once, and never varies per version.

## 2a. Model-version dispatch (new in R2)

Gap B (§0a): a shared evaluator whose semantics are unversioned can silently
change the meaning of an unchanged, identically-hashed artifact. Dispatch makes
the interpretation contract explicit and fails closed on anything unsupported.

```ts
function evaluatePolicy(
  policy: PublishedAuthorizationPolicy,
  input: PolicyInput,
): PolicyEvaluationResult {
  switch (policy.policyModelVersion) {
    case 1:
      return evaluatePolicyModelV1(policy, input);
    default:
      throw new Error(
        `Unsupported policyModelVersion: ${String(
          (policy as { policyModelVersion: unknown }).policyModelVersion,
        )}`,
      );
  }
}
```

**Bounds — this is deliberately not a framework (review prompt §15):**

- Only **model version 1** exists in F3 MVP. `evaluatePolicyModelV1` is one
  ordinary function.
- **No** plugin registry, **no** dynamic evaluator loading, **no** generic
  schema interpreter, **no** rules engine, **no** arbitrary version adapters.
- One `case`, one function. The sole purpose is historical semantic stability.
- Do **not** implement model v2 in F3.1.1a. It exists only as the documented
  escape hatch in §13a.

Because `policyModelVersion` is the **literal type `1`**, the `default` branch is
unreachable from well-typed code — it is reachable only from a deliberately cast
fixture, which is exactly what test §26.E constructs. The branch is kept
regardless: it is the fail-closed boundary for any artifact that ever reaches the
evaluator without passing the compiler (a future config-loaded or
differently-shipped artifact).

**Runtime enforcement (§16):** an unsupported `policyModelVersion` on the
**active** policy fails application **startup**, not merely the individual
evaluation — the same fail-closed posture as every other structural defect.

## 3. Policy publication / version registry — recommended approach

**Recommended: hybrid — policy matrix as versioned TypeScript data, selector
bindings as validated application configuration** (option C in the ADR-009
framing, applied asymmetrically). Unchanged from the prior plan except that
"TypeScript module" now means a data literal, not a function.

| Concern | Mechanism | Why |
|---|---|---|
| Classification × risk matrix | Versioned TypeScript **data** module per policy version | Environment-independent domain intent; exhaustiveness-checked over the closed `ChangeClassification`/`ChangeRiskLevel` unions; reviewed and diffable like any other code; no parser or schema to maintain |
| Selector → Catalog ref bindings | Validated `changeManagement.authorization` app-config, read via a `readAuthorizationConfig(config: Config)` function following the existing `readAdoProvisionerConfig` pattern | Which Catalog group is `cab-authority` genuinely differs dev vs. production; forcing a code deploy to rebind a group is unnecessary friction, and compiling environment refs into the domain would violate the "no provider/environment leakage" architecture guard |

**Rejected alternatives:**

- **All-TypeScript (option A):** would force a full application deploy to
  rebind a Catalog group per environment, and tempts hardcoding environment
  entity refs into domain code.
- **All-configuration (option B):** the classification/risk matrix would
  become a small policy DSL expressed in YAML/JSON, which ADR-009 explicitly
  rejects ("Policy DSL on day one — Rejected"), and loses TypeScript's
  exhaustiveness checking over the matrix. Note: making the *policy* data
  serializable (§1) is not the same as adopting option B — the matrix stays in
  reviewed TypeScript source, only the *shape* of one version's payload is now
  data instead of a closure. See §13 for why this is not a DSL either.
- **Mutable database admin table:** rejected outright — see §21. No runtime
  policy-editing UI, ever, in F3 MVP.

## 4. F3.1.0 field mapping (unchanged)

Every F3.1.1 output field maps onto an existing F3.1.0 persisted column; see
§19 (renumbered from the prior plan's §19 — content unchanged, reproduced
below for continuity).

| F3.1.1 concept | F3.1.0 persisted field | Source |
|---|---|---|
| `policyKey` | `AuthorizationRound.policyKey` | `authorization/types.ts` |
| `policyVersion` | `AuthorizationRound.policyVersion` | ″ |
| `policyArtifactSha256` | `AuthorizationRound.policyArtifactSha256` | ″ |
| `policyProvenance` | `AuthorizationRound.policyProvenance` | ″ |
| `policyInput` | `AuthorizationRound.policyInput` | ″ |
| `policyInputSha256` | `AuthorizationRound.policyInputSha256` | ″ |
| `matchedRuleProvenance` | `AuthorizationRound.matchedRuleProvenance` | ″ |
| `selectorBundleKey` | `AuthorizationRound.selectorBundleKey` | ″ |
| `selectorBundleVersion` | `AuthorizationRound.selectorBundleVersion` | ″ |
| `selectorBundleSha256`/`contentDigest` | `AuthorizationRound.selectorBundleSha256` | ″ |
| bundle provenance | `AuthorizationRound.selectorBundleProvenance` | ″ |
| `RequirementDefinition.requirementRole` | `ApprovalRequirement.sourceRef` | ″ + migration `.cjs` |
| `RequirementDefinition.kind` | `ApprovalRequirement.kind` (`individual/authority/cab`) | ″ |
| `RequirementDefinition.phase` | `ApprovalRequirement.phase` | ″ |
| `RequirementDefinition.mandatory` | `ApprovalRequirement.mandatory` | ″ |
| resolved principal snapshot | `ApprovalRequirement.principalSnapshot` (`PrincipalResolutionSnapshot`) | ″ |
| `RequirementDefinition.separationOfDutyKey` | `ApprovalRequirement.separationOfDutyKey` | ″ |
| `RequirementDefinition.sla.*` | `ApprovalRequirement.slaPolicyKey/slaPolicyVersion/slaDurationSeconds/slaAnchor` | ″ (`slaPolicyKey`/`slaPolicyVersion` are filled by F3.1.2 from the round's own policy identity — the rule no longer embeds them, see §12) |
| policy provenance (for `sourceProvenance`) | `ApprovalRequirement.sourceProvenance` | ″ |

One mapping nuance carried forward unchanged: the persisted `kind` column has
three values (`individual`/`authority`/`cab`) while `principal_type` has two
(`user`/`authority`). The resolver enforces `kind === 'individual' ⇔
principalType === 'user'` and `kind ∈ {'authority','cab'} ⇔ principalType ===
'authority'` — belongs to F3.1.1b's resolver, asserted by a startup check
across the **active** policy against the **active** bundle only (corrected —
see §11).

## 5. Publication manifest

A small, explicit, checked-in JSON file is the single source of truth for
"what has ever been published, and to what digest":

`packages/backend/src/modules/changeManagement/authorization/policy/published-manifest.json`

```json
{
  "manifestVersion": 1,
  "published": [
    {
      "artifact": "policy",
      "key": "default-change-authorization",
      "version": "2026-09-02.1",
      "digest": "9f1c2e...64-hex-chars"
    }
  ]
}
```

Design choices, each answering a §0/Defect-A concern:

- **`published` is an array, not an object keyed by `key@version`.** An object
  map would let `JSON.parse` silently apply last-write-wins on a duplicate key
  — undermining the very duplicate-identity detection this manifest exists
  for. An array makes duplicate `(artifact, key, version)` triples a detectable,
  hard-failing condition on both the CI validator (§6) and runtime startup
  (§16).
- **Entries carry `digest` only** — no timestamps, no free-text notes. This
  keeps "baseline entry unchanged" strictly equivalent to "digest unchanged";
  there is no ambiguous middle ground where an entry's metadata changed but its
  identity is claimed to be preserved.
- **`artifact: 'policy' | 'selector-bundle'`.** F3.1.1b (not built in this
  slice) appends selector-bundle entries to this **same** manifest and reuses
  the **same** validator — the §18/§19 recommendation in the review prompt:
  one publication-integrity model for both artifacts, not two incompatible
  ones (see §14).
- **JSON, not TypeScript.** It must be readable two ways: by runtime
  TypeScript (`resolveJsonModule: true` is already on in this repo's base
  `tsconfig.json`, confirmed against `4bad41d`) and by a standalone script that
  reads it out of a *different* git ref via `git show <ref>:<path>` (§6) — a
  `.ts` module cannot be evaluated that way without a build step.
- **The manifest itself is a repository file, not an external ledger** — by
  design (§7 of the review prompt: "do not build a general policy publishing
  system"). Its integrity comes entirely from being checked against **git
  history**, not from any property of the file in isolation — see §6.

`digest` for a `policy` entry is `policyArtifactSha256` (§12). `digest` for a
future `selector-bundle` entry (F3.1.1b) is that bundle's `contentDigest`.

**Manifest location — reviewed and kept in R2.** The path above still satisfies
all four required properties: the runtime can read the candidate manifest
(`resolveJsonModule: true`, confirmed at `4bad41d`); the repository validator can
read the baseline manifest via `git show <ref>:<path>`; F3.1.1b reuses the same
file for selector bundles; and no database is involved. No change.

## 5a. Genesis — the first publication (new in R2)

Gap A (§0a): the manifest does not exist at the trusted baseline `4bad41d`, so
the very first F3.1.1a publication has no baseline to compare against.

**Generic "manifest missing ⇒ empty baseline" behavior is forbidden.** It would
let anyone delete the manifest and re-publish an existing identity against a
synthesized empty baseline — defeating the entire append-only mechanism this
manifest exists to provide.

Instead, an empty baseline may be synthesized **only** under an intentionally
narrow, explicitly requested, one-time genesis rule.

### The approved genesis baseline

```ts
/** The one pre-manifest baseline from which a genesis publication is allowed. */
const AUTHORIZED_GENESIS_BASELINE_SHA =
  '4bad41d058edf5c5314d17275e0c8bdb5abf690f';
```

This is a **compiled-in constant in reviewed source**, not caller input.

> If implementation realities require a different direct descendant to become
> the pre-manifest baseline before F3.1.1a lands (e.g. an intervening authorized
> commit), the constant is updated to that exact SHA **in the F3.1.1a
> implementation PR**, as a reviewed one-line change. The invariant is unchanged:
> exactly one approved SHA, at which the manifest is absent.

### The five conditions

An empty baseline is synthesized **only when all five hold**:

1. the caller **explicitly** declared genesis mode (`--allow-genesis-from <sha>`);
2. that supplied `<sha>` equals `AUTHORIZED_GENESIS_BASELINE_SHA`;
3. the resolved `--baseline-ref` full commit SHA **also** equals that constant;
4. the manifest path is **absent** at that trusted ref;
5. the candidate manifest passes **full** structural and duplicate-identity
   validation (§6).

### Why the flag is an acknowledgement, not an authority

Conditions 1 and 2 are deliberately separate. If the caller-supplied flag alone
authorized genesis, an attacker could delete the manifest, pass
`--allow-genesis-from <whatever-sha-they-are-building-on>`, and obtain an empty
baseline — reintroducing exactly the bypass this rule closes. Requiring the flag
to match a **compiled-in constant** *and* the constant to match the **resolved
baseline ref** means the flag can only ever *acknowledge* a genesis that the
reviewed source already authorizes. It can never create one.

### Genesis is never inferred

Genesis is **never** inferred from any of: file-not-found alone; empty repository
state; branch name; current date; application version; `manifestVersion`; or an
environment variable without an explicit trusted baseline. Only the five
conditions above.

### Genesis is one-time and self-closing

Once the manifest exists in the trusted baseline history, the branch becomes
unreachable **by construction**, not by convention: every baseline ref at or
after that commit has the file present, so condition 4 can never hold again.

```text
approved pre-manifest baseline + explicit genesis + file absent  →  baseline = EMPTY
any normal baseline after genesis + file absent                  →  FAILURE
```

`manifest missing → FAIL` is the permanent rule; genesis is the single, narrow,
already-closed exception.

**Retention decision:** the genesis code path, its constant, and its tests are
**retained permanently**. The branch is dead once the manifest lands, but keeping
it preserves security cases B/C/D (§6a) under regression, rather than deleting
the cases along with the code that enforced them, and keeps the validator
self-contained if a future repository split or a new artifact kind ever needs its
own genesis.

## 6. Append-only publication-history validation

Merely checking "current manifest vs. current artifacts" is insufficient (per
§0/Defect-A) — it cannot detect a version's content and digest changing
together. The mechanical check that closes this gap:

```ts
type ManifestEntry = {
  artifact: 'policy' | 'selector-bundle';
  key: string;
  version: string;
  digest: string; // lowercase sha256 hex
};

type PublicationViolation =
  | { code: 'IDENTITY_REMOVED'; entry: ManifestEntry }
  | { code: 'DIGEST_CHANGED'; entry: ManifestEntry; candidateDigest: string }
  | { code: 'DUPLICATE_IDENTITY'; entry: ManifestEntry }
  | { code: 'MALFORMED_MANIFEST'; reason: string }
  // new in R2 — see §5a
  | { code: 'BASELINE_MANIFEST_MISSING'; baselineSha: string }
  | { code: 'GENESIS_NOT_AUTHORIZED'; baselineSha: string; reason: string };

/**
 * Corrected in R2: the baseline is a *resolution*, not a manifest that may or
 * may not exist. Absence is a first-class, explicitly handled input — never an
 * implicit empty default.
 */
type BaselineResolution =
  | { present: true; manifest: { published: ManifestEntry[] } }
  | { present: false };

function validatePublishedManifest(args: {
  baselineSha: string;             // resolved full commit SHA of --baseline-ref
  baseline: BaselineResolution;
  candidate: { published: ManifestEntry[] };
  genesis?: { allowedBaselineSha: string };
}): { ok: true } | { ok: false; violations: PublicationViolation[] };
```

Rules (pure, no I/O — the caller resolves and parses; the function decides):

| Case | Result |
|---|---|
| baseline present, identity present in candidate, same digest | OK |
| baseline present, candidate contains an identity absent from baseline | OK (append) |
| baseline present, baseline identity absent from candidate | **FAIL** `IDENTITY_REMOVED` |
| baseline present, baseline identity present with a different digest | **FAIL** `DIGEST_CHANGED` |
| duplicate `(artifact,key,version)` within either manifest | **FAIL** `DUPLICATE_IDENTITY` |
| malformed manifest (bad `manifestVersion`, non-sha256 digest, missing field) | **FAIL** `MALFORMED_MANIFEST` |
| **baseline absent, no `genesis` supplied** | **FAIL** `BASELINE_MANIFEST_MISSING` |
| **baseline absent, `genesis.allowedBaselineSha` ≠ compiled-in constant** | **FAIL** `GENESIS_NOT_AUTHORIZED` |
| **baseline absent, `genesis` authorized but `baselineSha` ≠ constant** | **FAIL** `GENESIS_NOT_AUTHORIZED` |
| **baseline absent, all five §5a conditions hold** | baseline treated as `{ published: [] }`, then the rules above apply normally |

Two properties of the genesis row, both deliberate:

- **`genesis` is ignored when the baseline is present.** It can only ever relax
  the missing-baseline case; it never weakens append-only validation against a
  baseline that exists. This is what makes case D (§6a) fail regardless of what
  flags the caller passes.
- **Genesis relaxes *only* the missing-baseline case.** The synthesized empty
  baseline still runs the full candidate validation — structural checks and
  duplicate-identity detection both apply (§5a condition 5). A genesis candidate
  **may carry more than one entry**: the constraint is validity, not entry count,
  so F3.1.1b can seed a `selector-bundle` entry the same way without amending
  the validator. Append-only applies strictly from the genesis commit onward.

SHA comparisons are normalized to lowercase and compared as **exact full
40-character commit SHAs** — never abbreviated, never prefix-matched.

**Trusted-baseline sourcing — the part that actually proves append-only
history:** the baseline is never the working tree. A repository command:

```
yarn validate:policy-publication --baseline-ref <git-ref> [--allow-genesis-from <exact-sha>]
```

implemented as `scripts/validate-policy-publication.mjs`:

1. resolves `--baseline-ref` to a **full commit SHA** via
   `git rev-parse <ref>^{commit}` — so a symbolic ref (`origin/main`, a tag,
   `git merge-base origin/main HEAD`) can be compared against the §5a constant;
2. reads the **baseline** manifest via `git show <resolved-sha>:packages/backend/src/modules/changeManagement/authorization/policy/published-manifest.json`,
   producing `{ present: true, manifest }`, or — **only** when git reports the
   path does not exist at that ref — `{ present: false }`. Any other git failure
   (bad ref, not a repository, I/O error) is a hard error, **never** an absent
   baseline;
3. reads the **candidate** manifest from the working tree;
4. calls the pure `validatePublishedManifest` above (loaded via
   `node --experimental-strip-types`, available on node 22.21+/24, already the
   engine range this repo declares — confirmed locally), passing `genesis` only
   when `--allow-genesis-from` was supplied;
5. exits non-zero and prints every violation if `ok: false`.

Step 2's narrowing matters: absence must mean *"this specific path is not in
that tree"*, not *"something went wrong reading it."* Collapsing the two would
turn any transient git error into a synthesized empty baseline.

`--baseline-ref` is **required** — there is no default and no environment
autodetection, per the review prompt's explicit instruction not to hide this
behind vague statements. In PR/CI validation the caller supplies the trusted
target branch or `git merge-base origin/main HEAD`; a developer working
locally supplies an explicit ref (e.g. `origin/main`, a tag, a prior commit).

`--allow-genesis-from` is **optional and, after the first publication,
permanently useless** (§5a). It is never defaulted, never inferred, and never
read from the environment.

## 6a. Genesis security cases (new in R2)

These are contract cases, not merely tests — each is required by §5a and gated
by §26.J.

| # | Scenario | Result |
|---|---|---|
| **A** | approved genesis baseline + manifest missing + explicit genesis flag | **OK** — compare against an empty baseline |
| **B** | approved genesis baseline + manifest missing + **no** genesis flag | **FAIL** `BASELINE_MANIFEST_MISSING` |
| **C** | **unapproved** baseline + manifest missing + genesis flag | **FAIL** `GENESIS_NOT_AUTHORIZED` |
| **D** | normal post-genesis baseline + manifest missing | **FAIL** `BASELINE_MANIFEST_MISSING` — regardless of any flag passed |
| **E** | normal post-genesis baseline + manifest present | normal append-only validation (§6) |
| **F** | candidate first manifest malformed | **FAIL** `MALFORMED_MANIFEST` |
| **G** | candidate first manifest contains duplicate identities | **FAIL** `DUPLICATE_IDENTITY` |

Case C has two distinct sub-cases, both failing, distinguished by the violation's
`reason`: the flag names a SHA that is not the approved constant, or the flag
matches the constant but `--baseline-ref` resolves somewhere else. Cases F and G
prove genesis does **not** relax candidate validation.

**Fallback if `--experimental-strip-types` proves unworkable at
implementation time:** keep `validatePublishedManifest` as a plain `.ts` unit
under jest (already exercised by §26.I tests) and give the `.mjs` script a
small hand-written re-implementation of the same table-driven rules, with a
fixture test asserting the two stay in lockstep. This is a documented
fallback, not the primary plan.

**Explicitly deferred, not built in this checkpoint:** wiring
`validate:policy-publication` into an actual CI pipeline. The repository has
**no pipeline definition at `4bad41d`** (confirmed — no `.github/`, no root
`azure-pipelines.yml`, no `.azuredevops/`); inventing one is out of scope for
F3.1.1a and forbidden by this checkpoint's STOP conditions. The command exists
and is testable standalone; pipeline integration is a follow-up once a
pipeline exists to integrate it into.

## 7. Policy version registry (unchanged mechanics)

```ts
function createPolicyRegistry(
  versions: PublishedAuthorizationPolicy[],
): { get(key: string, version: string): PublishedAuthorizationPolicy } {
  const map = new Map<string, PublishedAuthorizationPolicy>();
  for (const v of versions) {
    const id = `${v.policyKey}@${v.version}`;
    if (map.has(id)) {
      throw new Error(`Duplicate published policy version: ${id}`);
    }
    // R2: was Object.freeze(v) — shallow, see §0a Gap C / §17a.
    map.set(id, deepFreezeSerializable(v));
  }
  return {
    get(key, version) {
      const found = map.get(`${key}@${version}`);
      if (!found) {
        throw new Error(`Unknown published policy version: ${key}@${version}`);
      }
      return found;
    },
  };
}
```

Built once at startup from every version module the application ships (an
explicit array, not directory-scanning — keeps the registry statically
reviewable). Duplicate `key@version` is a **startup** failure (§16), *in
addition to* the separate CI-time duplicate-identity check in the manifest
(§6) — the registry check catches a duplicate among versions actually shipped
in this build; the manifest check catches a duplicate introduced across git
history, including versions no longer shipped.

## 8. Active policy selection (unchanged)

`changeManagement.authorization.activePolicy: { key: string; version: string }`
in app-config names exactly one published version. This is the *only* mutable
runtime input to policy selection, and it selects only for **new**
submissions.

The registry is a deep-frozen (§17a), in-memory map built at startup from every
`PublishedAuthorizationPolicy` the application ships — not filtered to the
active pin, so a shipped non-active version stays retrievable for rollback.

**Corrected in R2.** R continued: *"`registry.get(key, version)` is always
available for any previously-published version."* That is withdrawn — it holds
only for versions this build actually **ships** (§29, §30). ADR-009's clause
*"existing rounds remain bound to their recorded versions; a mutable 'current
policy' is never consulted to reinterpret history"* is satisfied regardless,
because it **forbids reinterpretation** rather than requiring runtime
availability: a historical round's authoritative record is the ledger snapshot
it already carries (§29), never a re-lookup.

F3.1.2 obtains `selectedPolicy` as: `registry.get(activePolicy.key,
activePolicy.version)`, evaluated once at submission time via `evaluatePolicy`
(§2), then embeds the resulting
`policyKey`/`policyVersion`/`policyArtifactSha256`/`policyProvenance` into the
new round. The round's own stored version is what all future reads consult —
never a re-lookup of "current" config.

**Rejected:** "always evaluate the greatest registered version" — this would
mean adding a new policy file silently changes behavior on the next deploy,
and rollback would require *deleting* a published version (which contradicts
immutability, and would also violate the append-only manifest — §6). An
explicit pin makes rollback a one-line config change to a still-registered
older version — see §11 for what rollback must then re-validate.

## 9. Policy input (unchanged)

Exactly two fields: `{ classification, risk }` — both already closed unions on
the canonical `Change` type. Nothing else is a policy input:

- **Not** `targetRef`, `ownerRef`, `systemRef`, `requestedWindow`,
  `executionPlan`, `rollbackPlan`, or `evidence` — the policy does not need
  them to select requirements under the accepted MVP matrix.
- **Not** provider, Azure DevOps, Teams, pipeline IDs, or repository IDs —
  these must never appear in policy input; the existing
  `architecture.test.ts` `FORBIDDEN_FIELDS` guard already forbids
  `repositoryId`, `pipelineId`, `buildId`, `deploymentId`, `environmentId`,
  `adoApprovalId`, `teamsUserId` and must be extended to cover the new
  `policy/` source directory (§16, §26.O).

The full Change snapshot is independently hashed into
`AuthorizationRound.changeSnapshotSha256` already — policy input fingerprinting
is a narrower, separate concern: `policyInputSha256 = sha256Canonical(input)`,
reusing [`canonical.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/authorization/canonical.ts)
verbatim. **No second canonicalization scheme is introduced.**

## 10. Policy output (unchanged)

The policy does **not** persist `ApprovalRequirement` rows. It returns
`RequirementDefinition[]` (§1) — pure intent, no round-local identity.

F3.1.2 turns each `RequirementDefinition` plus its resolved
`PrincipalResolutionSnapshot` (F3.1.1b) into a persisted `ApprovalRequirement`
by adding exactly the fields that are round-local, not policy-derived:
`requirementId` (generated), `createdAt` (server time), `source: 'policy'`,
`sourceRef: requirementRole`, `sourceProvenance: policyProvenance`,
`principalSnapshot` (from selector resolution), and `decision` (initially
absent) — plus, per the §12 correction below, `slaPolicyKey`/`slaPolicyVersion`
are filled from the round's own policy identity rather than read off the rule.

Deliberately **not** added to `RequirementDefinition`: any field that only
exists once a round is created.

## 11. Active vs. historical policy validation

**Corrected from the prior plan's blanket rule.** The prior plan asserted
"every registered policy version references only selectors present in the
active bundle," validated at startup for *all* registered versions. This
conflated two different needs:

- **A. Needing old policy identity/content for audit** — always true, and
  satisfied by the **ledger's** self-contained round evidence (§29), *not* — as
  R claimed — by the registry never evicting a version (corrected in R2, §29,
  §30).
- **B. Needing every old policy to participate in startup validation against
  the *current* active selector bundle** — not true, and actively harmful: it
  means an inactive historical policy can crash application startup solely
  because the *currently active* selector bundle dropped a selector that only
  that retired policy ever referenced.

**Corrected rule:** only the pair *(activePolicy, activeSelectorBundle)* must
be validated as complete and compatible at startup — every `selectorKey` the
active policy's matched rules reference must exist in the active bundle, and
`kind ⇔ principalType` must hold for each. A historical, non-active policy is
loaded into the registry (for §8's audit/replay guarantee) but is **not**
checked against the active bundle.

**Rollback consequence:** if `activePolicy` is rolled back to an older,
still-registered version, that version becomes the active pair member and
**must then** pass the same active-pair validation against whichever selector
bundle is active for new submissions at that time. Rollback is not exempt from
this check — it simply means the check runs against the newly-active pair, not
against every historical version simultaneously.

This directly answers review prompt §16's example: "inactive historical policy
references selector absent from active bundle → **NOT** a startup failure."

## 12. Policy artifact hash — exact semantics

**Corrected in R2** (Gap B, §0a). The digest now binds the interpretation
contract alongside the behavior it governs:

```ts
policyArtifactSha256 = sha256Canonical({
  policyModelVersion: policy.policyModelVersion,
  rules: policy.rules,
});
```

| Field | Hashed? | Reason |
|---|---|---|
| `policy.policyModelVersion` | **Yes** — new in R2 | The interpretation contract contributes to policy-specific behavior: identical `rules` under a different model version may mean something different (§0a Gap B, §2a, §13a) |
| `policy.rules` | **Yes** | The entire behavioral surface: which requirements a given `{classification, risk}` produces (§1, §2) |
| `policy.policyKey` | No — identity metadata | Renaming would not change behavior; identity/digest binding is the manifest's job (§5), not the digest's |
| `policy.version` | No — identity metadata | Same reasoning; also avoids a self-referential hash (the version string naming itself) |
| `policy.provenance` | No — publication metadata | A corrected PR link or commit citation must never read as a behavior change — see §13 |

Field order in the hashed literal is irrelevant: `canonicalJson` sorts keys
(verified in `authorization/canonical.ts` at `4bad41d`).

**Migration impact: none.** This changes the digest relative to R, but **nothing
has been published yet** — no manifest entry exists to revise. F3.1.1a simply
computes the seeded entry under the corrected formula. Had a version already been
published, changing the hash formula would have required a new policy version,
not an edited digest (§5, §6).

Corrections from the prior model, both closing gaps the review prompt raised:

- **No function serialization, ever.** Because `rules` is plain data (§1),
  there is no `evaluate` closure to (mis)represent in a hash. `evaluatePolicy`
  (§2) is shared, unversioned, ordinary application code — its own
  correctness is covered by regular unit tests (§26.D), not by this digest.
- **`sla.policyKey`/`sla.policyVersion` removed from `RequirementDefinition`
  (compare §1 here to the prior plan's §1).** The prior shape embedded the
  policy's own identity inside the hashed artifact — self-referential, and
  meant renaming a version changed `policyArtifactSha256` even though no rule
  changed. F3.1.2 fills the persisted `sla_policy_key`/`sla_policy_version`
  columns from the round's already-known policy identity instead.
- No filesystem timestamps, no runtime object identity, no deployment-specific
  values enter the hash — only `rules`.
- `canonicalJson`/`sha256Canonical` from `authorization/canonical.ts` reused
  verbatim (§14) — no second canonicalization algorithm, and no hashing of
  `undefined` fields (confirmed `canonicalJson` throws on those — the rule
  literal must omit rather than set-to-undefined any absent optional).
- **Deep-freezing does not change the digest** — see §17b for the verified
  argument and §26.M for the test.

## 13. Policy rule model is not a DSL

`match: { classification, risk }` is a **closed literal pair**, not a
predicate, expression, or condition string. `PolicyRule`/`PublishedAuthorizationPolicy`
express exactly the F3 MVP's two closed dimensions and nothing else:

- No boolean expression trees.
- No arbitrary predicates or scripting.
- No CEL, JSONLogic, CUE, Rego, or condition strings.
- No generic operators.
- No plugin rule types.

A small typed matrix — 6 total rows, one per `(classification, risk)` pair —
is data, not a DSL, exactly as ADR-009 permits ("deterministic reviewed
application code/configuration") while explicitly rejecting a generic policy
DSL. **Totality is required and enforced at load time (§16, §26.C): every
valid pair resolves to exactly one rule. No default fallthrough, no permissive
fallback.** Because `ChangeClassification`/`ChangeRiskLevel` have no runtime
enumeration elsewhere in the codebase (confirmed against `validation.ts`), the
policy slice defines its own `as const` tuples for both and a compile-time
bind (e.g. a mapped-type exhaustiveness check) so that adding a future union
member fails the build until a new rule is added — not merely a runtime
assertion.

## 13a. Model-version semantics are append-only (new in R2)

**Invariant: `policyModelVersion = 1` must keep the same interpretation
forever.**

This is the counterpart to publication immutability. The manifest (§5, §6) makes
a published *artifact* immutable; this invariant makes the *contract for
interpreting* that artifact immutable. Without both, an unchanged artifact with
an unchanged digest can still change meaning (§0a Gap B).

Consequences:

- **V1 semantics are never edited in place.** Not to fix a perceived
  infelicity, not to add a field's meaning, not to change how a rule matches.
- **When semantics must change, introduce `policyModelVersion = 2`** with its own
  rule shape and/or its own `evaluatePolicyModelV2`, dispatched by §2a. Existing
  V1 artifacts keep resolving to `evaluatePolicyModelV1`, unchanged.
- **The registry/runtime must never silently reinterpret `policyModelVersion: 1`
  through changed semantics** — an unsupported model version fails closed (§2a,
  §16) rather than falling back to the nearest implementation.
- Ordinary bug fixes to V1 that are provably semantics-preserving (a performance
  change, a clearer error message) remain allowed. Anything that could change an
  output for any input is a new model version, not a fix.

**Scope of enforcement.** This is an **application-code architecture
invariant**. It does **not** require a second publication manifest for evaluator
code in F3 MVP, and none is proposed. It is protected by:

- the V1 behavioral test matrix (§26.A–D), which pins V1 outputs for all six
  pairs and fails if any V1 output ever changes;
- the semantic architecture guards (§17);
- architecture review of any PR touching `evaluatePolicyModelV1`.

If a future slice finds those insufficient, versioning the evaluator artifact
itself is the escalation — deliberately **not** taken now, per the "do not build
a framework" bound (§2a).

## 14. Hashing / canonicalization (unchanged mechanism, corrected inputs)

No second canonicalization scheme. Every hash in F3.1.1 reuses
[`canonical.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/authorization/canonical.ts)
verbatim (`canonicalJson`, `sha256Canonical`, `assertSha256`):

| Value | Computed as |
|---|---|
| `policyArtifactSha256` | `sha256Canonical({ policyModelVersion, rules })` — see §12 (R2: now binds the interpretation contract as well as the rule data) |
| `policyInputSha256` | `sha256Canonical({ classification, risk })` |
| `selectorBundleSha256` / bundle `contentDigest` (F3.1.1b) | `sha256Canonical(bundle.selectors)` |
| `selectorDigest` (per requirement, F3.1.1b) | `sha256Canonical({ selectorKey, selectorVersion, principalType, principalRef })` |
| manifest entry `digest` (§5) | = `policyArtifactSha256` for a `policy` entry; = `contentDigest` for a `selector-bundle` entry |

## 15. Selector model, versioning, principal types, Catalog boundary, resolution vs. membership, emergency A/B, CAB, retrospective SLA, additional requirements

**Unchanged by this correction** — these sections of the prior plan
(previously §7–§15) contained no defect identified by review. They are
preserved verbatim in content; only cross-references were renumbered to match
this document, and one field removal cascades in from §12
(`sla.policyKey`/`sla.policyVersion` no longer appear on
`RequirementDefinition` — see §1 and §12). Summarized for continuity:

- **Selectors** remain generic configuration identities
  (`normal-primary-approver`, `emergency-approver-a`, `emergency-approver-b`,
  `cab-authority`); no corporate titles, employee names, or e-mail addresses.
- **Selector bundle versioning** stays one immutable bundle per deploy-time
  unit, not per-selector versions; per-entry `selectorDigest` still proves what
  one selector resolved to.
- **Principal types**: `user` → Catalog `User`; `authority` → Catalog `Group`.
  No e-mail ever snapshotted; an authority is never exploded into members.
- **Catalog integration boundary**: `parseEntityRef` + `catalog.getEntityByRef`
  via `auth.getOwnServiceCredentials()` (service credentials, not the
  requester's); startup validates syntax/declared-kind, submission validates
  actual existence/kind, fail-closed on both Catalog-unavailable (`503`,
  retryable) and missing/wrong-kind (`404`/`500`).
- **Resolution vs. membership**: F3.1.1 resolves `cab-authority` to a Catalog
  ref; whether the actual decision actor may act for that ref is decision-time
  membership, explicitly out of scope (F3.1.3/F3.1.4).
- **Emergency A/B**: both selectors restricted to `user`-typed principals for
  F3 MVP, distinct selector keys (startup check), distinct resolved refs
  (submission check), shared `separationOfDutyKey`; authority-typed A/B
  remains an open question for architecture review (§32), not resolved here.
- **CAB**: one collective authority requirement, `kind: 'cab'`, no quorum,
  voting, attendance, or per-member decisions.
- **Retrospective SLA**: `durationSeconds`/`anchor` carried on the
  `RequirementDefinition.sla` field of the rule (§1); changing the duration
  requires a new policy version, which is now additionally enforced by the
  publication manifest (§5, §6) — a duration change with no version bump is a
  `DIGEST_CHANGED` CI failure, not merely a "should" in prose.
- **Additional mandatory requirements**: compatibility-only, not implemented in
  F3.1.1; `RequirementDefinition` shape is shared with policy-generated
  requirements, additive-only downstream.

## 16. Failure model — three separated domains

**Corrected.** The prior plan's single "prevents startup / rejects submission"
table did not have a publication row at all, since the append-only manifest did
not exist. The failure model now has three explicitly separated domains, each
owned by a different actor at a different time.

### Publication-integrity validation (command-invoked — `yarn validate:policy-publication`, §6)

**Relabelled in R2.** This domain was headed "CI / publication validation,"
which implied a gate that does not exist. The command must be *invoked* to
fail anything — see §21a for the capability-vs-enforcement distinction.

| Failure | Result |
|---|---|
| Commit changes an existing published policy digest | **FAIL** — `DIGEST_CHANGED` |
| Commit removes a previously published version | **FAIL** — `IDENTITY_REMOVED` |
| Commit duplicates an existing `(artifact,key,version)` | **FAIL** — `DUPLICATE_IDENTITY` |
| Commit appends a new, previously-unseen version | **OK** |
| Manifest malformed | **FAIL** — `MALFORMED_MANIFEST` |
| **Baseline manifest missing, no authorized genesis** (R2) | **FAIL** — `BASELINE_MANIFEST_MISSING` |
| **Genesis requested from an unauthorized baseline** (R2) | **FAIL** — `GENESIS_NOT_AUTHORIZED` |
| **Genesis authorized per all five §5a conditions** (R2) | **OK** — validate against an empty baseline |

### Runtime startup validation (pure, no I/O)

| Failure | Result |
|---|---|
| No active policy configured / unknown `activePolicy` pin | Startup fails |
| **Active policy pin names a version not shipped in this build** (R2) | Startup fails — §29 |
| Duplicate `policyKey@version` registered in this build | Startup fails |
| Active policy matrix not total over classification×risk | Startup fails |
| **Unsupported `policyModelVersion` on a shipped policy** (R2) | Startup fails — fails closed, §2a, §13a |
| Active `policyArtifactSha256` (computed from shipped `{policyModelVersion, rules}`) disagrees with the manifest's entry for that `key@version` | Startup fails — content/manifest drift within one deploy |
| **A shipped policy has no manifest entry at all** (R2) | Startup fails — §28a |
| **A manifest entry has no shipped policy module** (R2) | **Startup succeeds** — publication history, not a runtime obligation, §28a, §29 |
| **Nested mutation attempted on a registered artifact** (R2) | Impossible / throws — §17a, §26.L |
| Active policy references a `selectorKey` absent from the **active** selector bundle | Startup fails (§11) |
| **Inactive historical** policy references a selector absent from the active bundle | **Startup succeeds** — corrected, see §11 |
| Duplicate selector key in active bundle (F3.1.1b) | Startup fails |
| Active selector bundle `contentDigest` mismatch (F3.1.1b) | Startup fails |
| Emergency A/B not both `user`-typed, or same selector key | Startup fails |
| Malformed post-execution SLA config (missing duration/anchor when `sla` present) | Startup fails |

### Submission validation (I/O, F3.1.2+ — not built in F3.1.1)

| Failure | Result |
|---|---|
| Missing Catalog User | Rejects `NOT_FOUND` (404) |
| Missing Catalog Group | Rejects `NOT_FOUND` (404) |
| Wrong principal entity kind at Catalog (drifted since startup) | Rejects `INTERNAL_ERROR` (500) |
| Catalog temporarily unavailable | Rejects `PROVIDER_UNAVAILABLE` (503, retryable) |
| Emergency A/B resolve to the same individual | Rejects `INTERNAL_ERROR` (500) |

No `default-approve`, `no-selector-means-skip`, or `catalog-failed-continue`
path exists anywhere in this design — every branch above either fails CI,
prevents startup, or rejects the specific submission; nothing silently
degrades.

## 17. Semantic architecture guards (corrected)

**The prior plan's guard — grepping source text for
`cto|director|manager|superintendent` — is removed.** It made architecture
correctness depend on arbitrary English words appearing anywhere in source
files; `manager` in particular is an ordinary software term (e.g. a
"connection manager" local variable) with no relation to corporate-title
semantics.

**Replacement: guards over domain shape and published-artifact content**,
extending the existing `architecture.test.ts` `sources`/`FORBIDDEN_FIELDS`
idiom (confirmed against `4bad41d`) rather than inventing a new mechanism:

- **Forbidden canonical field names** in the new `policy/` (and, later,
  selector) source files — extend `FORBIDDEN_FIELDS` with: `email`,
  `emailAddress`, `jobTitle`, `employeeName`, and any field using
  `displayName` as an identity key — in addition to the existing
  `repositoryId`/`pipelineId`/`buildId`/`deploymentId`/`environmentId`/
  `adoApprovalId`/`teamsUserId`/`sharePointItemId`/`jiraIssueKey`/
  `serviceNowSysId`.
- **Published artifact *data* assertions**, not source-text greps: every
  `selectorKey` string in the shipped policy's `rules` is a generic identifier
  (matches a selector-key naming convention, e.g. `^[a-z][a-z0-9-]*$`), and no
  string value anywhere in the published artifact matches an e-mail shape
  (`/\S+@\S+\.\S+/`). This asserts against the actual artifact object the
  application loads, not against arbitrary comments or variable names in
  source.
- (F3.1.1b, not built in this slice) principal refs parse as valid Catalog
  refs; `individual` ⇔ `User` ref; `authority`/`cab` ⇔ `Group` ref.
- **Explicit acceptance criterion:** a local variable, parameter, or comment
  containing the word `manager` (or `director`, `cto`, etc.) anywhere in any
  guarded source file must **not** fail any guard. This is a required test
  case in §26.N, not merely an assumption.

## 17a. Runtime deep immutability (new in R2)

Gap C (§0a): `Object.freeze(policy)` freezes only the top-level object.
Everything reachable below it stays mutable:

```ts
policy.policyKey = 'x';                       // blocked by Object.freeze
policy.rules.push(extraRule);                 // NOT blocked
policy.rules[0].match.risk = 'low';           // NOT blocked
policy.rules[0].requirements.push(extraReq);  // NOT blocked
policy.rules[0].requirements[0].sla.durationSeconds = 1; // NOT blocked
```

TypeScript `readonly` does not help: it is erased at compile time and constrains
only well-typed call sites. A *published immutable artifact* must be immutable at
runtime, against any caller.

**Recommended mechanism — one small reusable utility** (direction A of the review
prompt's §17: a validated deep-freeze, not immutable reconstruction, which would
allocate a parallel object graph for no added guarantee):

```ts
// packages/backend/src/modules/changeManagement/authorization/immutable.ts
export function deepFreezeSerializable<T>(value: T): T;
```

Properties:

- recursively freezes plain objects **and** arrays, bottom-up, returning the
  same reference;
- accepts **exactly** `canonicalJson`'s value domain — `string`, finite
  `number`, `boolean`, `null`, arrays, and plain objects (prototype
  `Object.prototype` or `null`) — and **throws** on anything else: functions,
  class instances, `Map`/`Set`/`Date`, and `undefined`;
- tracks visited nodes in a `WeakSet`, so it is cycle-safe and terminates.

**Placement — `authorization/immutable.ts`, beside `canonical.ts`, not inside
`policy/`.** F3.1.1b's selector bundles need the same guarantee; putting the
utility next to the canonicalization primitive it mirrors means the second
artifact kind reuses it instead of growing a second, subtly different freeze.

**Why the value domain is deliberately identical to `canonicalJson`'s.** The
same objects are both hashed and frozen. Sharing one domain makes "freezable"
and "hashable" the same invariant, so an artifact can never be publishable but
not freezable (or vice versa), and the utility doubles as a serializability
assertion at load time. Verified against `authorization/canonical.ts` at
`4bad41d`: it rejects non-plain prototypes and `undefined` properties, and
accepts only finite numbers.

**A `WeakSet`, not an `Object.isFrozen` short-circuit.** Short-circuiting on
`isFrozen` would wrongly skip a subtree whose root happens to be shallow-frozen
already — precisely the bug this section exists to fix.

**Consequence for the published artifact module (§24):** it exports a **plain
literal**, no longer an `Object.freeze`d one. A pre-frozen top level would be a
second, shallower freeze mechanism sitting in front of the real one, and would
be exactly the subtree the `isFrozen` trap above describes.

### Strict-mode semantics

Backstage backend sources compile to ES modules, which are always strict, so an
assignment to a frozen property **throws `TypeError`** rather than failing
silently. `Array.prototype.push` on a frozen array throws regardless of
strictness. §26.L therefore asserts **both** that the mutation throws **and**
that the value is unchanged — the second assertion is a semantics-independent
backstop if emitted test code is ever non-strict.

## 17b. Freeze ordering and digest invariance (new in R2)

**Required order:**

```text
load policy data
  ↓
validate shape / totality / model version   (§16)
  ↓
compute and check digest against manifest   (§12, §28a)
  ↓
deep freeze                                 (§17a)
  ↓
register                                    (§7)
```

Freezing happens **inside `createPolicyRegistry`**, after validation and digest
verification, replacing today's `Object.freeze(v)` (§7). The artifact is never
mutated after its digest is computed — there is no normalization, defaulting, or
back-filling step between digest and freeze.

Validation runs **before** the freeze, not after: freezing first would add no
guarantee (validation is read-only) and would only complicate any future
validator that wanted to normalize.

### Deep-freezing cannot change the digest

This is a **verified property**, not an assumption. Read against
`authorization/canonical.ts` at `4bad41d`, `canonicalJson` depends on exactly
four things, and `Object.freeze` changes none of them:

| `canonicalJson` depends on | Effect of `Object.freeze` |
|---|---|
| `typeof` / `value === null` | unchanged |
| `Array.isArray(value)` | unchanged — freezing an array leaves it an array |
| `Object.getPrototypeOf(value)` (must be `Object.prototype` or `null`) | unchanged — freezing does not alter the prototype |
| `Object.keys(record).sort()` and each property's value | unchanged — freezing alters neither enumerability nor values |

Therefore `digest(before freeze) === digest(after freeze)`, deterministically.
Pinned by test §26.M.

## 18. Selector bundle publication integrity (F3.1.1b direction — not built here)

**Corrected statement, direction only — F3.1.1b remains unimplemented in this
checkpoint.** A selector bundle's self-declared `version` + `contentDigest`
(§15) proves **content integrity of the currently loaded artifact only**. Per
§0/Defect-A, it does **not** by itself prove historical append-only
publication — the same swap-attack (same `key@version`, content and digest
changed together) applies equally to selector bundles.

**Recommendation: option A — reuse the identical publication-manifest +
append-only-validator mechanism from §5–§6 for selector bundles**, rather than
inventing a second, incompatible model (option B). A future `selector-bundle`
entry in the same `published-manifest.json` (§5) is validated by the same
`validatePublishedManifest` (§6) against the same trusted-baseline-ref command.
This is the smallest change that closes the same gap for both artifact kinds
and avoids two publication philosophies existing side by side.

## 19. Environment-specific selector bindings, reconciled with immutable identities

**Corrected direction, F3.1.1b scope.** The prior plan correctly identified
that selector principal bindings legitimately differ dev vs. production, but
did not reconcile that with immutable published identities. Correction:

- Each environment's selector bundle gets its **own explicit published
  identity** — e.g. `selector-bundle-dev@2026-09-02.1` and
  `selector-bundle-prod@2026-09-02.1` — each with its own manifest entry (§5)
  and its own append-only history (§6, §18).
- Config selects which bundle **key and version** is active per deployment; it
  never mutates an existing published identity's content.
- `environment = prod` is not encoded as authorization-domain meaning anywhere
  — it exists only as a naming convention within an otherwise-opaque bundle
  key, exactly as ADR-009's "no provider/environment leakage" guard requires.

## 20. No database by default (unchanged)

**Confirmed: F3.1.1 adds no database tables, no migration.** The F3.1.0 schema
(§4 mapping) already carries every field F3.1.1's output produces. Policy
versions and the selector bundle are frozen in-memory artifacts assembled at
startup from code, config, and the publication manifest; the ledger already
exists to snapshot the resulting *historical evidence* once a round is created
(F3.1.2). A mutable policy database, an artifact registry, or a runtime
publishing API would reintroduce exactly the "second policy authority"
ADR-009 rejects, and are explicitly not built here (per the review prompt's
§7: keep this small).

## 21. Operational model, configuration ownership, observability, security analysis, persistence decision

**Unchanged in substance from the prior plan** (previously §21–§25),
cross-referenced to the corrected sections above:

- **Operational model**: policy/selector change → code or config review →
  merge/deploy → registry rebuilt at next startup (validated per §11, §16) →
  new submissions may select the new version only if `activePolicy` is also
  repointed → existing rounds remain bound to their recorded version,
  untouched. Rollback is a config-only `activePolicy` change to a
  still-registered version, subject to §11's active-pair re-validation.
- **Configuration ownership**: Platform/DevOps authors policy/selector
  content; governance authority reviews and approves publication via required
  PR review (a process control, not an application permission in F3.1.1). No
  corporate title is encoded anywhere in code or configuration — enforced by
  the corrected guard in §17, not the removed job-title regex.
- **Observability**: same structured, low-cardinality event names as the prior
  plan (`change.authorization.policy.selected`,
  `change.authorization.selector.resolved`,
  `change.authorization.selector.resolution_failed`,
  `change.authorization.publication.invalid`,
  `change.authorization.policy.evaluation_failed`,
  `change.authorization.selector.entity_missing`,
  `change.authorization.separation_of_duty.violation`); add
  `change.authorization.publication.history_violation` — `{ artifact,
  key, version, code }` — emitted by the CI validator (§6), not at runtime.
  Never logged: e-mail addresses, employee names, job titles.
- **Security analysis**: version reuse (same `key@version`, different
  content) is now mitigated by the append-only manifest validator (§6) for
  **both** policy and (eventually) selector bundles, replacing the prior
  plan's "test-time digest for policy / startup digest for selectors" split,
  which Defect A showed was insufficient on its own. All other threats/
  mitigations from the prior plan (config tampering, spoofed Catalog refs,
  authority confusion, malformed policy, accidental permissive fallback)
  are unchanged.
- **Persistence decision**: no new database tables, routes, or UI — confirmed
  against §20 and the §4 field mapping.

## 21a. Publication capability vs. publication enforcement (new in R2)

The review found R's wording could be read as promising an automatic gate. It
does not exist. The distinction is now explicit and must be preserved in any
downstream summary:

| Statement | True after F3.1.1a? |
|---|---|
| "A publication-integrity **mechanism exists**" | **Yes** — manifest (§5), genesis rule (§5a), pure validator (§6), repository command, tests |
| "The publication **gate is mandatory in CI**" | **No** |

**Why not:** the repository has **no CI pipeline definition at `4bad41d`** —
re-verified for R2 via `git ls-tree -r HEAD`: no `.github/`, no `.azuredevops/`,
no root `azure-pipelines.yml`; root `scripts/` contains only
`build-showcase-techdocs.sh`. The only `azure-pipelines.yml` files present belong
to `templates/` scaffolding output, not this repository's own build.

Append-only publication becomes **enforced** only once a pipeline or branch
policy actually invokes `yarn validate:policy-publication --baseline-ref …`.
Until then it is a capability an author or reviewer runs deliberately.

**Creating that pipeline is out of scope for F3.1.1a** and is forbidden by this
checkpoint's STOP conditions. It is a follow-up, separately authorized, once a
pipeline exists to integrate into.

## 22. Performance / caching (unchanged)

Policy evaluation is pure/in-memory — no caching needed or beneficial.
Selector bundle lookup is an in-memory map read. Catalog validation involves
real backend service calls; no caching is introduced in F3.1.1 — caching is
precisely the mechanism by which a stale principal could leak into an
otherwise-immutable round snapshot.

## 23. F3.1.1 decomposition — re-evaluated, unchanged

The two-slice decomposition remains correct after these corrections, and R2
does not reopen it. The manifest and validator (§5, §6) are new work but have no
independent reviewable I/O boundary of their own — a manifest is meaningless
without the policy artifact it hashes, so it belongs inside F3.1.1a, not a third
slice.

**R2 re-check:** none of the three corrections adds an I/O boundary. Genesis
(§5a) is a rule inside the existing validator; `policyModelVersion` (§2a) is a
field plus a `switch`; deep freeze (§17a) is a pure in-memory utility. The slice
grows slightly; it does not split. Neither slice integrates with
`POST /changes` — **F3.1.2** performs submission integration.

| Slice | Scope | I/O boundary |
|---|---|---|
| **F3.1.1a — Policy domain + registry + publication integrity** | `PolicyInput`, `RequirementDefinition`, `PolicyRule`, `PublishedAuthorizationPolicy` (with `policyModelVersion: 1`), `PolicyEvaluationResult` types; the published `default-change-authorization@2026-09-02.1` data artifact implementing the ADR-009 matrix; `evaluatePolicy` + model-version dispatch (§2a); `createPolicyRegistry`; `deepFreezeSerializable` (§17a); `policyArtifactDigest` over `{policyModelVersion, rules}`; `published-manifest.json`; `validatePublishedManifest` incl. genesis (§5a); `scripts/validate-policy-publication.mjs`; corrected semantic architecture guards | None at runtime — pure, no config read, no Catalog, no DB. The manifest validator reads git history via a script, not via the application |
| **F3.1.1b — Selector bundle + resolver + selector publication integrity** | `SelectorBundle`/`SelectorEntry` types; `readAuthorizationConfig(config)` reader; startup validation (digest, coverage against the **active** policy only — §11, A/B individual-kind, duplicate keys); Catalog-backed resolver producing `PrincipalResolutionSnapshot`; environment-scoped bundle identities (§19); selector-bundle entries appended to the same manifest (§18); plugin wiring to add `coreServices.auth` | Config + Catalog (via `auth.getOwnServiceCredentials()`) |

Both slices remain fully unwired from `POST /changes`; F3.1.2 is the only
future slice that calls either of them from `ChangeManagementService`.

## 24. First implementation slice — F3.1.1a (revised, still NOT authorized)

**This section describes the smallest next authorized-implementation
candidate. It is not authorized by this planning checkpoint. A separate,
constrained implementation prompt must explicitly authorize it before any
code is written.**

### Objective

Implement a pure, deterministic, I/O-free authorization policy evaluator, its
published-version registry, and its append-only publication-integrity
validator, matching the ADR-009 "Normal-change policy baseline" and
"Emergency policy baseline" matrices exactly, with full unit test coverage and
zero integration into the F2 submission path.

### Exact code boundaries

New directory:
`packages/backend/src/modules/changeManagement/authorization/policy/`

New files:

- `types.ts` — `PolicyInput`, `RequirementDefinition`, `PolicyRule`,
  `PublishedAuthorizationPolicy` (incl. `policyModelVersion: 1`),
  `PolicyEvaluationResult` (§1).
- `published/default-change-authorization.2026-09-02.1.ts` — one exported
  `PublishedAuthorizationPolicy` **data** artifact implementing the six-row
  matrix (§1, §26.A), carrying `policyModelVersion: 1`, **a plain literal — no
  `Object.freeze`** (the registry deep-freezes it, §17a), no functions.
- `evaluatePolicy.ts` — the generic evaluator with explicit model-version
  dispatch (§2, §2a): `evaluatePolicy` + `evaluatePolicyModelV1`, plus
  `policyInputSha256` computation via `canonical.ts`.
- `registry.ts` — `createPolicyRegistry` (§7), deep-freezing at registration
  in the §17b order.
- `policyArtifactDigest.ts` — `sha256Canonical({ policyModelVersion, rules })`
  (§12).
- `published-manifest.json` — the append-only manifest (§5), seeded with the
  one policy entry above.
- `publication/validateManifestHistory.ts` — the pure
  `validatePublishedManifest` (§6), including `BaselineResolution`, the genesis
  rule, and `AUTHORIZED_GENESIS_BASELINE_SHA` (§5a).
- `scripts/validate-policy-publication.mjs` (repo-root `scripts/`, matching
  the existing convention alongside `build-showcase-techdocs.sh`) — the
  `--baseline-ref` / `--allow-genesis-from` CLI wrapper (§6).

**One new file outside `policy/`:**

- `authorization/immutable.ts` — `deepFreezeSerializable` (§17a), placed beside
  `canonical.ts` so F3.1.1b's selector bundles reuse it.
- Root `package.json` — one new script:
  `"validate:policy-publication": "node scripts/validate-policy-publication.mjs"`.

Test files (co-located, matching existing repo convention):

- `published/default-change-authorization.2026-09-02.1.test.ts` — matrix
  coverage (§26.A), determinism (§26.B), totality (§26.C).
- `registry.test.ts` — duplicate detection, unknown-version lookup (§26.K), and
  deep runtime immutability after registration (§26.L).
- `policyArtifactDigest.test.ts` — hash stability, identity-metadata exclusion
  (§26.G, §26.H), `policyModelVersion` inclusion (§26.F), freeze/digest
  invariance (§26.M).
- `evaluatePolicy.test.ts` — generic-evaluator behavior across versions
  (§26.D) and fail-closed unsupported `policyModelVersion` (§26.E).
- `publication/validateManifestHistory.test.ts` — the full table in §6
  (§26.I) **and all seven genesis security cases** (§6a, §26.J).
- `../immutable.test.ts` — `deepFreezeSerializable` unit behavior, including
  its rejected value domain (§26.L).

### Expected types/interfaces

Exactly the types in §1, §2, §2a, §5, §5a, §6, §7, and §17a of this plan — no
additions beyond what's specified there.

### R2 scope additions (relative to the R slice)

- `policyModelVersion: 1` on the published artifact (§1).
- Generic evaluator with explicit model-version dispatch and a fail-closed
  default (§2a).
- Canonical hash over `{ policyModelVersion, rules }` (§12).
- Genesis/bootstrap semantics: `BaselineResolution`, the compiled-in
  `AUTHORIZED_GENESIS_BASELINE_SHA`, the `--allow-genesis-from` flag, and the
  two new violation codes (§5a, §6, §6a).
- Runtime manifest/artifact agreement in the shipped ⇒ manifest direction
  (§28a).
- `deepFreezeSerializable` and the §17b freeze ordering (§17a).

Everything else in the slice is carried forward from R unchanged.

### Source paths likely involved

All new files listed above, under
`packages/backend/src/modules/changeManagement/authorization/policy/`, plus
one new root `scripts/` file and one root `package.json` script entry.

**One existing file extended, not rewritten:**
`packages/backend/src/modules/changeManagement/architecture.test.ts` — extend
the `sources` array (currently `types.ts`, `evaluators.ts`,
`AuthorizationLedgerRepository.ts`, `KnexAuthorizationLedgerRepository.ts`) to
also read the new `policy/` files, and add the corrected semantic guards
(§17, §26.N) in place of the removed job-title regex. The existing
`FORBIDDEN_FIELDS` check already applies to any file added to that `sources`
array, so it extends for free.

**No other existing file is modified.** In particular,
`ChangeManagementService.ts`, `changeManagementPlugin.ts`, `types.ts` (the
top-level module one, not the new policy one), `validation.ts`, `errors.ts`,
and every migration file are untouched by this slice.

### Non-goals for this slice

- No app-config read.
- No Catalog/`coreServices.auth` dependency.
- No selector bundle, no selector resolver (that is F3.1.1b).
- No `ChangeManagementService` change.
- No plugin wiring change.
- No new migration.
- No route change.
- No CI pipeline creation or modification (the repo has none at `4bad41d`;
  `validate:policy-publication` is a standalone command only — §6).

### Acceptance criteria

- All items in §26.A–Q pass (matrix, determinism, totality, generic evaluator,
  model-version dispatch, hash stability, registry duplicates, manifest
  append-only rules, genesis cases, manifest/artifact digest agreement, deep
  immutability, freeze/digest invariance, semantic guards).
- `architecture.test.ts` guard extensions pass, including on the new `policy/`
  sources, and including the "`manager` as a variable name passes" case.
- All 15 pre-existing F3.1.0 Change Management suites remain green and
  unmodified.
- `yarn workspace backend lint` passes.
- `yarn workspace backend build` passes.
- **Corrected in R2 — the genesis invocation.** The seeding publish must be
  validated as an explicit genesis, because the manifest does not exist at the
  trusted baseline:

  ```
  yarn validate:policy-publication \
    --baseline-ref 4bad41d058edf5c5314d17275e0c8bdb5abf690f \
    --allow-genesis-from 4bad41d058edf5c5314d17275e0c8bdb5abf690f
  ```

  R's criterion — run it against "this branch's own prior commit … trivially,
  nothing to compare against yet" — is **wrong** and is withdrawn: it is exactly
  the undefined-genesis assumption of Gap A (§0a), and under the corrected rules
  it fails `BASELINE_MANIFEST_MISSING` unless the baseline is the approved SHA
  *and* genesis is explicitly declared.
- The **negative** genesis invocations must also be demonstrated:
  omitting `--allow-genesis-from` fails `BASELINE_MANIFEST_MISSING`, and
  supplying a non-approved SHA fails `GENESIS_NOT_AUTHORIZED` (§6a cases B, C).
- Repository-wide `tsc --noEmit` error set is **set-identical** (by
  file/line/column/code, not just count — per the F3.1.0-H precedent) to the
  5 pre-existing errors present at `4bad41d`.

### STOP condition

Stop immediately once the acceptance criteria above are met and verified.
**Do not** begin F3.1.1b, do not touch `ChangeManagementService.ts` or
`changeManagementPlugin.ts`, do not add app-config keys, do not add a Catalog
dependency, do not create or modify a CI pipeline. Report results and wait for
the next explicit authorization.

## 25. F3.1.2 prerequisites (carried forward, unaffected by this correction)

1. **`buildChange()` called twice** in `ChangeManagementService.createChange`
   (`packages/backend/src/modules/changeManagement/ChangeManagementService.ts`)
   — causes divergent server-generated `activityId`/`createdAt` between the
   two snapshots. **Must be fixed before F3.1.2.** Not touched by F3.1.1
   planning, this correction, or the F3.1.1a slice.
2. **Cross-cutover idempotency invariant** — an existing reservation with
   `authorization_mode = 'LEGACY_PRE_F3'` must resume legacy F2 submission
   semantics on retry; only a genuinely *new* reservation may select
   `LEDGER_REQUIRED`. F3.1.1's policy/selector evaluation must only ever be
   invoked from the future ledger-aware (`LEDGER_REQUIRED`) submission path
   that F3.1.2 builds — never from a retry of a legacy reservation. Unaffected
   by this planning-only correction.
3. RBAC CSV / conditional-policy files referenced by committed app-config
   remain absent from ADO HEAD — an F3.1.4 prerequisite, unrelated to
   F3.1.1/F3.1.2.

## 26. Test strategy (revised in R2)

**Relabelled in R2 onto the canonical A–Q scheme.** R's ad-hoc lettering
collided once genesis, model-version, and deep-immutability cases were added.
Content is preserved; only the labels and the new R2 cases changed.

**A. Policy matrix** — six pairs

- `normal + low` → one `individual` requirement, `pre_execution`.
- `normal + medium` → `individual` + `cab`, both `pre_execution`.
- `normal + high` → `individual` + `cab`, both `pre_execution`.
- `emergency` (× low/medium/high — three total rows, per §13's totality
  requirement, no wildcard) → two `individual` `pre_execution` requirements
  (`emergency-approver-a`, `emergency-approver-b`, sharing a
  `separationOfDutyKey`) + one `cab` `post_execution` requirement carrying
  `sla`.

**B. Totality** — every one of the 6 `{classification, risk}` pairs resolves to
exactly one `PolicyRule`; no default fallthrough exists; a mapped-type
compile-time check fails the build if a union member is added without a matching
rule (§13).

**C. Determinism** — same immutable `PolicyInput` + same policy version ⇒
byte-identical `policyInputSha256` and identical `RequirementDefinition[]` across
repeated `evaluatePolicy` calls.

**D. Generic evaluator, model V1** — `evaluatePolicy` behaves identically across
two distinct published versions of the same `policyKey` given the same input
shape but different `rules` content, proving the evaluator carries no
version-specific behavior (§2, §2a).

**E. Unsupported `policyModelVersion` fails closed** — an artifact carrying a
model version other than `1` (constructed via a deliberate cast, since the field
is the literal type `1`) makes `evaluatePolicy` throw, and makes **startup fail**
when shipped (§2a, §16). No fallback to V1, no nearest-match. *(New in R2.)*

**F. Policy hash includes `policyModelVersion`** — holding `rules` fixed and
changing only `policyModelVersion` **does** change `policyArtifactSha256` (§12).
Pins the Gap B correction: the interpretation contract is part of policy
identity. *(New in R2.)*

**G. Identity/provenance changes do not alter the behavioral digest** — changing
`policyKey`, `version`, or `provenance` alone (rules and model version unchanged)
does **not** change the digest (§12). Digest stability also holds across repeated
calls and across object-identity-different-but-content-equal `rules`.

**H. Rule-content change alters the digest** — changing any `rules` content
changes `policyArtifactSha256`.

**I. Manifest append-only (§6)**

- baseline entry present unchanged in candidate → `ok: true`.
- candidate appends a new entry → `ok: true`.
- baseline entry's digest changed in candidate → `ok: false`, `DIGEST_CHANGED`.
- baseline entry absent from candidate → `ok: false`, `IDENTITY_REMOVED`.
- duplicate `(artifact,key,version)` within one manifest → `ok: false`,
  `DUPLICATE_IDENTITY`.
- malformed manifest (bad version, non-hex digest) → `ok: false`,
  `MALFORMED_MANIFEST`.

**J. Genesis / bootstrap (§5a, §6a)** — one case per security scenario.
*(New in R2.)*

| Case | Setup | Expected |
|---|---|---|
| **J1** | baseline = approved SHA, manifest absent, genesis flag = approved SHA | `ok: true` — validated against an empty baseline |
| **J2** | baseline = approved SHA, manifest absent, **no** genesis flag | `ok: false`, `BASELINE_MANIFEST_MISSING` |
| **J3** | baseline = approved SHA, manifest absent, genesis flag = **some other** SHA | `ok: false`, `GENESIS_NOT_AUTHORIZED` |
| **J4** | genesis flag = approved SHA but `baselineSha` resolves **elsewhere**, manifest absent | `ok: false`, `GENESIS_NOT_AUTHORIZED` |
| **J5** | post-genesis baseline, manifest absent at that ref, genesis flag supplied | `ok: false`, `BASELINE_MANIFEST_MISSING` — genesis cannot rescue a non-approved baseline |
| **J6** | genesis authorized, candidate malformed / containing duplicate identities | `ok: false`, `MALFORMED_MANIFEST` / `DUPLICATE_IDENTITY` — genesis does not relax candidate validation |
| **J7** | genesis flag supplied but baseline **present** | flag ignored; normal append-only rules apply unchanged |

J2–J5 are what make the genesis rule a rule rather than a bypass; J6–J7 pin its
exact blast radius.

**K. Artifact / manifest digest agreement** — the shipped policy's computed
`policyArtifactDigest` must equal the `published-manifest.json` entry for its
`key@version`; a deliberate mismatch (simulated) fails startup. Also asserts the
§28a direction: a manifest entry with **no** shipped module does **not** fail
startup, while a shipped policy with no manifest entry **does**.

**L. Deep runtime immutability (§17a)** — after `createPolicyRegistry` returns,
every one of these must fail, asserted at **nested** depth, not only top level.
*(New in R2.)*

| # | Attempted mutation |
|---|---|
| **L1** | `policy.policyKey = 'x'` |
| **L2** | `policy.rules.push(rule)` |
| **L3** | `policy.rules[0].match.risk = 'low'` |
| **L4** | `policy.rules[0].requirements.push(req)` |
| **L5** | `policy.rules[0].requirements[0].sla.durationSeconds = 1` |

Each case asserts **both** that the operation throws **and** that the value is
unchanged afterwards. The value-unchanged assertion is the semantics-independent
backstop, so the suite still proves immutability if emitted test code is ever
non-strict. Compile-time `readonly` alone is **not** accepted as evidence for any
of L1–L5.

Plus `deepFreezeSerializable` unit behavior: throws on a function, a class
instance, a `Map`/`Date`, and an `undefined` property; accepts strings, finite
numbers, booleans, `null`, arrays, and plain objects; cycle-safe; returns the
same reference.

**M. Freeze does not change the digest** — `policyArtifactDigest` computed before
`deepFreezeSerializable` equals the digest computed after, for the shipped
artifact and for a fixture exercising nested arrays and objects (§17b, §20).
*(New in R2.)*

**N. Semantic architecture guards (§17)**

- Extended `FORBIDDEN_FIELDS` (`email`, `emailAddress`, `jobTitle`,
  `employeeName`, `displayName`-as-identity) absent from guarded `policy/`
  sources.
- No e-mail-shaped string literal in the published artifact's `rules` data.
- Every `selectorKey` in the published artifact matches the generic identifier
  convention.
- **A guarded source file containing a variable, parameter, or comment named
  `manager` (or `director`, `cto`, `superintendent`) passes all guards** —
  explicit regression test for the removed job-title regex (§0/Defect C).

**O. No provider / Teams / ADO leakage** — the existing `FORBIDDEN_FIELDS` guard
(`repositoryId`, `pipelineId`, `adoApprovalId`, `teamsUserId`, etc.) extended
over the new `policy/` source files; no ADO/Teams-specific identifier appears in
`PolicyInput`, `RequirementDefinition`, or `PublishedAuthorizationPolicy`.

**P. TypeScript baseline** — repository-wide `tsc --noEmit` error set is
**set-identical** to the 5 known errors at `4bad41d`, compared by
file/line/column/code, not count (§27).

**Q. Existing F3.1.0 tests remain green** — all 15 existing Change Management
suites pass and are unmodified by F3.1.1a (excluding the `architecture.test.ts`
guard extensions in N/O, which extend rather than alter existing assertions).

**R. SLA configuration validation** (carried forward from R's §26.K) — a
`RequirementDefinition` with `sla` present but missing `durationSeconds`, or with
an `anchor` outside `'execution_completion'`, fails startup validation.

## 27. TypeScript baseline (carried forward, restated per prompt §32)

Repository-wide TypeScript baseline at `4bad41d` contains exactly **5** known
historical errors. Any future F3.1.1a implementation must prove its candidate
error set is identical to that baseline by file, line, column, and error
code — not merely by count — per the F3.1.0-H precedent (§26.M).

## 28. Required architecture review answers (R2)

The R answer table is superseded by this one; every R answer not restated below
remains valid and unchanged.

| # | Question | Answer |
|---|---|---|
| 1 | What ADO baseline was verified? | `4bad41d058edf5c5314d17275e0c8bdb5abf690f`, `feat/ado-repo-governance` — verified against the **Azure DevOps REST API**, not just a local `origin/*` ref. Unmoved |
| 2 | What backstage-docs baseline was used? | `d5fc3ff93a8916932860f0c18ad5f3863f9f163c` (the F3.1.1-R checkpoint this revises) |
| 3 | Why does the first manifest require a genesis rule? | The trusted baseline `4bad41d` **predates the manifest**, so `git show <ref>:<path>` finds nothing. Without an explicit rule the natural repair — "missing ⇒ empty" — is a permanent append-only bypass — §0a Gap A, §5a |
| 4 | What exact baseline is authorized for genesis? | `4bad41d058edf5c5314d17275e0c8bdb5abf690f`, as a **compiled-in constant** `AUTHORIZED_GENESIS_BASELINE_SHA` — §5a |
| 5 | Can generic "manifest missing = empty" behavior exist? | **NO** — forbidden outright. Missing baseline without authorized genesis is `BASELINE_MANIFEST_MISSING` — §5a, §6 |
| 6 | How is genesis explicitly requested? | `--allow-genesis-from <exact-sha>` on `yarn validate:policy-publication`, plus `genesis: { allowedBaselineSha }` on the pure validator. Never inferred, never defaulted, never from env — §5a, §6 |
| 7 | What happens if genesis is requested from the wrong SHA? | **FAIL** `GENESIS_NOT_AUTHORIZED` — whether the flag disagrees with the compiled-in constant, or the constant disagrees with the resolved `--baseline-ref` — §6, §6a case C |
| 8 | What happens if a post-genesis baseline has no manifest? | **FAIL** `BASELINE_MANIFEST_MISSING`, regardless of any flag passed — §6a case D |
| 9 | What is `policyModelVersion`? | The **technical interpretation contract** for the artifact's shape and the meaning of its `rules` — §1, §2a |
| 10 | How is it different from `policyVersion`? | `version` is **business** policy publication identity; `policyModelVersion` is **evaluator/rule semantics**. Never conflated — §1 "Business version vs. model version" |
| 11 | What model version exists in F3 MVP? | **1**, only — §2a |
| 12 | Is `policyModelVersion` included in `policyArtifactSha256`? | **YES** — §12 |
| 13 | What exact object is hashed? | `sha256Canonical({ policyModelVersion, rules })`. Excludes `policyKey`, `version`, `provenance` — §12 |
| 14 | Is function serialization used? | **NO** — `rules` is plain data; no closure, no runtime object identity, no timestamps — §12 |
| 15 | Can evaluator V1 semantics be edited in place later? | **NO** — model-version semantics are append-only — §13a |
| 16 | What happens when semantics must change? | Introduce **`policyModelVersion = 2`** with its own evaluator; V1 artifacts keep resolving to V1 — §13a |
| 17 | Does this create a plugin/rules framework? | **NO** — one `case`, one function. No plugin registry, dynamic loading, schema interpreter, or version adapters — §2a |
| 18 | Is `Object.freeze` alone sufficient? | **NO** — it is shallow; nested arrays/objects stay mutable, and `readonly` is compile-time only — §0a Gap C, §17a |
| 19 | What runtime deep-immutability mechanism is recommended? | One small reusable `deepFreezeSerializable(value)` in `authorization/immutable.ts`, accepting exactly `canonicalJson`'s value domain, `WeakSet` cycle-safe — §17a |
| 20 | At what point is the artifact deep-frozen? | load → validate → compute/check digest → **deep freeze** → register, inside `createPolicyRegistry` — §17b |
| 21 | Are nested arrays/objects immutable after registration? | **YES** — proven at nested depth by §26.L1–L5, each asserting both a throw and an unchanged value |
| 22 | Does freeze change the digest? | **NO** — verified against `canonical.ts`: freezing changes neither prototype, `Array.isArray`, key enumeration, nor values — §17b, §26.M |
| 23 | Is publication append-only automatically CI-enforced today? | **NO** — the repository has **no CI pipeline at `4bad41d`** (re-verified via `git ls-tree`) — §21a |
| 24 | What exists after F3.1.1a? | A publication-integrity **mechanism + command + tests** — *not* mandatory CI wiring. The gate becomes mandatory only once a pipeline or branch policy invokes the command — §21a |
| 25 | Does the manifest retain historical identities forever? | **YES** — append-only publication identity/digest history — §5, §30 |
| 26 | Must runtime ship every historical policy forever? | **NO** — R's "never evicts … forever" is **withdrawn** — §29, §30 |
| 27 | What historical evidence is already retained by the ledger? | Policy identity, artifact hash, provenance, policy input + fingerprint, matched-rule provenance, **full effective requirements**, resolved principal snapshots, and the Change snapshot + hash — verified in ADO `authorization/types.ts` — §29 |
| 28 | What is the recommended runtime registry retention model? | **Model B** — manifest keeps identity history forever, ledger keeps round evidence forever, runtime ships only the active version plus deliberately retained rollback versions — §30 |
| 29 | What happens if active config references a policy not shipped? | **Startup fails** — fail-closed, never a silent wrong-policy evaluation — §16, §30 |
| 30 | Do inactive manifest entries need runtime modules? | **NO** — shipped ⇒ manifest entry, but manifest entry ⇏ shipped module — §28a |
| 31 | Are emergency A/B semantics unchanged? | **YES** — individual `user`-typed selectors for F3 MVP, distinct resolved refs, shared `separationOfDutyKey`; not reopened — §15 |
| 32 | Is CAB semantics unchanged? | **YES** — one collective authority, no quorum/voting/member expansion — §15 |
| 33 | Does F3.1.1a add DB/routes/UI/Teams/ADO integration? | **NO** — §20, §24 |
| 34 | What is the final F3.1.1a scope? | §24, with the R2 additions enumerated in "R2 scope additions" |
| 35 | What tests were added to the plan? | §26 relabelled to A–R; **new**: E (unsupported model version), F (hash includes model version), J1–J7 (genesis), L1–L5 + utility domain (deep immutability), M (freeze/digest invariance) |
| 36 | Is buildChange-twice still a F3.1.2 blocker? | **YES** — untouched by this checkpoint — §25 |
| 37 | Is cross-cutover idempotency preserved? | **YES** — untouched; policy evaluation must never reinterpret a legacy retry — §25 |
| 38 | Was any ADO implementation modified? | **NO** — planning only; ADO working tree byte-identical, `HEAD` still `4bad41d` |
| 39 | Is F3.1.1a implementation authorized by this checkpoint? | **NO** |
| 40 | What backstage-docs SHA records F3.1.1-R2? | Recorded in the handoff once this revision is committed and pushed (see `implementation-progress.md`) |
| 41 | Is F3.1.1a ready for final architecture acceptance? | **Yes — ready for final review.** All three R2 gaps are closed and no ADR conflict was found. Not authorized for implementation until a separate constrained prompt says so |

### Carried forward from R, unchanged

Self-declared digests are insufficient (§0/Defect A); publication identity is
`(artifact, key, version) → digest`; append-only is proven against a **git ref**,
never the working tree; `IDENTITY_REMOVED`/`DIGEST_CHANGED` on tampering; no
database; no publishing API; no `evaluate()` closure; the rule model is not a DSL
(closed literal `match` pairs, 6 rows); inactive historical policies are not
validated against the active selector bundle (§11); the job-title regex stays
removed and a variable named `manager` must pass (§17, §26.N).

## 28a. Runtime manifest / artifact agreement (new in R2)

At startup, for the policy versions this build actually ships:

- every **shipped** published policy must have a matching manifest entry for its
  `(artifact='policy', key, version)`;
- its computed `policyArtifactSha256` (§12) must equal that entry's `digest`;
- duplicate identities in the manifest fail;
- a malformed manifest fails.

**Direction of the check matters, and R left it ambiguous:**

> **shipped policy ⇒ must have a manifest entry.**
> **manifest entry ⇏ must have a shipped policy module.**

A manifest entry with no corresponding shipped module is **allowed**. The
manifest is *publication history*; requiring a module for every historical entry
would force the runtime to carry every policy ever published, forever — the
over-promise withdrawn in §29.

## 29. Historical replay position — explicit recommendation (corrected in R2)

R asserted the registry "never evicts a registered version" and that
`registry.get` is "always available for historical-round replay/audit, for the
lifetime of the deployed application." **That is withdrawn.** Two models were
considered:

| Model | Runtime obligation |
|---|---|
| **A** | runtime ships **every** historical policy forever |
| **B** | the **ledger** is authoritative historical evidence; the runtime ships only the policies needed for active/operational use, while the manifest retains full publication identity history |

**Recommendation: Model B.**

### Why B, checked against ADR-009 and the shipped F3.1.0 schema

ADR-009 was read in full (see §0a's reconciliation table). It requires evidence
sufficient to **reproduce** outcomes and to let an auditor **reconstruct** the
sequence; it forbids consulting a mutable current policy to **reinterpret**
history. **No clause requires executable replay of every historical policy.**

The ledger already retains, per round — verified in ADO
`authorization/types.ts` at `4bad41d`, not assumed:

| Evidence | Persisted field |
|---|---|
| policy identity | `policyKey`, `policyVersion` |
| policy artifact hash | `policyArtifactSha256` |
| policy provenance | `policyProvenance` |
| policy input + fingerprint | `policyInput`, `policyInputSha256` |
| matched rule provenance | `matchedRuleProvenance` |
| **effective requirements, in full** | `requirements: ApprovalRequirement[]` — kind, phase, mandatory, source, sourceRef, sourceProvenance, separationOfDutyKey, `sla*` |
| resolved principals | `ApprovalRequirement.principalSnapshot` (`PrincipalResolutionSnapshot`) |
| the evaluated Change itself | `changeSnapshot`, `changeSnapshotSha256` |

That is a **self-contained record of what actually occurred** — exactly what
ADR-009's audit requirement asks an auditor to reconstruct. Re-running a
historical policy would, at best, reproduce a result the ledger already stores
verbatim.

**Therefore historical *re-evaluation* must not become a runtime availability
obligation** without a concrete use case, and none exists in F3 MVP. Model A
would turn every future application binary into an ever-growing archive of
retired policy artifacts to satisfy a requirement no ADR makes.

**ADR-009 impact: none. No amendment is proposed or required.**

## 30. Recommended retention model (new in R2)

| Layer | Retains | Duration |
|---|---|---|
| **Publication manifest** | publication identity + digest history — every `(artifact, key, version) → digest)` ever published | **forever**, append-only (§5, §6) |
| **Authorization ledger** | actual round evidence (§29 table) | **forever**, append-only (F3.1.0, already shipped) |
| **Runtime registry** | the policy artifacts this build supports — the **active** version plus any deliberately retained rollback versions | current build only |

Rules that follow:

- **Active config references a version not shipped → startup fails** (§16). This
  is the safety property that makes B safe: a retention mistake is a loud,
  immediate, fail-closed startup error, never a silent wrong-policy evaluation.
- **Inactive manifest entries need no runtime module** (§28a).
- **The shipped set is the retention declaration.** The registry is built from an
  explicit array of version modules (§7) — reviewing that array *is* reviewing
  what the deployment can still evaluate. No separate retention config is
  introduced.
- Rollback stays a one-line `activePolicy` config change to a still-shipped
  version, subject to §11's active-pair re-validation.

**Consequences accepted:** re-*evaluating* a round whose artifact is no longer
shipped is not possible from the runtime. Auditing that round remains fully
possible from the ledger, and its publication identity and digest remain
provable from the manifest forever.

## 31. Open questions for architecture review (carried forward)

1. **Emergency A/B principal type.** Unchanged from the prior plan: this
   correction does not touch the individual-only (`user`-typed) A/B narrowing.
   If architecture review wants authority-typed emergency approvers supported
   in a future slice, that requires either an authority-membership-overlap
   check (a new capability) or accepting that distinctness can only be proven
   at decision time — that would warrant its own ADR, not folded into this
   plan or ADR-009 directly (§15).
2. **`selectorBundleVersion` / environment-scoped bundle key syntax.** Not
   prescribed here beyond "opaque, unique per content, one identity per
   environment" (§19). Left to F3.1.1b implementation time as a
   config-authoring preference.
3. **New — fallback validator implementation.** If
   `node --experimental-strip-types` proves unworkable when F3.1.1a is
   actually implemented, §6's documented fallback (plain-jest-tested
   `.ts` + a synchronized `.mjs` re-implementation) should be used instead.
   Flagged so implementation does not silently pick a third option.

4. **New in R2 — the genesis constant's successor.** The approved genesis SHA
   (§5a) is `4bad41d`. If any authorized commit lands on
   `feat/ado-repo-governance` before F3.1.1a is implemented, the constant must be
   updated to that exact new pre-manifest SHA in the implementation PR. Flagged so
   the implementer updates it deliberately rather than discovering a
   `GENESIS_NOT_AUTHORIZED` failure and reaching for `--allow-genesis-from` with
   whatever SHA makes it pass.

None of these four requires or proposes amending ADR-009. Questions 3 and 4 are
implementation-detail flags, not architecture questions.

### Resolved in R2 (no longer open)

- **Historical replay obligation** — resolved as Model B against ADR-009's actual
  text and the shipped F3.1.0 schema (§29, §30). Was implicit in R's
  "never evicts … forever" wording rather than stated as a question.
- **Genesis code-path lifetime** — resolved: retained permanently with its tests
  (§5a).
- **Genesis candidate strictness** — resolved: any structurally valid,
  duplicate-free candidate; entry count is not constrained (§6).

## Gate

**F3.1.1 architecture/implementation planning is CORRECTED (R2) and under
FINAL review.**

**Recommendation: GO** for final architecture acceptance of the corrected first
slice, **F3.1.1a — Policy domain + registry + publication integrity** (§24),
subject to sign-off on:

1. the genesis contract and its single approved baseline (§5a, §6a);
2. the `policyModelVersion` interpretation contract and its append-only
   semantics (§2a, §13a);
3. the deep-immutability mechanism and freeze ordering (§17a, §17b);
4. the corrected historical retention model — **Model B** (§29, §30);
5. the emergency A/B narrowing (§31, question 1, carried forward unchanged).

**ADR-009 is unaffected.** Its clauses were read in full and reconciled against
the corrected model (§0a); no clause requires executable replay of historical
policies, and no amendment is proposed.

**F3.1.1 implementation itself remains NOT AUTHORIZED** by this checkpoint. No
ADO source, migration, configuration, script, test, pipeline, or runtime
behavior was modified to produce this correction. A separate, constrained
implementation prompt must explicitly authorize slice F3.1.1a before any code is
written.
