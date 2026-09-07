# Rollback / Emergency Runbook — Delivery / GitOps

## Status

Written to close **Condition C3** of the Production Adoption Review
(`docs/delivery/production-adoption-review.md` §10.3). This is
documentation of the existing mechanism, not new rollback automation — the
architecture already provides a sound revert path (`d1-prd` runs
`selfHeal:true, prune:true`, so a merged Git revert converges automatically).
What was missing was a written procedure naming who does what.

**This document requires review/approval by at least one human other than
its author before C3 is considered closed.** Until that approval lands,
treat this runbook as `IN_PROGRESS`, not final.

---

## 1. Trigger classification

Before doing anything, classify the incident. The action differs completely
by category — using the wrong one wastes the observation window or, worse,
bypasses the governed path for no reason.

| Symptom | Category | Action |
|---|---|---|
| The desired state in Git itself is wrong (bad manifest, bad digest, bad config) | **Git revert** | §2 below |
| Git desired state is correct, but Argo will not converge (`OutOfSync`, comparison error, admission rejection) | **Argo reconciliation issue** | Diagnose the specific sync/health error on the `Application`; do not revert Git for a problem Git doesn't have. Common causes: admission-controller rejection, resource conflict, a stale/broken image reference the desired state never actually specified incorrectly. Fix the underlying resource or image, then trigger/await the next sync. |
| A promotion never actually created the desired-state change at all (Kargo `Promotion` `Errored`, PR never opened, or the writer credential itself failed) | **Kargo/Delivery credential or promotion issue** | Check `Promotion.status` for the failed step. If it is a credential/auth failure, see the token-refresh failure path below. If it is a logic/template error (e.g. the nil-safe-guard class of bug from `p1-residual-closure.md`), fix the `Stage` promotion template directly — this is intentionally imperative, cluster-only state per the accepted D1 layout, not a Git-tracked object. No Git revert is needed because no incorrect desired state was ever written. |
| The application itself is unhealthy at runtime (crashing pod, bad application logic, a dependency outage) but the deployed manifest/digest is exactly what was intended | **Application-runtime incident — not a platform rollback** | This is not a Delivery/GitOps problem. Rolling back Git to an older digest is not automatically correct here (the "older" version may still exist / be unhealthy for the same reason, or masks an app-level bug that needs its own fix). Escalate to the owning team's own incident process. The platform owner may still assist if a Git-level rollback turns out to be the right application-level decision, but that decision belongs to the application owner, not to this runbook. |

**Quick discriminator:** if `kubectl -n <ns> get application <app> -o jsonpath='{.status.sync.status} {.status.health.status}'` shows `Synced Healthy` and the running workload still matches the intended digest, the platform did its job — any problem left is an application-runtime incident, not a rollback trigger.

---

## 2. Git rollback path (the only authorized rollback mechanism)

This is the **only** normal-path rollback. It reuses the exact governed
path already proven for forward promotions — no new authority or tooling
is introduced.

```text
1. Identify the known-good Git revision
   → `git log` on the governed branch (d1/desired-state, or the real
     production desired-state branch once designated) for the last commit
     known to have been Synced/Healthy before the incident.

2. Create a revert branch/commit
   → short-lived branch, `git revert <bad-commit>` (or a hand-authored
     revert if a straight revert would also undo unrelated later work —
     prefer the straight revert whenever possible for auditability).

3. Open a PR to the protected desired-state branch
   → same repository, same branch policy as every other desired-state
     change. Do not push directly — direct push to the protected branch is
     denied even for the org's top-privilege human account (proven in
     P1; TF402455).

4. Obtain required independent approval
   → at least one reviewer who is not the PR's own author/creator
     (branch policy 103: minimumApproverCount:1, creatorVoteCounts:false).
     Under incident pressure, this is still required — see §3 for who that
     approver is and the escalation path if they are unavailable.

5. Merge
   → same merge mechanics as any other desired-state PR.

6. Observe Argo reconciliation
   → the relevant Application(s) (e.g. `d1-prd`) have `selfHeal:true,
     prune:true`; convergence is automatic once the merge lands. No manual
     `argocd app sync` should be needed.

7. Verify health and deployment revision
   → `kubectl -n argocd get application <app> -o jsonpath='{.status.sync.revision} {.status.sync.status} {.status.health.status}'`
     confirm the revision matches the merge commit and status is
     `Synced Healthy`. Also confirm the actual running workload
     (`kubectl -n <target-ns> get deploy -o jsonpath='{.spec.template.spec.containers[0].image}'`)
     matches the reverted manifest.
```

