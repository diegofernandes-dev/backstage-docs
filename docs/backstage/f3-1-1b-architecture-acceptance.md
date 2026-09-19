# F3.1.1b — Architecture Implementation Acceptance

- **Status:** CLOSED / ACCEPTED IMPLEMENTED BASELINE
- **Date:** 2026-09-19
- **Verdict:** `ACCEPT`
- **Implementation:** Azure DevOps `platform-devops-developer-portal@188d8e9cc43423f3644b3cacfb9849257838a583`, branch `feat/ado-repo-governance`
- **Expected parent (verified):** accepted F3.1.1a baseline `d3c0751a15b908cec8f5595c97e52f41226344ed`
- **Documentation review baseline:** `backstage-docs@cf9f96cbf0ae8488a356c32f6f13c0ebb62b2ee3` (`origin/main` at review start)
- **Authority:** ADR-009 + F3.1.1-R2 plan + [`f3-1-1b-implementation-evidence.md`](./f3-1-1b-implementation-evidence.md) + this independent review

```text
F3.1.1b: CLOSED / ACCEPTED IMPLEMENTED BASELINE
Implementation: platform-devops-developer-portal@188d8e9cc43423f3644b3cacfb9849257838a583
F3.1.2 planning: GO
F3.1.2 implementation: NO-GO pending a separate reviewed plan and explicit authorization
```

---

## 1. Status / verdict

F3.1.1b architecture implementation acceptance: **ACCEPT**.

All mandatory gates **G1–G17 PASS**. No architecture blocker remains for closing this slice. Acceptance authorizes **F3.1.2 planning only**. It does not authorize F3.1.2 implementation, `POST /changes` wiring, AuthorizationRound creation, Teams/CAB UI, Delivery coupling, or production rollout.

---

## 2. Reviewed baselines and evidence limitations

### Docs baseline

| Item | Value |
|---|---|
| Repo | `diegofernandes-dev/backstage-docs` |
| Branch | `main` |
| `origin/main` SHA at review | `cf9f96cbf0ae8488a356c32f6f13c0ebb62b2ee3` |
| Prompt | `prompts/f3-1-1b-architecture-implementation-acceptance.md` |

Minimum reads completed: ADR-009, F3.1 / F3.1.1 plans, F3.1.1a acceptance, F3.1.1b implementation evidence, current-state, implementation-progress, and this acceptance prompt.

### Independent source verification: **PARTIAL**

| Check | Result |
|---|---|
| Local git object for candidate SHA | **YES** — inspected at detached worktree `/private/tmp/platform-devops-f311b-review` |
| Parent exact match | **YES** — `188d8e9^ == d3c0751` |
| Local `remotes/origin/feat/ado-repo-governance` tip | **equals** `188d8e9` |
| Fresh `git fetch` from Azure DevOps in this session | **FAILED** (`remote: One or more errors occurred.` / SSH) |
| Live ADO REST tip re-confirmation this session | **NOT performed** |
| Independent source-tree inspection at exact SHA | **YES** |
| Focused unit/architecture/publication tests re-run | **YES** (13 suites / 164 passed; live Catalog suite skipped) |
| Change Management service regressions re-run | **YES** (61 passed) |
| Backend lint / build / publication CLI re-run | **YES** (all exit 0) |
| Disposable PostgreSQL suite | **NOT re-executed** (no `CHANGE_MANAGEMENT_TEST_POSTGRES_URL`; suite skipped outside CI) |
| Live Catalog / second-backend startup negatives | **NOT re-executed** this session; retained as canonical evidence |

No architecture-critical claim depended solely on unreproduced live Catalog/Postgres evidence: fail-closed resolver behavior, non-expansion, digest/manifest rules, active-pair validation, and submission isolation were established from source inspection plus re-run unit tests. Canonical evidence remains credible for the live Catalog/startup negatives.

ADO implementation was **not modified** by this review.

---

## 3. Candidate lineage and scope

### Lineage

```text
4bad41d (F3.1.0-H)
  └─ d3c0751 (F3.1.1a ACCEPTED)
       └─ 188d8e9 (F3.1.1b candidate — this review)
```

Single commit on the slice: `feat(gmud): add selector bundle, Catalog resolver and selector publication`.

Local `feat/ado-repo-governance` checkout may lag; review used the exact candidate tree regardless. No local commit after `188d8e9` was found on that lineage. Session could not refresh the ADO remote; remote-tracking tip already equaled the candidate. No post-candidate F3.1.1b surface drift was observed in accessible objects.

