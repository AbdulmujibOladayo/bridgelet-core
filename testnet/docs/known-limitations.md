# Known Limitations on Testnet

## Purpose

The main `README.md` carries a "Current Stub Inventory" under **MVP Status**.
It is the authoritative statement of what is and is not real in this codebase,
and it matters more on testnet than anywhere else, because testnet is where
integrators discover the gaps. But it sits in general project documentation.

This page extracts the **testnet-relevant subset** and adds the limitations that
only show up when working against the shared testnet deployment, so an
integrator sees them without reading the whole README.

It summarises; it does not replace the README and does not change any contract.
When this page and the README disagree, the README and the source win, and this
page should be corrected.

## Status

**Derived from the README and the contract source. Not observed on a live
deployment.** `testnet/registry/contract-status.md` records all four contracts as
undeployed (contract IDs `TBD`, `Last Verified: Never`). Treat every row below
as a statement about the code, and check it against a live deployment before
depending on it.

## At a glance

| # | Limitation | Affects | Severity for integrators |
|---|---|---|---|
| 1 | `EphemeralAccount::sweep()` does not verify a signature | Anyone calling `sweep()` directly | High |
| 2 | `AccountFactory::batch_initialize` drops per-account error detail | Batch account creation | Medium |
| 3 | `SweepController` does not move tokens in the current source | Anyone expecting funds to arrive | High |
| 4 | `record_payment` records a payment; it does not verify or transfer one | Payment recording | High |
| 5 | `ReserveContract` is not wired into `EphemeralAccount` | Reserve behaviour | Low |
| 6 | Only two of four contracts are in the supported deployment tooling | Choosing what to call | Medium |
| 7 | The sweep nonce is global to the controller | Concurrent sweeps | Medium |
| 8 | Several documented error codes can never be returned | Error handling | Medium |
| 9 | CI does not deploy or fully test | Trusting a deployment | Medium |
| 10 | State is destroyed on a redeploy or an SDF reset | Everything stored by ID | High |

## 1. `EphemeralAccount::sweep()` does not verify a signature

`verify_sweep_authorization` ignores its `auth_signature` argument. What
authorises the call is `authorized_controller.require_auth()`, so it trusts
whichever address was stored as the controller at `initialize()`. Passing a
garbage signature is not rejected; it is ignored.

**Do this instead:** route sweeps through `SweepController::execute_sweep()`
(signed) or `SweepController::claim()` (recipient-signed, relayer-submitted).
`SweepController::verify_sweep_auth` performs a real Ed25519 verification over
`SHA256(destination || nonce || controller_id)`.

A successful direct `sweep()` is **not** evidence that any signature was
checked. Details: [`testnet/security/unverified-signature-path-warning.md`](../security/unverified-signature-path-warning.md).

## 2. `AccountFactory::batch_initialize` drops error detail

On a per-account failure the factory returns `success: false` with
`error: None`, in both `batch_initialize` and `batch_initialize_paginated`. You
learn **that** an account failed, never **why**.

**Do this instead:** after a batch, call `get_info` on each failed address. If it
returns data, initialisation actually succeeded. If it reports `NotInitialized`,
it genuinely failed and the reason has to be diagnosed out of band. See
[`testnet/runbooks/account-factory-batch-failure.md`](../runbooks/account-factory-batch-failure.md).

Also note that `batch_initialize` passes the **creator** as both
`authorized_controller` and `admin` for every account it creates, so accounts
from this path are not wired to your `SweepController`. `initialize` is one-shot.

## 3. `SweepController` does not move tokens in the current source

The README states that `sweep_account` "does **not** move tokens: the SDK
performs the custodial payout via Horizon", and `docs/security.md` lists token
transfers as not yet integrated. The contracts currently manage **state and
authorisation**, not funds.

