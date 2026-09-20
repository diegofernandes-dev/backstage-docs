# ADR-014 — Change resubmission authority and same-changeId identity boundary

- **Status:** Accepted
- **Date:** 2026-09-20
- **Related:** ADR-006, ADR-007, ADR-008, ADR-009, ADR-013
- **Does not supersede:** ADR-013 CAB governance / delegated low-risk autonomy
- **Fills an ADR-009 gap:** who may resubmit a rejected Change, and which fields remain the same business Change under the same `changeId`

## Context

ADR-009 already decided that a rejected round may be corrected and resubmitted under the **same** `changeId` with a new authorization round, and that a fundamentally different business change receives a **new** Change and `changeId`. It did not name who may perform that transition, and it did not freeze the identity fields that distinguish correction from retargeting.

The independent F3.1.3 plan review (`docs/backstage/f3-1-3-plan-architecture-review.md`) therefore REJECTED two F3.1.3b contracts:

1. **G13** — treating `change.create` plus requester / current `ownerRef` member / `platform_admin` as already-granted resubmission authority. ADR-009 lists those actors for **read** or **cancel**, not resubmit. CAB record is a different capability.
2. **G14** — allowing every user-editable create field, including `targetRef` / `ownerRef` / `systemRef`, to change under the same `changeId`. That can still represent a different governed target wearing the prior identity.

This ADR records the conservative reading of ADR-009 Q19 for F3.1.3b. It does not authorize implementation, production cutover, F3.1.4 UI, or F3.2 CAB autonomy. F3.1.3a decision-command contracts are out of scope.

## Decision

### 1. Resubmission business authority

An actor may resubmit a rejected Change iff **both** of the following hold:

```text
actor has server permission
  change-management.change.resubmit

AND

  (actorRef == original Change.requestedBy
   OR actor is a current member of the Change's immutable ownerRef group)
```

Permission and domain actor proof are both mandatory. Neither layer is sufficient alone.

Owner membership is a **current live Catalog membership** proof against the **immutable** `ownerRef` of the original Change identity:

- prefix-agnostic (`relations.memberOf` and `spec.memberOf`);
- not `ownershipEntityRefs` / Team-Project prefix filtering;
- fail closed if the membership source is unavailable;
- fail closed if the owner Group is missing or unresolvable;
- Catalog ownership drift does not substitute a new owner Group for this proof.

If the original Change has no `ownerRef`, the owner-membership path cannot succeed. Only the original requester with the resubmit permission may resubmit that Change.

### 2. What is not standalone resubmission authority

The following are **not** resubmission business authority by themselves:

- `role:default/platform_admin`;
- CAB membership or `change-management.change.authorization.cab.record`;
- participant read (`change.read`, `responsibleRef`, current Catalog owner after drift);
- `change-management.change.create`;
- `change-management.change.authorization.decide`.

RBAC may grant the literal `change-management.change.resubmit` capability to broad technical roles such as `contributor`, `template_executor`, or `platform_admin` **only if** the service still enforces the requester-or-immutable-owner domain proof. Permission alone must never permit resubmission. No generic governance override exists.

### 3. Same-changeId identity is frozen

Resubmission corrects the same business Change. For F3.1.3b these identity fields are immutable across rounds:

| Field | Rule |
|---|---|
| `changeId` | Unchanged |
| `requestedBy` | Original requester |
| identity `createdAt` | Original Change identity time |
| `targetRef` | Original governed target |
| `ownerRef` | Original snapshotted owner associated with that Change identity |
| `systemRef` | Original snapshotted System associated with that Change identity |

Do not re-resolve `ownerRef` / `systemRef` from Catalog during same-`changeId` resubmission merely because Catalog ownership drifted. Historical identity remains tied to the original governed-target snapshot.

### 4. Retargeting requires a new Change

If the operator needs a different `targetRef`, owner, or System, the correction is out of scope for the existing Change: create a new Change / new `changeId`.

