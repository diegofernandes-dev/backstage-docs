# F3.1.1b — Selector Bundle, Catalog Resolver, and Publication Integrity

## Role

Act as a senior Staff+/Principal Backstage backend engineer implementing the **next bounded GMUD authorization slice: F3.1.1b**.

This is an implementation-and-evidence checkpoint. It is **not** a new architecture design exercise.

The accepted architecture is already defined by ADR-009 and the reviewed F3.1.1 plan. F3.1.1a is an accepted implemented baseline. Your task is to implement **only** the selector-bundle/config/Catalog-resolution half that was deliberately deferred from F3.1.1a.

Core rule:

> Implement F3.1.1b exactly as the canonical plan defines it, keep it completely unwired from `POST /changes`, and STOP before F3.1.2.

The accepted Delivery/GitOps architecture from ADR-012 remains a separate concern. The current production-rollout readiness gate is intentionally **deferred until an actual production target exists** and does not block continued platform construction in this checkpoint.

---

# 0. Canonical baseline protocol

Before editing anything:

1. In `diegofernandes-dev/backstage-docs`:

   ```bash
   git fetch origin main
   ```

2. Record the exact current `origin/main` SHA.

3. Read, in this order:

   ```text
   docs/backstage/current-state.md
   docs/backstage/implementation-progress.md
   docs/backstage/f3-1-1a-architecture-acceptance.md
   docs/backstage/f3-1-1-implementation-plan.md
   docs/adr/ADR-009-change-authorization-model.md
   docs/adr/ADR-012-delivery-management-gitops-promotion.md
   docs/delivery/README.md
   ```

4. In the **Azure DevOps implementation repository** `platform-devops-developer-portal`, verify the actual current state of branch:

   ```text
   feat/ado-repo-governance
   ```

   Do not assume the branch still points to the last documented SHA. Canonical orientation says F3.1.1a was published at:

   ```text
   d3c0751a15b908cec8f5595c97e52f41226344ed
   ```

   but live ADO state wins.

5. Confirm that the F3.1.1a implementation is present and matches the accepted architectural shape before extending it:

   - policy domain/data artifact;
   - `policyModelVersion: 1`;
   - generic evaluator;
   - registry;
   - `deepFreezeSerializable`;
   - `published-manifest.json`;
   - append-only manifest validator;
   - `validate:policy-publication` command;
   - accepted semantic architecture guards.

6. If the implementation branch has advanced materially beyond the accepted F3.1.1a baseline, reconcile the delta with canonical docs before editing. Do not overwrite unrelated work.

### Repository separation — hard rule

This checkpoint belongs to the GMUD implementation line:

```text
platform-devops-developer-portal / feat/ado-repo-governance
```

Do **not** implement this slice on or merge it with the Delivery branch:

```text
feat/delivery-mvp-slice
```

Do not move Delivery code into GMUD merely because ADR-012 now exists.

---

# 1. Authorization

## GO — only F3.1.1b

Authorized:

- selector-bundle domain types;
- selector-bundle config reader;
- active selector-bundle selection;
- startup validation of the **active policy + active selector bundle pair**;
- selector bundle canonical digest;
- selector publication identity and append-only manifest entry using the existing F3.1.1a manifest/validator;
- Catalog-backed principal resolver;
- service-credential-based Catalog lookup using `coreServices.auth`;
- minimal backend plugin dependency wiring needed by the resolver;
- semantic architecture guard extensions for selector artifacts;
- unit/integration tests required to prove the above;
- factual canonical evidence updates after implementation.

## NO-GO

Do not implement any of the following:

- F3.1.2 submission integration;
- `AuthorizationRound` creation from `POST /changes`;
- `ChangeManagementService.createChange` authorization wiring;
- the `buildChange()`-twice fix unless the canonical plan has explicitly moved that fix into F3.1.1b since this prompt was authored;
- approval/decision commands;
- approver membership checks;
- CAB Workbench;
- Teams integration;
- Azure DevOps execution enforcement;
- Delivery `ChangeBinding` changes;
- ReleaseCandidate/DeploymentTarget changes;
- lifecycle transitions;
- ExecutionEligibility transport changes;
- new migrations;
- new HTTP routes;
- frontend work;
- generic rules engine / DSL / BPM / workflow engine;
- policy-admin UI;
- selector-admin UI;
- caching of Catalog resolution;
- CI pipeline creation just to invoke publication validation;
- production rollout readiness work;
- Deployments UX work.

If you believe any prohibited item is necessary for F3.1.1b, STOP and report the concrete dependency. Do not silently expand scope.

---

# 2. Accepted architectural invariants

Preserve all of these exactly.

