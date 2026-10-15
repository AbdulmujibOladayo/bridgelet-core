# Worked Example: Testing Locked-Destination Mode on Testnet

## Overview

`SweepController` has two destination modes, chosen once at `initialize()`:

| Mode | `authorized_destination` argument | Effect |
|---|---|---|
| Flexible | `None` | Any destination is accepted |
| **Locked** | `Some(<G... address>)` | Every sweep and claim may only go to that address; anything else fails with `UnauthorizedDestination` (code 13) |

This example deploys a controller in **locked mode** on testnet and proves
the lock actually holds: a sweep to a different destination is rejected, and —
the part worth the test — the rejection is *deterministic and cheap to
observe* because the destination check runs **before** any signature
verification.

> There is **no on-chain getter** for the authorized destination. The
> `dest_auth` event emitted at initialization, and the rejection behavior
> tested below, are the only records of the lock. That is why this example
> verifies via both.

---

## Prerequisites

Four things, following the placeholder convention in [`README.md`](README.md):

```bash
export NETWORK=testnet
export RPC_URL="https://soroban-testnet.stellar.org"
export READER="<funded-testnet-identity>"          # simulations only
export CREATOR="<controller-creator-identity>"     # will be the creator
export RELAYER="<sweep-submitter>"                 # pays fees for invoke

export EPHEMERAL_ACCOUNT_ID="<initialized-account-with-a-payment>"
export SWEEP_CONTROLLER_WASM_HASH="<verified-controller-wasm-hash>"
```

- `EPHEMERAL_ACCOUNT_ID` must be an `EphemeralAccount` that is initialized,
  has at least one payment recorded, has not expired, **and was initialized
  with `authorized_controller` pointing at the controller you are about to
  deploy**. A fresh create-pay flow is in [`README.md`](README.md).
- `CREATOR` is a funded identity; it signs the controller's `initialize`
  and (later, if needed) `update_authorized_destination`. Keep it — the
  locked destination cannot be changed by anyone else.
- Contract IDs for anything you do not deploy fresh:
  [`contract-status.md`](../registry/contract-status.md).

```bash
export DEST_LOCKED="<G_ADDRESS_THAT_SHOULD_RECEIVE_SWEEPS>"
export DEST_OTHER="<any_OTHER_funded_G_ADDRESS>"
```

---

## Step 1: Initialize the Controller in Locked Mode

Locked mode is simply `initialize` with a non-`None` destination. With the
stellar CLI, pass the optional argument:

```bash
export SWEEP_CONTROLLER_ID="$(stellar contract deploy \
  --wasm-hash "$SWEEP_CONTROLLER_WASM_HASH" \
  --network "$NETWORK" \
  --source "$CREATOR")"

export AUTHORIZED_SIGNER_PUBKEY="<32-byte-ed25519-pubkey-hex>"

stellar contract invoke \
  --id "$SWEEP_CONTROLLER_ID" \
  --network "$NETWORK" \
  --source "$CREATOR" \
  -- \
  initialize \
  --creator "$(stellar keys address "$CREATOR")" \
  --authorized_signer "$AUTHORIZED_SIGNER_PUBKEY" \
  --authorized_destination "$DEST_LOCKED"
```

The Ed25519 signer key is unrelated to destination locking — it authorizes
*sweeps*, not *where they go*. Generating one is covered in
[`signed-sweep-walkthrough.md`](signed-sweep-walkthrough.md); for this test
a throwaway keypair is fine, because the rejection cases below never reach
signature verification.

`initialize` may be called exactly once; a second call fails with
`AuthorizationFailed` (3).

## Step 2: Verify the Lock Was Recorded — the `dest_auth` Event

```bash
stellar contract events \
  --contract-id "$SWEEP_CONTROLLER_ID" \
  --start-ledger "$START_LEDGER" \
  --filter '{"topics": [["dest_auth"]]}' \
  --network testnet
```

Record `START_LEDGER` before Step 1. Expected payload (struct
`DestinationAuthorized` in `contracts/sweep_controller/src/lib.rs`, topics
per [`event-topics.md`](../registry/event-topics.md)):

```text
topic[0] = "dest_auth"
data     = { destination: <DEST_LOCKED> }
```

Exactly one `dest_auth` event exists per locked controller, for the life of
the deployment. A controller initialized in flexible mode emits none — which
is also how you can tell the two apart historically, given the missing
getter.

## Step 3: Prove a Sweep to a Different Destination Is Rejected

Here is the property that makes this test cheap: in
`SweepController::execute_sweep`, `validate_destination` runs **before** the
Ed25519 signature is verified. So you can pass any syntactically valid
64-byte signature — even all zeros — and the call must fail with
`UnauthorizedDestination` (13), not with a signature trap:

```bash
stellar contract invoke \
  --id "$SWEEP_CONTROLLER_ID" \
  --network "$NETWORK" \
  --source "$RELAYER" \
  -- \
  execute_sweep \
  --ephemeral_account "$EPHEMERAL_ACCOUNT_ID" \
  --destination "$DEST_OTHER" \
  --auth_signature "$(python3 -c 'print("00"*64)')"
# → Error::UnauthorizedDestination (13)
```

Interpretation:

- Getting **code 13** proves the destination lock works. The dummy signature
  was never examined; nothing about signing was tested.
