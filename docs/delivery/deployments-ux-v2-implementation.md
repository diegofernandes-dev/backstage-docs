# Deployments UX v2 Result

## 1. Baseline

- Docs baseline SHA (`diegofernandes-dev/backstage-docs`, `main`): `c1f32b32182275310bcca39dffb92c8f0bc6302e` (`docs(delivery): add approved Deployments UX v2 reference and prompt`).
- Implementation repo (`platform-devops-developer-portal`), branch `feat/delivery-mvp-slice`: before SHA `a48dedc4e67c9d2874284f9a53fe580e0a1e6488` (`fix(delivery): scope backend Kubernetes identity off ambient cluster-admin`), after SHA `0163a49` (`feat(delivery): implement Deployments UX v2`).
- **Corrupted normative asset, discovered and worked around**: `docs/delivery/assets/deployments-screen-v2.webp` at this SHA is truncated (RIFF header declares 37,138 bytes; the actual git blob and working-tree file are both 15,009 bytes). Confirmed independently with `dwebp`, ImageMagick, and macOS `sips` — all three fail with "not enough data" / "insufficient image data." The user supplied the correct reference image directly in the working conversation; it was inspected and found to closely match the textual spec in `deployments-ux-v2.md`. The docs repo asset itself should be re-exported and re-committed by its owner; that is outside this checkpoint's authorization.

## 2. Verdict

**CONDITIONAL_PASS.**

The approved v2 composition is implemented and live-validated against the real D1 sandbox (Kargo/Argo/Kubernetes) end to end, including a genuine `OUTSIDE_WINDOW` denial and the E1 non-completion message on a real succeeded PRD deployment. The ambiguous free-text release field is gone. No ADR-012 boundary was touched. Held to `CONDITIONAL_PASS` rather than `PASS` by a small number of narrow, nonblocking, truthfully-labeled data gaps (Commit/Branch/Aprovador fields have no backing data in the current domain model and are rendered as "não disponível" rather than fabricated — see §15) — none of the `FAIL` conditions apply (no fake selector, no fabricated data, no conflated Change/Delivery semantics, no unauthorized-dispatch implication, no architecture redesign).

## 3. Approved Reference Compliance

Implemented composition, top to bottom, matches the reference the user supplied and `docs/delivery/deployments-ux-v2.md`:
1. Heading row: "Deployments" + description (left); "Atualizar" + last-updated timestamp + "Dados atualizados" freshness indicator (right) — Atualizar is a secondary outlined button, not the primary CTA.
2. Full-width release candidate panel: selector with "Atual" badge, Commit / Publicado em / Imagem (digest) / Branch fields, primary "Solicitar promoção", secondary "Vincular GMUD", overflow menu.
3. ~70/30 two-column area: DEV → HML → PRD cards with a connector (solid when the prior stage succeeded, muted otherwise) on the left; "Contexto de mudança (GMUD)" governance panel on the right.
4. Footer two-column area: "Histórico de promoções" table (wider, left) and "Eventos recentes" list (right), both with a 5-row default and "Ver todos" toggle.

One implementation adaptation beyond the reference's literal desktop breakpoint, made during live validation on the user's own laptop: the governance/events panels stack full-width below MUI's `xl` (1920px) breakpoint rather than `md`/`lg`, because real laptop logical widths (1440–1728px) were still cramming the right column uncomfortably at the tighter breakpoints originally chosen. This is exactly the "small implementation adaptation... for responsiveness" the contract permits (§7); the 70/30 split still applies faithfully on genuinely wide external monitors.

## 4. UI Changes Implemented

`DeploymentsTab.tsx` (`packages/app/src/modules/catalogEntityTabs/`) is now a thin composition root; all UI moved into a new `deployments/` subfolder:
- `components/`: `DeploymentsHeader`, `ReleaseCandidatePanel`, `EnvironmentStatusSection`, `EnvironmentCard`, `EnvironmentConnector`, `GovernancePanel`, `PromotionHistoryTable`, `RecentEventsList`, `StatusBadge`.
- `model/`: `types.ts` (frontend mirror of backend types + `PromotionHistoryEntry`/`DeliveryEvent` + the fixed E1 copy constant), `semanticStatus.ts` (pure mapping function).
- `api/`: `DeliveryApi.ts` (api ref + `ApiBlueprint` extension), `DeliveryClient.ts`, `deliveryErrors.ts`.
- `hooks/`: `useReleaseCandidates`, `useDeploymentsForRc`, `usePromotionHistory`, `useRecentEvents`, `useEligibility`, `usePolling`, `useShowAll` — all built on `react-use`'s `useAsyncFn` (not plain `useAsync`) specifically so "Atualizar" and post-action refreshes can imperatively retrigger the same scoped reads.

