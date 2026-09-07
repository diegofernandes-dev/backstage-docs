# Deployments UX v2 Responsive Polish Result

## 1. Baseline

- Docs baseline SHA (`diegofernandes-dev/backstage-docs`, `main`): `1b181ef8175bfc7991f4b3a68147f757cd86455c` (`docs(prompts): set deterministic Deployments UX responsive polish as current`).
- Authoritative contract: `prompts/deployments-ux-v2-responsive-polish.md` (textual contract; corrupted `docs/delivery/assets/deployments-screen-v2.webp` was not used for layout decisions). Approved visual model also compared against the user-supplied Deployments v2 reference image during follow-up review.
- Implementation repo (`platform-devops-developer-portal`), branch `feat/delivery-mvp-slice`:
  - before SHA: `0163a4981db9e0f6217207ecfcc226ea82a5832d` (`feat(delivery): implement Deployments UX v2`);
  - after SHA: `b08e7b284f34e5d12c299b4ee6ab75697e21010f` (`fix(delivery): polish Deployments UX v2 responsive composition`).
- Theme breakpoints: no app override; MUI v4 defaults (`md=960`, `lg=1280`, `xl=1920`).

## 2. Verdict

**PASS** (with documented follow-up refinements after live MacBook review).

The approved model composition is reachable on typical laptop widths: DEV|HML|PRD with connectors, GMUD in a right column (~70/30), History|Events side-by-side (~65/35). Entity masthead densified so tab content reaches the fold. Card MetaRows no longer wrap mid-label or stretch full-bleed. Existing labels, actions, data truthfulness, and business behavior remain intact.

## 3. Layout Contract Compliance

