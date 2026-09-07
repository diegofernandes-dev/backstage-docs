# F3.1.1b — Selector Bundle, Catalog Resolver & Publication Integrity (implementation evidence)

- **Status:** IMPLEMENTED / PUBLISHED — architecture implementation acceptance **pending separate review**
- **Date:** 2026-09-07
- **Canonical docs baseline:** `backstage-docs@8de1ca430387c3f4b231f76b2c731347ec636c9a`
- **Authority:** ADR-009 + the F3.1.1-R2 implementation plan (§7–§19, §23 F3.1.1b row) + the F3.1.1b implementation prompt

This document records what was implemented and what was actually proven. It
does not revise F3.1.1a history and introduces no new architecture decision.

## 1. Implementation baseline

| Item | Value |
|---|---|
| Repository / branch | `platform-devops-developer-portal` / `feat/ado-repo-governance` |
| ADO baseline **before** (verified live via ADO REST API before any edit) | `d3c0751a15b908cec8f5595c97e52f41226344ed` |
| ADO SHA **after** (verified live via ADO REST API after the push) | `188d8e9cc43423f3644b3cacfb9849257838a583` |
| Relationship | Direct child of `d3c0751`; plain fast-forward, no force operation |
| Reconciliation needed? | No — the live branch still pointed at the documented F3.1.1a baseline |
| Delivery branch `feat/delivery-mvp-slice` | **Not used, not merged, not read from** |

Work was done in an isolated worktree off `d3c0751`, not in the Delivery
working tree.

**Publication transport note.** `git` over SSH to Azure DevOps was failing for
the whole session (`remote: One or more errors occurred.`). The commit was
pushed over HTTPS using a PAT through a temporary remote, which was removed
immediately afterwards; the resulting SHA was then confirmed independently via
the ADO REST API. The repository's `origin` remote is unchanged.

## 2. Changed paths

```
app-config.yaml                                                                     (M — +29, mandatory authorization block)
packages/backend/src/modules/changeManagement/architecture.test.ts                  (M — +157, additive guards only)
packages/backend/src/modules/changeManagement/authorization/policy/published-manifest.json  (M — +6, one appended entry)
packages/backend/src/plugins/changeManagementPlugin.ts                              (M — +21, 0 deletions)
packages/backend/src/modules/changeManagement/authorization/selector/               (A — 14 files)
```

All four modifications are **pure insertions** — 213 insertions, 0 deletions.

Untouched, confirmed by the pre-commit scope audit: `ChangeManagementService.ts`,
the Delivery domain, frontend, migrations, routes, approval commands,
Teams/CAB, `app-config.production.yaml`, and production rollout configuration.
No database table, migration, HTTP route, or CI pipeline was added.

## 3. Configuration shape actually implemented

`changeManagement.authorization` is **mandatory**: `readAuthorizationConfig`
uses required reads throughout, so a deployment without a complete, explicit
authorization configuration fails startup rather than booting unconfigured.

```yaml
changeManagement:
  authorization:
    activePolicy:
      key: default-change-authorization
      version: '2026-09-02.1'
    activeSelectorBundle:
      key: selector-bundle-dev
      version: '2026-09-07.1'
      provenance: backstage-docs@8de1ca430387c3f4b231f76b2c731347ec636c9a
      contentDigest: <64-hex>            # optional; validated when present
      selectors:                          # a LIST, so duplicate keys stay detectable
        - selectorKey: normal-primary-approver
          principalType: user
          principalRef: user:default/<catalog user>
        - selectorKey: emergency-approver-a
          principalType: user
          principalRef: user:default/<catalog user>
        - selectorKey: emergency-approver-b
          principalType: user
          principalRef: user:default/<catalog user>
        - selectorKey: cab-authority
          principalType: authority
          principalRef: group:default/<catalog group>
```

Neither pin is ever resolved to "the latest" anything. No secret value is part
of this configuration.

**Content in configuration, identity in the manifest.** This is how the
accepted invariant "selector bindings are environment-scoped validated
application configuration" is reconciled with §18/§19's requirement that each
bundle carry its own published identity: the *bindings* live in app-config, and
the *identity plus digest* of that binding set lives in the publication
manifest. Rebinding a selector therefore requires a version bump and a manifest
append — an in-place edit fails startup on the content-digest check, which is
proven below against a real running backend.

## 4. Selector bundle identity

