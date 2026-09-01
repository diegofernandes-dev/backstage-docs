# Backstage bridge migration — 2026-09-01

## Decision

The active Backstage architecture/documentation bridge moved from:

`diegofernandes-dev/poc-teams-approval`

to:

`diegofernandes-dev/backstage-docs`

Canonical branch: `main`.

The final legacy import baseline is:

`poc-teams-approval@fe4f8073f2a8785673e32ce51e5f70b7c322ad68`

## Authority after migration

| Concern | Authority |
|---|---|
| Backstage implementation/code/tests/config | Azure DevOps `platform-devops-developer-portal` |
| Architecture, ADRs, normative UX contracts | `backstage-docs@main` |
| Construction/handoff documentation | `backstage-docs@main` workstream folders |
| Historical Teams/Azure DevOps approval POC | `poc-teams-approval` archive |

The legacy repository must not receive new Backstage architectural decisions or active construction handoffs. It remains useful only when investigating historical POC implementation evidence.

## Imported canonical set

The migration established in `backstage-docs`:

- root repository authority/protocol README;
- shared ADR index and ADR-001 through ADR-009;
- GMUD current-state snapshot;
- active GMUD construction log / migration continuation point;
- accepted F3 authorization review packet;
- normative GMUD create/list/detail textual UI contracts;
- Golden Paths / Software Templates / Brownfield workstream documentation home.

ADR-009 is imported as **Accepted** after F3.0.1 convergence. ADO implementation remains at F2.2.1 (`6e28611`); F3.1 is planable but is not implemented/authorized for implementation merely by this migration.

## Historical assets

The legacy repository contains Teams/ADO POC implementation artifacts, old hands-on logs, and binary UI screenshots. Those are historical supporting evidence and were intentionally not made active architecture authority in the new bridge.

Binary screenshots may remain in the legacy archive until intentionally re-captured or migrated. The normative textual UI contracts are present in `backstage-docs` and win if historical screenshots contain superseded labels or flows.

## Cross-workstream protocol

New Backstage fronts should share this bridge instead of creating separate architecture repositories. Examples include:

- GMUD / Change Management;
- Software Templates / Golden Paths;
- legacy/brownfield application adoption;
- Catalog governance;
- developer experience/platform policy.

Workstream-specific documents belong under `docs/<workstream>/`. Decisions with platform-wide impact belong in `docs/adr/`, using the next available ADR number after fetching `origin/main`.

## Handoff contract

Every meaningful implementation checkpoint should leave a concise handoff in this repository containing:

1. ADO repository/branch/SHA;
2. `backstage-docs` baseline and resulting SHA;
3. architecture decisions applied;
4. implementation files/areas changed;
5. tests and functional validation;
6. deviations;
7. unresolved questions;
8. explicit GO/NO-GO for the next slice.

Do not copy proprietary application source trees or secrets into this repository.
