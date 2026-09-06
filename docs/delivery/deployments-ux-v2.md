# Deployments UX v2 — Approved Component View

## Status

**Approved visual/product reference for the Component → Deployments tab.**

This document is normative for the next Deployments UX implementation checkpoint. It refines the existing MVP screen without changing ADR-012 domain boundaries, provider authority, Change semantics, or production rollout status.

Visual reference:

![Approved Deployments v2 reference](./assets/deployments-screen-v2.webp)

The image is the **layout and information-hierarchy contract**. Example names, dates, versions, users, namespaces, error text, and provider values in the image are illustrative unless they already exist in the running sandbox. The implementing agent must use real backend data and must not fabricate state merely to match the screenshot.

## Why v2 is needed

The MVP Deployments tab proved the vertical architecture, but the current screen is not acceptable as the intended product UX. The present experience is essentially:

- one release-candidate text field whose interaction is unclear;
- three flat environment cards;
- weak hierarchy between release, promotion, provider state, and governance;
- very little historical context;
- too much unused space;
- provider details exposed without enough product framing;
- insufficient visual explanation of why PRD is blocked or eligible.

The v2 screen preserves the same domain model while composing it into a useful developer-facing product surface.

## Product question answered

`Catalog → Component → Deployments` answers:

> **What release am I looking at, where is it deployed, what is blocking the next promotion, and what governed change/evidence is associated with production?**

It is a component-scoped operational view. It is not the future global Delivery workbench.

## Architecture constraints

The screen must continue to express the accepted authority model:

> **Change Management authorizes. Delivery promotes. Git declares. Argo CD reconciles. Kubernetes executes. Backstage composes the experience.**

Consequences for UX:

1. **GMUD is governance context, not deployment status.**
2. **Deployment success does not complete a multi-activity Change.**
3. **PRD eligibility is distinct from Argo health/sync.**
4. **Kargo/Argo are operational projections, not the primary product vocabulary.**
5. **Backstage actions are affordances; backend authorization remains authoritative.**
6. **The same immutable ReleaseCandidate is the unit promoted across environments.**
7. **Do not introduce Azure DevOps pipeline/stage language as the canonical deployment UX.**

## Approved page composition

The page uses the existing Backstage Component header and tabs. Do not redesign the global shell, navigation rail, component masthead, or unrelated tabs.

Inside the `Deployments` tab, use the following vertical composition.

### 1. Page heading / freshness row

Left:

- title: `Deployments`;
- one concise explanatory sentence.

Right:

- `Atualizar` / refresh action;
- last-updated timestamp;
- compact freshness indicator such as `Dados atualizados`.

Do not put refresh as a dominant primary action.

### 2. Release candidate panel

One full-width horizontal panel directly below the heading.

Required content, from left to right:

1. **Release selector**
   - visible current version/tag or digest-short form;
   - current/selected badge when useful;
   - dropdown/autocomplete behavior if the backend supports listing candidates.
2. **Commit**
   - short SHA;
   - external/deep link when available.
3. **Published/created timestamp**.
4. **Artifact/image digest**
   - shortened for display;
   - copy/deep-link affordance if available.
5. **Source branch** only as provenance if available.
6. Primary action: **Solicitar promoção**.
7. Secondary action: **Vincular GMUD** when applicable.
8. Overflow menu only for truly secondary actions.

### Release-selector truthfulness rule

The control must not look like a search/select field unless it actually supports a meaningful selection interaction.

Preferred behavior:

- list recent ReleaseCandidates for the component;
- selected ReleaseCandidate drives the entire screen state.

If the current backend cannot list candidates yet, use a truthful selected-release summary plus an explicit action to choose/register a release. Do **not** keep the MVP's ambiguous free-text `Release candidate ID` field in the final v2 UI.

### 3. Main two-column area

Desktop target ratio is approximately **70/30**:

- left: environment/status progression;
- right: Change/GMUD governance context.

