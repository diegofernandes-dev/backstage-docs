# F3.1.1c — CAB-Safe Policy Publication (implementation evidence)

- **Status:** IMPLEMENTED / PUBLISHED — architecture/implementation acceptance **pending separate review**
- **Date:** 2026-09-19
- **Canonical docs baseline (start):** `backstage-docs@982bf0516dab64d9beeb0d3b4fc7dee663561a98`
- **Authority:** ADR-009 (partially superseded by ADR-013) + F3.1.1 plan ADR-013 follow-up + F3.1.2 ACCEPTED IMPLEMENTATION CONTRACT + `prompts/f3-1-1c-cab-safe-policy-implementation.md` (explicit launch)

This document records what was implemented and what was actually proven. It does
not authorize F3.1.2b or F3.2 and does not create AuthorizationRounds.

## 1. Implementation baseline

| Item | Value |
|---|---|
| Repository / branch | `platform-devops-developer-portal` / `feat/ado-repo-governance` |
| ADO baseline **before** (HTTPS `git ls-remote` + `az repos ref list`) | `ccee1e1676a2763e68880e5383ce1e5e48742843` (accepted F3.1.2a tip) |
| ADO SHA **after** (HTTPS `git ls-remote` + `az repos ref list`) | `3b302ab5c9caab38f96491b389b7ea9fe0b66c2f` |
| Parent | Exact `ccee1e1…` — direct child |
| Publication | Plain fast-forward `ccee1e1..3b302ab` (no force) |
| Source drift on policy/manifest/registry/active-policy contracts | **NONE** (F3.1.2a did not touch them) |
| Delivery branch `feat/delivery-mvp-slice` | **Not used, not merged, not read from** |

**Publication transport note.** SSH to Azure DevOps failed in-session
(`remote: One or more errors occurred.`). The push used HTTPS with the existing
osxkeychain credential helper. Remote tip was confirmed independently via
`git ls-remote` and `az repos ref list` (`objectId` =
`3b302ab5c9caab38f96491b389b7ea9fe0b66c2f`). Repository `origin` SSH URL is
unchanged.

## 2. Changed paths

```
app-config.yaml                                                                     (M — activePolicy pin → 2026-09-19.1)
packages/backend/src/modules/changeManagement/architecture.test.ts                  (M — additive F3.1.1c guards)
packages/backend/src/modules/changeManagement/authorization/policy/published-manifest.json  (M — append-only entry)
packages/backend/src/modules/changeManagement/authorization/policy/registry.test.ts (M — P4 dual-register)
packages/backend/src/modules/changeManagement/authorization/policy/published/default-change-authorization.2026-09-19.1.ts (A)
packages/backend/src/modules/changeManagement/authorization/policy/published/default-change-authorization.2026-09-19.1.test.ts (A)
packages/backend/src/modules/changeManagement/authorization/selector/bootstrapAuthorization.ts (M — ship both versions)
packages/backend/src/modules/changeManagement/authorization/selector/bootstrapAuthorization.test.ts (M — active pin)
```

8 files changed, 607 insertions, 3 deletions.

**Untouched (confirmed):** historical
`default-change-authorization.2026-09-02.1.ts` (byte-identical to `ccee1e1`),
selector-bundle identity / bindings, evaluator semantics, `policyModelVersion`
dispatch, `ChangeManagementService`, ledger, migrations, routes, frontend,
Delivery, RBAC, CAB Workbench, autonomy.

## 3. Published policy identity

| Field | Value |
|---|---|
| `policyKey` | `default-change-authorization` |
| `version` | `2026-09-19.1` |
| `policyModelVersion` | `1` |
| `provenance` | `backstage-docs@982bf0516dab64d9beeb0d3b4fc7dee663561a98` |
| `policyArtifactDigest` | `7df4cae497f3fca79f7fc17b5439eda0e1e3c150b5425a6f2a1e1c797065fe06` |
| Digest input | `sha256Canonical({ policyModelVersion, rules })` |

Historical identity retained unchanged:

| Field | Value |
|---|---|
| `version` | `2026-09-02.1` |
| Manifest digest | `a10560aedaf86b278f9a0963336da6a43d86114c0a63a9b783ff7b7d130e8252` |

## 4. Exact matrix change

| Pair | Behavior |
|---|---|
| `normal.low` | **Changed** — `normal-primary-approval` + `cab-approval` (`cab-authority`) |
| `normal.medium` | Unchanged — primary + CAB |
| `normal.high` | Unchanged — primary + CAB |
| `emergency.*` | Unchanged — A/B + post-execution CAB retrospective (`sla` 432000) |

Active selector bundle unchanged: `selector-bundle-dev@2026-09-07.1` (still
includes `cab-authority`).

Active policy pin:

```yaml
changeManagement.authorization.activePolicy:
  key: default-change-authorization
  version: '2026-09-19.1'
```

`POST /changes` remains legacy/F2: still reserves `LEGACY_PRE_F3`, still creates
**no** `AuthorizationRound`. Policy activation alone does not alter submission.

## 5. Mandatory proofs

| ID | Result | How proven |
|---|---|---|
| P1 — historical immutability | **PASS** | Byte-identical historical artifact vs `ccee1e1`; manifest entry digest unchanged; new version string distinct |
| P2 — exact matrix | **PASS** | Focused matrix tests; medium/high/emergency deep-equal to historical evaluation |
| P3 — determinism/digest | **PASS** | Digest pinned; identity metadata outside digest; model version remains 1 |
| P4 — registry/startup | **PASS** | Both versions register; bootstrap resolves `2026-09-19.1`; `cab-authority` covered |
| P5 — no autonomy | **PASS** | Source/JSON guards reject `skipCab` / autonomy / grant / waiver |
| P6 — regressions | **PASS** | F3.1.1a/b suites + full Change Management module green |

## 6. Validation

| Gate | Result |
|---|---|
| `yarn validate:policy-publication --baseline-ref ccee1e1676a2763e68880e5383ce1e5e48742843` | **PASS** (non-genesis append) |
| Focused policy/selector/architecture | 14 suites / 183 tests PASS (4 skipped opt-in) |
| Full Change Management module | 28 suites / **301 tests PASS**, 9 skipped |
| `yarn workspace backend lint` | PASS |
| `yarn workspace backend build` | PASS |
| TypeScript baseline vs `ccee1e1` | **Line-identical** — same 5 pre-existing duplicate-Knex errors in `changeManagementPlugin.ts` at `(88,61)`, `(98,64)`, `(99,64)`, `(100,60)` `TS2345` and `(101,11)` `TS2322` |
| `--forceExit` | **Not used** |
| Genesis mode | **Not used** |

## 7. Hard non-goals confirmed

- Historical `2026-09-02.1` edited/reused: **NO**
- `policyModelVersion` changed: **NO**
- Selector bundle identity changed: **NO**
- CAB autonomy / `skipCab` / waiver / grant: **NO**
- AuthorizationRound / F3.1.2b / ledger wiring: **NO**
- Migrations: **NO**
- RBAC / CAB Workbench / frontend / Delivery: **NO**

## 8. Next gate

Independent **F3.1.1c architecture/implementation acceptance review**.

F3.1.2b remains **NO-GO** until both F3.1.2a (already accepted) and F3.1.1c are
independently accepted.
