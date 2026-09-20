# F3.1.2 — Non-production LEDGER_REQUIRED activation (execution evidence)

- **Status:** OPERATIONAL ACTIVATION EXECUTED / **PASS** — independent activation-acceptance **pending**
- **Date:** 2026-09-20
- **Canonical docs baseline (start):** `backstage-docs@80a68153701582745eef784811831576808d8bbd`
- **Authority:** `prompts/f3-1-2-ledger-required-nonprod-activation.md` (explicit user launch) + F3.1.2 ACCEPTED IMPLEMENTATION CONTRACT + [`f3-1-2b-architecture-implementation-acceptance.md`](./f3-1-2b-architecture-implementation-acceptance.md) + ADR-009 (partially superseded by ADR-013) + ADR-013

This document records the live non-production activation that was actually
performed. It does **not** independently accept the activation, does **not**
flip the committed repository default, and does **not** authorize F3.1.3,
F3.1.4, F3.2, or production cutover.

```text
Activation checkpoint: PASS
Independent activation acceptance: PENDING
Committed default newSubmissionAuthorizationMode: LEGACY_PRE_F3
Final laptop overlay mode: LEDGER_REQUIRED
F3.1.3 implementation: NO-GO pending independent acceptance
F3.1.4 / F3.2: NO-GO
Production cutover: NOT AUTHORIZED
ADO source modified: NO
Migrations added: NO
Approvals fabricated: NO
Old-binary downgrade: NO
```

## 1. Baseline verification

| Item | Value |
|---|---|
| Docs `origin/main` at start | `80a68153701582745eef784811831576808d8bbd` |
| Implementation repo / branch | `platform-devops-developer-portal` / `feat/ado-repo-governance` |
| Local HEAD | `22495229502dabf2d99588599a156d862c5114fa` |
| Parent | Exact `f48dc825ab5d1d16fafc3f70ef772d28613df1a0` |
| Independent remote tip (`az repos ref list`) | `22495229502dabf2d99588599a156d862c5114fa` |
| Source drift after accepted F3.1.2b | **NONE** |
| Committed `app-config.yaml` | `newSubmissionAuthorizationMode: LEGACY_PRE_F3` (unchanged) |
| Working tree (ADO) | no source edits; gitignored overlay only |

**Publication transport note.** SSH `git fetch` to Azure DevOps failed in-session
(`remote: One or more errors occurred.`). Remote tip was confirmed independently
via `az repos ref list`. Repository `origin` SSH URL is unchanged. No ADO push
was attempted; this checkpoint does not modify implementation source.

## 2. Non-production target

No shared DEV/Kubernetes Backstage runtime was identifiable. Rancher Desktop
API connection was refused. No production cluster, namespace, GitOps
repository, or cloud database was invented.

The only real isolated product environment was the operator laptop runtime
already used to demonstrate the IDP:

| Field | Value |
|---|---|
| Environment name | Operator laptop Backstage product runtime |
| Runtime location | `yarn start` on the operator workstation |
| UI | `http://localhost:3000` |
| Backend | `http://localhost:7007` |
| Running SHA | `22495229502dabf2d99588599a156d862c5114fa` |
| Database engine | SQLite (`better-sqlite3`) |
| Database identity | `packages/backend/data/change-management.sqlite` (gitignored `**/data/`) |
| Config override | gitignored `app-config.local.yaml` (`*.local.yaml`) |
| Restart mechanism | stop `yarn start` listeners on `:3000`/`:7007`, then `yarn start` |
| Delivery reads | available (`delivery.deployment.read` ALLOW for the signed-in user) |
| Rollback operator | Diego Fernandes (`user:default/diego.fernandes_outlook.com`) |

Isolation: PASS. The SQLite file is local to this workstation, not shared with
production, and existing facts were not deleted to obtain a clean baseline.