Adopted `@backstage/core-components` (`InfoCard`, `Progress`, `WarningPanel`, `EmptyState`, `Table`, `Status*`, `Link`) throughout, matching the idiomatic pattern already used by the GMUD plugin — the MVP tab previously used only raw `@material-ui/core` primitives.

## 5. Data / API Changes

All inside `packages/backend/src/modules/delivery/`, **no new migrations**:
- **Added**: `listReleaseCandidatesForComponent` (repository + service `listReleaseCandidates`) → `GET /components/:namespace/:kind/:name/release-candidates`.
- **Added**: `DeliveryService.getPromotionHistory` (composes the existing `listRequestsForComponent`, never previously exposed past the "latest per target" collapse) → `GET .../promotion-history`.
- **Added**: `DeliveryService.listRecentEvents` — **synthesized**, not a new audit table. Derives up to 4 milestone events per request (`requested`, `change_bound`, `dispatched`, `succeeded`/`failed`) from existing timestamped fields, because `delivery_projections` is upsert-only and a durable event-sourcing table would be exactly what the contract forbids for a benefit (intermediate provider-flap history) the approved screen doesn't require → `GET .../events?limit=`.
- **Added**: `DeliveryService.getDeploymentsForReleaseCandidate`, wired as an optional `?releaseCandidateId=` query param on the **existing** `GET .../deployments` route (backward-compatible; absent param keeps today's "latest per target" behavior as the pre-selection default).
- **Skipped**: standalone `GET` for a `ChangeBinding` — already nested in the RC-scoped view; no consumer needs it independently.
- **Skipped**: backend-side semantic status mapping — implemented as a frontend pure function (`semanticStatus.ts`) instead, since it's pure UI vocabulary over fields the backend already returns unmodified.
- All four new/extended reads reuse the existing `deliveryDeploymentReadPermission` — no new permission.
- **One additive change outside Delivery**: widened the export barrel in `plugins/change-management/src/index.ts` (added `ChangeStatus`, `ExecutionActivity`, `ExecutionPlan`, `NormalizedRequestedWindow`, `CLASSIFICATION_LABELS`, `RISK_LABELS`, `STATUS_LABELS`) so the Deployments tab can type its cross-plugin `getChange()` call and reuse the existing Portuguese labels instead of inventing new copy. Purely additive re-exports of already-defined types/constants — no logic change, no GMUD redesign.

## 6. Release Candidate UX

Free-text `Release candidate ID` field fully removed. Replaced with a `Select` populated by `listReleaseCandidates`, defaulting to the newest (index 0) with an "Atual" badge. Selecting a different RC re-scopes the entire screen via `useDeploymentsForRc`. **Commit** and **Branch** are rendered as "não disponível" — the `ReleaseCandidate` domain type (`artifactRepoUrl`, `artifactDigest`, `freightName`, `createdAt`) genuinely has no commit-SHA or branch field; this is a real, load-bearing product/data gap, not an oversight, and is called out again in §15 rather than papered over with fabricated values. "Imagem (digest)" shows a shortened digest, linked when `artifactRepoUrl` is an `http(s)` URL.

## 7. Environment Status UX

`toSemanticStatus()` maps backend truth to the fixed vocabulary (`Sucesso, Em andamento, Falhou, Aguardando, Mudança necessária, Aguardando autorização, Elegível, Conflito, Desconhecido`) — unit-tested one case per row. Each `EnvironmentCard` shows the badge, timestamp, raw Argo Sync/Health (kept as their own labeled fields, distinct from the semantic badge, per the reference), namespace, a contextual callout for `change_required`/`awaiting_authorization`(pending)/`failed`, an inline "Ver detalhes" expand (no new route), and "Abrir no Argo" only when a `promotionUrl` is actually present. `EnvironmentConnector` renders solid only when the prior stage's request succeeded. Cards use `InfoCard`'s built-in `variant="gridItem"` so all three render at equal height regardless of callout content length — found necessary during live validation (see §13).

## 8. Governance / Eligibility UX

The RC-scoped `DeploymentView.binding`/`.request` for the PRD target feeds the governance panel directly (no new endpoint). `useEligibility` calls the existing, untouched, fail-closed `GET /deployment-requests/:id/eligibility` only when a binding exists; ALLOW renders the green "Elegível para produção" callout, DENY renders a reason-specific message from a fixed lookup table keyed on the backend's own `ExecutionEligibilityReason` values (no invented vocabulary). The E1 sentence ("A implantação bem-sucedida é evidência para esta atividade; não conclui a mudança.") renders whenever a binding exists and the PRD request is `succeeded` — verified live. **Aprovador** is rendered as "não disponível": the authorization ledger's `actorRef` is never exposed through any Change Management HTTP route reachable from Delivery or this tab; exposing it would require a new read endpoint in a different module, out of this checkpoint's scope.

## 9. History / Recent Events

`PromotionHistoryTable` uses `@backstage/core-components` `Table` with the exact required columns (`Data | Versão | Ambiente | Status | Solicitado por | Mudança | Ações`), a client-side 5-row default with "Ver todos" (no server pagination), and — after live feedback — truncates long freight-name hashes and full user entity refs with a hover tooltip, forces single-line cells, and wraps the table in a horizontally-scrolling container so it degrades gracefully rather than word-wrapping into oversized rows. `RecentEventsList` is a plain compact list (not a table) over the synthesized milestone events, same 5+"Ver todos" pattern via a shared `useShowAll` hook.

## 10. Browser Validation

Performed against the real running dev app (`http://localhost:3000`, backend `:7007`) and the real local D1 sandbox (Kargo/Argo/Kubernetes via `kargo-promoter.kubeconfig`, not the fake in-memory repository) on `idp-showcase-api`. Microsoft Entra ID interactive/MFA sign-in cannot and should not be automated headlessly, so validation was a hybrid: headless Playwright confirmed the unauthenticated shell (sign-in page, no console errors) after each restart, and the authenticated screen states were driven by the user in their own signed-in browser, with screenshots reviewed interactively in the working conversation rather than saved as files in this repo.

This process **found and fixed a real regression introduced during this checkpoint**, and two real responsive-layout defects — see §13. Confirmed working: release selection, environment cards, governance/eligibility panel, history table, events list, the Platform tab, and navigation back and forth — all with no console/network errors beyond the expected pre-auth 401.

## 11. State Evidence Matrix

| # | State | Evidence |
|---|---|---|
| 1 | Selected release shown | **LIVE** — real digest/timestamp/selector against the sandbox RC |
| 2 | DEV success | **FIXTURE/TEST** — `DeploymentsTab.test.tsx` (mocked DEV `succeeded`); not present in the single sandbox RC available at validation time |
| 3 | HML/provider failure | **LIVE** — real `Falhou` badge + callout, Argo Sync `Synced`/Health `Healthy` shown as separate truthful fields |
| 4 | PRD Change required/unbound | **FIXTURE/TEST** — `DeploymentsTab.test.tsx` |
| 5 | PRD Change bound but not eligible | **LIVE** — real `DENY/OUTSIDE_WINDOW` observed and rendered correctly |
| 6 | PRD ALLOW/eligible | **FIXTURE/TEST** — `DeploymentsTab.test.tsx` |
| 7 | PRD success with activity non-completion message | **LIVE** — real `Sucesso` + E1 sentence on a bound PRD request |
| 8 | Same-target busy/conflict | **FIXTURE/TEST, backend only** — `DeliveryService.test.ts` E1.3 concurrency suite proves the guarantee; the UI's `Conflito` mapping was not exercised end-to-end against a real concurrent dispatch |
| 9 | Stale/refresh selection correctness | **FIXTURE/TEST** — dedicated `useDeploymentsForRc.test.tsx` race test, plus the backend `getDeploymentsForReleaseCandidate` RC-scoping regression test |

## 12. Tests

- Backend: extended `packages/backend/src/modules/delivery/DeliveryService.test.ts` with 4 new `describe` blocks (7 new tests) — `listReleaseCandidates`, `getDeploymentsForReleaseCandidate` (×2, including the direct anti-stale-regression test), `getPromotionHistory`, `listRecentEvents` (×3). Full backend suite: **184 passed / 189 total** (5 pre-existing, unrelated skips).
- Frontend: new `semanticStatus.test.ts` (10 tests), `useDeploymentsForRc.test.tsx` (1 race-safety test), `DeploymentsTab.test.tsx` (4 RTL tests covering the eligibility states, E1 message, and history toggle). Full monorepo suite via `yarn test --all`: **263 passed / 268 total** (5 pre-existing, unrelated skips), 3 Jest projects, `packages/app/src/modules/catalogEntityTabs/index.test.ts` (tab-loader smoke guard) unchanged and green.
- `yarn tsc --noEmit` across the whole repo: exactly the same **9 pre-existing errors** (confirmed via `git stash` diff) before and after this checkpoint's changes — zero new type errors introduced.

## 13. Regressions / Defects Found

1. **Self-introduced, found and fixed during this checkpoint**: the Delivery API extension was first registered inside `catalogEntityTabsModule` (`pluginId: 'catalog'`) without an explicit `name`. That collided with the real `@backstage/plugin-catalog`'s own unnamed `catalogApiRef` extension in the same namespace, silently knocking out `catalogApiRef` **app-wide** (broke the GMUD list page and the catalog list provider, confirmed live by the user). First fix attempt (moving the extension into a new `pluginId: 'delivery'` module) was itself wrong — `createFrontendModule` extends an *existing* registered plugin, and no plugin with id `'delivery'` exists, so that module attached to nothing. Correct fix, verified against `@backstage/plugin-catalog`'s actual source: kept the extension inside `catalogEntityTabsModule` but gave it an explicit `name: 'delivery'`, producing a distinct extension id (`api:catalog/delivery`) instead of colliding with the built-in unnamed one. Confirmed fixed live after a full dev-server restart.
2. **Found live, fixed**: the three environment cards rendered at uneven heights (a card with a failure callout was taller than one without). Fixed with `InfoCard`'s built-in `variant="gridItem"`.
3. **Found live, fixed**: the promotion history table wrapped long freight-name hashes and full `user:default/...` entity refs across multiple lines, blowing up row height. Fixed with truncation + hover tooltips and forced single-line cells inside a horizontally-scrolling container.
4. **Found live, fixed (iteratively)**: the governance/events panels split to a narrow right column starting at MUI's `md` (960px) breakpoint, too early for real laptop logical widths (1440–1728px). Raised to `xl` (1920px) after two rounds of user feedback on an actual laptop screen.

## 14. Documentation Updated

This file. In-repo comments were added at the two highest-risk decision points for future readers: the `name: 'delivery'` collision fix in `DeliveryApi.ts`, and the events-synthesis-not-a-new-table rationale in `DeliveryService.ts`.

## 15. Known Limitations

- **Commit SHA and Branch**: not part of the `ReleaseCandidate` domain model at all; rendered as "não disponível" rather than fabricated. Adding them would require a new field on the domain model and its migration — out of this checkpoint's "smallest read/query change" scope.
- **Aprovador**: the authorization ledger's `actorRef` is never exposed through any Change Management HTTP route reachable from here; rendered as "não disponível" rather than substituting `requestedBy`/`boundBy` (requester/binder ≠ approver — that would be exactly the kind of fabrication the contract forbids).
- **Event granularity**: the recent-events feed shows 4 synthesized milestones per request (requested, change-bound, dispatched, terminal); it cannot show intermediate non-terminal Argo sync/health transitions, because `delivery_projections` is upsert-only and a durable event table was judged out of scope (see §5).
- **No retry-after-failure action**: a `failed` request has no UI path back to `dispatchable`/retry — this matches the original MVP's scope (it never had one either), but is worth flagging as an unaddressed product gap for a future checkpoint.
- **Same-target conflict UI path unverified live**: proven correct at the backend (E1.3 regression suite); the frontend's `Conflito` semantic-status mapping exists and is unit-tested, but was not exercised end-to-end against a genuine concurrent dispatch in the sandbox.
- **No screenshot files stored in this repo**: authenticated screens were validated through the user's own browser session and reviewed as images in the working conversation; committing them here was not practical given the session's file-access boundaries.
- **The corrupted reference asset** (§1) should be re-exported and re-committed by its owner in `backstage-docs`.

## 16. STOP

Per the execution contract: no production-adoption review, no continuation of P1 residual/credential hardening, no global Delivery workbench, and no redesign of the Platform tab or any other Backstage tab follows from this checkpoint. Production rollout remains **NO-GO**, independent of and untouched by this UX checkpoint.
