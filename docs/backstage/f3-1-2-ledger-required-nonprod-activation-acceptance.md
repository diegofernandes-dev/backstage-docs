# F3.1.2 — Non-Production LEDGER_REQUIRED Activation Acceptance

- **Status:** CLOSED / ACCEPTED
- **Date:** 2026-09-20
- **Verdict:** `ACCEPT`
- **F3.1.3 demo target readiness:** `READY`
- **Accepted runtime SHA:** `platform-devops-developer-portal@22495229502dabf2d99588599a156d862c5114fa`
- **Documentation review baseline:** `backstage-docs@55735d1282b390ab968f16f5bb612282bfe46a2c` (`origin/main` at review start)
- **Authority:** `prompts/f3-1-2-ledger-required-nonprod-activation-acceptance.md` + [`f3-1-2-ledger-required-nonprod-activation-evidence.md`](./f3-1-2-ledger-required-nonprod-activation-evidence.md) + [`f3-1-2b-architecture-implementation-acceptance.md`](./f3-1-2b-architecture-implementation-acceptance.md) + F3.1.2 ACCEPTED IMPLEMENTATION CONTRACT + ADR-009 (partially superseded by ADR-013) + ADR-013 + this independent review

```text
F3.1.2 non-production LEDGER_REQUIRED activation acceptance: ACCEPT
Activation baseline: ACCEPTED
Accepted runtime SHA: 22495229502dabf2d99588599a156d862c5114fa
Committed repository default: LEGACY_PRE_F3
Accepted non-production effective mode: LEDGER_REQUIRED
Production cutover: NOT AUTHORIZED
F3.1.3 demo target readiness: READY
F3.1.3 planning/prompt authoring: GO
F3.1.3 implementation: still requires separate explicit authorization
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
```

This review did **not** modify ADO source, committed `app-config.yaml`, the gitignored overlay, ledger data, or production.

---

## 1. Status / verdict

F3.1.2 non-production LEDGER_REQUIRED activation acceptance: **ACCEPT**.

All mandatory gates **G1–G15 PASS**. Independent runtime/database verification: **YES**. The operator-laptop Backstage runtime at accepted SHA `2249522` is adequate as the F3.1.3 demonstration target (`READY`). Absence of a shared Kubernetes DEV cluster does not invalidate that classification.

ADO implementation was **not modified** by this review. Production cutover remains **NOT AUTHORIZED**.

---

## 2. Reviewed baselines

### Docs baseline

| Item | Value |
|---|---|
| Repo | `diegofernandes-dev/backstage-docs` |
| Branch | `main` |
| `origin/main` SHA at review | `55735d1282b390ab968f16f5bb612282bfe46a2c` |
| Prompt | `prompts/f3-1-2-ledger-required-nonprod-activation-acceptance.md` |

Canonical authority files read: activation evidence, F3.1.2b acceptance + implementation evidence, F3.1.2 implementation plan, current-state, implementation-progress, prompts/README, activation prompt, ADR-009, ADR-013, product-convergence evidence.

### Independent ADO / runtime lineage: **YES**

| Check | Result |
|---|---|
| Local HEAD | `22495229502dabf2d99588599a156d862c5114fa` |
| Parent | Exact `f48dc825ab5d1d16fafc3f70ef772d28613df1a0` |
| Independent `az repos ref list` `objectId` | `22495229502dabf2d99588599a156d862c5114fa` (`refs/heads/feat/ado-repo-governance`) |
| Local `origin/feat/ado-repo-governance` | Same SHA |
| Source drift after accepted F3.1.2b | **NONE** (working tree: untracked `.vscode/` only; overlay and SQLite gitignored) |
| Committed `app-config.yaml` | `newSubmissionAuthorizationMode: LEGACY_PRE_F3` |

### Independent laptop runtime/database: **YES**

The activation process was no longer listening on `:3000`/`:7007` at review start. Durable SQLite and the gitignored overlay were still present. This review restarted the **same** accepted binary without changing overlay, config, or ledger data, then re-inspected HTTP/UI.