1. **Policy remains environment-independent reviewed TypeScript data.**
2. **Selector bindings are environment-scoped validated application configuration.**
3. **Selectors are generic identities**, not job titles, names, email addresses, or organizational semantics.
4. Principal types remain exactly:

   ```text
   user       -> Catalog User
   authority  -> Catalog Group
   ```

5. `ApprovalRequirement.kind` compatibility remains:

   ```text
   individual              -> principalType=user
   authority | cab         -> principalType=authority
   ```

6. An authority selector resolves to the **Group ref itself**. Do not expand the Group into members.
7. Resolution is not authorization-to-decide. Group membership / decision-actor authority belongs to later slices.
8. Emergency approver A/B remain **user-typed** in F3 MVP.
9. Emergency A/B must use distinct selector keys, and their resolved principals must be distinct when resolution is actually invoked.
10. Only the **active policy + active selector bundle** pair is validated for coverage at startup. An inactive historical policy must not crash startup because the current selector bundle lacks one of its old selectors.
11. No email address is canonical authorization identity or snapshot data.
12. No Azure DevOps, Teams, pipeline, environment, deployment, repository, or provider identifiers enter the canonical selector or policy domain.
13. Catalog resolution fails closed. No missing-selector, missing-Catalog, or provider-error path may silently continue.
14. No selector-resolution cache in this slice.
15. F3.1.1b remains completely unwired from Change submission.

---

# 3. Required F3.1.1b surface

Use the canonical F3.1.1 plan as normative for exact type names and field names. Do not invent alternate models when the plan already defines one.

The implementation must provide the bounded concepts below.

## 3.1 Selector bundle

Implement the plan-defined `SelectorBundle` / `SelectorEntry` model.

The bundle must have:

- stable bundle key;
- opaque version identity;
- provenance;
- selector entries;
- canonical `contentDigest` / `selectorBundleSha256` derived from **selector content**, using the existing canonical hashing primitives;
- immutable runtime representation using the existing F3.1.1a `deepFreezeSerializable` utility where applicable.

Do not introduce per-selector mutable versions if the canonical plan still specifies one immutable versioned bundle as the deployment-time unit.

## 3.2 Configuration reader

Implement the canonical:

```text
readAuthorizationConfig(config)
```

pattern using Backstage `Config`, following the repository's existing typed config-reader conventions.

The reader must make the active policy and active selector bundle explicit. Never select "latest" automatically.

Do not prescribe a new configuration schema if the canonical F3.1.1 plan already contains one; implement that shape.

Environment-specific Catalog refs belong here, not in policy source code.

## 3.3 Startup validation

At startup validate only the active pair.

Required checks include at minimum the canonical plan's checks:

- active policy configured;
- active policy exists in the shipped policy registry;
- active selector bundle configured;
- duplicate selector keys fail;
- selector-bundle digest is correct;
- shipped selector bundle has the required selector-bundle manifest entry;
- manifest digest matches the computed selector-bundle digest;
- every `selectorKey` referenced by the active policy exists in the active selector bundle;
- `individual <-> user` compatibility;
- `authority|cab <-> authority` compatibility;
- emergency A/B selector keys are distinct;
- emergency A/B are user-typed;
- malformed entity refs fail closed;
- forbidden canonical fields / email-shaped identity values fail the semantic guard.

Do **not** validate an inactive historical policy against the active selector bundle.

## 3.4 Catalog-backed resolver

Implement the canonical resolver that turns a selector into the already-defined F3.1.0 `PrincipalResolutionSnapshot`.

Use Backstage Catalog APIs and:

```text
coreServices.auth.getOwnServiceCredentials()
```

for backend-to-Catalog calls. Do not use the requester's user token to establish canonical principal resolution.

Resolver behavior must remain deterministic and fail closed:

- syntactically invalid configured ref -> configuration/startup failure where the plan assigns it;
- missing Catalog `User`/`Group` -> plan-defined not-found failure;
- wrong Catalog kind -> plan-defined internal/configuration failure;
- temporary Catalog/provider failure -> retryable provider-unavailable behavior;
- no email fallback;
- no display-name fallback;
- no member expansion for Groups;
- no cache.

Use `parseEntityRef` and existing Catalog access patterns already present in the repository. Reuse the existing execution-plan Catalog validation style where useful; do not create a second Catalog abstraction unless demonstrably necessary.

## 3.5 Selector digest

Reuse the existing canonical hashing implementation.

The canonical plan defines:

```text
selectorBundleSha256 / contentDigest = sha256Canonical(bundle.selectors)
```

and per resolved selector evidence:

```text
selectorDigest = sha256Canonical({
  selectorKey,
  selectorVersion,
  principalType,
  principalRef,
})
```

