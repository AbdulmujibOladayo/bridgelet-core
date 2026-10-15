# Worked Example: `SweepController::update_authorized_destination()` and Its `dest_upd` Event

## Overview

`update_authorized_destination(new_destination)` is the only way to change a
controller's locked destination after deployment. This example calls it on
testnet, verifies the resulting `DestinationUpdated` event, and pins down the
question the interface makes surprisingly easy to answer wrong: **who is
authorized to call it**.

The complete rule set, straight from
`contracts/sweep_controller/src/lib.rs`:

```rust
pub fn update_authorized_destination(env: Env, new_destination: Address) -> Result<(), Error>
```

| Condition | Outcome |
|---|---|
| Caller is the `creator` recorded at `initialize()` (with `creator.require_auth()`) **and** the controller nonce is `0` | Success; one `dest_upd` event |
| Caller is not the creator (or no creator is recorded) | `AuthorizationFailed` (3) |
| Nonce is `> 0` — any `execute_sweep`/`claim` has ever succeeded | `AccountAlreadySwept` (7) |

Two facts that are not obvious from the name:

1. **The authorized signer cannot do this.** The Ed25519 `authorized_signer`
   authorizes *sweeps*; it has no privileges over configuration. Only the
   `creator` address can update the destination.
2. **The call also locks a flexible controller.** If the controller was
   initialized with `authorized_destination = None`, its first successful
   `update_authorized_destination` *sets* a destination — converting it to
   locked mode. The event's `old_destination` is `None` in exactly this case.

---

## Prerequisites

```bash
export NETWORK=testnet
export RPC_URL="https://soroban-testnet.stellar.org"
export READER="<funded-testnet-identity>"
export CREATOR="<the-creator-identity-from-initialize>"
export IMPOSTOR="<any_other_funded_identity>"

export DEST_A="<original-locked-destination>"
export DEST_B="<new-locked-destination>"
```

- A `SweepController` you have just initialized in locked mode with
  destination `DEST_A` — the first half of
  [`testing-locked-mode.md`](testing-locked-mode.md) — with **no successful
  sweep yet** (nonce `0`).
- Or a flexible-mode controller, if you want to demonstrate the
  lock-a-flexible-controller behavior in Step 4.
- The `CREATOR` identity is the same address passed as `creator` in
  `initialize`. If you no longer have that key, the destination cannot be
  changed — there is no admin override.

---

## Step 1: Confirm the Nonce Is Still Zero

The update is only permitted before any sweep has consumed a nonce. Read it
first, both as a gate and as the value you will re-check in Step 5:

```bash
export CONTROLLER_ID="<locked-or-flexible-controller-id>"

NONCE=$(stellar contract invoke \
  --simulate-only \
  --id "$CONTROLLER_ID" \
  --network "$NETWORK" \
  --source "$READER" \
  -- \
  get_nonce)
echo "nonce: $NONCE"
# must print 0
```

If the nonce is not 0, stop — the rest of this example will fail with (7)
by design, and the lock is already permanent.

## Step 2: Update the Destination as the Creator

```bash
stellar contract invoke \
  --id "$CONTROLLER_ID" \
  --network "$NETWORK" \
  --source "$CREATOR" \
  -- \
  update_authorized_destination \
  --new_destination "$DEST_B"
# → ()
```

`--source` **is** the authorization. The contract calls
`creator.require_auth()`, so the submitting identity must be the creator
address itself; a relayer cannot be substituted unless it carries the
creator's signed auth entry. The argument name is the Rust parameter name,
`new_destination`.

## Step 3: Verify the `dest_upd` Event

```bash
stellar contract events \
  --contract-id "$CONTROLLER_ID" \
  --start-ledger "$START_LEDGER" \
  --filter '{"topics": [["dest_upd"]]}' \
  --network testnet
```

Record `START_LEDGER` before Step 2. Expected payload (struct
`DestinationUpdated` in `contracts/sweep_controller/src/lib.rs`; topic
encoding per [`event-topics.md`](../registry/event-topics.md)):

```text
topic[0] = "dest_upd"
data     = {
  old_destination: <DEST_A>,   // Option<Address>: was Some(DEST_A) for a
                               // locked controller; None if this call locked
                               // a previously flexible controller
  new_destination: <DEST_B>
}
```

Decoding checklist:

- `old_destination` is an `Option<Address>`. `null`/`None` is not an error —
  it means the controller had no lock before this call.
- `new_destination` must equal exactly the address you passed; a mismatch
  means you are reading events for a different controller.
- One event per successful update, in call order. Multiple `dest_upd` events
  can only exist while the nonce is still 0 — each successful update keeps
  it there.

