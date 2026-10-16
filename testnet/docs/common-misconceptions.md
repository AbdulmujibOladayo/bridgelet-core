# Common Misconceptions About Bridgelet on Testnet

## Purpose

A skimmable list of things new integrators commonly assume that are **not true**
of the current code and testnet setup. Each entry states the assumption, what is
actually the case, and where the detailed explanation lives.

Add to it when you find another. A misconception belongs here when it has been
formed by more than one person, or when acting on it is costly.

## Status

**Derived from the main `README.md`, the contract source and the existing
documents in this tree. Not observed on a live deployment.** No contracts are
recorded as deployed. Where two documents disagree, the entry says so rather
than picking a winner silently.

## Deployment and environment

| # | Misconception | What is actually true | Detail |
|---|---|---|---|
| 1 | "The contracts are deployed on testnet." | The status registry records all four as undeployed, with IDs `TBD`. The IDs in `deployments/testnet.json` are not corroborated and must be treated as unverified | [`../registry/contract-status.md`](../registry/contract-status.md) |
| 2 | "`account_factory` can be called on testnet." | It is not part of the supported deployment. It is not built by `scripts/build.sh`, not deployed by the script, and not tested in CI | [`overview.md`](overview.md) |
| 3 | "`reserve_contract` is deployed and in use." | It is not deployed by the tooling, and nothing in `EphemeralAccount` reads it | [`overview.md`](overview.md) |
| 4 | "Contract IDs are stable." | A redeploy mints new IDs for all four contracts and destroys every account. The SDF can also reset testnet without notice | [`../policies/reset-policy.md`](../policies/reset-policy.md) |
| 5 | "A merge to `main` redeploys testnet." | Nothing deploys automatically. The deploy workflow only validates the build, and its deploy steps are commented out | [`../../README.md`](../../README.md) |
| 6 | "The testnet passphrase ends in `November 2015`." | It is `Test SDF Network ; September 2015`. One document in this tree contains the wrong value | [`network-config.md`](network-config.md) |

## Signatures and sweeping

| # | Misconception | What is actually true | Detail |
|---|---|---|---|
| 7 | "`EphemeralAccount::sweep()` checks the signature I pass." | The `auth_signature` argument is ignored. The call is authorised by the stored controller's `require_auth()`, not by any signature. A successful direct `sweep()` proves nothing about a signature | [`../security/unverified-signature-path-warning.md`](../security/unverified-signature-path-warning.md) |
| 8 | "I should sweep with `EphemeralAccount::sweep()`." | Sweep through `SweepController::execute_sweep()` or `claim()`, which do real Ed25519 verification | [`../../README.md`](../../README.md) (MVP Status, Current Stub Inventory) |
| 9 | "A bad signature returns `SignatureVerificationFailed`." | That variant is never constructed. `ed25519_verify` **traps**, so there is no error code at all | [`../examples/error-handling.md`](../examples/error-handling.md) |
| 10 | "The nonce is per account." | There is one nonce for the whole controller, advanced by every successful sweep, including other integrators'. This is the usual cause of a correct signature failing | [`../config/nonce-tracking.md`](../config/nonce-tracking.md) |
| 11 | "A sweep moves the tokens." | The README says `sweep_account` does not move tokens and the SDK performs the payout. Some documents in this tree say the controller executes SEP-41 transfers. A search of the controller source finds none. Confirm against the deployed WASM before relying on either | [`../../README.md`](../../README.md) (MVP Status, Current Stub Inventory) |
| 12 | "I can change the authorised destination whenever I like." | `update_authorized_destination` is rejected once the controller's nonce is above zero | [`../examples/update-authorized-destination-flow.md`](../examples/update-authorized-destination-flow.md) |

## Payments and accounts

| # | Misconception | What is actually true | Detail |
|---|---|---|---|
| 13 | "`record_payment` receives or verifies a payment." | It stores an `(asset, amount)` entry. It neither checks that funds arrived nor moves any | [`../examples/error-handling.md`](../examples/error-handling.md) |
| 14 | "Only the controller can call `record_payment`." | The current source has no `require_auth()` on it, whatever other documents say. Do not build an access-control assumption on either claim | [`../examples/error-handling.md`](../examples/error-handling.md) |
| 15 | "A second payment of the same asset adds to the first." | One payment per asset. A repeat returns `DuplicateAsset` (13), not `PaymentAlreadyReceived` (3), which is never returned | [`../runbooks/multi-payment-edge-cases.md`](../runbooks/multi-payment-edge-cases.md) |
| 16 | "An account can hold any number of assets." | At most 10 distinct assets, then `TooManyPayments` | [`../runbooks/multi-payment-edge-cases.md`](../runbooks/multi-payment-edge-cases.md) |
| 17 | "Accounts expire on their own at `expiry_ledger`." | Nothing happens when the ledger passes. Someone must call `expire()` or `recover()`. A quiet account past its expiry is expected, not a fault | [`../examples/expiry-and-recovery.md`](../examples/expiry-and-recovery.md) |
| 18 | "`batch_initialize` tells me why an account failed." | It returns `success: false` and `error: None`. The reason is dropped. Confirm each failure with `get_info` | [`../runbooks/account-factory-batch-failure.md`](../runbooks/account-factory-batch-failure.md) |
| 19 | "Batch-created accounts are wired to my `SweepController`." | The factory sets the **creator** as both controller and admin. `initialize` is one-shot, so they are not wired to your controller | [`../../README.md`](../../README.md) (MVP Status, Current Stub Inventory) |

## Errors and events

| # | Misconception | What is actually true | Detail |
|---|---|---|---|
| 20 | "Error code `2` means one thing." | Codes are scoped to the contract. `2` is `NotInitialized`, `TransferFailed` or `ReserveNotSet` depending on which contract returned it | [`../examples/error-handling.md`](../examples/error-handling.md) |
| 21 | "`SweepController` has an error `12`." | The enum jumps from `11` to `13` (`UnauthorizedDestination`) | [`../examples/error-handling.md`](../examples/error-handling.md) |
| 22 | "Every payment emits `payment`." | Only the first. Later distinct assets emit `multi_pay` | [`../registry/event-topics.md`](../registry/event-topics.md) |
| 23 | "Listening to the account contract shows a whole sweep." | The sweep emits on both contracts. The controller's `sweep` event is on the other one | [`../examples/monitoring-a-single-account-lifecycle.md`](../examples/monitoring-a-single-account-lifecycle.md) |
| 24 | "`ReserveContract::set_base_reserve` changes sweep amounts." | `EphemeralAccount` uses its own hardcoded base reserve. Nothing reads the contract's value | [`../registry/reserve-contract-status.md`](../registry/reserve-contract-status.md) |

## How to use this list

If an assumption of yours appears here, follow the link before building on it. If
you were caught by something that is not listed, open an issue or add a row: a
good entry names the assumption, states the fact, and links the evidence.

## Related Documentation

- [`../../README.md`](../../README.md): the MVP stub inventory behind many of
  these rows
- [`faq.md`](faq.md): answers to questions rather than corrections
- [`overview.md`](overview.md): which contracts can be called
- [`../examples/error-handling.md`](../examples/error-handling.md): the error
  tables behind rows 9, 20 and 21
- [`webhook-vs-polling-tradeoffs.md`](webhook-vs-polling-tradeoffs.md): the
  event-delivery choices behind rows 22 and 23

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial list of common integrator misconceptions |
