# F3.1.1b — Architecture Implementation Acceptance Review

## Status

**REVIEW-ONLY. NO IMPLEMENTATION AUTHORIZED.**

This checkpoint independently reviews the implemented F3.1.1b selector-resolution slice and decides whether it becomes the accepted implementation baseline for the selector/configuration half of F3.1.1.

The implementation under review is:

- repository: Azure DevOps `platform-devops-developer-portal`
- branch: `feat/ado-repo-governance`
- candidate SHA: `188d8e9cc43423f3644b3cacfb9849257838a583`
- expected parent: accepted F3.1.1a baseline `d3c0751a15b908cec8f5595c97e52f41226344ed`

Canonical architecture/docs are `diegofernandes-dev/backstage-docs@main`.

This is an **architecture implementation acceptance**, not a new implementation slice and not a production-rollout review.

---

## 1. Objective

Answer one question:

> Does the implementation at `188d8e9cc43423f3644b3cacfb9849257838a583` faithfully implement the already accepted ADR-009 / F3.1.1-R2 selector-resolution architecture without widening authority, coupling Change Management to Delivery/providers, or prematurely implementing F3.1.2+ behavior?

The review must return exactly one implementation-acceptance verdict:

```text
F3.1.1b architecture implementation acceptance: ACCEPT
```

or

```text
F3.1.1b architecture implementation acceptance: REJECT
```

There is **no generic CONDITIONAL_ACCEPT** outcome. Non-blocking follow-ups may be recorded separately, but any unresolved architecture blocker means `REJECT`.

An `ACCEPT` closes F3.1.1b and authorizes only **planning/review preparation for F3.1.2**. It does **not** authorize F3.1.2 implementation.

---

## 2. Mandatory fresh-baseline procedure

Before reviewing:

1. `git fetch origin main` in `diegofernandes-dev/backstage-docs`.
2. Record the exact fetched `origin/main` SHA.
3. Read, at minimum:
   - `docs/adr/ADR-009-change-authorization-model.md`
   - `docs/backstage/f3-1-implementation-plan.md`
   - `docs/backstage/f3-1-1-implementation-plan.md`
   - `docs/backstage/f3-1-1a-architecture-acceptance.md`
   - `docs/backstage/f3-1-1b-implementation-evidence.md`
   - `docs/backstage/current-state.md`
   - `docs/backstage/implementation-progress.md`
   - this prompt.
4. Inspect the actual Azure DevOps source tree at the exact candidate SHA if the review environment can access it.
5. Verify the candidate's parent/lineage and compare `d3c0751..188d8e9`.
6. If the ADO branch head has advanced beyond `188d8e9`, review the exact candidate tree anyway and record the drift. If any later branch change touches the F3.1.1b review surface, do **not** silently treat `188d8e9` as the current branch baseline; record the conflict and stop acceptance until it is reconciled.

Do not reconstruct source behavior from memory.

### Evidence limitation rule

If the current environment cannot independently access Azure DevOps source:

- continue only with the canonical implementation evidence and already-recorded factual proof;
- explicitly label source/test claims that were **not independently re-verified** in this checkpoint;
- do not manufacture an independent-verification claim;
- lack of connector access alone is not an automatic `REJECT` if the canonical evidence is sufficient to evaluate the architecture;
- if an architecture-critical assertion cannot be established from either source inspection or canonical evidence, return `REJECT`.

---

## 3. Strict review-only boundary

You MUST NOT:

- modify `platform-devops-developer-portal` source;
- fix defects while reviewing;
- change selector configuration or publication-manifest content;
- modify migrations, routes, permissions, frontend, Delivery, Kargo, Argo, or production infrastructure;
- wire selector resolution into `POST /changes`;
- create an `AuthorizationRound`;
- begin F3.1.2, F3.1.3, or F3.1.4;
- introduce Teams/CAB UI;
- create a workflow engine, rules DSL, policy admin UI, or runtime publishing service;
- edit ADR-009 merely to make the implementation fit;
- treat production-rollout readiness as part of this review.

Read-only source inspection, test execution in an isolated worktree, local disposable databases, and read-only Catalog/backend verification are allowed when available.

If you find a defect, document it. Do not repair it.

---

## 4. Required implementation-scope verification

Independently confirm the candidate is a bounded F3.1.1b slice.

Expected implementation surface from the canonical evidence:

- `app-config.yaml`
- additive changes to `architecture.test.ts`
- one appended `selector-bundle` entry in the existing publication manifest
- bounded startup wiring in `changeManagementPlugin.ts`
- new selector-resolution implementation under:
  `packages/backend/src/modules/changeManagement/authorization/selector/`