| Field | Value |
|---|---|
| `artifact` | `selector-bundle` |
| `key` | `selector-bundle-dev` |
| `version` | `2026-09-07.1` |
| `contentDigest` | `6a0c2fb4f30037603848cb72700e77d60f68cf35a25e6db055c3690780d28b94` |
| Digest input | `sha256Canonical(bundle.selectors)`, reusing `canonical.ts` verbatim |

The digest was computed from the actual configured content and is recomputed
and compared by a test; it was never hand-typed.

## 5. What was implemented

Under `packages/backend/src/modules/changeManagement/authorization/selector/`:

- **`types.ts`** — `SelectorEntry` (`principalType` + `principalRef`, nothing
  else), `SelectorBundle` (key, version, provenance, `contentDigest`,
  `selectors`), and the typed configuration shapes. One immutable versioned
  bundle as the deploy-time unit; no per-selector versions.
- **`selectorDigest.ts`** — `selectorBundleContentDigest` and the per-resolution
  `selectorDigest` over `{selectorKey, selectorVersion, principalType,
  principalRef}`. Both reuse `sha256Canonical`; no second hashing scheme exists.
- **`config.ts`** — `readAuthorizationConfig(config)`, following the existing
  `readAdoProvisionerConfig` convention.
- **`bundle.ts`** — `createSelectorBundle` in the same order `createPolicyRegistry`
  uses: reject duplicate keys → compute digest → check the declared digest →
  require and check the manifest entry → `deepFreezeSerializable` (the F3.1.1a
  utility, reused unchanged).
- **`startupValidation.ts`** — pure, I/O-free validation of the **active pair only**.
- **`CatalogPrincipalResolver.ts`** — produces the existing
  `PrincipalResolutionSnapshot` (no field added) via `parseEntityRef` +
  `catalog.getEntityByRef` under `auth.getOwnServiceCredentials()`.
- **`bootstrapAuthorization.ts`** — the whole startup path in one function, so
  the plugin wiring is a single call and tests exercise exactly what the plugin
  runs.

### Emergency A/B narrowing, expressed generically

A/B narrowing is enforced by grouping the active policy's requirements on their
shared `separationOfDutyKey`, **not** by hardcoding `emergency-approver-a/b`.
Within a group every requirement must be `user`-typed and every `selectorKey`
must be distinct. The rule therefore holds for any future separation-of-duty
pair without the guard learning organizational vocabulary. Distinctness of the
two *resolved* refs is a resolution-time concern and actor-level separation is a
decision-time concern; neither is implemented here.

### Startup validation checks

Active policy configured; active policy present in the shipped registry; active
bundle configured; duplicate selector keys rejected; content digest correct;
declared digest (when present) correct; manifest entry required and its digest in
agreement; every `selectorKey` the **active** policy references present in the
active bundle; `individual ⇔ user` and `authority|cab ⇔ authority`; the bound
principal type matches the requirement's required type; every binding's ref
parses and its Catalog kind matches its declared principal type; A/B narrowing
as above. An **inactive historical** policy is deliberately not validated
against the active bundle — proven by a dedicated test.

### Resolver failure model

| Condition | Result |
|---|---|
| Selector absent from the active bundle | `INTERNAL_ERROR` (500), Catalog never called |
| Malformed configured ref | `INTERNAL_ERROR` (500), Catalog never called |
| Declared kind ≠ ref kind | `INTERNAL_ERROR` (500) |
| Catalog entity absent | `NOT_FOUND` (404) |
| Actual Catalog kind drifted from the declared type | `INTERNAL_ERROR` (500) |
| Catalog or credentials throw | `PROVIDER_UNAVAILABLE` (503, retryable) |

No cache, no member expansion, no address or human-readable-name fallback, and
no permissive branch anywhere.

## 6. Publication integrity — a normal append, not genesis

The `selector-bundle` artifact kind already existed in the F3.1.1a validator, so
**`validateManifestHistory.ts` was reused entirely unchanged**. One entry was
appended; the pre-existing policy entry is byte-identical.

Real command invocations:

```bash
# POSITIVE - append against the trusted F3.1.1a baseline. No --allow-genesis-from.
yarn validate:policy-publication --baseline-ref d3c0751a15b908cec8f5595c97e52f41226344ed
# OK: candidate publication manifest is a valid append-only extension ...   exit 0

# NEGATIVE - same key@version, changed content, against the new commit
yarn validate:policy-publication --baseline-ref 188d8e9cc43423f3644b3cacfb9849257838a583
# FAIL ... {"code":"DIGEST_CHANGED", ...}                                    exit 1

# NEGATIVE - removing the F3.1.1a policy identity
# FAIL ... {"code":"IDENTITY_REMOVED", ...}                                  exit 1
```

The genesis flag was **not used at any point**. A unit test additionally pins
that an absent baseline still fails closed even when a genesis argument is
supplied, so the genesis path cannot be reached from this slice.

## 7. Catalog evidence (functional, against a real running Backstage)

### 7.1 Live Catalog resolution

An opt-in integration suite drove the **production resolver, unmodified**
against the live Catalog HTTP API of the running development instance
(`:7007`, left untouched). All four cases passed:

- a real Catalog `User` resolved to the correct `PrincipalResolutionSnapshot`;
- a real Catalog `Group` resolved to the **Group ref itself** — the test first
  asserts that the live Group genuinely carries `hasMember` relations, then
  asserts none of them, and no count of them, appears in the snapshot;
- a deliberately non-existent principal failed closed with `NOT_FOUND`;
- `getOwnServiceCredentials()` was called for every lookup.

### 7.2 Real startup validation

A second backend was started from the F3.1.1b worktree on port `7008` with its
own SQLite database, using the **committed** `app-config.yaml` (only the port
was overridden, from a file outside the repository). msgraph synced 5 real users
and 3 real groups.

**Positive:**

```
change-management info Change authorization startup validated:
  policy default-change-authorization@2026-09-02.1,
  selector bundle selector-bundle-dev@2026-09-07.1
  (contentDigest 6a0c2fb4f30037603848cb72700e77d60f68cf35a25e6db055c3690780d28b94)
```

**Negative 1 — a selector rebound without a version bump:**

```
Plugin 'change-management' startup failed; caused by Error: Selector bundle
selector-bundle-dev@2026-09-07.1 declares contentDigest "6a0c2f..." but its
configured content hashes to "8ee709...".
```

The process shut down; no "startup validated" line was emitted.

**Negative 2 — an unpublished bundle identity (`2099-01-01.1`):**

```
Plugin 'change-management' startup failed; caused by Error: Selector bundle
selector-bundle-dev@2099-01-01.1 has no matching publication manifest entry.
```

**Restore:** the valid configuration was restored and the backend booted
cleanly again. `GET` and `POST /api/change-management/changes` returned `401`
unauthenticated on `:7008` — identical to the untouched `:7007` instance, i.e.
existing GMUD behavior is unchanged.

## 8. Semantic architecture guards

`architecture.test.ts` was extended **additively** (a new nested `describe`; the
F3.1.1a block is untouched), reusing the existing `sources`/`FORBIDDEN_FIELDS`
shape-guard idiom:

- forbidden provider/execution fields and forbidden canonical identity fields
  over the new `selector/` sources;
- no rules-engine/DSL/process-engine vocabulary;
- no provider/Teams/ADO/pipeline/environment identifier;
- **no Delivery runtime concept** (`ReleaseCandidate`, `DeploymentTarget`,
  `DeploymentRequest`, `ChangeBinding`, Kargo, Argo);
- no membership-expansion API surface (`hasMember`, `memberOf`, `getEntities(`,
  `.relations`, `queryEntities(`) — a **code-shape** guard, explicitly not a
  grep for English, so prose explaining that an authority is never expanded does
  not trip it;
- **data** assertions over the loaded bundle: every `selectorKey` matches
  `^[a-z][a-z0-9-]*$`; no string value is address-shaped; every entry carries
  exactly `principalType` + `principalRef` with a kind-appropriate ref prefix;
- the `manager`/`director`/`cto` regression case still passes;
- `ChangeManagementService.ts` contains no selector, principal-resolution,
  bootstrap, or round-creation reference.

**One design change came out of these guards.** `resolverProvenance` was
initially formatted `selector-bundle:<key>@<version>#<digest>`, which matches the
`/\S+@\S+\.\S+/` address shape used by the guard. It is now colon-separated, so
"no value a snapshot carries is address-shaped" is a genuinely assertable
invariant rather than one with a documented exception.

## 9. Tests, lint, build