| Field | Independent observation |
|---|---|
| Overlay | gitignored `app-config.local.yaml` — `newSubmissionAuthorizationMode: LEDGER_REQUIRED` only |
| Database | `packages/backend/data/change-management.sqlite` (gitignored `**/data/`) |
| Restart | `yarn start` → `Loaded config from app-config.yaml, app-config.local.yaml` |
| Live startup | `newSubmissionAuthorizationMode LEDGER_REQUIRED`; policy `default-change-authorization@2026-09-19.1`; selector bundle `selector-bundle-dev@2026-09-07.1` digest `6a0c2fb4f30037603848cb72700e77d60f68cf35a25e6db055c3690780d28b94` |
| Entra | signed-in `user:default/diego.fernandes_outlook.com`; catalog membership relations ready |

---

## 3. Mandatory gate matrix (G1–G15)

| Gate | Result | Concise independent basis |
|---|---|---|
| **G1** Source and runtime lineage | **PASS** | Accepted SHA `2249522` is local HEAD, origin tip, and `az repos` tip. Parent is exact `f48dc82`. No unaccepted Change/ledger/config source drift. Restarted runtime is that binary. |
| **G2** Non-production isolation | **PASS** | Operator-laptop `yarn start`, local SQLite, gitignored overlay. No production cluster/namespace/cloud DB/GitOps target. Overlay contains no secrets. |
| **G3** Activation mechanism | **PASS** | Overlay only. Committed YAML remains `LEGACY_PRE_F3` and architecture test still forbids `LEDGER_REQUIRED` there. Live startup logged LEDGER from `app-config.yaml, app-config.local.yaml`. No new config framework. |
| **G4** Pre-cutover legacy control | **PASS** | `CHG-2026-000002`: one completed reservation `LEGACY_PRE_F3`, index `submitted`, zero rounds/requirements/audits, created `2026-09-20T15:06:18.658Z` before `CHG-2026-000003`. Product UI titles match the activation control. |
| **G5** Live ledger submission | **PASS** | `CHG-2026-000003`: reservation+index `LEDGER_REQUIRED`, finalized, one provider row, Round 1 only, snapshot SHA `97d9da25…` matches stored canonical JSON, policy `default-change-authorization@2026-09-19.1`, selector bundle identity/digest as startup, exactly `normal-primary-approval` + `cab-approval` with `requirement_id = source_ref`, resolved principals present, zero decisions on this Change. |
| **G6** Canonical submission audit | **PASS** | Exactly five `CHG-2026-000003` audit rows: one `round_created`, one `policy_selected`, one `selector_bundle_bound`, two `requirement_materialized`. No decision audit on this Change. |
| **G7** Idempotent replay | **PASS** | Durable set remains one reservation/index/Round/two requirements/five audits/one provider record. Original runtime log `POST /changes` `201` at `2026-09-20T15:11:36.055Z` with `mode=LEDGER_REQUIRED source=reserved`. This review did not fire a new create. |
| **G8** Different-payload conflict | **PASS** | Original runtime log `POST /changes` `409` / 181 bytes at `2026-09-20T15:11:43.082Z`. No second Change/Round/requirement/audit/provider row exists. |
| **G9** Stored-mode-wins | **PASS** | Replay log at `2026-09-20T15:11:49.983Z`: `mode=LEGACY_PRE_F3 source=reserved` while runtime was LEDGER. `CHG-2026-000002` still LEGACY, still zero rounds. |
| **G10** Same-binary backout | **PASS** | Backout startup logged `LEGACY_PRE_F3`. `CHG-2026-000004` stored LEGACY, zero rounds. `CHG-2026-000003` remained LEDGER with unchanged Round 1. No historical round deleted. |
| **G11** Final steady state | **PASS** | Overlay still LEDGER; this review's restart again logged LEDGER. Committed default still LEGACY. |
| **G12** Product preservation | **PASS** | Independent UI: `/gmud` lists 000002/000003/000004; `/gmud/CHG-2026-000003` reachable; Catalog owned components reachable; Deployments tab selected on `idp-showcase-api`; `delivery.deployment.read` ALLOW. No `collision` / `duplicate extension` / `api:catalog/delivery` warnings. Pre-existing kubernetes-config warning unchanged. |
| **G13** Execution eligibility | **PASS** | Independent signed-in GET `2026-09-20T15:47:30.299Z`: `decision=DENY`, `reason=PENDING_AUTHORIZATION`, `roundNumber=1`. Not mocked. After the read: still 1 round, 2 requirements, 5 audits, 0 decisions for `CHG-2026-000003`. |
| **G14** Rollback safety | **PASS** | `SELECT COUNT(*) FROM change_idempotency WHERE authorization_mode='LEDGER_REQUIRED' AND state='pending'` = `0`. Old-binary downgrade was not performed. |
| **G15** Forbidden scope | **PASS** | ADO source unmodified; no migration added; production untouched; policy/selector mappings unchanged; no ApprovalDecision fabricated for the activation Change (the only decision row is pre-existing sandbox `CHG-2026-000001` dated 2026-09-05); no F3.1.3/F3.1.4/F3.2 behavior; no Kargo/Argo/GitOps mutation. |