**Limitation.** This is an ephemeral/local product runtime, not a persistent
shared DEV cluster. It is also the only identified F3.1.3 demonstration
environment. After the same-binary backout proof, the gitignored overlay was
restored to `LEDGER_REQUIRED` so this laptop can remain the live non-production
ledger target. Independent acceptance must still judge whether laptop SQLite is
an adequate persistent target for F3.1.3 product validation.

## 3. Activation mechanism

Existing Backstage environment overlay: `app-config.local.yaml`, already
gitignored by `*.local.yaml`. Loaded after `app-config.yaml` (`Loaded config
from app-config.yaml, app-config.local.yaml`).

Overlay used (no secrets):

```yaml
changeManagement:
  authorization:
    newSubmissionAuthorizationMode: LEDGER_REQUIRED
```

Hard constraints kept:

- committed `app-config.yaml` remains `LEGACY_PRE_F3`;
- no repository-wide default commit;
- no policy/selector edits;
- no feature-flag framework;
- no source-code change to activate the value;
- no production config;
- no Delivery/Kargo/Argo/GitOps desired-state change.

## 4. Preflight (before flip)

Startup under the committed default, before overlay:

- binary SHA `2249522`;
- active policy `default-change-authorization@2026-09-19.1`;
- selector bundle `selector-bundle-dev@2026-09-07.1` digest
  `6a0c2fb4f30037603848cb72700e77d60f68cf35a25e6db055c3690780d28b94`;
- F3.1.0 ledger migrations already present (`knex_migrations` ids 1–5,
  including `20260901200000_authorization_ledger_foundation.cjs` and
  `20260901220000_add_authorization_mode_to_change_idempotency.cjs`);
- Catalog Deployments tab present;
- GMUD create/list/detail reachable after Entra sign-in;
- effective mode `LEGACY_PRE_F3`.

Database counts **before** creating the activation control (did not delete
pre-existing facts):

| Query | Result |
|---|---|
| pending `LEGACY_PRE_F3` | 0 |
| pending `LEDGER_REQUIRED` | 0 |
| completed `LEDGER_REQUIRED` | 1 (`CHG-2026-000001`, pre-existing delivery-mvp sandbox) |
| existing Round 1 | 1 (`CHG-2026-000001`, sandbox policy `delivery-mvp-sandbox-policy@v1`) |

`CHG-2026-000001` was left untouched.

## 5. Legacy pre-cutover control

Created through the GMUD UI (`/gmud/new`) while the runtime was still
`LEGACY_PRE_F3`.

| Field | Value |
|---|---|
| changeId | `CHG-2026-000002` |
| Idempotency-Key | `cc25d2d6-3127-4bb4-ae98-6aa48e36247c` |
| actor | `user:default/diego.fernandes_outlook.com` |
| reservation mode | `LEGACY_PRE_F3` |
| state / status | `completed` / `submitted` |
| created_at | `2026-09-20T15:06:18.658Z` |
| payload_hash | `36b61ea3b29918d110c8308ca3c95a5c78a6e7c8c8cb5ba9a137194bafd9e019` |
| AuthorizationRound | **none** |
| classification / risk | normal / low |
| targetRef | `component:default/platform-engineering-core` |

## 6. Activate and restart

Overlay set to `LEDGER_REQUIRED`; `yarn start` restarted on the same accepted
binary.

Startup diagnostics:

```text
Loaded config from app-config.yaml, app-config.local.yaml
Change authorization startup validated: policy default-change-authorization@2026-09-19.1,
  selector bundle selector-bundle-dev@2026-09-07.1
  (contentDigest 6a0c2fb4f30037603848cb72700e77d60f68cf35a25e6db055c3690780d28b94),
  newSubmissionAuthorizationMode LEDGER_REQUIRED
Plugin initialization complete, newly initialized: ... 'change-management', 'delivery', 'catalog' ...
Listening on :7007
```

No `collision`, `duplicate extension`, or `api:catalog/delivery` warnings.
Pre-existing Kubernetes plugin warning (`valid kubernetes config is missing`)
is unchanged and out of scope.

## 7. Live ledger-governed submission