| Check | Result |
|---|---|
| New F3.1.1b tests | **105** across 7 new suites, plus the live suite (4, opt-in) |
| Full backend suite (SQLite) | **40 suites / 328 tests passed**, 1 suite skipped (opt-in live), 0 failed |
| PostgreSQL | Disposable `postgres:16` container; `authorization/postgres.test.ts` **genuinely executed** (not skipped) inside the full green run; container removed afterwards |
| F3.1.0 / F3.1.1a regressions | All pre-existing suites re-run unmodified; only `architecture.test.ts` changed, additively |
| Live Catalog suite | 4/4 passed against the running instance |
| `yarn workspace backend lint` | exit **0** |
| `yarn workspace backend build` | exit **0** (the JSON manifest import resolves in the packaged build) |

Baseline for comparison: F3.1.1a recorded 33 suites / 237 tests. The 8 added
suites are the 7 selector suites plus the opt-in live suite.

## 10. TypeScript baseline comparison

Captured inside the isolated worktree **before any edit** and again after.

Before (5 errors, all in `changeManagementPlugin.ts`):
`(67,61)`, `(77,64)`, `(78,64)`, `(79,60)` `TS2345`; `(80,11)` `TS2322`.

After: `(88,61)`, `(98,64)`, `(99,64)`, `(100,60)` `TS2345`; `(101,11)` `TS2322`.

**No error was added and none was removed.** The set is identical by file, code
and column; every line number shifted by exactly **+21**, which is exactly the
number of lines inserted into that file (`git diff --numstat` reports `21 0`),
and the source expression at each shifted line is byte-identical to the
expression at the corresponding pre-edit line (verified by extracting both).
This is the first F3 slice that had to touch `changeManagementPlugin.ts`, so a
literal line-for-line set comparison was not available; the identity was proven
at the level of file + code + column + source expression instead.

One new TypeScript error was introduced during implementation
(`selector/config.test.ts` fixture typing) and **fixed** before the final
capture. No unrelated pre-existing TypeScript error was touched.

## 11. Deviations and limitations

1. **`app-config.production.yaml` was deliberately not given an authorization
   block.** Backstage overlays the production config onto the base config, so
   the mandatory check is satisfied in every environment from `app-config.yaml`.
   Adding invented production Catalog refs would be fabrication. **Carried
   consequence:** a production deployment would currently inherit the *dev*
   selector bundle. Publishing a distinct `selector-bundle-prod` identity (its
   own key, version and manifest entry, per §19) is a prerequisite for
   production rollout — which is separately gated on a real production target
   existing.
2. **`contentDigest` in configuration is optional.** It is computed from the
   configured content and validated against the manifest regardless; the
   declared value is a second, independent operator statement, checked when
   present. The plan's "selector-bundle digest is correct" and "manifest digest
   matches the computed digest" are both enforced.
3. **`resolverProvenance` is colon-separated, not `key@version`** — see §8.
4. The bundle key `selector-bundle-dev` carries an environment word. This is
   permitted by §19 explicitly: it is a naming convention inside an otherwise
   opaque bundle key and carries no authorization-domain meaning. No
   environment value enters the policy or selector domain.

Carried forward, untouched by this checkpoint:

- `ChangeManagementService` still calls `buildChange()` twice — **MUST FIX
  BEFORE F3.1.2**.
- `LEGACY_PRE_F3` idempotency reservations must continue to resume the legacy
  path; never reinterpreted into ledger-required semantics.
- RBAC CSV / conditional-policy files remain an **F3.1.4 prerequisite**.
- Decision-time authority membership is not selector resolution.
- Production rollout readiness remains gated on a real production target.

## 12. Boundary statements

- **`POST /changes` remains completely unwired.** `ChangeManagementService` was
  not modified, no `AuthorizationRound` is created, no route was added, and the
  resolver is not reachable from any request path. It is constructed at startup
  and held in a local binding so that startup validation genuinely executes.
- **No Delivery runtime concept** entered selector configuration or principal
  resolution — enforced by a guard, not only by review.
- **No email address, human-readable name, job title, employee identifier,
  Entra object ID, Teams ID or Azure DevOps identity ID** is canonical
  authorization identity or snapshot data. The Catalog ref is the stable
  platform principal reference.

## 13. Gate

```text
F3.1.1b implementation: PASS
Architecture implementation acceptance: PENDING SEPARATE REVIEW
F3.1.2: NO-GO
Production rollout: separate / deferred
```