If the exact canonical field names in the live accepted F3.1.1 plan differ, the plan wins. Do not invent an additional digest scheme.

## 3.6 Publication integrity

Append selector-bundle identity to the **existing**:

```text
packages/backend/src/modules/changeManagement/authorization/policy/published-manifest.json
```

using:

```text
artifact = selector-bundle
key
version
digest = contentDigest
```

Reuse the F3.1.1a append-only history validator unchanged unless a concrete selector-bundle bug proves a narrowly-scoped change necessary.

### Critical rule: this is NOT genesis

F3.1.1a already created the publication manifest. Therefore F3.1.1b must validate as an ordinary append against the trusted accepted implementation baseline.

Do **not** use `--allow-genesis-from` merely to make validation pass.

Resolve the actual trusted F3.1.1a baseline commit first, then run:

```bash
yarn validate:policy-publication --baseline-ref <trusted-f3.1.1a-ref>
```

Expected semantics:

- all pre-existing policy manifest entries unchanged;
- new selector-bundle identity appended;
- no identity removed;
- no digest changed;
- no duplicate identity;
- no genesis path involved.

A negative test that edits the selector-bundle content while reusing the same `key@version` must fail via the existing append-only mechanism.

---

# 4. Principal-resolution snapshot boundary

F3.1.1b may construct the already-defined `PrincipalResolutionSnapshot` value, but it must **not persist a new AuthorizationRound**.

The snapshot must carry only provider-neutral, audit-relevant principal identity/provenance defined by ADR-009/F3.1.0.

Do not add:

- email;
- display name as identity;
- employee/job-title information;
- Entra object ID unless already canonically defined by the accepted model;
- Teams ID;
- ADO identity ID;
- mutable Catalog entity body;
- Group member lists.

The Catalog ref is the stable platform principal reference.

---

# 5. Plugin wiring boundary

Minimal backend wiring is authorized only to make the selector bundle and resolver constructible/testable.

The F3.1.1 plan explicitly expects plugin wiring for `coreServices.auth`.

Allowed:

- add `coreServices.auth` dependency where required;
- add/reuse Catalog client dependency already used by Change Management;
- instantiate/configure the F3.1.1b selector bundle/resolver services at plugin startup;
- run startup validation of the active policy + active selector bundle pair.

Not allowed:

- call selector resolution from `POST /changes`;
- create an `AuthorizationRound`;
- alter `ChangeManagementService.createChange` business behavior;
- add routes merely to exercise the resolver.

Tests may call the resolver directly.

---

# 6. Test contract

Add focused tests proving the implementation rather than broadening the architecture.

At minimum prove:

### Config / bundle

- valid active policy + bundle loads;
- duplicate selector key fails;
- missing active bundle fails;
- active policy references missing selector -> startup failure;
- inactive historical policy references missing selector -> startup succeeds;
- bundle digest deterministic;
- nested selector bundle mutation is prevented if bundle is registered/frozen;
- manifest/artifact digest agreement;
- selector bundle append validates against trusted F3.1.1a baseline;
- identity reuse with changed bundle content is rejected by publication-history validation.

### Type compatibility

- `individual -> user` accepted;
- `authority -> authority` accepted;
- `cab -> authority` accepted;
- incompatible pairs rejected;
- emergency A/B user-type narrowing enforced;
- emergency A/B duplicate key rejected.

### Catalog resolution

- User selector resolves a real/mocked Catalog `User` to the correct `PrincipalResolutionSnapshot`;
- authority/CAB selector resolves a Catalog `Group` ref without expanding members;
- backend uses service credentials (`getOwnServiceCredentials`) rather than requester credentials;
- missing entity fails with the canonical not-found classification;
- wrong entity kind fails closed;
- Catalog unavailable fails with the canonical retryable/provider-unavailable classification;
- no email/display-name fallback;
- repeated calls do not rely on a new resolution cache.

### Architecture guards

Extend the existing semantic guards to selector artifacts/config model:

- no provider IDs;
- no ADO/Teams IDs;
- no email identity field;
- no job-title/employee-name canonical field;
- no email-shaped values in published selector artifact content;
- generic selector keys only;
- ordinary source variable/comment words such as `manager` must not trip the guard.

### Regression

Re-run all accepted F3.1.0 and F3.1.1a Change Management tests without modifying their semantics.

Run at minimum:

```bash
yarn workspace backend lint
yarn workspace backend build
```

and the canonical Change Management / authorization suites documented by the repository.

Compare repository-wide TypeScript errors to the accepted pre-existing baseline using **set identity** by file/line/column/code, not merely count. Do not "fix unrelated TypeScript errors" in this checkpoint.

---

# 7. Functional evidence