Created through the GMUD UI after the LEDGER restart.

| Field | Value |
|---|---|
| changeId | `CHG-2026-000003` |
| Idempotency-Key | `16f81279-4e3d-4430-b7ce-67a8ab840f6c` |
| actor | `user:default/diego.fernandes_outlook.com` |
| reservation mode | `LEDGER_REQUIRED` |
| index mode | `LEDGER_REQUIRED` |
| state / status | `completed` / `submitted` |
| created_at | `2026-09-20T15:08:35.751Z` |
| payload_hash | `883059f7ec0f6b65159d136abccc0e2e85802264298109c4daae81c841a5977c` |
| provider record | one `development_change_records` row |
| ApprovalDecision | **none** |

Round 1 (exactly one):

| Field | Value |
|---|---|
| round_number | 1 |
| created_at | `2026-09-20T15:08:35.772Z` |
| change_snapshot_sha256 | `97d9da255e0039d162c4d059993d9f1f522a853420958af893ffe8c55baa70cf` |
| policy | `default-change-authorization@2026-09-19.1` |
| policy_artifact_sha256 | `7df4cae497f3fca79f7fc17b5439eda0e1e3c150b5425a6f2a1e1c797065fe06` |
| policy_provenance | `backstage-docs@982bf0516dab64d9beeb0d3b4fc7dee663561a98` |
| matched_rule_provenance | `normal.low` |
| selector bundle | `selector-bundle-dev@2026-09-07.1` |
| selector_bundle_sha256 | `6a0c2fb4f30037603848cb72700e77d60f68cf35a25e6db055c3690780d28b94` |
| selector_bundle_provenance | `backstage-docs@8de1ca430387c3f4b231f76b2c731347ec636c9a` |

Requirements (exactly two; `requirementId` equals `requirementRole` /
`source_ref`):

| requirement_id | kind | selector_key | resolved_principal_ref |
|---|---|---|---|
| `normal-primary-approval` | individual | `normal-primary-approver` | `user:default/diego.fernandes_outlook.com` |
| `cab-approval` | cab | `cab-authority` | `group:default/cloud_azure_devops_platform_devops` |

Canonical submission audit events (five rows, each type once plus one
`requirement_materialized` per requirement):

| event_type | count |
|---|---|
| `change.authorization.round_created` | 1 |
| `change.authorization.policy_selected` | 1 |
| `change.authorization.selector_bundle_bound` | 1 |
| `change.authorization.requirement_materialized` | 2 |

No ledger rows were inserted by hand.

## 8. Idempotency and conflict

Same actor + same key + same payload via `ChangeManagementClient` (product
discovery `http://localhost:7007/api/change-management`):

- HTTP `201`, body `{ changeId: "CHG-2026-000003", status: "submitted" }` at
  `2026-09-20T15:11:36.055Z`;
- log `change.create.authorization_mode mode=LEDGER_REQUIRED source=reserved`;
- still one reservation, one index row, one Round 1, two requirements, five
  audit events, one provider record.

Same actor + same key + deliberately different harmless title, `2026-09-20T15:11:43.082Z`:

- HTTP `409` (181-byte conflict body);
- no second reservation, round, requirement, audit event, or provider record.

## 9. Stored-mode-wins across the live cutover

Replay of `CHG-2026-000002` with original actor/key/payload while the runtime
default was already `LEDGER_REQUIRED` (`2026-09-20T15:11:49.983Z`):

- HTTP `201`, same `changeId` `CHG-2026-000002`;
- log `change.create.authorization_mode mode=LEGACY_PRE_F3 source=reserved`;
- still zero AuthorizationRounds for that change.

Current runtime default did not reinterpret the pre-cutover reservation.

## 10. Product surfaces (LEDGER active)