Same-`changeId` resubmission that supplies a different `targetRef`, or any other source-level identity field mismatch, fails closed before Round, provider, or index mutation.

### 5. Correctable non-identity fields

Subject to the existing create validation contract, F3.1.3b may correct:

- title;
- summary/description;
- classification;
- risk;
- requested execution window;
- rollback information;
- evidence/references supported by the current create contract;
- execution-plan content.

A successful correction creates a new Round with currently published policy/selectors, new requirements, and new activity IDs. Prior rounds, requirements, decisions, and audits remain immutable history.

### 6. Catalog ownership drift does not rewrite identity or authority

```text
Round 1 Change identity snapshot:
targetRef = component/system X
ownerRef  = Team A
systemRef = System X

Catalog later says target X is now owned by Team B
```

Same-`changeId` resubmission still uses Team A / System X as the immutable Change identity snapshot for authority and history. Team B ownership drift does not silently transfer resubmission rights for the existing Change.

If governance later wants ownership transfer to affect an in-flight or rejected Change, that is a separate future policy/administrative operation. It is not part of F3.1.3b.

### 7. Execution-plan hidden retarget

ADR-008 allows an optional activity `targetRef` of kind `Component` that may differ from `Change.targetRef`. Current create validation only checks kind and Catalog existence; it does not bind the activity Component to the Change System.

F3.1.3b therefore fail-closes a corrected execution plan that would cross the immutable Change System identity:

- omitted activity `targetRef` remains allowed;
- when present, the activity Component's resolved `systemRef` must equal the Change's immutable `systemRef` (including both absent);
- a different System, or introducing a System where the Change identity has none, is `VALIDATION_ERROR` `execution_plan_identity_mismatch`;
- activity `targetRef` must never rewrite `Change.targetRef` / `ownerRef` / `systemRef`.

This closes the hidden retarget path without inventing fields that do not exist in source.

### 8. No runtime authorization from this ADR

This ADR authorizes no production or runtime behavior by itself. Implementation remains gated on an independently accepted F3.1.3 plan and a later explicit implementation prompt.

## Consequences

Positive:

- resubmission authority is an explicit capability plus a domain actor proof, not an inferred admin/CAB/create shortcut;
- same-`changeId` history stays bound to one governed target, owner, and System;
- Catalog drift cannot launder identity or transfer correction rights;
- Model C current projection may still show corrected non-identity fields while identity columns stay frozen.

Costs:

- a dedicated `change-management.change.resubmit` permission must be registered and authorized;
- live Catalog owner-membership proof is required at resubmit time, prefix-agnostic and fail-closed;
- retargeting is an operator-visible new Change, not an in-place edit;
- execution-plan activity targets on resubmit gain a System-identity check that create does not currently enforce.

## Rejected alternatives

| Alternative | Decision |
|---|---|
| Reuse `change.create` as the literal resubmit permission | Rejected — create authorizes a new identity, not rewrite of an existing one |
| `platform_admin` as standalone resubmit authority | Rejected — technical admin is not business-correction authority |
| CAB membership / `cab.record` as standalone resubmit authority | Rejected — CAB decides a requirement; it does not own the Change identity |
| Participant read / `responsibleRef` as resubmit authority | Rejected — ADR-009 read-only |
| Re-resolve owner/system from Catalog on resubmit | Rejected — ADR-007 snapshot identity; Catalog drift must not rewrite history |
| Allow `targetRef` change under the same `changeId` | Rejected — that is a new business Change per ADR-009 Q19 |
| Generic governance override | Rejected — no override plane exists |

## Gate

This ADR is **Accepted**.

Immediate plan consequences:

```text
F3.1.3b resubmission authority: dedicated permission + requester OR live immutable-owner member
F3.1.3b identity: targetRef / ownerRef / systemRef / requestedBy / createdAt frozen
F3.1.3a decision-command contract: unchanged
F3.1.3 implementation: still NO-GO until the revised plan is independently ACCEPTed
Migration required: NO
```

This ADR does not authorize implementation.