### Scope (`d3c0751..188d8e9`)

19 paths, **+2687 / −0** (pure insertions on modified files):

- `app-config.yaml` (+29 authorization block)
- `architecture.test.ts` (additive F3.1.1b guards)
- `published-manifest.json` (+1 selector-bundle entry; policy entry byte-identical)
- `changeManagementPlugin.ts` (+21 lines, 0 deletions)
- `authorization/selector/**` (14 new files)

**Confirmed untouched:** `ChangeManagementService.ts`, migrations, HTTP route set, frontend, Delivery, approval/decision commands, Teams/CAB, `app-config.production.yaml`, CI pipeline creation. No unexplained scope widening.

---

## 4. G1–G17 gate matrix

| Gate | Result | Concise basis |
|---|---|---|
| **G1** ADR-009 authority boundary | **PASS** | Selectors produce `PrincipalResolutionSnapshot` only; Catalog is resolution I/O; no ADO/Teams/Kargo/Argo/ITSM identifier in selector domain; no workflow/BPM. |
| **G2** Generic selector semantics | **PASS** | Keys are `normal-primary-approver` / `emergency-approver-a|b` / `cab-authority`; entries are only `principalType` + `principalRef`; config may point at real Catalog refs without making titles/emails domain semantics. |
| **G3** Principal type / Catalog-ref | **PASS** | `individual⇔user`, `authority|cab⇔authority`/`Group`; malformed/kind mismatch fail closed; snapshot retains Catalog ref, not display name/email. |
| **G4** Authority-group non-expansion | **PASS** | Resolver never reads `relations`/`hasMember`; snapshot is the Group ref; membership APIs forbidden by architecture guards. |
| **G5** Backend-service credentials | **PASS** | `auth.getOwnServiceCredentials()` only; requester Catalog ACL cannot skew resolution; deterministic server-side boundary. |
| **G6** Fail-closed resolver | **PASS** | Missing selector / malformed ref / kind mismatch → `INTERNAL_ERROR`; missing entity → `NOT_FOUND`; Catalog/cred throw → `PROVIDER_UNAVAILABLE`; no cache/fallback/default. |
| **G7** Active-pair startup validation | **PASS** | Validates only pinned active policy + active bundle; inactive historical policy not required to match today's bundle (dedicated test). |
| **G8** Bundle identity / digest / immutability | **PASS** | One versioned bundle; `sha256Canonical(selectors)` via existing `canonical.ts`; manifest digest mandatory; `deepFreezeSerializable`; optional app-config `contentDigest` still always checked vs computed+manifest (see §5). |
| **G9** Publication-history integrity | **PASS** | Ordinary append vs trusted `d3c0751`; policy entry unchanged; one selector-bundle entry; genesis unused; CLI + unit tests cover DIGEST_CHANGED / IDENTITY_REMOVED / duplicate fail-closed. |
| **G10** Separation-of-duty layers | **PASS** | Layer 1 enforced (distinct user-typed selector keys via `separationOfDutyKey`); layers 2–3 explicitly deferred and not falsely claimed complete. |
| **G11** Submission-path isolation | **PASS** | `bootstrapAuthorization` result held locally; not passed to `ChangeManagementService`; `createChange` still defaults/`LEGACY_PRE_F3`; no round creation; architecture guard asserts service has no selector wiring. |
| **G12** Cross-cutover idempotency | **PASS** | F3.1.1b does not touch reservation/`authorization_mode` logic; new submissions remain legacy; no version-triggered regime flip. |
| **G13** Semantic architecture guards | **PASS** | Forbidden fields / membership API shape / Delivery concepts / email-shaped values; `manager`/`director`/`cto` prose still passes. |
| **G14** Environment / production separation | **PASS** | `selector-bundle-dev` naming and prod overlay inheritance classified as **production-rollout prerequisites**, not selector-resolution architecture blockers (see §5). |
| **G15** Resolver provenance | **PASS** | Colon-separated `selector-bundle:<key>:<version>#<digest>` accepted: deterministic, unambiguous, avoids address-shape false positive. |
| **G16** No hidden F3.1.2+ wiring | **PASS** | Plugin constructs resolver at startup only; no new routes; no decision/lifecycle/participant authority changes. |
| **G17** Test/evidence credibility | **PASS** | Independent re-run: selector+policy+architecture 164 pass; CM service 61 pass; lint 0; build 0; `validate:policy-publication --baseline-ref d3c0751` OK. Postgres/live Catalog limitations recorded; canonical evidence sufficient for those claims. |