## Step 4 (Optional): Watch a Flexible Controller Become Locked

On a flexible-mode controller (`authorized_destination = None` at
`initialize`, hence no `dest_auth` event in its history):

```bash
stellar contract invoke \
  --id "$FLEXIBLE_CONTROLLER_ID" \
  --network "$NETWORK" \
  --source "$CREATOR" \
  -- \
  update_authorized_destination \
  --new_destination "$DEST_B"

# event shows old_destination = None, new_destination = DEST_B
```

From this point the controller rejects every other destination with
`UnauthorizedDestination` (13), exactly as if it had been deployed locked.
If your system assumed "no `dest_auth` event ⇒ flexible", this is the case
that breaks the assumption — derive the current lock from behavior (or track
`dest_upd` events), not from the absence of one event.

## Step 5: Prove the Authorization Boundaries

### Negative: a non-creator cannot update

```bash
stellar contract invoke \
  --id "$CONTROLLER_ID" \
  --network "$NETWORK" \
  --source "$IMPOSTOR" \
  -- \
  update_authorized_destination \
  --new_destination "$IMPOSTOR"
# → Error::AuthorizationFailed (3)
```

This fires regardless of the nonce — the creator check runs first. Note that
the Ed25519 `authorized_signer` key is *not* a credential here; signing with
it changes nothing. The only credential is the creator address's auth.

### Negative: after the first sweep, the lock is permanent

Run one successful signed sweep (any destination, per
[`signed-sweep-walkthrough.md`](signed-sweep-walkthrough.md)), then:

```bash
stellar contract invoke \
  --simulate-only \
  --id "$CONTROLLER_ID" \
  --network "$NETWORK" \
  --source "$CREATOR" \
  -- \
  update_authorized_destination \
  --new_destination "$DEST_A"
# → Error::AccountAlreadySwept (7)
```

The nonce check (`nonce > 0`) is what earns the error its name — it counts
*sweeps*, not accounts, and it is controller-global. A `claim()` consumes the
nonce the same way, so a claim-only controller is equally locked
afterwards.

---

## Expected Outputs

| Step | Call | Expected |
|---|---|---|
| 1 | `get_nonce` | `0` |
| 2 | `update_authorized_destination` as creator | `Ok(())` |
| 3 | events filter `["dest_upd"]` | one event: `old_destination: Some(DEST_A)`, `new_destination: DEST_B` |
| 4 | same call on a flexible controller | `Ok`; event shows `old_destination: None` |
| 5 | same call as non-creator | `AuthorizationFailed` (3) |
| 5 | same call as creator after any successful sweep/claim | `AccountAlreadySwept` (7) |
| — | sweep to `DEST_A` after the update | `UnauthorizedDestination` (13) — the old lock is gone |
| — | sweep to `DEST_B` after the update | destination check passes; proceeds to signature verification |

## Failure Modes

| Symptom | Cause | Fix |
|---|---|---|
| `AuthorizationFailed` (3) as the "creator" | The submitting identity's address differs from the `creator` recorded at `initialize` (different network profile, wrong key) | Compare `stellar keys address` with the creator in the original initialize; there is no recovery if the key is lost |
| `AccountAlreadySwept` (7) unexpectedly | A concurrent sweep (shared controller, [`nonce-tracking.md`](../config/nonce-tracking.md)) consumed the nonce | The lock is permanent; deploy a new controller if you needed the change |
| No `dest_upd` event found | Query start ledger is after the update, or wrong contract ID | Use the ledger recorded before the call; verify the ID per [`contract-not-found-error.md`](../runbooks/contract-not-found-error.md) |
| Sweeps to the new destination still fail with (13) | You are querying/invoking a different controller deployment than the one updated | Correlate `--id` with the `dest_upd` event's contract |
| Flexible controller rejects nothing | `update` never actually ran (prior (3)/(7)) | Check Step 1's nonce and the caller before assuming the lock took |

## Related Documentation

- [`testing-locked-mode.md`](testing-locked-mode.md) — deploying locked and proving the rejection
- [`signed-sweep-walkthrough.md`](signed-sweep-walkthrough.md) — consuming the nonce (Step 5's permanent lock)
- [`gas-free-claim-walkthrough.md`](gas-free-claim-walkthrough.md) — `claim()` also consumes the nonce
- [`event-topics.md`](../registry/event-topics.md) — `dest_upd` topic and `DestinationUpdated` payload
- [`error-handling.md`](error-handling.md) — codes 3, 7, and 13 in the handling map
- [`nonce-tracking.md`](../config/nonce-tracking.md) — the global nonce this feature races against

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-10-15 | Initial update_authorized_destination worked example |