Confirm that the candidate did **not** introduce unrelated behavior into:

- `ChangeManagementService.ts`
- HTTP routes
- database migrations
- frontend
- Delivery
- approval/decision commands
- execution lifecycle
- Teams/CAB
- production rollout configuration
- CI pipeline creation.

A change outside the documented slice is not automatically a rejection, but it must be justified as strictly necessary to F3.1.1b. Unexplained scope widening is a blocker.

---

## 5. Mandatory architecture gates

Evaluate every gate below as `PASS` or `FAIL`. Every gate must `PASS` for `ACCEPT`.

### G1 — ADR-009 authority boundary

Confirm:

- selectors resolve platform principals; they do not become authorization decisions;
- Catalog is a principal-resolution dependency, not authorization authority;
- Backstage UI, ADO, Teams, Kargo, Argo and ITSM providers remain non-canonical for authorization;
- no provider or execution identifier enters canonical selector/principal data;
- no workflow/BPM semantics were introduced.

### G2 — Generic selector semantics

Confirm canonical selector semantics remain generic:

- no employee names as domain semantics;
- no email address as identity;
- no job-title/corporate-role semantics;
- no Entra object ID, ADO identity ID, Teams ID, pipeline ID, environment ID, repository ID, deployment ID, Kargo ID, or Argo ID;
- selector keys are configuration identities only.

The configured dev values may point to real Catalog refs. That is allowed. The domain must not depend on organization-specific titles/names.

### G3 — Principal type and Catalog-ref correctness

Confirm:

- `individual` requirements bind only to `user` principals / Catalog `User`;
- `authority` and `cab` requirements bind only to `authority` principals / Catalog `Group`;
- malformed refs and kind mismatches fail closed;
- the snapshot retains the stable Catalog principal ref, not an email/display name.

### G4 — Authority-group non-expansion

Confirm a resolved Group remains a Group ref.

The implementation must not:

- snapshot current group members;
- count group members as an authorization fact;
- derive N approval requirements from one authority;
- use `hasMember` / `memberOf` expansion as selector resolution;
- cache membership into the immutable principal snapshot.

Decision-time proof that an actor can act for an authority remains a later decision-command concern.

### G5 — Backend-service credential boundary

Confirm Catalog resolution uses Backstage backend/service credentials (for example `auth.getOwnServiceCredentials()`) and does not depend on the requesting end user's delegated Catalog credential.

Explain why this preserves a deterministic server-side resolution boundary.

### G6 — Fail-closed resolver behavior

Verify the documented failure model remains strict:

- missing selector → failure, no fallback;
- malformed ref → failure;
- declared/actual kind mismatch → failure;
- missing Catalog entity → failure;
- Catalog/service-credential failure → retryable provider-unavailable style failure;
- no “best effort”, display-name lookup, email fallback, or permissive default;
- no Catalog cache that could snapshot stale principal identity.

### G7 — Active-pair startup validation

Confirm startup validates exactly the active policy + active selector-bundle compatibility required for new submissions.

It must not require every historical policy version to remain compatible with today's active selector bundle.

Confirm an inactive historical policy can remain in publication/audit history without crashing startup solely because one of its retired selector keys is absent from the active bundle.

### G8 — Selector-bundle identity, digest, and runtime immutability

Confirm:

- one selector bundle is the versioned deployment unit;
- no per-selector versioning framework was introduced;
- selector-bundle content digest is computed canonically from the selector bindings using the existing canonical hashing primitive;
- configured content is always checked against the published manifest identity/digest;
- runtime data is immutable after validation;
- no second canonicalization/hash scheme was invented.

Explicitly review the implementation choice that `contentDigest` in app-config is optional while the computed content digest is still always checked against the publication manifest. Decide whether this preserves the accepted contract. Record the reasoning.

### G9 — Publication-history integrity

Confirm F3.1.1b reused the F3.1.1a append-only publication model rather than inventing another one.

Specifically verify:

- the existing policy manifest entry was not mutated;
- exactly the new selector-bundle identity/digest was appended;
- validation was performed as a **normal append** against the trusted F3.1.1a baseline;
- the genesis escape hatch was not used for F3.1.1b;
- same `artifact/key/version` with changed digest fails;
- identity removal fails;
- duplicate identity remains fail-closed.

### G10 — Separation-of-duty layer boundaries

The review must distinguish three different concepts and ensure the implementation does not claim the wrong one:

1. **configuration/startup** — emergency A/B requirements use distinct selector keys and the required `user` principal type;
2. **submission/resolution** — future F3.1.2 must fail closed if A/B resolve to the same effective person for the submitted round;
3. **decision time** — future decision logic must ensure the required human decision actors are distinct in the round.

F3.1.1b is not required to implement layers 2 or 3 because `POST /changes` is intentionally unwired. It **is** required to leave those boundaries explicit and not falsely claim they are already enforced.

A same-person separation-of-duty gap that would be silently accepted once F3.1.2 is wired is a mandatory carried-forward blocker to address in the F3.1.2 design.

### G11 — Submission-path isolation

Confirm:

- `POST /changes` remains behaviorally on the pre-F3 submission path;
- no `AuthorizationRound` is created;
- no selector resolution is reachable from `ChangeManagementService.createChange`;
- no existing F2 logical request is reinterpreted;
- startup bootstrap/wiring does not imply submission integration.

This gate is critical.

### G12 — Cross-cutover idempotency invariant preserved

Confirm F3.1.1b did not weaken the F3.1.0 invariant:

> one logical idempotent submission never changes authorization regime across retry or deployment boundaries.

In particular:

- existing `LEGACY_PRE_F3` reservations must continue legacy semantics;
- a future F3.1.2 path may select `LEDGER_REQUIRED` only for a genuinely new reservation;
- selector/policy evaluation must never be invoked merely because the application version changed.

### G13 — Architecture guards are semantic, not brittle prose policing

Review the new architecture guards for false confidence and false positives.

They should enforce meaningful domain/code-shape invariants such as forbidden canonical fields, provider leakage, membership expansion, and selector data shape.

They must not become fragile English-word grep tests where ordinary source words like `manager`, `director`, or `cto` accidentally fail architecture.

### G14 — Environment scoping and production separation

Review these known implementation choices explicitly:

- bundle key `selector-bundle-dev` contains an environment word;
- `app-config.production.yaml` does not yet publish a distinct production selector bundle;
- current production overlay would inherit the base/dev bundle if someone attempted deployment today;
- production rollout itself is separately gated and no real production target was defined at implementation time.

Classify whether these are:

- valid F3.1.1b architecture choices with a **production-rollout prerequisite**, or
- architecture blockers to selector resolution itself.

Do not invent production principal refs to make this green.

### G15 — Resolver provenance representation

Review the deviation that resolver provenance is colon-separated rather than using a `key@version` string.

Accept it only if:

- it remains deterministic and auditable;
- bundle identity/version/digest are preserved unambiguously;
- it avoids being mistaken for an email/address-shaped identity;
- no downstream contract requires the original formatting.

### G16 — No F3.1.2+ behavior hidden in plugin wiring

Inspect `changeManagementPlugin.ts` carefully.

The F3.1.1b bootstrap may validate and construct the resolver at backend startup, but it must not:

- invoke the resolver on submission;
- persist a round or requirement;
- authorize a decision;
- expose a new route;
- add participant decision authority;
- change lifecycle.

### G17 — Test/evidence credibility

When source access permits, independently rerun or inspect enough evidence to establish:

- focused selector/config/startup/publication tests;
- F3.1.1a policy/publication regressions;
- relevant Change Management regressions;
- architecture guards;
- backend lint;
- backend build;
- TypeScript baseline did not gain a new error.

A disposable PostgreSQL run and live Catalog test should be re-run when practical and available, but inability to reproduce external infrastructure in this review is not itself a rejection if the canonical implementation evidence is specific and credible. Record the limitation.

Do not change production or shared infrastructure merely to reproduce evidence.

---

## 6. Specific deviations that require an explicit review decision

Do not silently inherit the implementation checkpoint's classification. Explicitly decide each:

1. optional `contentDigest` in app-config, while computed digest vs. manifest remains mandatory;
2. `resolverProvenance` colon-separated representation;
3. `selector-bundle-dev` naming;
4. production config currently inheriting the dev selector bundle;
5. resolver constructed at startup but not reachable from request handling;
6. emergency A/B resolved-person distinctness intentionally deferred to the submission slice.

For each, record:

```text
ACCEPTED AS WITHIN F3.1.1b
or
BLOCKER
```

with a concise architecture reason.

---

## 7. Carried-forward constraints that acceptance must preserve

Whether the candidate is accepted or rejected, the final review document must restate these without “fixing” them:

### Before F3.1.2 implementation

