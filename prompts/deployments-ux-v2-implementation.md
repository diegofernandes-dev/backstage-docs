# Deployments UX v2 — Guided Implementation Checkpoint

You are acting as a senior Staff+/Principal Platform Engineer with strong product/Backstage UX judgment.

Your task is **not to invent a new Deployments experience**.

Your task is to implement the already-approved `Deployments UX v2` reference faithfully, using the existing Delivery/Change architecture and only the minimum provider-neutral read/query changes needed to make the screen truthful.

Core rule:

> **Implement the approved screen. Do not reinterpret it.**

If the approved visual/spec conflicts materially with the accepted architecture or actual backend truth, STOP and report the conflict instead of creatively redesigning the page.

---

## 0. Canonical source protocol

Canonical documentation repository:

```text
diegofernandes-dev/backstage-docs
branch: main
```

Always begin with:

```bash
git fetch origin main
```

Record the docs SHA used as your execution baseline.

Read at minimum:

```text
docs/delivery/deployments-ux-v2.md
docs/delivery/assets/deployments-screen-v2.webp
docs/adr/ADR-012-delivery-management-gitops-promotion.md
docs/delivery/README.md
docs/delivery/mvp-vertical-delivery-slice.md
docs/delivery/mvp-demo-hardening.md
docs/delivery/e1-multi-activity-concurrency.md
docs/delivery/eligibility-window-toctou.md
docs/delivery/p1-production-authority-hardening.md
docs/golden-paths/current-state.md
```

The implementation source of truth is the current Azure DevOps Backstage repository. Inspect its actual branch/SHA and working tree before editing.

Do not rely on remembered SHAs or earlier chat summaries when live repository evidence is available.

---

## 1. Current architecture baseline — immutable for this checkpoint

ADR-012 is **Accepted**.

The authority model is:

```text
Change Management authorizes.
Delivery promotes.
Git declares.
Argo CD reconciles.
Kubernetes executes.
Backstage composes the experience.
```

You must preserve these boundaries.

Specifically:

- Change is not a DeploymentRequest.
- Change does not gain Kargo/Argo/ADO/deployment IDs.
- `ChangeBinding` remains Delivery-owned and activity-scoped where applicable.
- one successful deployment does not complete a multi-activity Change.
- Backstage UI is not runtime authority.
- Kargo/Argo remain provider/execution projections, not business-approval authorities.
- backend eligibility remains authoritative even when UI buttons are disabled.

Production rollout remains **NO-GO** independently of this UX checkpoint.

Do not touch that decision.

---

## 2. Product problem

The current Deployments tab is a proven MVP surface but poor product UX.

Observed problems include:

- ambiguous `Release candidate ID` free-text/search-looking field;
- effectively three isolated cards and little else;
- unclear relationship between selected release, current environment state, promotion intent, and GMUD;
- raw provider states without enough product framing;
- poor use of screen space;
- no useful component-scoped promotion history;
- no concise recent-event context;
- weak explanation of PRD blocking/eligibility;
- low confidence that a stakeholder or squad member understands what action to take next.

This checkpoint exists to make the already-proven Delivery MVP feel like an intentional Backstage product surface.

---

## 3. Normative visual contract

The primary UX contract is:

```text
docs/delivery/deployments-ux-v2.md
docs/delivery/assets/deployments-screen-v2.webp
```

Open and inspect the image before coding.

The final desktop implementation must be **recognizably faithful** to that reference.

### What is normative from the image

Normative:

- page composition;
- content hierarchy;
- section ordering;
- approximate relative widths;
- release panel anatomy;
- DEV/HML/PRD card composition;
- right-side governance panel;
- lower promotion-history + recent-events composition;
- primary vs secondary action hierarchy;
- restrained semantic use of green/red/blue;
- compact Backstage-style density.

Illustrative only:

- exact version/tag;
- exact timestamps;
- exact names/users;
- exact Change ID;
- exact namespace names;
- exact failure string;
- exact branch;
- exact provider values.

Use factual data from the running system.

### Creativity constraint

Do **not** replace the approved layout with:

- your preferred dashboard;
- another tab structure;
- charts;
- a stepper-heavy wizard;
- kanban;
- a timeline-first design;
- giant provider panels;
- a generic deployment-detail application;
- a new global Delivery workbench.

Small implementation adaptations for Backstage component APIs/responsiveness are allowed only when they preserve the approved information architecture.

---

## 4. Required rendered structure

Implement the content in this exact order.

### 4.1 Page heading and freshness

Inside the existing Component → Deployments tab:

Left:

```text
Deployments
<one-line description>
```

Right:

```text
Atualizar | last-updated timestamp | freshness indicator
```

Do not make refresh the primary CTA.

### 4.2 Release candidate panel

One full-width horizontal card/panel.

Left-to-right content:

1. selected ReleaseCandidate selector/summary;
2. commit short SHA + deep link when available;
3. created/published timestamp;
4. artifact/image digest short form;
5. source branch as provenance only when available;
6. **Solicitar promoção** as primary action;
7. **Vincular GMUD** as secondary action where applicable;
8. overflow only for truly secondary actions.

#### Mandatory removal of misleading MVP interaction

The existing ambiguous free-text `Release candidate ID` control must not survive as-is.

Preferred implementation:

- list/select recent ReleaseCandidates belonging to the component;
- selected release drives all status/history/event presentation.

If listing candidates is not currently supported, implement the smallest provider-neutral Delivery query necessary.

If that cannot be done safely inside this checkpoint, use a truthful selected-release summary with a clear explicit selection/registration action.

Do not render a fake autocomplete/search field.

### 4.3 Main area: approximately 70/30

Desktop:

```text
| Status por ambiente (~70%) | Contexto de mudança (~30%) |
```

Responsive stacking is allowed below normal desktop width.

### 4.4 Environment section

Header:

```text
Status por ambiente
Visualize o status da versão selecionada em cada ambiente.
```

One row:

```text
DEV → HML → PRD
```

A subtle connector is permitted, matching the approved reference.

Do not implement a workflow/state-machine visualization.

#### Environment card contract

All cards use the same anatomy:

- environment title;
- semantic product-status badge;
- deployment/promotion timestamp;
- Argo Sync projection;
- Argo Health projection;
- namespace/target when available;
- concise contextual message if blocked/failed;
- `Ver detalhes`;
- optional provider deep link such as `Abrir no Argo`.

Keep provider labels secondary.

#### DEV

Success treatment only when selected release/current state supports it.

#### HML

Failure treatment must show one human-readable reason, not a wall of provider text.

#### PRD

Use a waiting/governance treatment where appropriate.

Examples:

- Change required;
- Change bound, awaiting authorization;
- outside window;
- eligible;
- promotion in progress;
- succeeded;
- target busy/conflict.

Do not show `failed` solely because a historical Kargo Promotion errored while Argo currently has the selected artifact healthy. Correctly distinguish current selected-release state from historical attempt state.

### 4.5 Governance side panel

Heading:

```text
Contexto de mudança (GMUD)
```

When bound, show factual fields where available:

- Change ID/link;
- bound badge;
- type;
- whole-Change lifecycle status;
- activity ID/title;
- requested/approved window;
- authorization/approver summary when factual;
- execution eligibility for the selected governed PRD request.

#### Eligibility callout

Make eligibility visually prominent:

- ALLOW → green `Elegível para produção`;
- pending → neutral/amber;
- denied/outside window → red/amber with concise reason;
- missing binding → informational Change-required state.

Never infer eligibility client-side from UI metadata if the backend exposes a real eligibility result.

#### E1 multi-activity invariant

Preserve explicit copy equivalent to:

```text
A implantação bem-sucedida é evidência para esta atividade; não conclui a mudança.
```

Do not add activity `status`, `completed`, or workflow-task semantics.

### 4.6 Promotion history

Bottom-left, wider panel.

Heading:

```text
Histórico de promoções
```

Default to a compact recent list/table, ideally 5 rows.

Preferred columns:

```text
Data | Versão | Ambiente | Status | Solicitado por | Mudança | Ações
```