| Requirement | Final implementation |
|---|---|
| Section order Header → RC → Status → GMUD → History → Events | PASS |
| Model composition: DEV\|HML\|PRD + connectors; GMUD right (~70/30) | PASS from **`lg` (~1280)** (raised from prompt's `xl`-only after MacBook review — `xl`-only forced unusable full-bleed stacks on real laptops and diverged from the approved visual model) |
| History\|Events side-by-side | PASS from **`lg`** |
| DEV/HML/PRD one card per row + no connectors | Below **`md` (~960)** only |
| No page-level horizontal overflow on laptop | PASS |
| No new icons/fonts/libraries/domain concepts/backend fields | PASS |

Exact breakpoint implementation:

- Outer composition: [`DeploymentsTab.tsx`](../../platform-devops-developer-portal/packages/app/src/modules/catalogEntityTabs/DeploymentsTab.tsx) `Grid` items `xs={12}` / `lg={8|4}`.
- Environment pipeline: `useMediaQuery(theme.breakpoints.up('md'))` in `EnvironmentStatusSection` — row + solid/dashed connectors from `md`; column stack below `md`.
- Card internals: dense `MetaRow` (label does not wrap mid-word; long provider IDs ellipsize + tooltip); reserved callout band so actions align across DEV/HML/PRD.
- Catalog entity masthead: global densify via `CompactEntityHeaderStyles` (`AppRootElementBlueprint`) — reduced `BackstageHeader` padding/title size without replacing EntityHeader behavior.
- Tab header: `Deployments` as `h5` + body2 description; Atualizar remains secondary outlined.

## 4. Wide-Screen Verification

Viewport: `1920×1200` and `1512×1000` via CDP.

At `1512` (MacBook-class):

- DEV/HML/PRD same row with 2 connectors; GMUD right of environments; History|Events same row.
- Overflow: `scrollWidth === clientWidth`.

Screenshots (checkpoint capture set):

- `docs/delivery/assets/deployments-ux-v2-responsive-wide-xl.png`
- `docs/delivery/assets/deployments-ux-v2-responsive-wide-environments.png`

## 5. MacBook/Laptop Verification

After follow-up (composition at `lg`):

- `1440` / `1512`: model layout (env+GMUD, history+events) side-by-side.
- Below `lg`: GMUD/History/Events stack full-width; env cards remain inline until below `md`.
- Below `md`: env cards stack; connectors hidden.

Earlier checkpoint captures (pre-`lg` follow-up, stacked laptop composition) retained for history:

- `docs/delivery/assets/deployments-ux-v2-responsive-laptop-1440.png`
- `docs/delivery/assets/deployments-ux-v2-responsive-laptop-gmud-history.png`

## 6. Release Candidate Panel

- Field order (aligned to approved visual model): Commit → Publicado em → Imagem (digest) → Branch.
- Commit/Branch remain truthful `não disponível` (no fabricated provenance; domain model still lacks those fields).
- Actions unchanged: `Solicitar promoção`, `Vincular GMUD`, overflow menu.

## 7. Environment Cards

- Anatomy, labels, actions, failure/GMUD callouts unchanged in meaning.
- Connectors: solid success-colored when prior stage succeeded; dashed muted otherwise (theme-aware).
- Dense MetaRows + truncated detail IDs with tooltip.

## 8. GMUD Panel

- Title `Contexto de mudança (GMUD)`; Change ID as InfoCard `subheader` link when bound; `Vinculada` chip.
- Fields: Tipo → Status → Atividade → Solicitação (`deploymentRequestId`) → Janela → Aprovador (`não disponível`).
- Eligibility callout + E1 evidence sentence unchanged.

## 9. History / Events

- Columns, 5-row default, `Ver todos`, truncation, table horizontal scroll unchanged.
- Side-by-side from `lg`; stacked below `lg`.

## 10. Functional Regression

Live authenticated session against running app (`:3000`) / backend (`:7007`) on `idp-showcase-api`:

- Release selector, env cards, GMUD eligibility, history, events, refresh affordances present.
- No backend API or ADR-012 boundary changes in this checkpoint.

## 11. Tests

```text
cd packages/app && CI=true yarn test \
  src/modules/catalogEntityTabs/deployments/components/EnvironmentStatusSection.test.tsx \
  src/modules/catalogEntityTabs/DeploymentsTab.test.tsx \
  src/modules/catalogEntityTabs/index.test.ts \
  src/modules/catalogEntityTabs/deployments/model/semanticStatus.test.ts \
  --no-coverage --watchAll=false
→ suites green (incl. new EnvironmentStatusSection responsive tests)

cd packages/backend && CI=true yarn test \
  src/modules/delivery/DeliveryService.test.ts \
  --no-coverage --watchAll=false
→ 26 passed
```

## 12. Screenshots

| Asset | Notes |
|---|---|
| `assets/deployments-ux-v2-responsive-wide-xl.png` | Wide reference |
| `assets/deployments-ux-v2-responsive-wide-environments.png` | Env row |
| `assets/deployments-ux-v2-responsive-laptop-1440.png` | Early stacked laptop capture |
| `assets/deployments-ux-v2-responsive-laptop-gmud-history.png` | Early GMUD+History stack capture |

## 13. Files Changed

Implementation (`platform-devops-developer-portal`):

- `packages/app/src/modules/catalogEntityTabs/DeploymentsTab.tsx` (`lg` Grid split)
- `packages/app/src/modules/catalogEntityTabs/index.ts` (`AppRootElementBlueprint` compact header styles)
- `packages/app/src/modules/catalogEntityTabs/CompactEntityHeaderStyles.tsx` (new)
- `packages/app/src/modules/catalogEntityTabs/deployments/components/DeploymentsHeader.tsx`
- `packages/app/src/modules/catalogEntityTabs/deployments/components/EnvironmentStatusSection.tsx`
- `packages/app/src/modules/catalogEntityTabs/deployments/components/EnvironmentStatusSection.test.tsx` (new)
- `packages/app/src/modules/catalogEntityTabs/deployments/components/EnvironmentCard.tsx`
- `packages/app/src/modules/catalogEntityTabs/deployments/components/EnvironmentConnector.tsx`
- `packages/app/src/modules/catalogEntityTabs/deployments/components/GovernancePanel.tsx`

Docs (`backstage-docs`):

- `docs/delivery/deployments-ux-v2-responsive-polish.md` (this file)
- `docs/delivery/assets/deployments-ux-v2-responsive-*.png`
- `docs/delivery/README.md` status entry

## 14. Deviations

- Prompt originally required wide layout only at `xl`. Live MacBook review + approved visual model required bringing the 70/30 and History/Events split down to **`lg`**, otherwise the page remained far from the model on real laptop widths. Env card stacking retained only below `md`.
- Pre-existing truthful data gaps retained: Commit/Branch/Aprovador = `não disponível`.

## 15. Documentation Updated

This file + Delivery README status line. Historical evidence documents not rewritten.

## 16. STOP

Per the execution contract: no production-adoption review, no P1 residual hardening, no global Delivery workbench, and no unrelated UI work follows from this checkpoint.
