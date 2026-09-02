# GMUD F3.1.1 — Published Policy & Selector Resolution implementation plan

- Status: **PLANNING COMPLETE / UNDER REVIEW — IMPLEMENTATION NOT AUTHORIZED**
- Date: 2026-09-02
- ADO implementation baseline (verified): `platform-devops-developer-portal@4bad41d`
  (full SHA `4bad41d058edf5c5314d17275e0c8bdb5abf690f`), branch `feat/ado-repo-governance`.
  `origin/feat/ado-repo-governance` confirmed identical to local `HEAD`; ADO has not moved
  since F3.1.0-H closure.
- Documentation baseline used to produce this plan: `backstage-docs@f4dc268b6f7ac09600710e59d9c7d35d918fdc15`
- Architecture authority: [ADR-009](../adr/ADR-009-change-authorization-model.md) (unchanged by this checkpoint)
- Prior checkpoint: [F3.1.0 implementation plan](./f3-1-implementation-plan.md) — **F3.1.0 CLOSED**

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
5. what immutable provenance must be captured for a future `AuthorizationRound`.

F3.1.1 does **not** create an `AuthorizationRound` during `POST /changes` — that
is F3.1.2. This document is architecture/implementation **planning only**; no
ADO source, migration, configuration, test, or runtime behavior was modified to
produce it.

## Method

This plan was produced by reading the actual F3.1.0 implementation at ADO
`4bad41d`, not by re-deriving requirements from ADR-009 alone:

- [`authorization/types.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/authorization/types.ts) — the exact persisted shape of `AuthorizationRound`/`ApprovalRequirement`/`ApprovalDecision`/`AuthorizationAuditEvent`
- [`authorization/canonical.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/authorization/canonical.ts) — the existing canonical-JSON/SHA-256 primitives
- [`authorization/evaluators.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/authorization/evaluators.ts) — the pure `AuthorizationEvaluation`/`GovernanceEvaluation` derivations
- [`authorization/AuthorizationLedgerRepository.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/authorization/AuthorizationLedgerRepository.ts) — the insert/read-only repository contract
- `packages/backend/migrations/change-management/20260901200000_authorization_ledger_foundation.cjs` — the actual ledger table columns (source of truth for §7 field mapping)
- [`architecture.test.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/architecture.test.ts) — the guards F3.1.1 code must keep passing
- [`executionPlanValidator.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/executionPlanValidator.ts) — the existing Catalog-ref validation pattern to reuse for selector resolution
- [`idpProvisionerConfig.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/idpProvisioner/idpProvisionerConfig.ts) — the existing `read*Config(config: Config)` pattern to reuse for selector bundle configuration
- `changeManagementPlugin.ts` — current plugin dependency wiring (no `coreServices.auth` yet)

## 1. Policy model

`AuthorizationPolicy` is a published, immutable, deterministic artifact —
reviewed application code, not a rules engine, DSL, BPM/workflow engine, or
mutable database row.

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
    policyKey: string;
    policyVersion: string;
    durationSeconds: number;
    anchor: 'execution_completion';
  };
};

type PolicyEvaluationResult = {
  policyKey: string;
  policyVersion: string;
  policyArtifactSha256: string;
  policyProvenance: string;
  matchedRuleProvenance: string;
  input: PolicyInput;
  policyInputSha256: string;
  requirements: RequirementDefinition[];
};

type AuthorizationPolicyVersion = {
  policyKey: string;
  version: string;               // e.g. '2026-09-02.1'
  artifactSha256: string;        // frozen digest of this version's matrix
  evaluate: (input: PolicyInput) => PolicyEvaluationResult;
};
```

- **policyKey**: a stable identifier, e.g. `default-change-authorization`.
- **policyVersion**: an opaque, monotonically-meaningless string, e.g.
  `2026-09-02.1`. No syntax is prescribed beyond "unique per key, never reused
  for different content" — see §21 (content-integrity).
- **Publication identity**: `policyKey@policyVersion`, unique in the registry
  (§10).
- **Provenance**: `policyProvenance` names the reviewing PR/commit that
  published the version; `matchedRuleProvenance` names which row of the
  classification/risk matrix produced the output for a given input (audit
  trail down to the specific rule, not just the version).
- **Deterministic input**: `PolicyInput` only — see §8.
- **Output**: `RequirementDefinition[]`, pure — see §9.
- **Active/current version selection**: an explicit config pin, never
  "whatever is latest" — see §11.
- **Validation**: startup-time totality and shape checks — see §21.
- **Startup/load failure behavior**: fail application startup on any
  structural defect — see §21.
- **Version immutability**: enforced by a frozen per-version content digest
  checked in a unit test, not at runtime (compiled code cannot be tampered
  with post-deploy the way config can) — see §22.
- **New versions**: added as a new file exporting a new
  `AuthorizationPolicyVersion`, registered alongside old ones — never editing
  an existing version's file.
- **Old versions remain interpretable**: the registry never evicts a
  registered version; `registry.get(key, version)` is always available for
  historical-round replay/audit, for the lifetime of the deployed application.

A policy version has a stable key such as `default-change-authorization` and
an immutable version such as `2026-09-02.1`, matching ADR-009's example
(not prescriptive of syntax beyond uniqueness).

## 2. Policy publication / version registry — recommended approach

**Recommended: hybrid — policy matrix as versioned TypeScript, selector
bindings as validated application configuration** (option C in the ADR-009
framing, applied asymmetrically).

| Concern | Mechanism | Why |
|---|---|---|
| Classification × risk matrix | Versioned TypeScript module per policy version | Environment-independent domain intent; exhaustiveness-checked over the closed `ChangeClassification`/`ChangeRiskLevel` unions already in [`types.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/types.ts); reviewed and diffable like any other code; no parser or schema to maintain |
| Selector → Catalog ref bindings | Validated `changeManagement.authorization` app-config, read via a `readAuthorizationConfig(config: Config)` function following the existing `readAdoProvisionerConfig` pattern | Which Catalog group is `cab-authority` genuinely differs dev vs. production; forcing a code deploy to rebind a group is unnecessary friction, and compiling environment refs into the domain would violate the "no provider/environment leakage" architecture guard |

**Rejected alternatives:**

- **All-TypeScript (option A):** would force a full application deploy to
  rebind a Catalog group per environment, and tempts hardcoding environment
  entity refs into domain code — exactly what the existing
  `architecture.test.ts` forbidden-fields guard exists to prevent.
- **All-configuration (option B):** the classification/risk matrix would
  become a small policy DSL expressed in YAML/JSON, which ADR-009 explicitly
  rejects ("Policy DSL on day one — Rejected"), and loses TypeScript's
  exhaustiveness checking over the matrix.
- **Mutable database admin table:** rejected outright — see §25. No runtime
  policy-editing UI, ever, in F3 MVP.

## 3. Active policy selection

`changeManagement.authorization.activePolicy: { key: string; version: string }`
in app-config names exactly one published version. This is the *only* mutable
runtime input to policy selection, and it selects only for **new**
submissions.

The registry is a frozen, in-memory map built at startup from every
`AuthorizationPolicyVersion` the application ships — not filtered to the
active pin. `registry.get(key, version)` is always available for any
previously-published version, so a historical `AuthorizationRound`'s
`(policyKey, policyVersion)` is always re-derivable/auditable, satisfying
ADR-009's "existing rounds remain bound to their recorded versions; a mutable
'current policy' is never consulted to reinterpret history."

F3.1.2 obtains `selectedPolicy` as: `registry.get(activePolicy.key,
activePolicy.version)`, evaluated once at submission time, then embeds the
resulting `policyKey`/`policyVersion`/`policyArtifactSha256`/`policyProvenance`
into the new round. The round's own stored version is what all future reads
consult — never a re-lookup of "current" config.

**Rejected:** "always evaluate the greatest registered version" — this would
mean adding a new policy file silently changes behavior on the next deploy,
and rollback would require *deleting* a published version (which contradicts
immutability). An explicit pin makes rollback a one-line config change to a
still-registered older version.

## 4. Policy input

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
  `policy/` source directory (§16, §Test strategy G/H).

The full Change snapshot is independently hashed into
`AuthorizationRound.changeSnapshotSha256` already — policy input fingerprinting
is a narrower, separate concern: `policyInputSha256 = sha256Canonical(input)`,
reusing [`canonical.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/authorization/canonical.ts)
verbatim. **No second canonicalization scheme is introduced.**

## 5. Policy output

The policy does **not** persist `ApprovalRequirement` rows. It returns
`RequirementDefinition[]` (§1) — pure intent, no round-local identity.

F3.1.2 turns each `RequirementDefinition` plus its resolved
`PrincipalResolutionSnapshot` (§7) into a persisted `ApprovalRequirement` by
adding exactly the fields that are round-local, not policy-derived:
`requirementId` (generated), `createdAt` (server time), `source: 'policy'`,
`sourceRef: requirementRole`, `sourceProvenance: policyProvenance`,
`principalSnapshot` (from selector resolution), and `decision` (initially
absent).

Deliberately **not** added to `RequirementDefinition`: any field that only
exists once a round is created. This is the "policy intent vs. round-local
persisted evidence" split the review packet asks for — challenged and kept
minimal rather than mirroring the persisted shape field-for-field.

## 6. Policy version registry (recommended)

```ts
function createPolicyRegistry(
  versions: AuthorizationPolicyVersion[],
): { get(key: string, version: string): AuthorizationPolicyVersion } {
  const map = new Map<string, AuthorizationPolicyVersion>();
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
reviewable). Duplicate `key@version` is a **startup** failure (§21).

## 7. Selector model

Selectors remain generic configuration identities, exactly as ADR-009 lists:
`normal-primary-approver`, `emergency-approver-a`, `emergency-approver-b`,
`cab-authority`. No corporate titles, employee names, or e-mail addresses
anywhere in the selector domain.

```ts
type SelectorEntry = {
  principalType: PrincipalType;  // 'user' | 'authority'
  principalRef: string;          // Catalog entity ref, e.g. 'group:default/cab-authority'
};

type SelectorBundle = {
  selectorBundleKey: string;
  selectorBundleVersion: string;
  contentDigest: string;         // sha256Canonical of the bundle content, excluding this field
  selectors: Record<string, SelectorEntry>;
};
```

A **resolved** selector, as consumed by the F3.1.0 `PrincipalResolutionSnapshot`
type (already persisted, unchanged):

```ts
type PrincipalResolutionSnapshot = {
  selectorKey: string;
  selectorVersion: string;       // = selectorBundleVersion (see §8)
  principalType: PrincipalType;
  resolvedPrincipalRef: string;
  resolvedAt: string;
  resolverProvenance: string;
  selectorDigest: string;
};
```

## 8. Selector versioning — one immutable bundle, not per-selector versions

**Recommended: version selectors as one immutable bundle**, not individually.

Rationale, read directly off the persisted schema: `AuthorizationRound` carries
exactly one `selectorBundleKey`/`selectorBundleVersion`/`selectorBundleSha256`/
`selectorBundleProvenance` per round (not one per selector). Per-selector
independent versioning would have no persisted home at the round level and
would only complicate the "what changed together" audit story. A bundle gives
one deploy-time unit: rebinding any selector requires bumping the whole
bundle's version, which is exactly the granularity F3.1.0 already committed to
storing.

Per-entry provability is *not* lost: each requirement's own
`selector_digest` (already a persisted, non-nullable column) is
`sha256Canonical({ selectorKey, selectorVersion, principalType,
principalRef })` — the individual selector's resolved content, digested
per-requirement even though the version number is shared bundle-wide. A future
round can therefore prove exactly what one selector resolved to without
needing per-selector version numbers.

`selectorVersion` on each `PrincipalResolutionSnapshot`/`ApprovalRequirement`
= the bundle's `selectorBundleVersion` at resolution time.

## 9. Principal types

Two `PrincipalType` values, matching the existing persisted
`principal_type CHECK IN ('user','authority')` column:

| `PrincipalType` | Resolves to | ADR-009 requirement kind |
|---|---|---|
| `user` | Catalog `User` entity ref | `individual` |
| `authority` | Catalog `Group` entity ref | `authority` or `cab` |

Mapping enforced by the resolver: `RequirementDefinition.kind === 'individual'`
requires `requiredPrincipalType === 'user'`; `kind ∈ {'authority','cab'}`
requires `requiredPrincipalType === 'authority'`.

- No e-mail address is ever snapshotted — only the Catalog entity ref.
- An authority is **never** exploded into N individual members at resolution
  or persistence time; the round snapshots the authority ref itself, matching
  ADR-009's "retains the group/authority ref rather than expanding it to N
  required individuals."
- Whether the *actual future decision actor* is currently authorized to act
  for a snapshotted authority is **decision-time membership resolution** —
  explicitly out of scope for F3.1.1 (belongs to F3.1.3/F3.1.4, per §16 below).

## 10. Catalog integration boundary

Reuses the exact pattern already proven in
[`executionPlanValidator.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/executionPlanValidator.ts):
`parseEntityRef` for syntax/kind, `catalog.getEntityByRef(ref, { credentials })`
for existence.