### Explicitly not part of the normal path

Do **not**, as a routine rollback action:

- `kubectl patch` or `kubectl edit` the production Deployment directly;
- `argocd app sync` with a manual override or a source that bypasses Git;
- force-push to the protected branch;
- approve your own revert PR, or otherwise bypass branch policy 103.

Any of the above is break-glass territory (§4), not routine rollback.

---

## 3. Roles / ownership

| Role | Who | Notes |
|---|---|---|
| **Rollback initiator** | Any platform-team member, or the on-call/escalation contact below if outside business hours | Anyone can open the revert PR; merging still requires independent approval per branch policy 103 |
| **Required approver** | The platform owner (currently `Diego Fernandes`, `diego.fernandes@outlook.com` — the same identity that has approved every desired-state PR proven in P1 and the P1 residual closure, e.g. PR #81) | Must not be the PR's own author; branch policy 103 enforces this mechanically |
| **Platform owner** | `Diego Fernandes` | Overall accountability for the Delivery/GitOps mechanism; the named "platform-owner-attended" role required by the authorized first-rollout envelope (`production-adoption-review.md` §11) |
| **Escalation contact if the approver is unavailable** | **Not yet designated — named gap.** Today's platform team is a single named human. This is a real, factual single point of failure for the approval step, not something this runbook can responsibly paper over by inventing a second name. **Action required before the first real production rollout:** the platform owner must name a second qualified approver (or an explicit alternate escalation path) before an unattended incident window can be considered safe. |
| **Break-glass decision-maker** | The platform owner. No one else is authorized to invoke break-glass under this runbook until a second approver/escalation path is designated. |

---

## 4. Break-glass boundary

Break-glass (any direct, non-PR-gated mutation — a manual `kubectl patch`,
a direct `argocd app sync` override, or equivalent) is described here as a
boundary, **not built as new automation** in this checkpoint. It exists
only for the case where the normal PR-gated path (§2) cannot be used in
time to prevent unacceptable impact — e.g., the approver and every
alternate are genuinely unreachable during an active incident, or the Git
host itself is unreachable (per the review's fail-closed analysis: both
Argo and Kargo fail closed if Git is unreachable, so a Git-path outage
blocks the *normal* rollback route specifically).

If break-glass is invoked:

1. It must be **exceptional** — the normal path in §2 was genuinely not
   usable, not merely slower than preferred.
2. It must be **attributable** — record who performed the direct mutation,
   exactly what command was run, against which resource, and why the
   normal path was not used, in a durable location (an ADO work item or
   equivalent — not only verbal/Slack).
3. It must be **followed by restoration of Git authority** — as soon as
   the emergency condition is resolved, the actual live state must be
   reconciled back into Git (either by confirming Git already matches what
   was manually applied and doing nothing further, or by opening the
   normal PR to bring Git in line with what was manually applied) so that
   Argo's `selfHeal` does not silently revert the emergency fix, and so
   that Git remains the durable source of truth going forward.
4. Only the platform owner may authorize invoking break-glass (§3). This
   checkpoint does not grant any identity new direct-mutation permissions
   to make this easier — the existing negative-authority proofs (writer SP
   cannot force-push or bypass policy; ordinary squad/pipeline identities
   are denied on Deployment patch) remain intact and are not weakened by
   this runbook.

---

## 5. Token-refresh failure path (cross-reference)

If the underlying cause is a token-refresh failure rather than a bad
desired-state commit, this is **not** a Git-revert scenario — see the
"Kargo/Delivery credential or promotion issue" row in §1. The mechanism
now raises a visible ADO work-item alert on failure (Condition C4; see
`docs/delivery/pre-rollout-condition-closure.md`), including a watchdog
that catches the case where the refresh pod fails before its own
in-script alert can run. If the alert itself did not fire, check the
watchdog CronJob's own job history directly
(`kubectl -n idp-token-refresh get jobs`) — the alert path shares the same
SP credentials as the refresh mechanism, so if both SP secrets are
themselves expired/revoked, neither the refresh nor the alert can
authenticate, and this must be diagnosed manually (`az ad app credential
list` for expiry dates, then `az ad app credential reset` to recover).

---

## STOP

This runbook documents the existing bounded mechanism. It does not build
rollback automation, does not add a new break-glass tool, and does not
change any RBAC/branch-policy authority beyond what was already proven.
The one open item (§3, second approver/escalation contact) is named
explicitly rather than fabricated, and is recorded as a pending action in
`docs/delivery/pre-rollout-condition-closure.md`.