On narrower layouts, the governance panel may stack below the environment area using normal responsive Backstage behavior.

## Environment/status area

Heading: `Status por ambiente` or equivalent.

Subtitle: one short sentence explaining that the selected version is shown across environments.

Display three environment cards on one row for the current MVP topology:

```text
DEV → HML → PRD
```

A subtle progression connector may be shown between cards. It must not imply that environment state is a workflow engine or that a failed HML automatically defines Change lifecycle.

### Card anatomy

Each card must use the same structure:

1. environment name;
2. prominent semantic status badge;
3. selected/current version or `—` when not deployed;
4. deployment/promotion timestamp where meaningful;
5. Argo Sync projection;
6. Argo Health projection;
7. namespace/target label when available;
8. concise blocking/failure explanation when applicable;
9. `Ver detalhes`;
10. optional `Abrir no Argo` or provider deep link as a secondary technical affordance.

Provider names should be secondary to the product status.

### Semantic status examples

Use product-level states such as:

- `Sucesso` / deployed;
- `Em andamento`;
- `Falhou`;
- `Aguardando`;
- `Mudança necessária`;
- `Aguardando autorização`;
- `Elegível`;
- `Conflito` / target busy, when surfaced;
- `Desconhecido` only when projection really is unavailable.

Do not derive a red failure badge solely from historical provider noise if the current deployment is healthy. The selected release/request context must be clear.

### DEV example

Green/success treatment only when supported by actual current selected-release deployment state.

Show:

- deployed timestamp;
- Argo Synced;
- Argo Healthy;
- namespace/target;
- details/deep link.

### HML failure example

Red/error treatment.

Show:

- latest attempt;
- sync/health state;
- one concise human-readable reason, e.g. image pull/readiness failure;
- provider raw details behind `Ver detalhes`, not dumped into the card.

### PRD waiting example

Neutral/blue waiting treatment.

If a production request requires governance before promotion:

- clearly say `Requer mudança (GMUD)` or the exact current state;
- explain the required next action;
- do not present unavailable provider values as failures;
- the primary promotion action must remain disabled/redirected appropriately until backend eligibility permits dispatch.

## Governance side panel

Heading: `Contexto de mudança (GMUD)`.

When a Change is bound, show:

- Change ID as a link;
- bound state badge;
- type;
- whole-Change status;
- **activity ID/title** bound to this DeploymentRequest;
- requested/approved execution window;
- approver/authorization summary only when supported by actual data;
- execution eligibility summary for the selected PRD request.

### Eligibility block

This is an important visual block, not a tiny metadata row.

Examples:

- green: `Elegível para produção` / ALLOW;
- amber/neutral: awaiting authorization;
- red: outside window / authorization denied;
- informational: Change required but not bound.

The exact backend reason should remain authoritative.

### Multi-activity non-completion message

When an activity is bound, preserve the E1 semantic explicitly:

> A implantação bem-sucedida é evidência para esta atividade; não conclui a mudança.

Equivalent concise wording is acceptable. Do not introduce per-activity lifecycle/status fields merely for UI decoration.

### No Change bound

The panel should explain the next step rather than appear empty. For PRD requests requiring Change, provide the existing safe `Criar/Vincular GMUD` path.

## 4. Lower information area

Use approximately two columns on desktop:

- left, wider: `Histórico de promoções`;
- right: `Eventos recentes`.

These sections are part of the approved v2 experience. They are not decorative filler.

### Promotion history

Show a compact table for recent component-scoped records.

Preferred columns:

- date/time;
- version/release;
- environment;
- status;
- requested by / actor when available;
- Change ID when present;
- action/details.

Keep the default table small (for example 5 recent rows) with `Ver todos` when more history exists.

The table must use durable Delivery facts where possible, not depend solely on provider CR retention.

### Recent events

Show a compact timeline/list for the selected release/component.

Useful event categories include:

- release registered/published;
- promotion requested;
- promotion accepted/dispatched;
- provider transition;
- Git desired-state update/PR event if already projected;
- Argo sync/health transition;
- deployment failure/success;
- governance/eligibility transition when it materially affects promotion.

Do not build a generic event-sourcing subsystem for this page. Use the facts already available from Delivery/provider projections and add only the smallest product-specific read model if necessary.

## Action rules

### Solicitar promoção

The primary action is contextual and must never imply an unauthorized execution is possible.

It may:

- create/request the next valid DeploymentRequest;
- route the user to missing Change binding;
- show awaiting authorization;
- dispatch only when the backend permits it.

The backend remains authoritative; UI disablement is not a security control.

### Vincular GMUD

Use the existing Delivery-owned `ChangeBinding` semantics (`changeId + activityId`).

Do not add deployment/provider IDs to canonical Change.

### Atualizar

Refresh projections/read models. Do not use refresh as a hidden imperative Argo sync.

### Deep links

Provider links (`Abrir no Argo`, Git PR, etc.) are secondary troubleshooting affordances. They must not become the main happy path.

## Visual rules

The approved reference intentionally uses:

- existing Backstage shell/header;
- white/light content surface;
- restrained blue accents;
- semantic green/red/blue status treatments;
- thin borders and modest radius;
- strong whitespace hierarchy without large dead areas;
- compact metadata;
- one visually dominant primary action;
- no decorative dashboard charts.

Do not introduce:

- gradients or custom brand chrome inside the content area beyond what Backstage already provides;
- oversized hero cards;
- pie/bar charts for the MVP state;
- animated pipeline diagrams;
- kanban/workflow metaphors;
- duplicated GMUD approval buttons;
- provider-specific giant cards;
- a second global navigation system.

## Required states to verify in browser

At minimum, validate real or controlled fixture/projection states for:

1. selected release visible;
2. DEV success;
3. HML failure or another provider failure state;
4. PRD Change required/unbound;
5. PRD Change bound but not eligible;
6. PRD ALLOW/eligible;
7. PRD success with activity-bound non-completion message;
8. same-target conflict/busy error surfaced cleanly if reachable;
9. stale/refresh behavior does not show an arbitrary old request as current.

Do not fabricate sandbox history to satisfy screenshot cosmetics. When a state cannot be driven safely, verify the rendering through existing focused UI test fixtures and record that it was fixture-validated rather than live-validated.

## Data/API guidance

Prefer existing Delivery APIs and persisted facts.

Do not create a new workflow engine, analytics subsystem, or canonical domain concept merely to fill the page.

If the current APIs cannot support a required v2 block, first identify the precise read-model gap. The authorized UX checkpoint may add the **smallest provider-neutral read/query surface** necessary for:

- listing recent component ReleaseCandidates;
- recent promotion history;
- recent event/projection entries.

Any such addition must remain inside Delivery and preserve ADR-012 boundaries.

## Non-goals

This v2 checkpoint does not authorize:

- production rollout GO;
- P1 credential follow-up work;
- HA/DR;
- break-glass;
- rollback automation;
- supply-chain expansion;
- the global Delivery workbench;
- redesign of GMUD;
- redesign of Platform tab;
- new environment topology beyond the current DEV/HML/PRD MVP;
- a generic workflow engine.

## Acceptance criteria

The v2 implementation is acceptable when:

- the rendered page is recognizably faithful to the approved reference image;
- the ambiguous MVP ReleaseCandidate free-text field is removed or replaced by a truthful interaction;
- release context is clearly separated from environment state and governance context;
- DEV/HML/PRD status is scannable without reading raw provider strings;
- GMUD and execution eligibility are clear without conflating Change and Deployment;
- activity-bound success does not imply whole-Change completion;
- promotion history and recent events provide useful context from factual data;
- actions are contextual and backend-authoritative;
- Backstage Platform and other component tabs remain unbroken;
- browser validation is performed against the running application;
- screenshots of the implemented result are captured and compared against this reference;
- no accepted ADR-012 boundary is changed.