| Surface | Result |
|---|---|
| `/gmud` list | reachable; showed `CHG-2026-000002` and `CHG-2026-000003` |
| `/gmud/CHG-2026-000003` | reachable |
| Catalog | reachable at `/` (catalog is mounted at `/`; `/catalog` is 404 by app-config) |
| Component Deployments tab | `/catalog/default/component/idp-showcase-api/deployments` visible |
| Delivery reads | `delivery.deployment.read` ALLOW; HTTP 200/304 on release-candidates / deployments / events |
| Extension collisions | none (`api:catalog/delivery` not reported) |

No F3.1.3 decision commands exist. No approvals were fabricated.

## 11. Execution eligibility

Existing read path `GET /api/change-management/changes/CHG-2026-000003/execution-eligibility`
evaluated the live Round 1 without rewriting it.

Re-read after restoring the LEDGER overlay (`2026-09-20T15:17:35.440Z`):

```json
{
  "eligibility": {
    "changeId": "CHG-2026-000003",
    "decision": "DENY",
    "reason": "PENDING_AUTHORIZATION",
    "roundNumber": 1,
    "evaluatedAt": "2026-09-20T15:17:35.440Z"
  }
}
```

`decisionId` is an evaluation identifier, not an `ApprovalDecision`. After this
read: still 1 round, 2 requirements, 5 audits, 0 decisions.

## 12. Same-binary config backout

Overlay switched to `LEGACY_PRE_F3`; same accepted binary restarted. Startup
logged `newSubmissionAuthorizationMode LEGACY_PRE_F3`.

New reservation via product client:

| Field | Value |
|---|---|
| changeId | `CHG-2026-000004` |
| Idempotency-Key | `15b78668-9621-4feb-8e96-cc90c33e62e9` |
| mode | `LEGACY_PRE_F3` |
| created_at | `2026-09-20T15:13:43.271Z` |
| AuthorizationRound | **none** |

Replay of LEDGER key `16f81279-4e3d-4430-b7ce-67a8ab840f6c` after backout:

- `{ changeId: "CHG-2026-000003", status: "submitted" }`;
- reservation remained `LEDGER_REQUIRED`;
- Round 1 / requirements / audits unchanged (not removed or rewritten).

Mandatory old-binary drain query after backout:

```sql
SELECT COUNT(*)
FROM change_idempotency
WHERE authorization_mode = 'LEDGER_REQUIRED'
  AND state = 'pending';
```

Result: `0`. Old-binary downgrade was **not** performed.

## 13. Final non-production steady state

Because this laptop is the intended (and only identified) F3.1.3 demonstration
environment, the gitignored overlay was restored to `LEDGER_REQUIRED` and the
same binary restarted. Startup again logged
`newSubmissionAuthorizationMode LEDGER_REQUIRED`. Plugins
`change-management` and `delivery` initialized.

Committed `app-config.yaml` remains `LEGACY_PRE_F3`.

Final database identities:

| changeId | mode | rounds |
|---|---|---|
| `CHG-2026-000001` | `LEDGER_REQUIRED` (pre-existing sandbox) | 1 (sandbox policy) |
| `CHG-2026-000002` | `LEGACY_PRE_F3` | 0 |
| `CHG-2026-000003` | `LEDGER_REQUIRED` (activation proof) | 1 (CAB-safe policy) |
| `CHG-2026-000004` | `LEGACY_PRE_F3` (backout proof) | 0 |

## 14. Forbidden-scope checklist

| Constraint | Observed |
|---|---|
| F3.1.2b source modified | NO |
| `LEDGER_REQUIRED` committed as default | NO |
| Production activated / modified | NO |
| F3.1.3 / F3.1.4 / F3.2 implemented | NO |
| ApprovalDecision fabricated | NO |
| Policy or selector mappings changed | NO |
| Migrations added | NO |
| Ledger facts rewritten/deleted | NO |
| Kargo/Argo/GitOps mutated | NO |
| Binary downgraded | NO |

## 15. Next gate

Independent **F3.1.2 non-production LEDGER_REQUIRED activation acceptance
review**. That review may only ACCEPT or REJECT. It must not modify ADO code,
must not treat this PASS as acceptance, and must not start F3.1.3
implementation.