- Getting a **trap in `ed25519_verify`** instead would mean the destination
  check *passed* — i.e. the controller is not locked, or `DEST_OTHER` equals
  the locked value. Check the Step 2 event and your two variables.
- Getting `AccountNotReady`/`NoPaymentReceived` is impossible here (the
  destination check precedes the sweep), and would indicate a different
  controller than the one you deployed.

The same function ordering applies to the claim path —
`claim(recipient, ephemeral_account)` runs `recipient.require_auth()` first,
then validates the destination — so a claim whose recipient is not the locked
destination *can* also fail with 13, but only **after** the recipient's auth
has been satisfied (see
[`gas-free-claim-walkthrough.md`](gas-free-claim-walkthrough.md) for who signs
what):

```bash
# Without DEST_OTHER's signature available, this fails at require_auth —
# an auth failure with no contract error code — never reaching the lock.
stellar contract invoke \
  --id "$SWEEP_CONTROLLER_ID" \
  --network "$NETWORK" \
  --source "$RELAYER" \
  -- \
  claim \
  --recipient "$DEST_OTHER" \
  --ephemeral_account "$EPHEMERAL_ACCOUNT_ID"

# To observe code 13 on the claim path, have DEST_OTHER sign the claim
# auth entry (SDK envelope, per gas-free-claim-walkthrough.md). Once its
# auth passes, the destination check rejects with:
# → Error::UnauthorizedDestination (13)
```

Note the contrast with the signed path: on `execute_sweep` the lock is
checked *before* any signature work, so a dummy signature still yields a
clean 13. On the claim path the recipient's auth gates first, so code 13 is
only reachable with a correctly signed entry.

## Step 4: Positive Control — the Locked Destination Is Accepted

A rejection test without a success control proves nothing about *which*
check rejected. Run the real signed sweep to `DEST_LOCKED`, following
[`signed-sweep-walkthrough.md`](signed-sweep-walkthrough.md) end to end
(fresh `get_nonce` → sign → `execute_sweep`), with the destination pinned:

```bash
stellar contract invoke \
  --id "$SWEEP_CONTROLLER_ID" \
  --network "$NETWORK" \
  --source "$RELAYER" \
  -- \
  execute_sweep \
  --ephemeral_account "$EPHEMERAL_ACCOUNT_ID" \
  --destination "$DEST_LOCKED" \
  --auth_signature "$SIG"
```

Success is the full checklist from that walkthrough — `get_status` is `2`,
`can_sweep` is `false`, the nonce advanced by one, and both the account's
`swept_mul` and the controller's `sweep` events are present with
`destination == DEST_LOCKED`.

---

## Expected Outputs

| Step | Call | Expected |
|---|---|---|
| 1 | `initialize(..., Some(DEST_LOCKED))` | `Ok(())`; one `dest_auth` event |
| 2 | events filter `["dest_auth"]` | exactly one event, `destination == DEST_LOCKED` |
| 3 | `execute_sweep` → `DEST_OTHER`, dummy signature | `UnauthorizedDestination` (13) — **not** a signature trap |
| 3 | `claim` → `DEST_OTHER`, unsigned | auth failure with no contract code (`require_auth` runs first) |
| 3 | `claim` → `DEST_OTHER`, recipient-signed | `UnauthorizedDestination` (13) |
| 4 | `execute_sweep` → `DEST_LOCKED`, valid signature | `Ok`; status `Swept`; nonce +1; `sweep`/`swept_mul` events |
| — | second `initialize` on the same controller | `AuthorizationFailed` (3) |

## After the Test: Changing the Lock

While the controller's nonce is still 0 (no sweep has succeeded yet), the
creator can replace the locked destination with
[`update_authorized_destination.md`](update-authorized-destination-flow.md) —
including the failure signatures once the nonce has moved. Once any
`sweep`/`claim` succeeds, the lock is permanent: further updates fail with
`AccountAlreadySwept` (7).

---

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Expecting a getter for the locked destination | Looking for `get_authorized_destination` that does not exist | Verify via `dest_auth` event and rejection behavior |
| Reading a code-13 rejection as "signature failed" | Wrong runbook | The destination check runs first; 13 is never a signature problem |
| Reusing a controller that already swept | Update attempts fail with (7) | Deploy a fresh controller per test, or test the update flow while nonce is 0 |
| Testing with an account whose `authorized_controller` is a different ID | Sweeps fail in the account, not the controller | The ephemeral account must name this controller at `initialize` |
| Flexible-mode controller assumed locked | `DEST_OTHER` sweep succeeds | No `dest_auth` event ⇒ no lock; deploy with `Some(destination)` |

## Related Documentation

- [`signed-sweep-walkthrough.md`](signed-sweep-walkthrough.md) — the positive-control sweep
- [`gas-free-claim-walkthrough.md`](gas-free-claim-walkthrough.md) — the claim path's lock semantics
- [`update-authorized-destination-flow.md`](update-authorized-destination-flow.md) — changing the lock while the nonce is 0
- [`event-topics.md`](../registry/event-topics.md) — `dest_auth` / `dest_upd` topic encoding
- [`error-handling.md`](error-handling.md) — `UnauthorizedDestination` (13) in the error map
- [`checking-account-status.md`](checking-account-status.md) — pre-flight probes for the account used here

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-10-15 | Initial locked-mode testing example |