Use durable Delivery records where possible.

`Ver todos` may be a simple expansion/navigation affordance if supported. Do not build a new reporting product.

### 4.7 Recent events

Bottom-right panel.

Heading:

```text
Eventos recentes
```

Compact vertical timeline/list.

Only show meaningful Delivery/product events available from factual data, for example:

- release registered;
- promotion requested;
- promotion dispatched;
- provider transition;
- Git desired-state update/PR when already projected;
- Argo sync/health transition;
- failure/success;
- governance/eligibility transition.

No generic event-sourcing framework is authorized.

---

## 5. Data/read-model rules

First inspect what the existing Delivery APIs/tables already expose.

Reuse them before adding anything.

You may add only the smallest provider-neutral read/query capability necessary to support the approved page, specifically for one or more of:

- recent component ReleaseCandidate selection;
- promotion history;
- recent product-relevant events.

If a new endpoint/query is needed:

- keep it in Delivery;
- keep provider IDs/details as projections;
- do not change Change canonical contract;
- do not introduce a generic analytics/event platform;
- persist only if existing durable Delivery facts are insufficient and the minimal new fact clearly belongs to Delivery.

If you discover a larger domain-model requirement, STOP rather than expanding scope.

---

## 6. Action behavior

### Solicitar promoção

The main CTA must be contextual.

It may create/request the next legitimate promotion intent and must reflect current selected release/target state.

Do not allow UI to appear to bypass:

- Change required;
- authorization deny;
- outside-window deny;
- target-busy conflict.

Backend dispatch/eligibility remains authoritative.

### Vincular GMUD

Use existing Delivery `ChangeBinding` semantics.

Activity binding must remain explicit where required.

### Atualizar

Refresh Delivery/provider projections only.

Do not translate `Atualizar` into imperative `argocd app sync`.

### Provider deep links

Argo/Kargo/Git links are technical drill-down paths, not primary product actions.

---

## 7. Visual implementation guardrails

Stay inside the established Backstage design language.

Prefer existing Backstage/Material components and theme tokens already used by this application.

Target characteristics:

```text
clear
compact
professional
easy to scan
restrained
credible for stakeholder demo
```

Preserve:

- existing component header;
- existing global nav;
- existing tab bar;
- existing theme.

Do not redesign the shell.

Do not introduce:

- new global colors/theme;
- decorative charts;
- flashy gradients in page content;
- animation-heavy progression;
- huge empty cards;
- custom iconography when existing icons suffice.

Approximate desktop proportions and spacing should match the reference image. Pixel-perfect reproduction is not required, but changing the information architecture is not allowed.

---

## 8. Browser-driven validation is mandatory

You can navigate the running Backstage UI. Use that capability extensively for this checkpoint.

Do not stop at source inspection or tests.

At minimum:

1. start/verify the real Backstage application;
2. navigate to the IDP Showcase API or current test Component;
3. open `Deployments`;
4. verify the final composition visually;
5. exercise release selection if implemented;
6. exercise refresh;
7. inspect details/deep links where safe;
8. navigate to the bound GMUD and back;
9. verify Platform tab still opens;
10. verify no console/network errors caused by the UX refactor.

Capture screenshots of the final implementation.

### Visual comparison requirement

Compare the implemented screen side-by-side against:

```text
docs/delivery/assets/deployments-screen-v2.webp
```

Report differences explicitly.

A screen that is functionally correct but materially diverges from the approved composition is not `PASS`.

---

## 9. Required state validation

Prove rendering/behavior for these states using live sandbox data when safe, otherwise focused test fixtures that use the same component contracts:

1. selected release shown;
2. DEV success;
3. HML/provider failure;
4. PRD Change required/unbound;
5. PRD Change bound but not eligible;
6. PRD ALLOW/eligible;
7. PRD success with activity non-completion message;
8. same-target busy/conflict, if reachable;
9. stale/current request selection behaves correctly.

Label evidence as `LIVE` or `FIXTURE/TEST`.

Do not mutate the sandbox merely to make the screenshot pretty.

---

## 10. Tests

Add/adjust focused tests for product behavior affected by the refactor.

