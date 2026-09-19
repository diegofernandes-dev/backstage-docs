# ADR-013 — CAB governance and delegated low-risk autonomy

- **Status:** Accepted
- **Date:** 2026-09-19
- **Related:** ADR-006, ADR-007, ADR-008, ADR-009, ADR-010, ADR-012
- **Supersedes narrowly:** ADR-009 normal-low baseline that allowed one primary approval with no CAB requirement, and ADR-009's absolute "policy requirements can never be waived" rule only for the specific low-risk CAB-autonomy case defined here.

## Context

ADR-009 established the provider-neutral authorization ledger and modeled CAB as one collective governance authority. Its initial F3 baseline allowed a normal low-risk Change to become AUTHORIZED after one configured primary approval, while normal medium/high required primary + CAB.

The governance target is now intentionally stricter:

- CAB remains the default governance authority for normal changes, including low risk;
- a team may earn bounded autonomy for low-risk Changes;
- only CAB governance may grant or revoke that autonomy;
- the requester never self-selects a bypass;
- historical authorization facts remain immutable;
- the autonomy mechanism must not grow into a generic waiver/rules/BPM engine.

The intended operating model is therefore:

```text
default:
  normal + low     -> primary approval + CAB
  normal + medium  -> primary approval + CAB
  normal + high    -> primary approval + CAB

if a valid CAB low-risk autonomy grant applies:
  normal + low     -> primary approval only

emergency:
  Approver A + Approver B before execution
  mandatory CAB retrospective after execution
  no low-risk autonomy path
```

CAB becomes both a Change-decision authority and a governance authority over delegated autonomy.

## Decision

### 1. Two distinct CAB authority planes

CAB exposes two semantically separate capabilities.

**Change-decision authority**
- records a collective CAB approval or rejection for one specific requirement/round;
- remains an append-only `ApprovalDecision` concern.

**Governance authority**
- grants, revokes, renews, narrows, and audits bounded low-risk autonomy;
- never fabricates a CAB approval for a Change it did not review.

The same Backstage Workbench may expose both, but the backend contracts, permissions, commands, and audit facts remain distinct.

## 2. Default-deny policy baseline

The current target policy for normal Changes is:

| Classification / risk | Mandatory pre-execution baseline |
|---|---|
| Normal + low | Primary approval + CAB |
| Normal + medium | Primary approval + CAB |
| Normal + high | Primary approval + CAB |

A future valid low-risk autonomy grant may remove only the **CAB pre-execution requirement** from the **effective requirements** of a `normal + low` round.

It may not:
- remove the primary approval;
- affect medium/high;
- affect emergency pre-approvers;
- affect emergency CAB retrospective;
- remove any unrelated/additive mandatory requirement.

This is the only waiver-like behavior authorized by this ADR.

## 3. CAB autonomy is not a pre-approval

A CAB autonomy grant must never be represented as:

```text
ApprovalDecision {
  decision: approved,
  reason: "delegation"
}
```

for a Change the CAB never reviewed.

The canonical truth is:

> CAB previously granted bounded autonomy, so the CAB requirement was not part of this round's effective requirements.

A round that uses autonomy must preserve immutable evidence of why CAB was not required.

## 4. Narrow domain: CabAutonomyGrant

The initial capability is deliberately not generic.

Conceptual contract:

```text
CabAutonomyGrant

grantId
authorityRef              # the CAB authority
subjectRef                # Catalog Group / team
systemRefs[]              # bounded business/system scope
classification = normal
maxRisk = low
validFrom
validUntil
grantedByActorRef
grantedAt
reason
authorityEvidence
provenance
```

Initial constraints:

- `subjectRef` is a Catalog Group, not an individual user;
- scope is bounded by one or more Catalog Systems;
- classification is fixed to `normal`;
- risk eligibility is fixed to `low`;
- validity is finite;
- no wildcard enterprise-wide grant in the initial model;
- no user-carried privilege that follows a person between teams;
- no arbitrary expression language;
- no generic `skipRequirement` field.

ADR-010 System semantics are used because a System represents the business/product boundary. Component-level narrowing may be considered later but is not required by this ADR.

## 5. Applicability

A grant is applicable only when all conditions hold:

1. the Change is `normal + low`;
2. the subject team matches the governed ownership/responsibility scope required by the implementation contract;
3. every relevant execution scope is covered by the grant's `systemRefs`;
4. the grant is active and not revoked;
5. `validFrom` is at or before the authoritative submission/Round boundary;
6. `validUntil` covers the full requested execution window;
7. the server can prove applicability from canonical platform/Catalog facts.