- `ChangeManagementService` currently calls `buildChange()` twice, allowing server-generated `activityId` / `createdAt` divergence. This is **MUST FIX BEFORE F3.1.2**.
- Existing `LEGACY_PRE_F3` idempotency reservations must resume the legacy path.
- F3.1.2 must fail submission if required emergency A/B effective principals resolve to the same person.
- F3.1.2 must create one canonical Change snapshot, evaluate one pinned policy + selector bundle, create Round 1 atomically with its effective requirements, and preserve fail-closed behavior. This is a planning constraint only; do not design/implement it in this review.
- Delivery/ADR-012 provider-neutral boundaries remain in force; do not hardwire ADO pipeline/environment semantics into Change authorization.

### Later than F3.1.2

- decision-time authority membership / actual actor authorization belongs to the server-authoritative decision command work;
- RBAC CSV / conditional-policy completeness remains an F3.1.4 prerequisite unless canonical docs have since superseded it;
- production selector-bundle publication waits for a real production target/principal set;
- production rollout readiness remains a separate workstream.

---

## 8. Decision rule

Return `ACCEPT` only if all architecture gates G1–G17 pass.

Non-blocking implementation-quality observations may be recorded as follow-ups, but they must not be disguised blockers.

Return `REJECT` if any of the following is true:

- ADR-009 authority semantics are violated;
- provider/Delivery/execution coupling enters canonical authorization selector semantics;
- group membership is expanded/snapshotted as selector resolution;
- Catalog resolution can degrade permissively;
- publication identity can be edited/reused without detection;
- `POST /changes` or round creation was wired prematurely;
- cross-cutover idempotency semantics were weakened;
- an architecture-critical claim cannot be established;
- candidate scope/lineage cannot be reconciled.

Do not introduce `CONDITIONAL_ACCEPT`.

---

## 9. Required output document

Write:

`docs/backstage/f3-1-1b-architecture-acceptance.md`

Use this structure:

1. Status / verdict
2. Reviewed baselines and evidence limitations
3. Candidate lineage and scope
4. G1–G17 gate matrix
5. Explicit deviation decisions
6. Architecture findings
7. Carried-forward blockers/constraints
8. Decision
9. Next gate
10. STOP statement

If `ACCEPT`, the status must say:

```text
F3.1.1b: CLOSED / ACCEPTED IMPLEMENTED BASELINE
Implementation: platform-devops-developer-portal@188d8e9cc43423f3644b3cacfb9849257838a583
F3.1.2 planning: GO
F3.1.2 implementation: NO-GO pending a separate reviewed plan and explicit authorization
```

If `REJECT`, state the smallest concrete corrective checkpoint and keep F3.1.2 planning/implementation `NO-GO` where the blocker requires it.

---

## 10. Canonical documentation updates

After writing the review result, update only documentation needed to make the checkpoint unambiguous:

- `docs/backstage/current-state.md`
- `docs/backstage/implementation-progress.md`
- `prompts/README.md`

Do not rewrite historical evidence.

If accepted:

- mark F3.1.1b as the accepted implemented baseline;
- mark the F3.1.1b implementation and acceptance prompts as completed/historical;
- record that the next authorized activity is **F3.1.2 planning only**;
- do not create or execute an F3.1.2 implementation prompt in this checkpoint.

If rejected:

- record the blocker factually;
- leave the candidate as implemented-but-not-accepted;
- do not “correct” code from this review.

Commit documentation changes only.

---

## 11. Final report contract

The final response must include:

```text
Docs baseline reviewed: <sha>
Implementation candidate: 188d8e9cc43423f3644b3cacfb9849257838a583
Independent source verification: YES | PARTIAL | NO
Architecture gates: <N>/17 PASS
F3.1.1b architecture implementation acceptance: ACCEPT | REJECT
F3.1.2 planning: GO | NO-GO
F3.1.2 implementation: NO-GO
ADO implementation modified by this review: NO
Production rollout modified by this review: NO
Canonical review document: docs/backstage/f3-1-1b-architecture-acceptance.md
Final docs SHA: <sha>
```

Then summarize only material findings and explicit follow-ups.

---

## 12. STOP

STOP immediately after the review document and canonical documentation updates are committed.

Do **not**:

- fix code;
- begin F3.1.2;
- write F3.1.2 production code;
- create approval/decision endpoints;
- touch Teams/CAB UI;
- alter Delivery;
- reopen Deployments UX;
- resume production rollout readiness;
- “helpfully” implement the next slice.

The only purpose of this checkpoint is to decide whether F3.1.1b at `188d8e9` becomes an accepted implemented baseline.