At minimum retain existing Delivery regressions and add coverage for:

- release selector/selection mapping;
- semantic environment status mapping;
- Change/eligibility panel state;
- activity non-completion message;
- history ordering/current component scoping;
- recent-event ordering/scoping if introduced;
- current-request selection does not regress to stale arbitrary request;
- component tab loader remains healthy.

Do not create a broad end-to-end test framework if existing patterns suffice.

Run the relevant existing backend/frontend suites after implementation.

---

## 11. Scope authorization — GO

You are authorized to:

- refactor the existing Deployments tab layout/components;
- replace the ambiguous ReleaseCandidate free-text UX;
- add the minimum component-scoped release list/query;
- add the minimum promotion-history read model/query;
- add the minimum recent-events read/query projection;
- improve semantic status mapping;
- fix directly related stale/projection bugs found while implementing v2;
- add focused tests;
- use browser navigation to refine implementation toward the approved reference;
- update factual canonical docs with implementation evidence.

---

## 12. Explicit NO-GO

Do **not**:

- change ADR-012 boundaries;
- redesign Change Management;
- add provider IDs to canonical Change;
- create per-activity lifecycle/workflow tasks;
- implement global Delivery workbench;
- change production rollout gate;
- continue P1 credential/token-refresh hardening;
- implement HA/DR;
- implement break-glass;
- implement automatic rollback;
- expand supply-chain/security controls;
- redesign Platform tab;
- redesign the Backstage shell/theme;
- introduce a generic workflow/state-machine/event platform;
- add new environments beyond current MVP merely for visual completeness.

If implementation requires one of these, STOP and ask for a separate authorization.

---

## 13. Documentation/evidence output

Update canonical documentation only with factual implementation results.

Create a result record preferably at:

```text
docs/delivery/deployments-ux-v2-implementation.md
```

Record:

- docs baseline SHA;
- implementation repo/branch/before/after SHA;
- files changed;
- APIs/read models added, if any;
- tests and results;
- browser route tested;
- state matrix (`LIVE` vs `FIXTURE`);
- screenshots of final screen (store in docs if practical);
- explicit visual differences from the approved reference;
- known limitations;
- verdict.

Do not rewrite historical checkpoint evidence.

---

## 14. Acceptance gate

### PASS

Only if all are true:

- final page is recognizably faithful to the approved reference;
- misleading release-ID free text is gone;
- release context clearly drives the screen;
- DEV/HML/PRD states are scannable and product-level;
- governance panel clearly expresses Change + activity + eligibility;
- deployment success cannot be read as whole-Change completion;
- history and recent events are factual/useful;
- actions are contextual and backend-authoritative;
- no architecture boundary changed;
- browser validation completed;
- relevant tests green;
- Platform and core Component tabs still work.

### CONDITIONAL_PASS

Only for a narrow nonblocking UX/data limitation with a documented truthful fallback.

Do not use `CONDITIONAL_PASS` to hide:

- a fake release selector;
- materially different layout;
- stale/wrong deployment state;
- broken actions;
- broken Platform/Component navigation.

### FAIL

If:

- approved reference is materially ignored;
- UX remains essentially the three-card MVP;
- data is fabricated or misleading;
- Change/Delivery semantics are conflated;
- UI implies unauthorized production dispatch;
- implementation requires architecture redesign.

---

## 15. Final report format

Use exactly these sections:

```markdown
# Deployments UX v2 Result

## 1. Baseline
## 2. Verdict
## 3. Approved Reference Compliance
## 4. UI Changes Implemented
## 5. Data / API Changes
## 6. Release Candidate UX
## 7. Environment Status UX
## 8. Governance / Eligibility UX
## 9. History / Recent Events
## 10. Browser Validation
## 11. State Evidence Matrix
## 12. Tests
## 13. Regressions / Defects Found
## 14. Documentation Updated
## 15. Known Limitations
## 16. STOP
```

Final `STOP` means:

- do not start production adoption review;
- do not continue P1 residual hardening;
- do not build the global Delivery workbench;
- do not redesign another Backstage tab.