If any required fact is missing, ambiguous, unavailable, outside scope, or cannot be proven, CAB remains required.

Autonomy therefore fails **closed to CAB-required**, not open to bypass.

## 6. Multi-activity guard rail

A Change may contain multiple execution activities.

A grant must never "leak" autonomy from one covered activity/team/system to an uncovered activity.

Initial invariant:

> CAB may be omitted only when **all relevant execution scopes** for the Change are covered by applicable autonomy grants under the final F3.2 implementation contract.

If one relevant activity/system is uncovered or ambiguous, the round keeps its CAB requirement.

The implementation may intentionally start even more conservatively (for example one ownership/system boundary only) but may not be less strict than the all-covered invariant.

## 7. Validity and requested window

An autonomy grant must cover the execution window, not only the submission instant.

At minimum:

```text
grant.validFrom <= authoritative submission/Round time
AND
grant.validUntil >= requestedWindow.endsAtUtc
```

This prevents a Change from receiving CAB autonomy for an execution window that extends past the grant's expiry.

The grant is snapshotted/applied at Round creation. Later expiry does not rewrite an already committed Round.

## 8. Revocation and renewal

Autonomy history is append-only.

Conceptually:

```text
CAB-AUT-0042 GRANTED
CAB-AUT-0042 REVOKED

CAB-AUT-0043 GRANTED
supersedes: CAB-AUT-0042
```

Do not silently mutate the historical grant record.

Rules:

- revocation is prospective for new rounds;
- renewal creates a new immutable grant/version/fact rather than rewriting history;
- a Round that legitimately used a grant remains historically valid after later revocation;
- preventing an already-submitted Change requires a separate hold/cancel/reject governance action, not retroactive grant mutation.

## 9. Grant/revoke race

Grant applicability and Round materialization must not use a vulnerable long-lived "check now, create Round later" sequence.

The future F3.2 implementation must define a durable ordering so that:

```text
revocation becomes authoritative first
  -> new Round cannot use the grant

Round application becomes authoritative first
  -> that Round retains the snapshotted grant evidence
  -> revocation applies prospectively
```

The exact storage/transaction mechanism belongs to F3.2 planning. No distributed lock, generic workflow engine, or mutable historical repair is implied.

## 10. RBAC and authority membership

Participant read, platform administration, CAB Change decisions, and CAB autonomy governance are distinct powers.

Conceptual permissions:

```text
change.authorization.cab.record
change.authorization.cab.autonomy.read
change.authorization.cab.autonomy.manage
change.authorization.audit.read
```

Literal names may be refined during implementation planning, but the separation is mandatory.

To record a CAB decision or manage autonomy, the actor must satisfy both:

```text
server-side RBAC capability
AND
current authority to act for the configured CAB authority
```

`platform_admin` does not automatically imply CAB business authority.

The frontend is never the authority.

## 11. CAB Workbench

The preferred Backstage CAB Workbench is a future interaction surface, not canonical state.

Target information architecture:

```text
CAB Workbench
├── Approvals
│   ├── normal/medium/high CAB requirements
│   └── emergency retrospectives
├── Autonomies
│   ├── active grants
│   ├── grant
│   ├── revoke
│   ├── renew
│   └── scope/expiry
└── Governance / History
    ├── grant usage
    ├── decisions
    ├── revocations
    ├── audit evidence
    └── future incident/change-quality context
```

Teams may later invoke the same backend commands, but neither Teams nor Backstage UI becomes canonical authority.

## 12. Accountability and feedback

Using autonomy does not remove accountability.

For every round where CAB is omitted due to autonomy, the authorization ledger/audit must preserve enough immutable evidence to reconstruct:

- the grant identity;
- CAB authority;
- subject/team;
- governed System scope;
- validity interval;
- grant provenance/evidence;
- who granted it;
- why it applied to this Round;
- authoritative application time.

Future incident/change-quality integrations may show change failure rate, rollback rate, incidents, or policy violations to support CAB renewal/revocation decisions.

The initial model does **not** automatically revoke autonomy from a score or threshold. Such metrics are decision support until separately governed.

## 13. Side effects on current F3 plan

### F3.1.0 — ledger foundation

No rewrite required. Existing append-only Round/Requirement/Decision/Audit foundation remains valid.

