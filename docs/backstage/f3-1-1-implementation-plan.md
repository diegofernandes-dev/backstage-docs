# GMUD F3.1.1 — Published Policy & Selector Resolution implementation plan

- Status: **PLAN CORRECTED / UNDER REVIEW — IMPLEMENTATION NOT AUTHORIZED**
- Date: 2026-09-02
- ADO implementation baseline (verified): `platform-devops-developer-portal@4bad41d`
  (full SHA `4bad41d058edf5c5314d17275e0c8bdb5abf690f`), branch `feat/ado-repo-governance`.
  `origin/feat/ado-repo-governance` confirmed identical to local `HEAD`; ADO has not moved
  since F3.1.0-H closure.
- Documentation baseline used to produce this correction: `backstage-docs@75abd9d60facc6ff1dfc54af4a46192f4c287c9e`
  (the prior F3.1.1 planning checkpoint this document corrects).
- Architecture authority: [ADR-009](../adr/ADR-009-change-authorization-model.md) (unchanged by this checkpoint)
- Prior checkpoint: [F3.1.0 implementation plan](./f3-1-implementation-plan.md) — **F3.1.0 CLOSED**

## Correction notice (F3.1.1-R)

This is a **corrective revision** of the F3.1.1 plan previously published at
`75abd9d`. Architecture review found three defects in that version, described in
[§0](#0-what-changed-and-why) below. The plan is corrected in place — sections
are renumbered where the correction reshapes the model; the F3.1.1 scope,
decomposition, and non-goals are otherwise preserved. **F3.1.1a implementation
remains unauthorized**; nothing in ADO was touched to produce this correction.

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
§17, §26.I.

Two further corrections fall out of A–C:

- **Historical vs. active validation.** An inactive historical policy must not
  fail application startup merely because the *active* selector bundle no
  longer contains a selector only that inactive policy references — §11, §16.
- **Three separated failure domains** — CI/publication, runtime startup,
  submission — previously blended into one failure table — §16.

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
// (see §12); `rules` IS the hash (see §12).
type PublishedAuthorizationPolicy = {
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
- **Old versions remain interpretable**: the registry never evicts a
  registered version; `registry.get(key, version)` is always available for
  historical-round replay/audit, for the lifetime of the deployed application.

## 2. The generic evaluator

```ts
function evaluatePolicy(
  policy: PublishedAuthorizationPolicy,
  input: PolicyInput,
): PolicyEvaluationResult;
```

One function, shared by every published version. It:

1. finds the single `PolicyRule` in `policy.rules` whose `match` equals
   `input` (totality guaranteed at load time — §16, §26.C — so this lookup
   never falls through);
2. computes `policyInputSha256 = sha256Canonical(input)`;
3. returns `PolicyEvaluationResult` with `matchedRuleProvenance = rule.ruleId`
   and `requirements = rule.requirements`.

This directly answers §12's original concern: because `rules` — not an
`evaluate` closure — is what a policy version *is*, `policyArtifactSha256`
covering `rules` now covers **all** policy-specific behavior. `evaluatePolicy`
itself is identical, reviewed, tested once, and never varies per version.

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
  | { code: 'MALFORMED_MANIFEST'; reason: string };

function validatePublishedManifest(args: {
  baseline: { published: ManifestEntry[] };
  candidate: { published: ManifestEntry[] };
}): { ok: true } | { ok: false; violations: PublicationViolation[] };
```

Rules (pure, no I/O — the caller supplies both manifests already parsed):

| Case | Result |
|---|---|
| baseline identity present in candidate, same digest | OK |
| candidate contains an identity absent from baseline | OK (append) |
| baseline identity absent from candidate | **FAIL** `IDENTITY_REMOVED` |
| baseline identity present in candidate with a different digest | **FAIL** `DIGEST_CHANGED` |
| duplicate `(artifact,key,version)` within either manifest | **FAIL** `DUPLICATE_IDENTITY` |
| malformed manifest (bad `manifestVersion`, non-sha256 digest, missing field) | **FAIL** `MALFORMED_MANIFEST` |

**Trusted-baseline sourcing — the part that actually proves append-only
history:** the baseline is never the working tree. A repository command:

```
yarn validate:policy-publication --baseline-ref <git-ref>
```

implemented as `scripts/validate-policy-publication.mjs`:

1. reads the **baseline** manifest via `git show <baseline-ref>:packages/backend/src/modules/changeManagement/authorization/policy/published-manifest.json`;
2. reads the **candidate** manifest from the working tree;
3. calls the pure `validatePublishedManifest` above (loaded via
   `node --experimental-strip-types`, available on node 22.21+/24, already the
   engine range this repo declares — confirmed locally);
4. exits non-zero and prints every violation if `ok: false`.

`--baseline-ref` is **required** — there is no default and no environment
autodetection, per the review prompt's explicit instruction not to hide this
behind vague statements. In PR/CI validation the caller supplies the trusted
target branch or `git merge-base origin/main HEAD`; a developer working
locally supplies an explicit ref (e.g. `origin/main`, a tag, a prior commit).

**Fallback if `--experimental-strip-types` proves unworkable at
implementation time:** keep `validatePublishedManifest` as a plain `.ts` unit
under jest (already exercised by §26.G tests) and give the `.mjs` script a
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
    map.set(id, Object.freeze(v));
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

The registry is a frozen, in-memory map built at startup from every
`PublishedAuthorizationPolicy` the application ships — not filtered to the
active pin. `registry.get(key, version)` is always available for any
previously-published version, so a historical `AuthorizationRound`'s
`(policyKey, policyVersion)` is always re-derivable/auditable, satisfying
ADR-009's "existing rounds remain bound to their recorded versions; a mutable
'current policy' is never consulted to reinterpret history."

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
  `policy/` source directory (§16, §26.H).

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

- **A. Needing old policy identity/content for audit/replay** — always true,
  satisfied simply by the registry never evicting a version (§8).
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

`policyArtifactSha256 = sha256Canonical(policy.rules)`.

| Field | Hashed? | Reason |
|---|---|---|
| `policy.rules` | **Yes** — this *is* the digest | The entire behavioral surface: which `PolicyRequirementRule`s a given `{classification, risk}` produces (§1, §2) |
| `policy.policyKey` | No — identity metadata | Renaming would not change behavior; identity/digest binding is the manifest's job (§5), not the digest's |
| `policy.version` | No — identity metadata | Same reasoning; also avoids a self-referential hash (the version string naming itself) |
| `policy.provenance` | No — publication metadata | A corrected PR link or commit citation must never read as a behavior change — see §13 |

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

## 14. Hashing / canonicalization (unchanged mechanism, corrected inputs)

No second canonicalization scheme. Every hash in F3.1.1 reuses
[`canonical.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/authorization/canonical.ts)
verbatim (`canonicalJson`, `sha256Canonical`, `assertSha256`):

| Value | Computed as |
|---|---|
| `policyArtifactSha256` | `sha256Canonical(policy.rules)` — see §12 (corrected: `rules` data, not a matrix description of a function) |
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
table did not have a CI/publication row at all, since the append-only manifest
did not exist. The failure model now has three explicitly separated domains,
each owned by a different actor at a different time:

### CI / publication validation (`yarn validate:policy-publication`, §6)

| Failure | Result |
|---|---|
| PR/commit changes an existing published policy digest | **FAIL** — `DIGEST_CHANGED` |
| PR/commit removes a previously published version | **FAIL** — `IDENTITY_REMOVED` |
| PR/commit duplicates an existing `(artifact,key,version)` | **FAIL** — `DUPLICATE_IDENTITY` |
| PR/commit appends a new, previously-unseen version | **OK** |
| Manifest malformed | **FAIL** — `MALFORMED_MANIFEST` |

### Runtime startup validation (pure, no I/O)

| Failure | Result |
|---|---|
| No active policy configured / unknown `activePolicy` pin | Startup fails |
| Duplicate `policyKey@version` registered in this build | Startup fails |
| Active policy matrix not total over classification×risk | Startup fails |
| Active `policyArtifactSha256` (computed from shipped `rules`) disagrees with the manifest's entry for that `key@version` | Startup fails — content/manifest drift within one deploy |
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
  case in §26.I, not merely an assumption.

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

## 22. Performance / caching (unchanged)

Policy evaluation is pure/in-memory — no caching needed or beneficial.
Selector bundle lookup is an in-memory map read. Catalog validation involves
real backend service calls; no caching is introduced in F3.1.1 — caching is
precisely the mechanism by which a stale principal could leak into an
otherwise-immutable round snapshot.

## 23. F3.1.1 decomposition — re-evaluated, unchanged

The two-slice decomposition remains correct after these corrections. The
manifest and validator (§5, §6) are new work but have no independent
reviewable I/O boundary of their own — a manifest is meaningless without the
policy artifact it hashes, so it belongs inside F3.1.1a, not a third slice.

| Slice | Scope | I/O boundary |
|---|---|---|
| **F3.1.1a — Policy domain + registry + publication integrity** | `PolicyInput`, `RequirementDefinition`, `PolicyRule`, `PublishedAuthorizationPolicy`, `PolicyEvaluationResult` types; the published `default-change-authorization@2026-09-02.1` data artifact implementing the ADR-009 matrix; `evaluatePolicy`; `createPolicyRegistry`; `policyArtifactDigest`; `published-manifest.json`; `validatePublishedManifest`; `scripts/validate-policy-publication.mjs`; corrected semantic architecture guards | None at runtime — pure, no config read, no Catalog, no DB. The manifest validator reads git history via a script, not via the application |
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
  `PublishedAuthorizationPolicy`, `PolicyEvaluationResult` (§1).
- `published/default-change-authorization.2026-09-02.1.ts` — one exported
  `PublishedAuthorizationPolicy` **data** artifact implementing the six-row
  matrix (§1, §26.A), `Object.freeze`d, no functions.
- `evaluatePolicy.ts` — the single generic evaluator (§2) plus
  `policyInputSha256` computation via `canonical.ts`.
- `registry.ts` — `createPolicyRegistry` (§7).
- `policyArtifactDigest.ts` — `sha256Canonical(policy.rules)` (§12).
- `published-manifest.json` — the append-only manifest (§5), seeded with the
  one policy entry above.
- `publication/validateManifestHistory.ts` — the pure
  `validatePublishedManifest` (§6).
- `scripts/validate-policy-publication.mjs` (repo-root `scripts/`, matching
  the existing convention alongside `build-showcase-techdocs.sh`) — the
  `--baseline-ref` CLI wrapper (§6).
- Root `package.json` — one new script:
  `"validate:policy-publication": "node scripts/validate-policy-publication.mjs"`.

Test files (co-located, matching existing repo convention):

- `published/default-change-authorization.2026-09-02.1.test.ts` — matrix
  coverage (§26.A), determinism (§26.B), totality (§26.C).
- `registry.test.ts` — duplicate detection, unknown-version lookup (§26.F).
- `policyArtifactDigest.test.ts` — hash stability, identity-metadata exclusion
  (§26.E).
- `publication/validateManifestHistory.test.ts` — the full table in §6
  (§26.G).

### Expected types/interfaces

Exactly the types in §1, §2, §5, §6, and §7 of this plan — no additions beyond
what's specified there.

### Source paths likely involved

All new files listed above, under
`packages/backend/src/modules/changeManagement/authorization/policy/`, plus
one new root `scripts/` file and one root `package.json` script entry.

**One existing file extended, not rewritten:**
`packages/backend/src/modules/changeManagement/architecture.test.ts` — extend
the `sources` array (currently `types.ts`, `evaluators.ts`,
`AuthorizationLedgerRepository.ts`, `KnexAuthorizationLedgerRepository.ts`) to
also read the new `policy/` files, and add the corrected semantic guards
(§17, §26.I) in place of the removed job-title regex. The existing
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

- All items in §26.A–I pass (matrix, determinism, totality, generic
  evaluator, hash stability, registry duplicates, manifest append-only rules,
  manifest/artifact digest agreement, semantic guards).
- `architecture.test.ts` guard extensions pass, including on the new `policy/`
  sources, and including the "`manager` as a variable name passes" case.
- All 15 pre-existing F3.1.0 Change Management suites remain green and
  unmodified.
- `yarn workspace backend lint` passes.
- `yarn workspace backend build` passes.
- `yarn validate:policy-publication --baseline-ref <this-branch's-own-prior-commit>`
  passes on the seeded single-entry manifest (trivially — nothing to compare
  against yet beyond the initial publish).
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

## 26. Test strategy (revised)

**A. Policy matrix**

- `normal + low` → one `individual` requirement, `pre_execution`.
- `normal + medium` → `individual` + `cab`, both `pre_execution`.
- `normal + high` → `individual` + `cab`, both `pre_execution`.
- `emergency` (× low/medium/high — three total rows, per §13's totality
  requirement, no wildcard) → two `individual` `pre_execution` requirements
  (`emergency-approver-a`, `emergency-approver-b`, sharing a
  `separationOfDutyKey`) + one `cab` `post_execution` requirement carrying
  `sla`.

**B. Determinism** — same immutable `PolicyInput` + same policy version ⇒
byte-identical `policyInputSha256` and identical `RequirementDefinition[]`
across repeated `evaluatePolicy` calls.

**C. Totality** — every one of the 6 `{classification, risk}` pairs resolves
to exactly one `PolicyRule`; no default fallthrough exists; a mapped-type
compile-time check fails the build if a union member is added without a
matching rule (§13).

**D. Generic evaluator** — `evaluatePolicy` behaves identically across two
distinct published versions of the same `policyKey` given the same input
shape but different `rules` content — proving the evaluator itself carries no
version-specific behavior (§2).

**E. Artifact hash stability** — `policyArtifactDigest(policy)` is stable
across repeated calls and across object-identity-different-but-content-equal
`rules`; changing `policyKey`, `version`, or `provenance` alone (rules
unchanged) does **not** change the digest (§12); changing any `rules` content
does change it.

**F. Duplicate registry identity** — registering two versions with the same
`key@version` string fails registry construction (§7); the older version
remains retrievable via `registry.get` and evaluates identically to when it
was active (version isolation).

**G. Publication manifest (§6)**

- baseline entry present unchanged in candidate → `ok: true`.
- candidate appends a new entry → `ok: true`.
- baseline entry's digest changed in candidate → `ok: false`,
  `DIGEST_CHANGED`.
- baseline entry absent from candidate → `ok: false`, `IDENTITY_REMOVED`.
- duplicate `(artifact,key,version)` within one manifest → `ok: false`,
  `DUPLICATE_IDENTITY`.
- malformed manifest (bad version, non-hex digest) → `ok: false`,
  `MALFORMED_MANIFEST`.

**H. Policy artifact / manifest digest agreement** — the shipped policy's
computed `policyArtifactDigest` must equal the `published-manifest.json`
entry for its `key@version`; a deliberate mismatch (simulated in the test)
fails.

**I. Semantic architecture guards (§17)**

- Extended `FORBIDDEN_FIELDS` (`email`, `emailAddress`, `jobTitle`,
  `employeeName`, `displayName`-as-identity) absent from guarded `policy/`
  sources.
- No e-mail-shaped string literal in the published artifact's `rules` data.
- Every `selectorKey` in the published artifact matches the generic
  identifier convention.
- **A guarded source file containing a variable, parameter, or comment named
  `manager` (or `director`, `cto`, `superintendent`) passes all guards** —
  explicit regression test for the removed job-title regex (§0/Defect C).

**J. No provider leakage** — the existing `FORBIDDEN_FIELDS` guard
(`repositoryId`, `pipelineId`, `adoApprovalId`, `teamsUserId`, etc.) extended
over the new `policy/` source files; no ADO/Teams-specific identifier appears
in `PolicyInput`, `RequirementDefinition`, or `PublishedAuthorizationPolicy`.

**K. SLA configuration validation** — a `RequirementDefinition` with `sla`
present but missing `durationSeconds` or an `anchor` outside
`'execution_completion'` fails startup validation.

**L. Regression** — all 15 existing F3.1.0 Change Management suites remain
green and are not modified by F3.1.1 changes (excluding the
`architecture.test.ts` guard extensions in I/J, which extend rather than alter
existing assertions).

**M. TypeScript baseline** — repository-wide `tsc --noEmit` error set is
set-identical to the 5 known errors at `4bad41d`, compared by
file/line/column/code, not count (§27).

## 27. TypeScript baseline (carried forward, restated per prompt §32)

Repository-wide TypeScript baseline at `4bad41d` contains exactly **5** known
historical errors. Any future F3.1.1a implementation must prove its candidate
error set is identical to that baseline by file, line, column, and error
code — not merely by count — per the F3.1.0-H precedent (§26.M).

## 28. Required architecture review answers

| # | Question | Answer |
|---|---|---|
| 1 | What ADO baseline was verified? | `4bad41d058edf5c5314d17275e0c8bdb5abf690f`, `feat/ado-repo-governance` — confirmed unmoved |
| 2 | What backstage-docs baseline was used? | `75abd9d60facc6ff1dfc54af4a46192f4c287c9e` (the plan this correction supersedes) |
| 3 | Why is a self-declared digest insufficient for historical immutability? | It only detects "content changed, digest forgotten," not "content and digest both changed together under the same identity" — §0/Defect A, §5 |
| 4 | What exactly is the trusted publication identity? | `(artifact, key, version) → digest`, one entry per published policy or selector-bundle version — §5 |
| 5 | What is the policy publication manifest? | `published-manifest.json`, a checked-in JSON array of `{artifact, key, version, digest}` entries — §5 |
| 6 | How is it proven append-only? | A pure `validatePublishedManifest(baseline, candidate)` function requiring every baseline entry present unchanged in the candidate — §6 |
| 7 | Against what trusted baseline is it compared? | A previous **git ref** (`--baseline-ref`), read via `git show <ref>:<path>` — never the working tree alone — §6 |
| 8 | What happens if an old identity disappears? | CI/publication validator fails: `IDENTITY_REMOVED` — §6, §16 |
| 9 | What happens if an old digest changes? | CI/publication validator fails: `DIGEST_CHANGED` — §6, §16 |
| 10 | Can a new version be appended? | Yes — new `(artifact,key,version)` entries always pass — §6 |
| 11 | Does this require a database? | **No** — §20 |
| 12 | Does this require a policy publishing API? | **No** — a checked-in file + a standalone script — §5, §6, §20 |
| 13 | Is a policy version still represented by an arbitrary `evaluate()` function? | **No** — §1, §2 |
| 14 | What serializable policy artifact replaces that model? | `PublishedAuthorizationPolicy { policyKey, version, provenance, rules }` — §1 |
| 15 | What generic evaluator consumes it? | `evaluatePolicy(policy, input): PolicyEvaluationResult` — §2 |
| 16 | Does `policyArtifactSha256` now cover all policy-specific behavior? | Yes — it hashes `rules`, which is the entire behavioral surface; the evaluator is shared, unversioned code — §2, §12 |
| 17 | Is function serialization used? | **No** — §12 |
| 18 | Does the rule model become a generic DSL? | **No** — closed literal `match` pairs only, 6 total rows — §13 |
| 19 | What exact dimensions can the MVP policy express? | `classification ∈ {normal, emergency}` × `risk ∈ {low, medium, high}` → `RequirementDefinition[]` — §1, §13 |
| 20 | How is totality proven? | Every one of 6 pairs has exactly one rule, checked at load time plus a compile-time exhaustiveness bind — §13, §26.C |
| 21 | Is the active policy pin still used only for new submissions? | Yes — §8 |
| 22 | Must inactive historical policies validate against the active selector bundle? | **No** — corrected — §11 |
| 23 | What historical policy information must remain available? | Identity/content for audit/replay via `registry.get`, for the app's lifetime — §8, §11 |
| 24 | What historical selector information is already snapshotted in the ledger? | Bundle identity/version/hash/provenance per round; selectorKey/version/principalType/resolvedPrincipalRef/resolverProvenance/selectorDigest per requirement — §15 (F3.1.0, unchanged) |
| 25 | Does a self-declared selector digest alone prove historical immutability? | **No** — §18 |
| 26 | What publication-integrity model is recommended for selector bundles? | The same manifest + validator used for policies (option A) — §18 |
| 27 | Are environment-specific selector bindings still supported? | Yes — §19 |
| 28 | How are they represented without mutating an existing bundle identity? | Each environment gets its own explicit bundle key/version/manifest entry; config only selects which is active — §19 |
| 29 | Was source-wide job-title regex scanning removed from the plan? | **Yes** — §0/Defect C, §17 |
| 30 | What semantic guards replace it? | Forbidden canonical field names + published-artifact data assertions (no e-mail shapes, generic selector keys, valid Catalog ref kinds) — §17 |
| 31 | Can a normal variable named `manager` exist in source? | **Yes** — explicit test case §26.I |
| 32 | Are emails canonical principals? | **No** |
| 33 | Are job titles canonical authorization semantics? | **No** |
| 34 | Are emergency A/B still individual-only for F3 MVP? | Yes — §15 |
| 35 | Is CAB still one collective authority? | Yes — §15 |
| 36 | Does F3.1.1 add DB tables? | **No** — §20 |
| 37 | Does it add routes/UI/Teams/ADO integration? | **No** |
| 38 | What is the revised F3.1.1 decomposition? | Unchanged two-slice split; F3.1.1a absorbs the manifest/validator (no independent third slice) — §23 |
| 39 | What is the exact revised F3.1.1a scope? | §24 |
| 40 | What tests gate F3.1.1a? | §26.A–M |
| 41 | Is `buildChange()` still a pre-F3.1.2 blocker? | **Yes** — §25 |
| 42 | Is cross-cutover idempotency preserved? | **Yes** — §25 |
| 43 | Was any ADO implementation modified in this checkpoint? | **No** |
| 44 | Is F3.1.1a implementation authorized by this checkpoint? | **No** |
| 45 | What backstage-docs SHA records the corrected plan? | Recorded in the handoff after this commit is pushed (see `implementation-progress.md` checkpoint) |
| 46 | Is the revised F3.1.1a ready for architecture acceptance? | Yes — ready for review; not authorized for implementation |

## 29. Open questions for architecture review (carried forward)

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

Neither of the first two questions requires or proposes amending ADR-009. The
third is an implementation-detail fallback, not an architecture question.

## Gate

**F3.1.1 architecture/implementation planning is CORRECTED and under
review.**

**Recommendation: GO** for architecture review of the corrected first slice,
**F3.1.1a — Policy domain + registry + publication integrity** (§24), subject
to architecture sign-off on the emergency A/B narrowing (§29, question 1,
carried forward unchanged) and on the corrected publication-manifest model
(§5, §6).

**F3.1.1 implementation itself remains NOT AUTHORIZED** by this checkpoint. No
ADO source, migration, configuration, test, pipeline, or runtime behavior was
modified to produce this correction. A separate, constrained implementation
prompt must explicitly authorize slice F3.1.1a before any code is written.