**Credentials:** resolve with `auth.getOwnServiceCredentials()` (the pattern
already used in `orgCatalogBootstrapModule.ts`,
`teamProjectGroupDisplayModule.ts`, `staleEntraGovernanceGroupCleanupModule.ts`,
`adoProjectAccessPlugin.ts`), **not** the requesting user's credentials. A
requester submitting a Change must not need read visibility of the
`cab-authority` group for their own submission to resolve correctly, and
selector resolution must be deterministic and requester-independent.
`coreServices.auth` is not yet a dependency of `changeManagementPlugin.ts` —
adding it is in scope for slice F3.1.1b (§18).

**Two validation moments, deliberately split:**

| When | What | Why |
|---|---|---|
| **Startup** (pure, no I/O) | Selector ref syntax parses; declared `principalType` is consistent with a resolvable Catalog kind (`user`→`User`, `authority`→`Group`); no duplicate selector keys in the bundle; every `selectorKey` a registered policy version references exists in the active bundle; bundle `contentDigest` matches computed digest | Fully deterministic, catches misconfiguration before serving traffic |
| **Submission** (I/O, via Catalog) | Selector's Catalog entity actually exists; entity's actual `kind` matches the declared `principalType`'s expected kind | Catalog reachability/content is external, transient state — cannot be a boot-time hard dependency without turning a temporary Catalog outage into an application crash loop |

**Answers to the boundary questions:**

- Should User refs be validated against Catalog? **Yes**, at submission
  (existence) and startup (ref syntax/declared kind).
- Should authority refs be Backstage Group refs? **Yes.**
- How is entity kind validated? `parseEntityRef(ref).kind` checked at startup
  against the selector's declared `principalType`; the *actual* Catalog
  entity's kind checked again at submission (config can drift from reality
  between startup and any given submission).
- Catalog temporarily unavailable? Submission **rejects fail-closed** with
  `PROVIDER_UNAVAILABLE` (503) — retryable, not a silent skip.
- Selector points to a missing entity? Submission **rejects fail-closed** with
  `NOT_FOUND` (404).
- Selector resolves to the wrong kind? This means startup validation and
  Catalog reality have drifted (e.g. someone changed the entity's kind after
  deploy) — submission **rejects fail-closed** with `INTERNAL_ERROR` (500),
  because this is a misconfiguration the requester cannot fix.
- Should submission fail closed? **Yes, unconditionally** — confirmed; no
  exception.

## 11. Resolution vs. membership