Because F3.1.1b has a real Catalog I/O boundary, do not stop at pure unit tests if a running Backstage/Catalog environment is available.

Where practical and safe:

1. start or use the current Backstage development runtime;
2. validate startup with a correct selector configuration;
3. validate one User selector against a real Catalog User;
4. validate one authority selector against a real Catalog Group;
5. prove the Group is not expanded into members;
6. deliberately reference a non-existent principal in a controlled configuration/test and prove fail-closed behavior;
7. restore the valid config;
8. verify existing GMUD create/list/detail functionality has not changed.

Do not manufacture a production environment. This is functional F3.1.1b evidence, not production rollout proof.

---

# 8. Delivery boundary — mandatory

ADR-012 is now Accepted, but F3.1.1b must **not** consume Delivery runtime concepts yet.

Do not add any of the following to selector configuration or principal resolution:

- ReleaseCandidate;
- DeploymentTarget;
- DeploymentRequest;
- ChangeBinding;
- Kargo Stage/Promotion IDs;
- Argo Application IDs;
- GitOps repository/branch identity;
- production namespace;
- Azure DevOps pipeline/environment IDs.

F3.1.1b resolves **who/which authority** a policy requirement refers to. It does not decide **what deployment is being authorized**.

That boundary becomes relevant only in later F3.1.2+/execution-eligibility integration.

---

# 9. Explicitly carried blockers — do not fix here

Keep these visible but untouched:

1. `ChangeManagementService` still calls `buildChange()` twice. Canonical docs classify this as **MUST FIX BEFORE F3.1.2**, not F3.1.1b.
2. Legacy idempotency reservations remain `LEGACY_PRE_F3` and must never be reinterpreted into ledger-required semantics.
3. RBAC CSV / conditional-policy files remain an F3.1.4 prerequisite.
4. Decision-time authority membership is not selector resolution.
5. Production rollout readiness remains gated on a real production target and is not part of this GMUD slice.

---

# 10. Scope-diff check before commit

Before committing implementation, inspect the complete diff and classify every changed file.

Expected categories are narrowly bounded to:

- F3.1.1b selector/config/resolver implementation;
- selector tests;
- existing policy publication manifest append;
- narrowly required plugin/config wiring;
- architecture guard extension;
- factual evidence docs.

If you find changes to:

```text
ChangeManagementService submission behavior
Delivery domain
frontend UI
migrations
routes
approval commands
Teams/CAB
production rollout configuration
```

remove/revert them unless the canonical docs explicitly advanced before execution and clearly authorize them.

---

# 11. Result semantics

Return exactly one implementation checkpoint verdict:

```text
PASS
CONDITIONAL_PASS
FAIL
```

`PASS` means the F3.1.1b implementation contract itself is satisfied.

It does **not** mean F3.1.2 is authorized automatically.

Even on `PASS`, record:

```text
F3.1.1b implementation: PASS
Architecture implementation acceptance: PENDING SEPARATE REVIEW
F3.1.2: NO-GO
Production rollout: separate / deferred
```

Use `CONDITIONAL_PASS` only for a finite, objective implementation/evidence dependency that does not change architecture.

Use `FAIL` for architectural deviation, unsafe permissive fallback, regression, broken publication integrity, or inability to satisfy the canonical contract.

---

# 12. Canonical evidence update

After implementation/testing:

1. record implementation branch and before/after SHAs;
2. record exact changed paths;
3. record configuration shape actually implemented;
4. record selector-bundle identity/version/digest without any secret values;
5. record publication-manifest append evidence and trusted baseline ref;
6. record Catalog positive/negative proof;
7. record test/lint/build results;
8. record repository-wide TypeScript baseline comparison;
9. record deviations, if any;
10. explicitly state that `POST /changes` remains unwired;
11. explicitly state F3.1.2 remains `NO-GO` pending separate review/authorization.

Update canonical files factually, including as appropriate:

```text
docs/backstage/implementation-progress.md
docs/backstage/current-state.md
```

Create a focused evidence document if needed:

```text
docs/backstage/f3-1-1b-implementation-evidence.md
```

Do not rewrite historical F3.1.1a evidence.

---

# 13. Final report

Report exactly these sections:

1. Canonical docs baseline SHA
2. ADO implementation baseline before
3. ADO implementation SHA after
4. Scope-diff summary
5. Selector bundle/config implemented
6. Startup validation evidence
7. Catalog resolver evidence
8. Publication-integrity evidence
9. Semantic architecture-guard evidence
10. Regression / lint / build evidence
11. TypeScript baseline comparison
12. Deviations / limitations
13. F3.1.1b verdict
14. F3.1.2 gate
15. Documentation commits
16. STOP

End with:

```text
STOP
```

Do not continue into F3.1.2.