**Gates: 15 / 15 PASS.**

Independent SQLite identities at review:

| changeId | mode | rounds | notes |
|---|---|---|---|
| `CHG-2026-000001` | `LEDGER_REQUIRED` | 1 (sandbox policy) | pre-existing; one historical decision; left untouched |
| `CHG-2026-000002` | `LEGACY_PRE_F3` | 0 | pre-cutover control |
| `CHG-2026-000003` | `LEDGER_REQUIRED` | 1 (CAB-safe policy) | activation proof |
| `CHG-2026-000004` | `LEGACY_PRE_F3` | 0 | same-binary backout proof |

---

## 4. F3.1.3 demo target readiness: READY

The laptop runtime is not a shared Kubernetes DEV cluster. That absence alone does not make it invalid.

Proven product-validation properties:

- the accepted `2249522` binary restarts reliably with the existing gitignored overlay;
- local SQLite ledger facts survive process abort and restart (logical identities/counts unchanged after this review's restart and eligibility re-read);
- real Entra sign-in and Catalog selector resolution work (`user:default/diego.fernandes_outlook.com`, group membership indexed);
- real HTTP/UI flows work for GMUD list/detail, Catalog, Deployments, Delivery reads, and execution-eligibility evaluation without production dependencies.

Limitation (not a REJECT): the process is operator-laptop `yarn start` and the overlay is gitignored. A later shared DEV target remains preferable, but this laptop is adequate for F3.1.3 product validation.

This is **not** a production-readiness finding.

---

## 5. Decision

```text
F3.1.2 non-production LEDGER_REQUIRED activation acceptance: ACCEPT
Activation baseline: ACCEPTED
Accepted runtime SHA: 22495229502dabf2d99588599a156d862c5114fa
Committed repository default: LEGACY_PRE_F3
Accepted non-production effective mode: LEDGER_REQUIRED
Production cutover: NOT AUTHORIZED
F3.1.3 demo target readiness: READY
F3.1.3 planning/prompt authoring: GO
F3.1.3 implementation: still requires separate explicit authorization
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
```

---

## 6. Final report

```text
Docs baseline reviewed: 55735d1282b390ab968f16f5bb612282bfe46a2c
ADO/runtime SHA verified: 22495229502dabf2d99588599a156d862c5114fa
Independent runtime/database verification: YES
Source/lineage gate: PASS
Isolation gate: PASS
Activation mechanism gate: PASS
Legacy control gate: PASS
Live ledger submission gate: PASS
Canonical audit gate: PASS
Idempotent replay gate: PASS
Different-payload conflict gate: PASS
Stored-mode-wins gate: PASS
Same-binary backout gate: PASS
Final steady-state gate: PASS
Product preservation gate: PASS
Execution eligibility gate: PASS
Rollback safety gate: PASS
Forbidden-scope gate: PASS
F3.1.2 non-production LEDGER_REQUIRED activation acceptance: ACCEPT
F3.1.3 demo target readiness: READY
ADO/source modified by review: NO
Production modified by review: NO
Final docs SHA: recorded by the documentation commit that publishes this review
```

---

## 7. STOP

```text
STOP
ADO implementation modified: NO
Production modified: NO
F3.1.2 non-production LEDGER_REQUIRED activation: ACCEPT
F3.1.3 demo target readiness: READY
Next authorized activity: F3.1.3 planning/prompt authoring
F3.1.3 implementation: still requires separate explicit authorization
F3.1.4 implementation: NO-GO
F3.2 implementation: NO-GO
Production cutover: NOT AUTHORIZED
```
