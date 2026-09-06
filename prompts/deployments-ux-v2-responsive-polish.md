# Deployments UX v2 — Deterministic Responsive Polish

## Role

You are acting as a senior Backstage frontend engineer executing a **strictly deterministic visual/refinement checkpoint** on the already-implemented Component → Deployments page.

This is **not** a product-design task. It is **not** an invitation to improve the UX creatively. The target visual structure and responsive behavior are already decided.

Your job is to make the existing Deployments UX v2 render **exactly as specified below**, within the actual capabilities of the Backstage/MUI stack already present in the repository.

Core rule:

> **Do not redesign. Do not reinterpret. Do not improvise. Implement the specified composition and responsive behavior exactly.**

If a requested visual detail is not technically available in the existing Backstage/MUI stack, use the closest native Backstage/MUI equivalent **without changing hierarchy, wording, ordering, or information grouping**. Document that substitution explicitly.

---

## 0. Canonical baseline protocol

Before changing code:

```bash
git fetch origin main
```

Canonical docs repository:

```text
diegofernandes-dev/backstage-docs
branch: main
```

Read, in this order:

```text
docs/delivery/deployments-ux-v2.md
docs/delivery/deployments-ux-v2-implementation.md
docs/delivery/README.md
docs/adr/ADR-012-delivery-management-gitops-promotion.md
```

Implementation repository:

```text
platform-devops-developer-portal
branch: feat/delivery-mvp-slice
known implemented UX v2 baseline: 0163a49 (verify before use; do not assume if branch moved)
```

Important known fact from the implementation evidence:

- the previously committed visual asset `docs/delivery/assets/deployments-screen-v2.webp` was found truncated/corrupted;
- therefore **the textual contract in this prompt is authoritative for this checkpoint**;
- do not use the corrupted asset as a basis for layout decisions.

Verify the live branch/SHA before edits and record the actual baseline.

---

## 1. Scope of this checkpoint

This checkpoint is only a **responsive/layout polish** of the already-working Deployments UX v2.

Authorized:

- layout restructuring;
- responsive breakpoints;
- spacing/alignment polish;
- minor typography hierarchy using existing Backstage/MUI primitives;
- status-chip/callout alignment using components already available in the codebase;
- compacting sections that currently waste space;
- improving truthful rendering of unavailable values;
- preserving all existing actions and data behavior;
- focused UI tests for the deterministic responsive contract.

Not authorized:

- new architecture;
- new domain concepts;
- new workflow engine;
- new global Delivery workbench;
- P1 hardening;
- production-adoption review;
- new provider integrations;
- new data fields merely to make the screen prettier;
- changes to GMUD semantics;
- changes to ADR-012 boundaries;
- changing the global Backstage shell/header/navigation;
- redesigning unrelated tabs;
- inventing new charts, dashboards, timelines, or workflow visuals.

---

## 2. Absolute no-creativity rule

The following are **forbidden** unless this prompt explicitly says otherwise:

- changing the order of the sections;
- changing labels/copy;
- replacing cards with a different visualization;
- introducing charts;
- introducing stepper/pipeline components;
- introducing a new visual language;
- adding decorative icons merely for aesthetics;
- adding gradients/custom brand surfaces inside the content area;
- adding a side drawer;
- moving GMUD into a modal;
- turning History or Events into new page routes;
- inventing fields not backed by current data;
- forcing desktop multi-column layouts into laptop-sized screens;
- replacing Backstage/MUI typography/fonts with custom fonts.

Do not "improve" the design beyond this contract.

---

# 3. Exact information architecture

The page body inside the existing Component → Deployments tab must render in **exactly this vertical order**:

```text
1. Page heading / freshness row
2. Release candidate panel
3. Status por ambiente section
4. Contexto de mudança (GMUD) section
5. Histórico de promoções + Eventos recentes
```

No section may move above/below another.

The existing Backstage component masthead and tabs remain untouched.

---

# 4. Exact desktop / wide-screen layout

## Breakpoint rule

Use the existing MUI breakpoint system already present in the project.

The **wide reference layout is allowed only at `xl` and above**.

Assume the standard MUI v4 breakpoint unless the repository theme overrides it:

```text
xl ≈ 1920px viewport width
```

Do not reduce this breakpoint just to keep more columns visible.

At `xl` and above:

```text
[ Deployments heading ---------------------------------------------- ]
[ Release candidate ------------------------------------------------ ]
[ DEV ][ HML ][ PRD ]               [ Contexto de mudança (GMUD)    ]
[ Histórico de promoções --------- ][ Eventos recentes ------------ ]
```

Precise proportions:

- main row (environment area + GMUD): approximately 70/30;
- environment area: three equal-width cards;
- history/events row: approximately 65/35 or 2/3 + 1/3;
- all panels align to the same outer content width;
- no panel should visually float outside the content grid.

At `xl`, the GMUD section is a right-hand panel aligned with the top of the environment section.

---

# 5. Exact laptop / MacBook 13-inch responsive layout

This is a **hard requirement**, not a suggestion.

For any viewport below `xl` (including typical MacBook 13-inch browser widths), the screen must NOT compress the wide desktop composition.

Use exactly this stacking behavior:

```text
[ Deployments heading ---------------------------------------------- ]
[ Release candidate ------------------------------------------------ ]
[ DEV -------------------------------------------------------------- ]
[ HML -------------------------------------------------------------- ]
[ PRD -------------------------------------------------------------- ]
[ Contexto de mudança (GMUD) -------------------------------------- ]
[ Histórico de promoções ------------------------------------------ ]
[ Eventos recentes ------------------------------------------------ ]
```

Requirements below `xl`:

- DEV/HML/PRD are **one card per row**, never three squeezed columns;
- GMUD is full width below PRD;
- History is full width;
- Events is full width below History;
- no horizontal squeeze of the main cards;
- no 70/30 split below `xl`;
- no two-column History/Events below `xl`;
- content must remain readable without zooming out;
- only tables may use horizontal scrolling when necessary;
- the page itself must not create horizontal overflow.

This behavior is specifically intended for the user's 13-inch MacBook experience.

Do not invent a tablet-specific alternate composition. The rule is simple:

```text
>= xl: wide layout
<  xl: stacked layout
```

---

# 6. Page heading / freshness row — exact contract

Left side:

```text
Deployments
Acompanhe o status de implantação, promova versões entre ambientes e visualize o contexto de mudança (GMUD).
```

Right side:

- outlined/secondary `Atualizar` button;
- last-updated timestamp when available;
- compact freshness indicator `Dados atualizados` when appropriate.

Rules:

- `Atualizar` is never the dominant CTA;
- do not add extra header actions;
- below `xl`, the right-side freshness controls may wrap below the title, but must remain right-aligned when space allows;
- do not hide refresh on laptop.

---

# 7. Release candidate panel — exact contract

Panel title:

```text
Release candidate
```

Subtitle:

```text
Selecione uma versão para visualizar o status nos ambientes ou solicitar uma promoção.
```

Information order is fixed:

```text
[ Release selector + Atual badge ]
[ Commit ]
[ Branch ]
[ Publicado em ]
[ Imagem (digest) ]
[ Solicitar promoção ]
[ Vincular GMUD ]
[ overflow menu ]
```

At `xl`:

- keep this as one horizontal panel;
- selector first and visually largest metadata control;
- metadata fields remain compact;
- actions remain on the right.

Below `xl`:

- the panel remains full width;
- allow the metadata/action region to wrap into additional rows naturally;
- preserve the exact field order above;
- buttons must remain visible and must not shrink into icon-only variants;
- selector should remain the first item and preferably occupy a full row when width is constrained.

Truthfulness rules:

- selector remains a real select over available ReleaseCandidates;
- do not restore the free-text ReleaseCandidate ID field;
- `Commit`, `Branch`, or other unavailable values must display `não disponível` unless the backing data exists;
- do not fabricate commit/branch values;
- do not replace unavailable values with fake examples;
- do not add a data migration just for this polish checkpoint.

Button labels are fixed:

```text
Solicitar promoção
Vincular GMUD
```

Do not rename them.

---

# 8. Status por ambiente — exact contract

Heading:

```text
Status por ambiente
```

Subtitle:

```text
Visualize o status da versão selecionada em cada ambiente.
```

Environment order is fixed:

```text
DEV
HML
PRD
```

Do not sort dynamically.

## 8.1 Environment card structure

Every environment card must preserve the exact anatomy and order:

```text
Environment name
Semantic status badge
Short state description
Timestamp row
Argo Sync
Argo Health
Namespace
Optional contextual callout
Actions
```

### Row labels

Use only the following existing labels as applicable:

```text
Última implantação
Última tentativa
Argo Sync
Argo Health
Namespace
```

### Actions

Use:

```text
Ver detalhes
Abrir no Argo
```

`Abrir no Argo` renders only when a truthful provider URL exists.

## 8.2 Status treatment

Keep semantic color usage restrained and Backstage/MUI-native:

- success: green;
- failure: red;
- waiting/informational: blue/neutral;
- unknown: neutral.

Do not create custom gradients or oversized iconography.

Use existing Backstage status primitives / Material icons already in the repository when possible.

Do not introduce a new icon library.

## 8.3 Failure callout

For a failed environment, keep one concise callout inside the card.

Structure:

```text
Erro na promoção
<human-readable short reason>
```

Do not show raw multiline provider logs in the card.

## 8.4 PRD governance-related callout

When PRD requires governance, use the existing truthful state, e.g.:

```text
Requer mudança (GMUD)
Vincule uma GMUD e aguarde aprovação para solicitar a promoção.
```

Do not show unavailable Argo fields as failures.

---

# 9. Environment connector rule

At `xl` only, the subtle DEV → HML → PRD connector may remain if already implemented.

Below `xl`:

- do not render horizontal connectors between stacked cards;
- do not add vertical pipeline arrows;
- do not add stepper UI.

The stacked cards themselves communicate sequence by their fixed order.

---

# 10. Contexto de mudança (GMUD) — exact contract

Heading:

```text
Contexto de mudança (GMUD)
```

When bound, show the existing `Vinculada` badge.

Metadata field order is fixed:

```text
Change
Tipo
Status
Atividade
Solicitação
Janela de execução
Aprovador
```

Rules:

- Change ID is a link when the route exists;
- do not turn the metadata into a dense HTML table;
- use a simple MUI grid/list layout;
- at `xl`, two compact metadata columns are acceptable inside the panel;
- below `xl`, stack the metadata to avoid squeezed labels/values;
- preserve labels exactly;
- do not add new GMUD fields.

Unavailable approver:

```text
não disponível
```

Do not substitute requester/binder as approver.

## 10.1 Eligibility block

Place the eligibility callout **below the metadata**.

It must remain visually distinct from deployment health.

Examples already supported by backend truth:

```text
Elegível para produção
Não elegível para produção
Aguardando autorização
```

For OUTSIDE_WINDOW, preserve the factual reason:

```text
Não elegível para produção
Fora da janela de execução aprovada.
```

Do not infer new eligibility states.

## 10.2 Multi-activity message

When applicable, render exactly:

```text
A implantação bem-sucedida é evidência para esta atividade; não conclui a mudança.
```

Do not rewrite this sentence.

This message is informational and should be visually separate from the eligibility callout.

---

# 11. Histórico de promoções — exact contract

Heading:

```text
Histórico de promoções
```

Subtitle:

```text
Últimas promoções realizadas para este componente.
```

Use the existing compact Backstage table.

Column order is fixed to the currently-backed dataset:

```text
DATA
VERSÃO
AMBIENTE
STATUS
SOLICITADO POR
MUDANÇA
AÇÕES
```

Do **not** add `Duração` unless the current backend already exposes a truthful duration field at the start of this checkpoint. Do not derive or fabricate it for visual parity.

Rules:

- default to 5 rows;
- `Ver todos` remains available when more rows exist;
- long release IDs and actor refs remain truncated with tooltip/title;
- cells remain single-line where practical;
- horizontal scrolling is allowed for this table below `xl`;
- do not reduce font size to unreadable values merely to avoid scrolling;
- do not convert the table into cards.

---

# 12. Eventos recentes — exact contract

Heading:

```text
Eventos recentes
```

Subtitle:

```text
Eventos relacionados à versão selecionada.
```

Keep the existing compact event list/timeline semantics.

Rules:

- default to 5 events;
- `Ver todos` when more exist;
- newest first;
- use existing semantic status indicators;
- do not add a vertical timeline package/library;
- do not add a new event store;
- use existing Delivery-derived events only.

At `xl`, this panel is beside History.

Below `xl`, this panel must be full width below History.

---

# 13. Exact responsive spacing rules

Use the repository's existing theme spacing; do not introduce pixel-heavy custom CSS unless required.

Required behavior:

- outer page spacing consistent with other Component tabs;
- vertical gap between major sections visually consistent;
- cards/panels should not touch;
- no giant dead whitespace;
- do not reduce spacing so aggressively that cards look cramped;
- metadata rows should remain readable at 100% browser zoom on a 13-inch MacBook.

Do not change global font family.

Do not introduce custom font files.

Do not rely on icons unavailable in the current dependencies.

---

# 14. Deterministic component constraints

Prefer and reuse the components already present in the current implementation:

```text
DeploymentsHeader
ReleaseCandidatePanel
EnvironmentStatusSection
EnvironmentCard
EnvironmentConnector
GovernancePanel
PromotionHistoryTable
RecentEventsList
StatusBadge
```

Do not replace this component model with a new UI framework.

Use existing:

- `@backstage/core-components`;
- `@material-ui/core` / the MUI version already in the repository;
- the existing Material icon dependencies already used by the application.

Do not add a new design-system dependency.

---

# 15. Functional behavior must remain unchanged

This checkpoint is not allowed to change business behavior.

Must continue to work exactly as before:

- release selection;
- component-scoped ReleaseCandidate listing;
- selected RC scopes the whole screen;
- refresh/polling;
- promotion request action;
- Change binding;
- eligibility reads;
- history reads;
- recent events reads;
- provider details;
- stale-response protection;
- Platform tab and all other Component tabs.

Do not change backend APIs unless a concrete regression proves that the current responsive polish cannot work without a tiny correction. A visual preference is not sufficient justification for a backend change.

---

# 16. Browser validation matrix — mandatory

You must navigate the real running Backstage UI.

Do not declare completion from unit tests alone.

Validate at minimum these viewport classes:

## A. Wide reference viewport

Use a viewport >= 1920px wide.

Expected:

```text
Release: full width
Environment area: DEV/HML/PRD 3 columns
GMUD: right column beside environments
History + Events: 2 columns
```

## B. MacBook 13-inch / laptop viewport

Use a realistic laptop viewport below 1920px, preferably close to the user's actual browser width from the supplied screenshots.

Expected:

```text
Release: full width
DEV: full width
HML: full width
PRD: full width
GMUD: full width
History: full width
Events: full width
```

There must be no main-page horizontal overflow.

The History table may scroll horizontally inside its own container.

## C. Narrower sanity viewport

Use one additional narrower viewport to ensure the stacked composition degrades safely.

This is not a mobile-redesign task. Only verify that the existing stacked layout does not break.

---

# 17. Screenshot comparison — mandatory

Capture at minimum:

1. wide `xl` screenshot;
2. MacBook/laptop screenshot below `xl`;
3. if possible, one screenshot of GMUD + History/Events in the laptop composition.

Review the screenshots against this contract before declaring PASS.

Do not merely write "looks good".

For each screenshot, explicitly verify:

- section order;
- breakpoint behavior;
- environment card stacking/columns;
- GMUD placement;
- History/Events placement;
- action labels;
- no invented fields;
- no clipped text;
- no page-level horizontal overflow.

If any item differs, fix it before completion.

---

# 18. Tests required

At minimum:

- existing Deployments frontend tests remain green;
- existing tab-loader smoke guard remains green;
- existing Delivery backend tests remain green if backend files are untouched or affected;
- add focused layout/render tests only if they can assert breakpoint structure without brittle pixel snapshots;
- do not introduce broad screenshot-test infrastructure solely for this checkpoint.

Record exact commands and results.

---

# 19. PASS / CONDITIONAL_PASS / FAIL

## PASS

Only if:

- wide layout matches the exact specified composition;
- below `xl`, DEV/HML/PRD are stacked one-per-row;
- below `xl`, GMUD is below PRD;
- below `xl`, History and Events are stacked;
- Release panel preserves exact field/action order;
- no visible labels were creatively renamed;
- no new visual paradigm was introduced;
- no fake data was added;
- no page-level horizontal overflow on the laptop viewport;
- all existing functional behavior remains intact;
- browser validation and screenshots are completed;
- required regressions are green.

## CONDITIONAL_PASS

Only for a narrow, documented limitation caused by an existing Backstage/MUI capability or existing missing data that does not materially change the requested layout.

Do not use CONDITIONAL_PASS to hide visual deviations that could have been implemented.

## FAIL

If any of these occur:

- the agent redesigns the page;
- 3 environment cards remain squeezed below `xl`;
- GMUD remains a narrow side panel below `xl`;
- History/Events remain forced into two columns below `xl`;
- a new icon/font/design library is introduced;
- labels/copy are materially changed without authorization;
- fake Commit/Branch/Aprovador values are introduced;
- accepted Delivery/GMUD semantics regress;
- the final browser output is materially different from this contract.

---

# 20. Documentation update

Create/update a factual result document under:

```text
docs/delivery/deployments-ux-v2-responsive-polish.md
```

Record:

- docs baseline SHA;
- implementation before/after SHA;
- exact files changed;
- exact breakpoint implementation;
- wide screenshot result;
- laptop screenshot result;
- tests run;
- deviations, if any;
- final verdict.

Do not rewrite historical evidence documents.

---

# 21. Final report format

Use exactly:

```markdown
# Deployments UX v2 Responsive Polish Result

## 1. Baseline
## 2. Verdict
## 3. Layout Contract Compliance
## 4. Wide-Screen Verification
## 5. MacBook/Laptop Verification
## 6. Release Candidate Panel
## 7. Environment Cards
## 8. GMUD Panel
## 9. History / Events
## 10. Functional Regression
## 11. Tests
## 12. Screenshots
## 13. Files Changed
## 14. Deviations
## 15. Documentation Updated
## 16. STOP
```

End with:

```text
STOP
```

Do not proceed into production-adoption review, P1 residual hardening, global Delivery workbench, or any unrelated UI work.
