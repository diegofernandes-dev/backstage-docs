# Backstage Documentation Bridge

This repository is the canonical GitHub documentation bridge for the Backstage platform work.

## Authority model

- **Implementation source of truth:** Azure DevOps repository `platform-devops-developer-portal`.
- **Architecture, ADRs, UI contracts, handoffs and cross-workstream documentation source of truth:** this repository, `diegofernandes-dev/backstage-docs`, branch `main`.
- **Legacy bridge:** `diegofernandes-dev/poc-teams-approval` is retained as historical POC evidence only. It must not be used as the active architecture/documentation authority after the migration recorded here.

The implementation repository answers **what is implemented**. This repository answers **what should be implemented**, why architectural decisions were made, and the current reviewed construction state.

## Working protocol

Before starting any architecture or implementation checkpoint that depends on these documents:

1. fetch `origin/main` from this repository;
2. compare the local documentation state with `origin/main`;
3. read the relevant ADRs/current-state/handoff documents before proposing changes;
4. after implementation in Azure DevOps, update the corresponding construction documentation here;
5. record both the Azure DevOps implementation SHA and the `backstage-docs` documentation SHA in the handoff.

Do not reconstruct missing history from memory when the canonical documents exist here.

## Documentation areas

- `docs/adr/` — cross-workstream architecture decisions.
- `docs/backstage/` — current platform state and implementation/construction handoffs.
- `docs/ui/` — normative Backstage UI contracts and visual notes.
- `docs/architect-review-*.md` — active architecture review packets.
- Future workstreams should add focused documentation areas under `docs/` while sharing the same ADR set when decisions affect the overall Backstage platform.

Examples of independent workstreams that can coexist here include:

- GMUD / Change Management;
- application Software Templates / Golden Paths;
- brownfield / legacy application adoption;
- Catalog governance;
- developer experience and platform policy.

## Migration provenance

The initial canonical documentation set is migrated from `diegofernandes-dev/poc-teams-approval@fe4f8073f2a8785673e32ce51e5f70b7c322ad68` on 2026-09-01.

That repository remains useful for historical Teams/Azure DevOps POC implementation evidence, but new Backstage architecture and construction documentation must be written here.

Historical binary screenshots that were produced by the old POC may remain referenced from the legacy archive until they are intentionally re-captured or migrated; they are supporting evidence, not architecture authority.
