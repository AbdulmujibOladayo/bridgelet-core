# Recommended Idempotency-Key Handling for Consuming Applications

## Purpose

`SweepController` protects sweeps against replay with an on-chain nonce (see
[`../examples/idempotency-considerations.md`](../examples/idempotency-considerations.md)).
The other write paths have **no equivalent on-chain replay protection**: creating
an account and recording a payment can each be submitted twice, and the contract
will not always tell you that the second submission was a duplicate.

This document recommends a pattern for **your application's own idempotency
keys** around those calls. It is distinct from the nonce: the nonce answers
"may this signed sweep run again", while an idempotency key answers "did my
system already ask for this operation".

## Status

**Reasoned from source; not observed.** No contracts are recorded as deployed
(`testnet/registry/contract-status.md`), so none of the failure behaviour below
has been watched on a live deployment. Every claim cites the code that supports
it, and those that depend on network behaviour are marked as needing
verification.

## Scope

| In scope | Out of scope |
|---|---|
| Account creation (`AccountFactory`, `EphemeralAccount::initialize`) | Sweep replay protection (the nonce covers it) |
| Payment recording (`EphemeralAccount::record_payment`) | Your HTTP framework's idempotency middleware |
| A key lifecycle and storage schema | Changing any contract |

## What the contracts give you, and what they do not

| Operation | On-chain protection | Gap |
|---|---|---|
| `EphemeralAccount::initialize` | `AlreadyInitialized` (code 1) on a second call | Rejects *any* second call, even one with identical arguments. A duplicate is indistinguishable from a conflict |
| `EphemeralAccount::record_payment` | `DuplicateAsset` (13) for an asset that already has a payment; `TooManyPayments` (14) at 10 assets | Protection is **per asset**, not per request. A second call with a different amount is rejected, but you are not told whether the first amount matches yours |
| `AccountFactory::batch_initialize` | None dedicated. Each account address is derived from the factory and a salt equal to the **index in the batch** | The salt does not depend on the creator or the request contents. Resubmitting a batch re-derives the same addresses |
| Network level | A transaction is identified by its hash and consumes one source-account sequence number | Resubmitting the *identical signed transaction* is the safe retry. Building a *new* transaction for the same intent is not |

The last row is the foundation of the pattern below: the safe unit of retry is a
specific signed transaction, not "the operation".

## The pattern

Keep a record per operation, keyed by a client-supplied idempotency key, and make
the record, not the contract, the source of truth for "did we already do this".

### 1. Accept or generate a key

Require callers to send an `Idempotency-Key` header (a UUID is enough). The key
identifies one **business intent**, for example "create the account for order
4521". A retry of that intent reuses the key; a different intent gets a new one.

### 2. Fingerprint the request

Store a hash of the request fields that matter (creator, expiry, asset, amount).
If the same key arrives with a different fingerprint, reject it with `422`: the
caller reused a key for a different operation, which is a bug on their side.

### 3. Record before you submit

Write the record **before** sending anything to the network, in state `PENDING`,
then update it as you go:

```text
PENDING   -> built, not yet submitted
SUBMITTED -> signed transaction sent, hash stored
CONFIRMED -> transaction succeeded on chain
FAILED    -> transaction failed with a known contract error
```

Store the transaction hash and the signed envelope when you reach `SUBMITTED`.
Without them you cannot tell, after a crash, whether the network ever saw the
request.

### 4. On a retry, look up first

| Record state | Action |
|---|---|
| None | Create it and proceed |
| `PENDING` | The previous attempt never submitted. Rebuild and submit |
| `SUBMITTED` | **Do not build a new transaction.** Look up the stored hash. If it is final, update the record and return that result. If it is still unknown, resubmit the *same* signed envelope |
| `CONFIRMED` / `FAILED` | Return the stored result without touching the network |

### 5. Reconcile against the chain for unknown outcomes

When a submission times out, the outcome is unknown. Resolve it by reading
state, not by guessing:

| Path | Read | Conclusion |
|---|---|---|
| Account creation | `get_info` on the expected address | `Ok` means it was initialised. `NotInitialized` means it was not |
| Payment recording | `get_info`, and check `payments` for the asset | Present with your amount: done. Present with another amount: conflict. Absent: not recorded |