Selector **resolution** (F3.1.1's scope): `cab-authority` →
`group:default/cab-authority`. A pure Catalog-ref lookup, snapshotted once at
submission.

Authority **membership** (explicitly **not** F3.1.1): whether
`user:default/alice`, the actual authenticated actor recording a decision, may
currently act for `group:default/cab-authority`. That check happens at
decision-record time and belongs to F3.1.3 (decision command) /F3.1.4
(permission boundaries). F3.1.1 does not implement, stub, or partially
implement CAB decision authorization.

## 12. Emergency A/B separation of duty

**Recommendation: F3 MVP requires `emergency-approver-a` and
`emergency-approver-b` to declare `requiredPrincipalType: 'user'`.** This is a
startup-enforced constraint on the policy's `RequirementDefinition`s for those
two roles, not a selector-bundle constraint (the bundle only says what a
*named* selector currently resolves to; the policy is what says A and B must
be individuals).

**Why:** ADR-009 requires A and B to "produce distinct human decision actors
in the same round" and states "resolution to the same effective person fails
submission closed." Two distinct *authority/group* refs prove nothing about
the humans who will eventually act for them — an authority-typed A/B pair
cannot be proven distinct at either submission or decision time without
building authority-membership overlap detection, which ADR-009 defers past
F3.1. Restricting A and B to `user` kind makes distinctness a directly
provable equality check on two Catalog `User` refs.

**What F3 MVP allows for emergency A/B selectors:**

- Both selectors must resolve to `PrincipalType: 'user'`.
- Both selectors must be distinct selector *keys* in the bundle (a
  configuration cannot alias `emergency-approver-a` and
  `emergency-approver-b` to the same selector key) — startup check.

**What can be validated at submission (F3.1.2, using F3.1.1's resolver):**

- The two resolved `resolvedPrincipalRef` values differ. If they resolve to
  the same `User` ref, submission fails closed
  (`ChangeManagementError('INTERNAL_ERROR', ...)` — this is a selector
  misconfiguration, not a bad request).
- Both requirements are snapshotted sharing one `separationOfDutyKey` (e.g.
  `emergency-ab`), so the constraint is provable from the persisted round
  alone without re-resolving selectors.

**What still must be validated at decision time (F3.1.3, not F3.1.1):**

- A single authenticated `actorRef` must not be permitted to record `approved`
  or `rejected` for two requirements in the same round that share a
  `separationOfDutyKey`. This is a decision-command-boundary check, over
  `actorRef` rather than `resolvedPrincipalRef` — even though F3.1.1 makes the
  two principal refs provably distinct at submission, F3.1.3 must still guard
  against, e.g., a delegated/impersonated actor scenario at decision time.
  Documented here as a carried-forward requirement on the F3.1.3 command
  boundary, not implemented now.

**How a same-human A+B attempt fails closed:** at submission, because A and B
must resolve to distinct `User` refs by construction (§ above) — there is no
same-human A+B state reachable past submission in F3 MVP, since both are
individual selectors resolved independently.

**Ambiguity surfaced for architecture review — not resolved by amending
ADR-009:** ADR-009 does not explicitly forbid A/B being authority-typed; this
plan *narrows* the MVP to individuals only, as the smallest closed
implementation that provably satisfies the stated "distinct human decision
actors" requirement. If review decides authority-typed emergency
approvers must be supported (Q28 in the review packet: "What if A/B selectors
point to authorities with overlapping membership?"), that is a genuine
architecture decision — not representable safely in F3 MVP without either (a)
an authority-membership-overlap check at submission (a new capability) or (b)
deferring the distinctness proof entirely to decision time. That would warrant
its own reviewed ADR addendum (see §19), not a silent change here.

## 13. CAB semantics

CAB remains one collective authority, never N individual approvals:

- `kind: 'cab'`, `requiredPrincipalType: 'authority'`, `selectorKey:
  'cab-authority'` → resolves to one Group ref.
- The policy produces exactly **one** `RequirementDefinition` with
  `kind: 'cab'` for normal medium/high (`phase: 'pre_execution'`) and one
  separate `RequirementDefinition` with `kind: 'cab'` for the emergency
  retrospective (`phase: 'post_execution'`, `requirementRole:
  'emergency-cab-retrospective'`, carrying `sla`).
- No attendance tracking, voting, quorum engine, or one-decision-per-member
  semantics — matches ADR-009's "CAB does not require one click per
  participant by default."

## 14. Emergency retrospective SLA configuration

The mandatory post-execution CAB retrospective's SLA facts are carried as a
field of the *policy version*, not a separately published artifact:

```ts
sla: {
  policyKey: 'default-change-authorization',
  policyVersion: '2026-09-02.1',
  durationSeconds: 259200,       // example only — not prescribed here
  anchor: 'execution_completion',
}
```

This maps directly onto the F3.1.0 `ApprovalRequirement` columns already
present and nullable: `sla_policy_key`, `sla_policy_version`,
`sla_duration_seconds`, `sla_anchor` (checked `IN ('execution_completion')`).
Changing the SLA duration therefore *requires* publishing a new policy
version — which is the correct governance behavior per ADR-009 (SLA duration
is "deliberately a published policy value, not an open domain decision"), and
avoids introducing a third independently-versioned artifact beyond policy and
selector bundle.

No scheduler, no escalation automation, no execution-completion timestamp
fabrication — F3.1.1 supplies only the rule metadata; the existing
`evaluateGovernance` in
[`evaluators.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/authorization/evaluators.ts)
already derives overdue status from exactly these snapshotted fields plus a
future `executionCompletedAt`, unmodified by F3.1.1.

## 15. Additional mandatory requirements — compatibility only

F3.1.1 does **not** implement the additive-requirement submission path. The
policy model is shaped so F3.1.2 composition works without rework:

```
effective requirements = policy-generated RequirementDefinition[]
                        + authorized additional RequirementDefinition[]
```

Both use the identical `RequirementDefinition` shape; `source: 'additional'`
vs. `source: 'policy'` is set by F3.1.2 when persisting, not by the policy
evaluator. Additional requirements may only *add* — the policy evaluator's
output is never filtered, replaced, or reordered by anything downstream in
this design.

## 16. Failure model

| Failure | Prevents **startup** | Rejects **submission** |
|---|---|---|
| No active policy configured / unknown `activePolicy` pin | ✅ | |
| Duplicate `policyKey@version` registered | ✅ | |
| Invalid/malformed policy version (matrix not total over classification×risk) | ✅ | |
| Unsupported classification/risk combination | ✅ (caught by totality check) | |
| Missing selector (policy references a `selectorKey` absent from bundle) | ✅ | |
| Duplicate selector key in bundle | ✅ | |
| Invalid selector bundle version / missing `contentDigest` | ✅ | |
| Selector `contentDigest` mismatch (content changed, version didn't) | ✅ | |
| Emergency A/B not both `requiredPrincipalType: 'user'`, or same selector key | ✅ | |
| Malformed post-execution SLA config (missing duration/anchor when `sla` present) | ✅ | |
| Missing Catalog User | | ✅ `NOT_FOUND` (404) |
| Missing Catalog Group | | ✅ `NOT_FOUND` (404) |
| Wrong principal entity kind at Catalog (drifted since startup) | | ✅ `INTERNAL_ERROR` (500) |
| Catalog temporarily unavailable | | ✅ `PROVIDER_UNAVAILABLE` (503, retryable) |
| Emergency A/B resolve to the same individual | | ✅ `INTERNAL_ERROR` (500) |
| Configuration changed without version bump | ✅ (via `contentDigest` mismatch) | |

Startup failures are **deterministic misconfiguration** — always wrong,
regardless of any particular submission — so the application must refuse to
serve any traffic. Submission failures are **externally-discovered state**
(Catalog content, entity existence) that can only be known per-request, so
only that submission is rejected; the application keeps serving other
requests.

No `default-approve`, `no-selector-means-skip`, or `catalog-failed-continue`
path exists anywhere in this design — every branch above either prevents
startup or rejects the specific submission.

## 17. Immutability / publication / content-integrity

A published policy or selector bundle version is immutable: if content
changes, the version **must** change. Enforcement is differentiated by where
each artifact lives, because the two have different tamper surfaces:

| Artifact | How it can change | Integrity check |
|---|---|---|
| Policy version (compiled TypeScript) | Only via code review + redeploy | **Test-time**: a frozen `key@version → sha256Canonical(matrix)` digest table in the test suite; editing a published version's file without bumping its exported `version` fails CI |
| Selector bundle (app-config) | By anyone with deploy/config access, without a code review | **Runtime startup**: the bundle's declared `contentDigest` must equal `sha256Canonical` of the bundle's `selectors` map; a config edit that bumps content but not `selectorBundleVersion` (or vice versa) fails application startup |

A uniform runtime digest-pin on compiled code would be pure ceremony — the
code cannot be edited without a deploy, which is already gated by review. A
test-only check on production configuration would be useless, since
configuration is exactly the artifact most likely to be edited outside a
reviewed code change. This differentiated design directly answers §22's "same
key/version, different content" concern at the point where it can actually
occur.

## 18. Hashing / canonicalization

No second canonicalization scheme. Every hash in F3.1.1 reuses
[`canonical.ts`](../../../platform-devops-developer-portal/packages/backend/src/modules/changeManagement/authorization/canonical.ts)
verbatim (`canonicalJson`, `sha256Canonical`, `assertSha256`):

| Value | Computed as |
|---|---|
| `policyArtifactSha256` | `sha256Canonical(policyVersion.requirements-generating-matrix content)` — frozen per version, checked in a unit test (§17) |
| `policyInputSha256` | `sha256Canonical({ classification, risk })` |
| `selectorBundleSha256` / bundle `contentDigest` | `sha256Canonical(bundle.selectors)` |
| `selectorDigest` (per requirement) | `sha256Canonical({ selectorKey, selectorVersion, principalType, principalRef })` |

## 19. F3.1.0 contract integration — field mapping

Every F3.1.1 output field maps onto an existing F3.1.0 persisted column; no
mismatch requiring a future schema adjustment was found.

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
| `RequirementDefinition.sla.*` | `ApprovalRequirement.slaPolicyKey/slaPolicyVersion/slaDurationSeconds/slaAnchor` | ″ |
| policy provenance (for `sourceProvenance`) | `ApprovalRequirement.sourceProvenance` | ″ |

One mapping nuance worth flagging explicitly (not a mismatch, a translation
F3.1.2 must apply): the persisted `kind` column has three values
(`individual`/`authority`/`cab`) while `principal_type` has two
(`user`/`authority`). The resolver enforces `kind === 'individual' ⇔
principalType === 'user'` and `kind ∈ {'authority','cab'} ⇔ principalType ===
'authority'` — this rule belongs in F3.1.1b's resolver and should be asserted
by a startup check across every registered policy version against the active
bundle.

## 20. No database by default

**Confirmed: F3.1.1 adds no database tables, no migration.** The F3.1.0 schema
(§19 mapping) already carries every field F3.1.1's output produces. Policy
versions and the selector bundle are frozen in-memory artifacts assembled at
startup from code and config; the ledger already exists to snapshot the
resulting *historical evidence* once a round is created (F3.1.2). A mutable
policy database would reintroduce exactly the "second policy authority"
ADR-009 and the review packet both explicitly reject.

## 21. Operational model

```
policy matrix / selector config change
  → code review (policy) or config review (selectors)
  → merge to main / deploy
  → registry rebuilt at next application startup
  → new submissions may select the new version (only if activePolicy pin also updated)
  → existing rounds remain bound to their recorded version — untouched
```

- **Rollback**: revert `activePolicy` to an older, still-registered
  `key@version` — a config-only change, no code rollback needed as long as the
  older version's file is still present in the registry (it always is, since
  versions are never deleted).
- **Startup validation**: registry build, bundle digest check, A/B
  individual-kind check, selector-coverage check, SLA-shape check — all fail
  fast (§16).
- **Observability**: see §22.
- **Audit of what version was selected**: the `AuthorizationRound` record
  itself is the audit trail — `policyKey`/`policyVersion` persisted per round,
  immutable once written; no separate "which version served this request"
  log is required beyond the structured event at selection time (§22).

No administrative UI is built. Who is allowed to *approve* a policy-content
change is an organizational/PR-review process, not an application feature —
consistent with §23's ownership split below.

## 22. Configuration ownership

- **Platform/DevOps** authors and operates the policy TypeScript and selector
  bundle configuration — same team that owns the rest of
  `packages/backend/src/modules/changeManagement`.
- **Governance authority** reviews and approves *publication* — in practice,
  required PR review/approval on the commit that adds a policy version or
  changes the selector bundle. This is a process control (branch protection,
  required reviewers), not an application-level permission in F3.1.1 — no
  `change.authorization.policy.admin` capability is implemented here (that
  belongs to F3.1.4's permission boundary work, if a runtime admin surface is
  ever built at all).
- No corporate title (CTO, director, manager) is encoded anywhere in code or
  configuration — enforced by an architecture-guard test addition (§ Test
  strategy G).

## 23. Observability

Structured, low-cardinality event names, no personal data beyond stable
principal refs:

- `change.authorization.policy.selected` — `{ policyKey, policyVersion, changeId candidate }`
- `change.authorization.selector.resolved` — `{ selectorKey, selectorVersion, principalType, resolvedPrincipalRef }`
- `change.authorization.selector.resolution_failed` — `{ selectorKey, reason }`
- `change.authorization.publication.invalid` — startup-time, `{ artifact: 'policy'|'selector-bundle', reason }`
- `change.authorization.policy.evaluation_failed` — `{ policyKey, policyVersion, reason }`
- `change.authorization.selector.entity_missing` — `{ selectorKey, principalRef }`
- `change.authorization.separation_of_duty.violation` — `{ separationOfDutyKey, resolvedPrincipalRef }`

Never logged: e-mail addresses, employee names, job titles. Only Catalog
entity refs, selector keys, and policy identity.

## 24. Security analysis

| Threat | Mitigation |
|---|---|
| Config tampering (selector bundle edited without version bump) | `contentDigest` startup check (§17) |
| Selector mapping silently changed in prod without review | Same digest check; bundle changes should also go through the same PR-review process as code (organizational control, §23) |
| Version reuse (same `key@version`, different content) | Test-time digest for policy (§17); startup digest for selectors (§17) |
| Spoofed/malformed Catalog refs in config | Startup ref-syntax + declared-kind check (§10); submission-time actual-entity-kind check |
| Selector entity missing at submission | Fail closed `NOT_FOUND` (§16) |
| Authority confusion (authority resolved where individual required, or vice versa) | Startup enforcement of `kind ⇔ principalType` (§19); emergency A/B individual-only constraint (§12) |
| Startup with malformed policy | Registry build fails, application does not start (§16) |
| Accidental permissive fallback | None exists — every failure path in §16 is closed, none silently allow |

No "default approve," "no selector means skip," or "Catalog failed, continue
anyway" path exists in this design (re-confirmed).

## 25. Persistence decision

**No new database tables. No new routes. No new UI.** Confirmed against §20;
the persisted-field mapping in §19 shows the existing F3.1.0 schema is
sufficient.

## 26. Test strategy

**A. Policy matrix**

- `normal + low` → one `individual` requirement, `pre_execution`.
- `normal + medium` → `individual` + `cab`, both `pre_execution`.
- `normal + high` → `individual` + `cab`, both `pre_execution`.
- `emergency` → two `individual` `pre_execution` requirements
  (`emergency-approver-a`, `emergency-approver-b`, sharing a
  `separationOfDutyKey`) + one `cab` `post_execution` requirement carrying
  `sla`.

**B. Determinism** — same immutable `PolicyInput` + same policy version ⇒
byte-identical `policyInputSha256` and identical `RequirementDefinition[]`
across repeated evaluations.

**C. Version isolation** — two published versions of the same `policyKey`
produce distinct `policyArtifactSha256`/`matchedRuleProvenance`; the older
version remains retrievable via `registry.get` and evaluable identically to
when it was active.

**D. Invalid publication** — registering two versions with the same
`key@version` string fails registry construction; a selector bundle whose
`contentDigest` doesn't match its `selectors` content fails startup.

**E. Selector resolution**

- Valid `User` selector resolves successfully.
- Valid `Group`/authority selector resolves successfully.
- Missing selector key (referenced by policy, absent from bundle) fails
  startup.
- Wrong entity kind (Catalog entity is `Component`, selector declared `user`)
  fails at startup (declared-kind check) and/or submission (actual-kind
  check).
- Missing Catalog entity fails submission `NOT_FOUND`.

**F. Emergency separation** — A and B selectors resolving to the same `User`
ref fails submission closed; distinct refs succeed and share a
`separationOfDutyKey`.

**G. Generic semantics guard** — extend `architecture.test.ts`'s guarded
source list to the new `policy/` and selector-resolver files; assert no
job-title tokens (`cto|director|manager|superintendent`, case-insensitive) and
no e-mail-shaped literal appears in any guarded source.

**H. No provider leakage** — extend the existing `FORBIDDEN_FIELDS` guard
(`repositoryId`, `pipelineId`, `adoApprovalId`, `teamsUserId`, etc.) over the
new policy/selector source files; assert no ADO/Teams-specific identifier
appears in `PolicyInput`, `RequirementDefinition`, or `SelectorBundle`.

**I. SLA configuration validation** — a `RequirementDefinition` with `sla`
present but missing `durationSeconds` or an `anchor` outside
`'execution_completion'` fails startup validation.

**J. Regression** — all 15 existing F3.1.0 Change Management suites remain
green and are not modified by F3.1.1 changes (excluding the
`architecture.test.ts` guard extensions in G/H, which extend rather than
alter existing assertions).

## 27. Performance / caching

Policy evaluation is pure/in-memory — no caching needed or beneficial.
Selector bundle lookup is an in-memory map read — no caching needed. Catalog
validation involves real backend service calls; **no caching is introduced in
F3.1.1** — 2–3 lookups per submission is an acceptable cost, and caching is
precisely the mechanism by which a stale principal could leak into an
otherwise-immutable round snapshot. If a future slice needs Catalog-read
caching for load reasons, that is a separate, explicitly-scoped decision, not
a default here.

## 28. F3.1.1 decomposition

The prompt's suggested three-way split (a: policy domain + registry; b:
selector config + resolver; c: validation + provenance contract) is
**rejected as over-decomposed**. "Validation + provenance contract" is not
independently testable — provenance fields are produced by *both* the policy
and selector slices and are consumed only by F3.1.2's round-creation code, so
a standalone slice for it would have no independent I/O boundary or
acceptance test of its own; it would just be a shared-types PR wedged between
the two real slices.

**Recommended: two independently testable, non-overlapping slices.**

| Slice | Scope | I/O boundary |
|---|---|---|
| **F3.1.1a — Policy domain + registry** | `PolicyInput`, `RequirementDefinition`, `PolicyEvaluationResult` types; the published `default-change-authorization@2026-09-02.1` version implementing the ADR-009 §6 matrix; `createPolicyRegistry`; pure `evaluatePolicy`; frozen per-version content digests | None — pure, no config read, no Catalog, no DB |
| **F3.1.1b — Selector bundle + resolver** | `SelectorBundle`/`SelectorEntry` types; `readAuthorizationConfig(config)` reader (app-config → bundle); startup validation (digest, coverage, A/B individual-kind, duplicate keys); Catalog-backed resolver producing `PrincipalResolutionSnapshot`; plugin wiring to add `coreServices.auth` as a dependency | Config + Catalog (via `auth.getOwnServiceCredentials()`) |

Both slices remain fully unwired from `POST /changes`; F3.1.2 is the only
future slice that calls either of them from `ChangeManagementService`.

## 29. First implementation slice — F3.1.1a (recommended starting point)

**This section describes the smallest next authorized-implementation
candidate. It is not authorized by this planning checkpoint. A separate,
constrained implementation prompt must explicitly authorize it before any
code is written.**

### Objective

Implement a pure, deterministic, I/O-free authorization policy evaluator and
its published-version registry, matching the ADR-009 §"Normal-change policy
baseline" and §"Emergency policy baseline" matrices exactly, with full unit
test coverage and zero integration into the F2 submission path.

### Exact code boundaries

New directory:
`packages/backend/src/modules/changeManagement/authorization/policy/`

New files:

- `types.ts` — `PolicyInput`, `RequirementDefinition`, `PolicyEvaluationResult`,
  `AuthorizationPolicyVersion` (as specified in §1 of this plan).
- `versions/default-change-authorization.2026-09-02.1.ts` — one exported
  `AuthorizationPolicyVersion` implementing the four-row matrix (§1/§26.A),
  `Object.freeze`d.
- `registry.ts` — `createPolicyRegistry` (§6).
- `evaluatePolicy.ts` — thin wrapper: `(policyVersion, input) =>
  policyVersion.evaluate(input)`, plus `policyInputSha256` computation via
  `canonical.ts`.
- `digests.ts` — a frozen `Record<'key@version', sha256>` table checked by a
  test (§17, §26.D).

Test files (co-located, matching existing repo convention):

- `versions/default-change-authorization.2026-09-02.1.test.ts` — matrix
  coverage (§26.A), determinism (§26.B).
- `registry.test.ts` — duplicate detection, unknown-version lookup (§26.D).
- `digests.test.ts` — content-integrity pin (§17, §26.D).

### Expected types/interfaces

Exactly the types in §1 and §6 of this plan — no additions beyond what's
specified there.

### Source paths likely involved

All new files listed above, under
`packages/backend/src/modules/changeManagement/authorization/policy/`.

**One existing file extended, not rewritten:**
`packages/backend/src/modules/changeManagement/architecture.test.ts` — extend
the `sources` array (currently `types.ts`, `evaluators.ts`,
`AuthorizationLedgerRepository.ts`, `KnexAuthorizationLedgerRepository.ts`) to
also read the new `policy/` files, and add two new guard assertions: no
job-title tokens, no e-mail-shaped literals (§26.G). The existing
`FORBIDDEN_FIELDS` check already applies to any file added to that `sources`
array, so it extends for free.

**No other existing file is modified.** In particular, `ChangeManagementService.ts`,
`changeManagementPlugin.ts`, `types.ts` (the top-level module one, not the new
policy one), `errors.ts`, and every migration file are untouched by this
slice.

### Non-goals for this slice

- No app-config read.
- No Catalog/`coreServices.auth` dependency.
- No selector bundle, no selector resolver (that is F3.1.1b).
- No `ChangeManagementService` change.
- No plugin wiring change.
- No new migration.
- No route change.

### Acceptance criteria

- All items in §26.A–D pass (matrix, determinism, version isolation, invalid
  publication).
- `architecture.test.ts` guard extensions pass, including on the new `policy/`
  sources.
- All 15 pre-existing F3.1.0 Change Management suites remain green and
  unmodified.
- `yarn workspace backend lint` passes.
- `yarn workspace backend build` passes.
- Repository-wide `tsc --noEmit` error set is **set-identical** (by
  file/line/column/code, not just count — per the F3.1.0-H precedent) to the
  5 pre-existing errors present at `4bad41d`.

### STOP condition

Stop immediately once the acceptance criteria above are met and verified.
**Do not** begin F3.1.1b, do not touch `ChangeManagementService.ts` or
`changeManagementPlugin.ts`, do not add app-config keys, do not add a Catalog
dependency. Report results and wait for the next explicit authorization.

## 30. F3.1.2 prerequisites (carried forward, not addressed here)

1. **`buildChange()` called twice** in `ChangeManagementService.createChange`
   (`packages/backend/src/modules/changeManagement/ChangeManagementService.ts`,
   calls at the two `buildChange` sites feeding the index snapshot and the
   provider snapshot respectively) — causes divergent server-generated
   `activityId`/`createdAt` between the two snapshots. **Must be fixed before
   F3.1.2.** Not touched by F3.1.1 planning or by the F3.1.1a slice.
2. **Cross-cutover idempotency invariant** — an existing reservation with
   `authorization_mode = 'LEGACY_PRE_F3'` must resume legacy F2 submission
   semantics on retry; only a genuinely *new* reservation may select
   `LEDGER_REQUIRED`. F3.1.1's policy/selector evaluation must only ever be
   invoked from the future ledger-aware (`LEDGER_REQUIRED`) submission path
   that F3.1.2 builds — never from a retry of a legacy reservation. This plan
   introduces no submission-path wiring, so the invariant is unaffected by
   F3.1.1 itself, but F3.1.2 must respect it when it calls into F3.1.1's
   registry/resolver.
3. RBAC CSV / conditional-policy files referenced by committed app-config
   remain absent from ADO HEAD — an F3.1.4 prerequisite, unrelated to
   F3.1.1/F3.1.2.

## 31. Required architecture review answers

| # | Question | Answer |
|---|---|---|
| 1 | Verified ADO baseline SHA | `4bad41d058edf5c5314d17275e0c8bdb5abf690f` |
| 2 | Was `current-state.md` corrected to `4bad41d`? | Yes |
| 3 | What is an `AuthorizationPolicy` in F3 MVP? | A published, immutable, deterministic TypeScript artifact mapping `{classification, risk}` to a `RequirementDefinition[]` — see §1 |
| 4 | Where are published policy definitions stored? | Versioned TypeScript modules in `authorization/policy/versions/`, registered in an in-memory registry at startup — see §2 |
| 5 | Why is that better than a database-backed rules engine? | No mutable second policy authority; code review is the approval gate; exhaustiveness-checked over closed unions; matches ADR-009's explicit rejection of a policy DSL — see §2 |
| 6 | What makes a policy version immutable? | Compiled code, changeable only via reviewed redeploy, plus a test-time frozen content digest per `key@version` — see §17 |
| 7 | How is version-content reuse detected/prevented? | Policy: test-time digest table (§17). Selectors: runtime startup `contentDigest` check (§17) |
| 8 | What exact Change facts are policy inputs? | `classification` and `risk` only — see §4 |
| 9 | What is the policy output? | `RequirementDefinition[]`, pure intent — see §5 |
| 10 | Does the policy directly persist ApprovalRequirements? | **No** — see §5 |
| 11 | How does a new submission select a policy version? | An explicit `activePolicy: {key, version}` config pin, resolved via the registry — see §3 |
| 12 | Can an existing round be reinterpreted under a newer policy? | **No** — a round stores its own policy identity permanently; the registry never re-evaluates history — see §3 |
| 13 | What is a selector? | A generic configuration identity resolving to a Catalog principal ref — see §7 |
| 14 | How are selectors versioned? | As one immutable bundle (`selectorBundleKey`/`Version`), not per-selector — see §8 |
| 15 | Where are selector definitions stored? | Validated `changeManagement.authorization` app-config, read via a `readAuthorizationConfig` function — see §2, §10 |
| 16 | What principal types are supported? | `user` and `authority` — see §9 |
| 17 | Are emails canonical principals? | **No** | 
| 18 | Are corporate job titles canonical semantics? | **No** |
| 19 | How is a User selector validated? | Startup ref-syntax/declared-kind check + submission-time Catalog existence/actual-kind check — see §10 |
| 20 | How is an authority selector validated? | Same two-stage validation, expecting a `Group` ref — see §10 |
| 21 | What happens when Catalog resolution fails? | Submission rejects fail-closed: `NOT_FOUND` (missing entity) or `PROVIDER_UNAVAILABLE` (transient) — see §10, §16 |
| 22 | Is Catalog group membership snapshotted at round creation? | **No** — only the authority ref itself is snapshotted — see §9 |
| 23 | When is authority membership checked? | At decision-record time, in F3.1.3/F3.1.4 — not F3.1.1 — see §11 |
| 24 | How is CAB represented? | One `kind: 'cab'` requirement resolving `cab-authority` to one Group ref — see §13 |
| 25 | Does CAB mean one approval per member? | **No** — see §13 |
| 26 | How are emergency A/B separation-of-duty rules enforced? | Startup: both must be `user`-typed, distinct keys. Submission: resolved refs must differ. Decision time (F3.1.3): one actor may not decide both requirements sharing a `separationOfDutyKey` — see §12 |
| 27 | Can A and B resolve to the same individual? | **No** — fails submission closed — see §12 |
| 28 | What if A/B selectors point to authorities with overlapping membership? | Not representable in F3 MVP — A/B are restricted to individual (`user`) selectors precisely to avoid this case; flagged as an open question for architecture review, not resolved by amending ADR-009 — see §12, §32 |
| 29 | What retrospective SLA facts come from policy? | `slaPolicyKey`, `slaPolicyVersion`, `slaDurationSeconds`, `slaAnchor` — carried as a field of the policy version — see §14 |
| 30 | Does F3.1.1 implement SLA scheduling? | **No** — see §14 |
| 31 | Does F3.1.1 add database tables? | **No** — see §20 |
| 32 | Does it add routes? | **No** |
| 33 | Does it add UI? | **No** |
| 34 | Does it integrate Teams? | **No** |
| 35 | Does it integrate Azure DevOps? | **No** |
| 36 | How does the design reuse F3.1.0 canonical hashing/provenance? | Every hash uses `canonicalJson`/`sha256Canonical` from `authorization/canonical.ts` verbatim — see §18 |
| 37 | What failures prevent startup? | Registry/duplicate errors, bundle digest mismatch, selector coverage gaps, A/B individual-kind violation, malformed SLA shape — see §16 |
| 38 | What failures reject a future submission? | Missing/unavailable Catalog entity, wrong entity kind at Catalog, A/B resolving to the same individual — see §16 |
| 39 | What is the proposed F3.1.1 decomposition? | Two slices: F3.1.1a (policy domain + registry, pure) and F3.1.1b (selector config + Catalog resolver) — see §28 |
| 40 | What is the FIRST implementation slice? | F3.1.1a — see §29 |
| 41 | What tests will gate that first slice? | §26.A–D plus extended `architecture.test.ts` guards — see §29 |
| 42 | What remains explicitly out of scope? | Selector resolution runtime, Catalog integration, config reading, submission integration, decision commands, permissions, frontend, Teams, ADO — see §29, and the top-level scope statement |
| 43 | Is buildChange-twice still a prerequisite for F3.1.2? | **Yes** — see §30 |
| 44 | Is the cross-cutover idempotency invariant preserved? | **Yes** — unaffected by this planning-only checkpoint; explicitly carried forward for F3.1.2 to respect — see §30 |
| 45 | Was any ADO source modified during this planning checkpoint? | **No** |
| 46 | Was F3.1.1 implementation performed? | **No** |
| 47 | What backstage-docs baseline SHA was used? | `f4dc268b6f7ac09600710e59d9c7d35d918fdc15` |
| 48 | What backstage-docs SHA contains the completed plan? | Recorded in the handoff after this commit is pushed (see implementation-progress.md checkpoint) |
| 49 | Is F3.1.1 ready for implementation review? | Yes — this plan is ready for review of slice F3.1.1a |
| 50 | Is implementation authorized by this checkpoint? | **No** |

## 32. Open questions for architecture review

1. **Emergency A/B principal type.** This plan restricts
   `emergency-approver-a`/`emergency-approver-b` to `user`-typed selectors
   only, as the smallest closed way to satisfy ADR-009's distinct-human
   requirement. If architecture review wants authority-typed emergency
   approvers supported in a future slice, that requires either an
   authority-membership-overlap check (a new capability, likely at decision
   time) or an explicit acceptance that distinctness can only be proven at
   decision time, not submission. Recommend leaving this as an MVP
   narrowing unless review identifies a concrete near-term need for
   authority-typed A/B.
2. **`selectorBundleVersion` syntax.** Not prescribed here beyond "opaque,
   unique per content." If Platform/DevOps wants a specific convention (date-
   based, semver, etc.) to match the `policyVersion` convention, that's a
   config-authoring preference, not an architecture decision — can be settled
   at F3.1.1b implementation time.

Neither question requires or proposes amending ADR-009. If review resolves
question 1 in favor of authority-typed A/B, that decision would need its own
ADR (next available number per `docs/adr/README.md`) in a separate checkpoint
— not folded into this plan or into ADR-009 directly.

## Gate

**F3.1.1 architecture/implementation planning is COMPLETE and under review.**

**Recommendation: GO** for implementation review of the first slice,
**F3.1.1a — Policy domain + registry** (§29), subject to architecture
sign-off on the emergency A/B narrowing (§32, question 1).

**F3.1.1 implementation itself remains NOT AUTHORIZED** by this checkpoint. No
ADO source, migration, configuration, test, or runtime behavior was modified
to produce this plan. A separate, constrained implementation prompt must
explicitly authorize slice F3.1.1a before any code is written.
