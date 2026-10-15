# Who Can Request a Testnet Reset

## Purpose

A Bridgelet testnet redeploy changes every contract ID that every integrator has
stored, and destroys all on-chain state (see
[`reset-policy.md`](reset-policy.md#what-is-destroyed-and-what-is-not)). It is
therefore a decision about **shared infrastructure**, not a personal action, and
it needs a defined path: who may *ask* for one, who may *approve* one, and who
may *carry it out*.

This document proposes that path. It exists so that no individual contributor
resets shared testnet state unilaterally, and so that the coordination the
reset procedure assumes (an announcement, a changelog entry, updated registry
files) is actually triggered by a decision somebody is accountable for.

## Status

**Proposal. Nothing in this document is in force until a maintainer adopts it.**

| Statement | Basis |
|---|---|
| No file in the repository names who may approve a redeploy | `reset-policy.md`, "Authority": *Undecided* |
| `testnet/docs/contact-and-ownership.md` lists a Testnet Infrastructure Lead and a Contract Security Team | That document |
| The deploy workflow is `workflow_dispatch`-only with no environment protection | `reset-policy.md`, "Authority" |
| `testnet/reset/README.md` does not exist; `testnet/reset/` holds only `communication-template.md` | Directory listing |
| No contracts are currently recorded as deployed | `testnet/registry/contract-status.md` |

This document reuses the roles already named in
`testnet/docs/contact-and-ownership.md` rather than inventing new ones. Whether
those roles are filled by the people the file implies is for the maintainers to
confirm.

## Which kind of reset this covers

| | Platform-level reset | Bridgelet redeploy |
|---|---|---|
| Decided by | The Stellar Development Foundation | This project |
| Can anyone here request it? | **No.** It cannot be requested or approved by this project. | Yes, through the path below |
| Covered by this document | Only the *response* (see [Recovering from a platform reset](#recovering-from-a-platform-reset)) | Yes |

## Roles

| Role | Who | May do |
|---|---|---|
| **Requester** | Anyone: a maintainer, contributor or integrator | Open a reset request issue. Nothing more. |
| **Approver** | The Testnet Infrastructure Lead | Approve or decline a redeploy request. |
| **Security consult** | The Contract Security Team | Must be consulted when the stated reason is security-related. May request an emergency redeploy. |
| **Operator** | Whoever holds the deployer key | Run the redeploy, **only** after a recorded approval. The key grants the ability, not the authority. |

The operator and the approver may be the same person. They may not be the same
*decision*: the approval is recorded before the deploy is run, not inferred from
the deploy having been run.

## The request

A reset request is a GitHub issue that states:

| Field | Content |
|---|---|
| Reason | One of the triggers in `reset-policy.md` ("Triggers"), or a described alternative |
| Contracts affected | All four, or the specific ones |
| Why an in-place `upgrade()` is not enough | Required, because a redeploy is the more destructive option |
| Proposed date and time (UTC) | Leaves room for the notice period below |
| Integrator impact | Known dependent tests, fixtures or documents |

## Approval

1. The approver records **approve** or **decline**, with a reason, on the issue.
2. If the reason is security-related, the Security consult comments first. A
   security reason does not skip approval; it shortens the notice period.
3. The approval comment names the **operator** and the **agreed time**.

### Notice before a planned reset

`reset-policy.md` records that no notice commitment exists. This proposal adds a
minimum for *planned* redeploys, which the maintainers may change:

| Situation | Minimum notice |
|---|---|
| Planned redeploy | Announcement published before the deploy is run, using `testnet/reset/communication-template.md` |
| Security-driven redeploy | Announcement as soon as it is safe to publish; approval is still required |

## Emergencies

A real emergency (for example a suspected exploit of an unauthenticated path,
see `testnet/runbooks/reentrancy-suspicion.md`) may justify acting before a
full notice period. It does **not** justify skipping the record:

- The Security consult may request the redeploy, and the approver (or, if
  unreachable, a second maintainer) approves it on the issue.
- The rationale is written down at the time, not reconstructed later.
- `testnet/runbooks/reentrancy-suspicion.md` already states that a redeploy
  must not be used as an investigation step. That rule stands.

## After approval

The redeploy is complete only when every artefact listed in
`reset-policy.md` ("What must change in the same change") has been updated, and
the changelog entry links back to the approval issue so the decision is
traceable.

## Recovering from a platform reset

An SDF testnet reset takes the deployment down without anyone here approving it.
The recovery redeploy still needs an approver and an operator on record, but the
*reason* is already settled, so the issue can be short and the approval quick.
Detection steps are in `reset-policy.md` ("Detecting that your world is no
longer valid").

## Undecided

| Question | State |
|---|---|
| Is the Testnet Infrastructure Lead the right approver, and is the role filled? | Not confirmed in the repository |
| Should the deploy workflow gain environment protection requiring this approval? | Not proposed here; `.github/workflows/` and `scripts/` are out of scope |
| Is a second approver needed when the operator is also the approver? | Open |
| Where do requests live if the repository has no issue template? | Plain issues, until a template exists |

## Related Documentation

- [`reset-policy.md`](reset-policy.md): triggers, what is destroyed, and the
  "Authority" gap this document proposes to fill
- [`../docs/contact-and-ownership.md`](../docs/contact-and-ownership.md): the
  roles reused here
- [`../reset/communication-template.md`](../reset/communication-template.md):
  the announcement to publish before a planned redeploy
- [`../security/admin-key-hygiene.md`](../security/admin-key-hygiene.md): why
  holding the deployer key is not the same as being authorised to use it
- [`../runbooks/reentrancy-suspicion.md`](../runbooks/reentrancy-suspicion.md):
  why a redeploy is not an investigation step
- [`../docs/changelog.md`](../docs/changelog.md): where the approved redeploy is
  recorded

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial proposal for who may request, approve and operate a testnet reset |