**Architecture gates: 17/17 PASS**

---

## 5. Explicit deviation decisions

| # | Deviation | Decision | Reason |
|---|---|---|---|
| 1 | Optional `contentDigest` in app-config; computed digest vs manifest still mandatory | **ACCEPTED AS WITHIN F3.1.1b** | Manifest remains the publication authority; optional declared digest is an extra operator assertion, not a bypass. |
| 2 | `resolverProvenance` colon-separated | **ACCEPTED AS WITHIN F3.1.1b** | Preserves auditability without looking like an email; no downstream contract requires `key@version`. |
| 3 | Bundle key `selector-bundle-dev` | **ACCEPTED AS WITHIN F3.1.1b** | Opaque configuration identity with an environment naming convention; not authorization semantics. |
| 4 | Production overlay inherits base/dev bundle | **ACCEPTED AS WITHIN F3.1.1b** | Valid with an explicit **production-rollout prerequisite** to publish a distinct prod bundle once a real prod principal set exists. Not a blocker to selector-resolution architecture. |
| 5 | Resolver constructed at startup, unreachable from request handling | **ACCEPTED AS WITHIN F3.1.1b** | Ensures startup validation executes without implying submission integration. |
| 6 | Emergency A/B resolved-person distinctness deferred | **ACCEPTED AS WITHIN F3.1.1b** | Correct layering; **MUST address in F3.1.2 design** before wiring submission. |

---

## 6. Architecture findings

Material findings (none blocking):

1. **Faithful ADR-009 selector half.** Configuration identities resolve to snapshotted Catalog refs under service credentials; Catalog is not an authorization authority; providers stay non-canonical.
2. **Publication model reused, not reinvented.** F3.1.1a append-only validator unchanged; F3.1.1b is a normal append.
3. **Submission remains F2/legacy.** Critical isolation gate holds; F3.1.2 is not smuggled into plugin bootstrap.
4. **Production selector publication is a later gate**, not a reason to invent prod principal refs now.

Non-blocking follow-ups (quality only):

- Prefer refreshing ADO remote connectivity before the next implementation push so tip verification does not depend on prior remote-tracking state.
- Re-run disposable Postgres and live Catalog suites during F3.1.2 planning verification when infrastructure is available.

---

## 7. Carried-forward blockers / constraints

### Before F3.1.2 implementation

- `ChangeManagementService` still calls `buildChange()` twice → **MUST FIX BEFORE F3.1.2**.
- Existing `LEGACY_PRE_F3` idempotency reservations must continue the legacy path; retries must never be reinterpreted into `LEDGER_REQUIRED`.
- F3.1.2 must fail closed if required emergency A/B effective principals resolve to the same person.
- F3.1.2 must create one canonical Change snapshot, evaluate one pinned policy + selector bundle, create Round 1 atomically with effective requirements, and remain fail-closed. Planning constraint only — not designed or implemented here.
- Delivery / ADR-012 provider-neutral boundaries remain in force; do not hardwire ADO pipeline/environment semantics into Change authorization.

### Later than F3.1.2

- Decision-time authority membership / actual actor authorization belongs to the decision-command work.
- RBAC CSV / conditional-policy completeness remains an **F3.1.4** prerequisite unless later canonical docs supersede it.
- Production selector-bundle publication waits for a real production target/principal set.
- Production rollout readiness remains a separate workstream.

---

## 8. Decision

```text
F3.1.1b architecture implementation acceptance: ACCEPT
```

F3.1.1b at `188d8e9` is the accepted implemented baseline for the selector/configuration half of F3.1.1.

---

## 9. Next gate

| Activity | Status |
|---|---|
| F3.1.1b implementation | CLOSED / ACCEPTED |
| F3.1.2 planning / review preparation | **GO** |
| F3.1.2 implementation | **NO-GO** until a separate reviewed plan and explicit authorization |
| Production rollout | Separate / deferred |

---

## 10. STOP statement

STOP after this acceptance document and the related canonical documentation updates are committed.

This checkpoint must **not**:

- fix ADO code;
- begin F3.1.2 implementation;
- wire `POST /changes` or create AuthorizationRounds;
- create approval/decision endpoints;
- touch Teams/CAB UI, Delivery, Deployments UX, or production rollout infrastructure.

The only purpose of this checkpoint was to decide whether F3.1.1b at `188d8e9` becomes an accepted implemented baseline. That decision is **ACCEPT**.