These are read-only calls. They are what make a retry safe when the contract has
no replay protection of its own.

## Path-specific guidance

### Account creation

- **Single account.** Treat `AlreadyInitialized` as "may be your own earlier
  success". Read `get_info` and compare `creator`, `expiry_ledger` and
  `recovery_address` with the request before reporting a conflict.
- **Batch.** Because the salt is only the batch index, a batch resubmitted from
  index `0` targets the same addresses as the first attempt. Do not rely on the
  factory to deduplicate. Persist the batch result per index, and when
  retrying, resubmit only the indices that were not confirmed, using the
  paginated variant with a stored cursor where the batch is large.
- **Failures carry no reason.** The factory sets `error: None` on every result,
  so a failed item must be confirmed with `get_info` before you mark it failed.
- **Distinct creators share the index space.** Two callers each submitting a
  batch start from the same salt values. Serialise batches per factory in your
  service, or key them so two cannot run at once. Confirm this against a live
  deployment before relying on either behaviour.

### Payment recording

- **One recorded payment per asset.** Use `(account, asset)` as part of the
  idempotency key's scope: a second request for the same pair is either a retry
  or a conflict, and the stored fingerprint tells you which.
- **Treat `DuplicateAsset` as success only after a check.** Read the recorded
  payment. If the amount matches the stored request, mark the operation
  `CONFIRMED`. If not, report a conflict.
- **`record_payment` does not move funds.** It records a payment; it does not
  verify or transfer the underlying asset (`error-handling.md`). Your key
  protects the *record*, and any real transfer needs its own protection.
- **Respect the cap.** `TooManyPayments` is permanent for that account. Do not
  retry it.

## Storage schema

```sql
CREATE TABLE idempotency_records (
  idempotency_key  TEXT PRIMARY KEY,
  operation        TEXT NOT NULL,        -- 'create_account' | 'record_payment'
  fingerprint      TEXT NOT NULL,        -- hash of the request fields
  state            TEXT NOT NULL,        -- PENDING | SUBMITTED | CONFIRMED | FAILED
  tx_hash          TEXT,                 -- set at SUBMITTED
  signed_envelope  TEXT,                 -- set at SUBMITTED, for exact resubmission
  result           JSONB,                -- final response body
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Never store a secret key or signing seed in this table. The signed envelope is
public once submitted, but it is still data to protect.

### Retention

Keep records at least as long as a client could plausibly retry, and for as long
as the account can still be reconciled. Testnet resets delete on-chain state
(see `testnet/policies/reset-policy.md`), so a record that points at a deleted
contract is history, not a live reference. Do not "retry" against it.

## Common mistakes

| Mistake | Why it hurts |
|---|---|
| Building a new transaction on retry | A new envelope is a new submission; the original may still land |
| Treating `DuplicateAsset` or `AlreadyInitialized` as always success | They also fire for a conflicting request |
| Keying only on the idempotency header, not the fingerprint | A reused key silently returns the wrong result |
| Using the batch index as a stable identity | It is a position, not an identity |
| Deleting records on failure | The next retry has nothing to reconcile against |

## Undecided

| Question | State |
|---|---|
| Whether `AccountFactory` should derive salts from request contents | Contract design; out of scope here |
| How long a network keeps a transaction's final status queryable | Needs verification against a live deployment |
| Whether concurrent batches from different creators really collide | Reasoned from the salt derivation; not observed |

## Related Documentation

- [`../examples/idempotency-considerations.md`](../examples/idempotency-considerations.md):
  the sweep case, covered by the on-chain nonce
- [`error-code-to-http-status-mapping.md`](error-code-to-http-status-mapping.md):
  status codes for the duplicate and conflict outcomes
- [`../examples/error-handling.md`](../examples/error-handling.md): what
  `DuplicateAsset`, `AlreadyInitialized` and the factory's dropped errors mean
- [`../runbooks/account-factory-batch-failure.md`](../runbooks/account-factory-batch-failure.md):
  diagnosing partial batch failures
- [`../config/nonce-tracking.md`](../config/nonce-tracking.md): the on-chain
  nonce this document is distinct from
- [`../policies/reset-policy.md`](../policies/reset-policy.md): why stored
  references can be invalidated

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial idempotency-key recommendation for account creation and payment recording |