### F3.1.1a — published policy

The accepted implementation remains historical evidence and is not edited in place.

Before F3.1.2b enables ledger submissions, publish a **new immutable policy version** where:

```text
normal.low    -> primary + CAB
normal.medium -> primary + CAB
normal.high   -> primary + CAB
emergency     -> unchanged
```

This narrow publication checkpoint is tracked as **F3.1.1c — CAB-safe policy baseline publication**.

Do not mutate or reuse the existing published policy identity.

### F3.1.1b — selector runtime

No redesign required. The existing `cab-authority` selector remains usable.

### F3.1.2a — canonical Change

No impact. May proceed after the F3.1.2 implementation-plan gate accepts.

### F3.1.2b — Round 1 integration

F3.1.2b consumes the CAB-safe policy baseline.

Until the later autonomy workstream exists, a `normal + low` ledger submission materializes both primary and CAB requirements.

**F3.1.2b must not implement a bypass, grant registry, or autonomy UI.**

### F3.1.3 — decision commands

CAB decision commands remain part of the ordinary requirement-decision model.

### F3.1.4 — read/RBAC boundaries

Must preserve separate capability boundaries for participant read, individual decisions, CAB record, audit read, and future autonomy administration. Autonomy-management permissions may be introduced in the later workstream rather than forced into F3.1.4.

### F3.2 — CAB Governance & Delegated Autonomy

A new bounded workstream follows the core F3.1 authorization path.

Recommended decomposition:

1. **F3.2.0 — Autonomy contract + append-only storage**
   - `CabAutonomyGrant` contract;
   - grant/revoke/renew history;
   - bounded Group + System scope;
   - finite validity;
   - no UI.

2. **F3.2.1 — Server-authoritative governance commands + RBAC**
   - grant/revoke/renew commands;
   - CAB authority membership check;
   - dedicated permissions;
   - idempotency/audit.

3. **F3.2.2 — Round applicability integration**
   - deterministic low-risk grant resolution;
   - all-covered multi-activity rule;
   - no-CAB effective-requirement composition;
   - immutable autonomy-applied audit evidence;
   - grant/revoke concurrency contract.

4. **F3.2.3 — CAB Workbench**
   - Approvals;
   - Autonomies;
   - Governance/History.

Metrics/incident correlation and automated trust scoring remain later work unless explicitly authorized.

## 14. Rejected alternatives

| Alternative | Decision |
|---|---|
| Low risk always bypasses CAB | Rejected — opens the gate by classification alone |
| Requester chooses `skipCab` | Rejected — requester is not CAB authority |
| Grant autonomy directly to a user | Rejected for initial model — privilege should belong to bounded team/system scope |
| Generic waiver/exception engine | Rejected — unnecessary BPM/policy-engine expansion |
| Reuse CAB ApprovalDecision as an autonomy grant | Rejected — fabricates approval of a Change CAB did not review |
| Automatically revoke from a quality score | Deferred — metrics are initially decision support |
| Let `platform_admin` manage CAB autonomy automatically | Rejected — technical admin does not imply business authority |
| Retroactively add CAB to existing rounds when a grant is revoked | Rejected — violates immutable historical evidence |
| Apply grant when only one activity is covered | Rejected — autonomy cannot leak to uncovered execution scope |

## 15. Consequences

Positive:

- default governance is conservative;
- autonomy is earned and bounded rather than assumed;
- CAB can evolve from a transaction bottleneck into a governance body;
- every bypass of CAB scrutiny has an accountable CAB-issued basis;
- historical audit remains truthful;
- medium/high and emergency protections stay unchanged;
- the model remains provider-neutral.

Costs:

- a new immutable governance concept and later persistence/commands are required;
- Round creation eventually gains one additional server-side applicability input;
- CAB Workbench grows beyond simple approval inbox;
- multi-activity and grant/revoke concurrency need explicit tests;
- the current F3.1.1 policy baseline must be superseded by a new immutable publication before F3.1.2b.

## 16. Gate

This ADR is **Accepted**.

Immediate plan consequences:

```text
F3.1.2 concurrency-corrected plan: must be re-reviewed with ADR-013 dependency
F3.1.2a: architecture-independent from CAB autonomy
F3.1.1c: required before F3.1.2b
F3.1.2b: CAB required for normal.low until F3.2 exists
F3.2: future CAB autonomy workstream
```

This ADR does not authorize implementation.