**Documents disagree.** Other pages in this tree describe `SweepController`
executing SEP-41 `TokenClient::transfer()` calls (for example the comparison
table in the unverified-signature warning). A search of
`contracts/sweep_controller/src/lib.rs` for a token transfer finds none. Until
that is reconciled, assume a successful sweep **marks the account swept and emits
`SweepCompleted` without moving the underlying asset**, and confirm against the
deployed WASM.

## 4. `record_payment` is a record, not a payment

`record_payment` stores an amount and an asset address. It does not check that
the asset arrived and does not transfer anything. It also has no `require_auth()`
in the current source, so do not rely on the statement elsewhere that only the
controller may call it. Do not build an access-control assumption on either
statement; verify against the deployed WASM.

Each account holds at most **10 distinct assets**, and one payment per asset.

## 5. `ReserveContract` is not wired into `EphemeralAccount`

`EphemeralAccount` uses its own `BASE_RESERVE_STROOPS` constant and its own
reserve bookkeeping. `ReserveContract` stores an admin-settable base reserve that
nothing reads. Calling `ReserveContract::set_base_reserve` does **not** change
sweep behaviour today.

## 6. Only two contracts are in the supported deployment

The deploy tooling covers `ephemeral_account` and `sweep_controller`.
`reserve_contract` and `account_factory` are not deployed by it, and
`account_factory` is not built by `scripts/build.sh` or tested in CI. Do not
build on the other two on testnet until the tooling covers them. Per-contract
status: [`overview.md`](overview.md).

## 7. The sweep nonce is global

`SweepController` keeps **one** nonce for the whole controller deployment, and
every successful sweep advances it, including other integrators' sweeps. On a
shared testnet this is the most common reason a correctly-built signature fails.
A rejected signature is also a **trap**, not an error code. Read `get_nonce()`
immediately before signing. See
[`testnet/config/nonce-tracking.md`](../config/nonce-tracking.md).

## 8. Some documented error codes never fire

Six of the twelve `SweepController` variants and two `EphemeralAccount` variants
are never constructed by the current source. The README and API reference list
`SignatureVerificationFailed` as the symptom of a bad signature; it is not, the
real symptom is a trap. The enum also skips code `12`. See
[`testnet/examples/error-handling.md`](../examples/error-handling.md).

## 9. CI does not deploy or fully test

`.github/workflows/test.yml` is entirely commented out, and
`deploy-testnet.yml` validates the build but its deploy, upload and summary
steps are commented out. Nothing runs automatically on push, and nothing deploys
automatically. A merge to `main` therefore does not change testnet.

## 10. State does not survive a redeploy or a reset

A redeploy changes every contract ID and destroys every account, recorded
payment and nonce. The Stellar Development Foundation can also reset testnet
itself without notice. Do not store anything you would miss in a testnet
ephemeral account. See
[`testnet/policies/reset-policy.md`](../policies/reset-policy.md).

## Limits that are working as designed

These are constraints, not defects, but they surprise people:

| Constraint | Value |
|---|---|
| Distinct assets per account | 10 |
| Payments per asset | 1 |
| Account lifetime | Fixed at `initialize()` by `expiry_ledger`; sweeping after expiry fails with `AccountExpired` |
| Destination changes | `update_authorized_destination` is rejected once the nonce is above zero |

## What would change this page

A live deployment that has been verified, a decision on token movement, a fix to
the signature path, and error detail returned by the factory would each remove a
row. Update this page in the same change as the fix.

## Related Documentation

- [`overview.md`](overview.md): which contracts are deployed and callable
- [`../../README.md`](../../README.md): the stub inventory this page summarises
- [`../../docs/security.md`](../../docs/security.md): the design-level known limitations
- [`../security/unverified-signature-path-warning.md`](../security/unverified-signature-path-warning.md):
  limitation 1 in detail
- [`../examples/error-handling.md`](../examples/error-handling.md): limitation 8
  in detail
- [`../registry/contract-status.md`](../registry/contract-status.md): the
  deployment status all of this depends on
- [`../policies/reset-policy.md`](../policies/reset-policy.md): limitation 10

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial testnet-specific summary of known limitations |
