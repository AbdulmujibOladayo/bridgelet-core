# Suggested HTTP Status Mapping for Contract Errors

## Purpose

Teams that wrap Bridgelet contract calls behind their own REST API all face the
same question: when `EphemeralAccount` returns `Error(Contract, #11)`, what does
the HTTP client see? Left undecided, every team invents a different answer, and
the same condition becomes `400` in one service, `409` in another and `500` in a
third.

This document proposes **one starting convention**. It maps each contract error
condition to an HTTP status, a stable machine-readable error code for your API,
and whether a client may retry. It is a suggestion, not a standard: change it
where your API has different conventions, but change it deliberately and in one
place.

It is the HTTP companion to
[`../examples/error-handling.md`](../examples/error-handling.md), which owns the
question "what should the integrator do about this error". This document does
not repeat that reasoning; it only assigns a status to each outcome.

## Status

**Derived from source; unverified against a live deployment.** All four
contracts are recorded as undeployed in `testnet/registry/contract-status.md`.
Which variants can actually fire is taken from the reachability analysis in
`error-handling.md`, which was read from the contract bodies, not observed.

## Scope

| In scope | Out of scope |
|---|---|
| Status codes for each reachable contract error | Your API's authentication (`401`) and rate limiting (`429`) |
| A stable error body shape | A specific web framework or SDK |
| Failures that are not contract errors (traps, auth, RPC) | Changing any contract error code |

## Principles

1. **Key on the pair, never on the integer alone.** The same number means
   different things in different contracts (`2` is `NotInitialized`,
   `TransferFailed` or `ReserveNotSet`). Your mapping table is keyed by
   `(contract, code)`.
2. **4xx means the caller can change the outcome; 5xx means they cannot.**
   A condition that only an operator can fix (a controller with no signer) is a
   5xx even though the contract reports it as an ordinary error.
3. **Use `409` for "the resource is in the wrong state".** Most Bridgelet errors
   are state conflicts: already swept, not yet expired, already initialized.
4. **Never leak the raw host string.** `HostError: Error(Contract, #7)` is not
   an API response. Return your own code and a human message.
5. **A read of an expired or swept account is still `200`.** The status codes
   below describe *actions* (sweep, claim, record a payment), not reads. A
   `GET` on an expired account succeeds and reports `status: "Expired"`.

## Error body

```json
{
  "error": {
    "code": "ACCOUNT_EXPIRED",
    "message": "This account has expired and can no longer be swept.",
    "contract": "EphemeralAccount",
    "contractCode": 11,
    "retryable": false
  }
}
```

`contract` and `contractCode` are for debugging and support. Clients should
branch on `code`, which you control, and not on `contractCode`.

## `EphemeralAccount`

Reachability follows `error-handling.md`. Codes marked *unreachable* are never
constructed by the current source.

| Code | Variant | HTTP | Your API `code` | Retryable | Notes |
|---:|---|---:|---|---|---|
| 1 | `AlreadyInitialized` | 409 | `ACCOUNT_ALREADY_INITIALIZED` | No | Deploy a fresh instance |
| 2 | `NotInitialized` | 404 | `ACCOUNT_NOT_FOUND` | No | The ID does not resolve to a usable account |
| 3 | `PaymentAlreadyReceived` | n/a | n/a | n/a | *Unreachable.* If seen, return 500 |
| 4 | `InvalidAmount` | 422 | `INVALID_AMOUNT` | No | Fix the argument |
| 5 | `InvalidExpiry` | 422 | `INVALID_EXPIRY` | Yes, after recomputing | Re-read the ledger and add a margin |
| 6 | `NotExpired` | 409 | `ACCOUNT_NOT_EXPIRED` | Later | Poll `is_expired`, do not loop |
| 7 | `AlreadySwept` | 409 | `ACCOUNT_ALREADY_SWEPT` | No | Expected result of a duplicate submission |
| 8 | `Unauthorized` | 403 | `FORBIDDEN` | No | Wrong caller, not a missing login |
| 9 | `InvalidSignature` | n/a | n/a | n/a | *Unreachable.* If seen, return 500 |
| 10 | `NoPaymentReceived` | 409 | `NO_PAYMENT_RECORDED` | After recording | Precondition for a sweep |
| 11 | `AccountExpired` | 410 | `ACCOUNT_EXPIRED` | No | Route to recovery |
| 12 | `InvalidStatus` | 409 | `INVALID_ACCOUNT_STATUS` | No | A one-shot transition was already used |
| 13 | `DuplicateAsset` | 409 | `PAYMENT_ALREADY_RECORDED` | No | See the note on idempotency below |
| 14 | `TooManyPayments` | 409 | `PAYMENT_LIMIT_REACHED` | No | The cap is 10 distinct assets |
| 15 | `NotUpgradeAdmin` | 500 | `INTERNAL_ERROR` | No | Operational problem, alert |

`410 Gone` for `AccountExpired` is deliberate: the account will never become
sweepable again, which is what `410` communicates and `409` does not. Use `410`
only on the action that failed, never on a read.

`DuplicateAsset` is `409` here, but if your API treats recording the same
payment twice as a harmless retry, translate it to `200` after confirming the
recorded amount matches. See
[`idempotency-key-recommendation.md`](idempotency-key-recommendation.md).

## `SweepController`

| Code | Variant | HTTP | Your API `code` | Retryable | Notes |
|---:|---|---:|---|---|---|
| 1 | `InvalidAccount` | n/a | n/a | n/a | *Unreachable.* If seen, return 500 |
| 2 | `TransferFailed` | 502 | `TRANSFER_FAILED` | Once, after re-reading state | The underlying cause is discarded by the contract |
| 3 | `AuthorizationFailed` | 409 | `CONTROLLER_STATE_CONFLICT` | No | Already configured, or never initialised |
| 4 | `InsufficientBalance` | n/a | n/a | n/a | *Unreachable* |
| 5 | `AccountNotReady` | 409 | `ACCOUNT_NOT_READY` | After re-reading `get_info` | May mean the sweep actually completed |
| 6 | `AccountExpired` | n/a | n/a | n/a | *Unreachable* here; expect `EphemeralAccount` #11 |
| 7 | `AccountAlreadySwept` | 409 | `DESTINATION_LOCKED` | No | The nonce is above zero, so the destination is fixed |
| 8 | `InvalidSignature` | n/a | n/a | n/a | *Unreachable* |
| 9 | `SignatureVerificationFailed` | n/a | n/a | n/a | *Unreachable*; a bad signature traps instead |
| 10 | `AuthorizedSignerNotSet` | 503 | `SERVICE_NOT_CONFIGURED` | No | An operator must initialise the controller |
| 11 | `InvalidNonce` | n/a | n/a | n/a | *Unreachable* |
| 13 | `UnauthorizedDestination` | 403 | `DESTINATION_NOT_AUTHORIZED` | No | Re-sign for the correct destination |

There is no code `12`. The enum jumps from `11` to `13`.

## `ReserveContract`

| Code | Variant | HTTP | Your API `code` | Retryable | Notes |
|---:|---|---:|---|---|---|
| 1 | `InvalidAmount` | 422 | `INVALID_AMOUNT` | No | The value is in stroops |
| 2 | `ReserveNotSet` | 503 | `SERVICE_NOT_CONFIGURED` | No | Prefer the `Option`-returning getter |
| 3 | `Unauthorized` | n/a | n/a | n/a | *Unreachable* |
| 4 | `AlreadyInitialized` | 409 | `ALREADY_INITIALIZED` | No | |
| 5 | `NotInitialized` | 503 | `SERVICE_NOT_CONFIGURED` | No | An operator must initialise it |
| 6 | `AmountTooLarge` | 422 | `AMOUNT_TOO_LARGE` | No | Check units: stroops, not XLM |

## `AccountFactory`

The factory defines no error enum. `batch_initialize` returns a result per
account with `success: true | false` and no reason (the contract always sets
`error: None`).

| Outcome | HTTP | Notes |
|---|---:|---|
| Every item succeeded | 200 | |
| Some items failed | 207 | Report each item; confirm each failure with `get_info` before reporting it |
| The call itself failed (RPC, auth) | see the next table | |

Do not invent a reason for a failed item. Report that it failed and that the
cause is not available from the factory.

## Failures that are not contract errors

| Failure | HTTP | Your API `code` | Retryable | Notes |
|---|---:|---|---|---|
| `Error(Auth, InvalidAction)` | 403 | `FORBIDDEN` | No | The signer did not authorise the call |
| `ed25519_verify` trap on a sweep | 409 | `SIGNATURE_REJECTED` | Once, after re-reading the nonce | Most often a stale nonce, so confirm with `get_nonce` before blaming the signer |
| Network passphrase mismatch | 500 | `INTERNAL_ERROR` | No | Your configuration, not the caller's |
| RPC unreachable or unhealthy | 503 | `UPSTREAM_UNAVAILABLE` | Yes, with `Retry-After` | Check `getHealth` first |
| Submitted, outcome unknown (timeout) | 202 | `PENDING` | Poll by transaction hash | Do not resubmit blindly |
| `Error(WasmVm, InvalidAction)` | 500 | `INTERNAL_ERROR` | No | A build problem; escalate |

A timed-out submission is the case that most needs care. Return `202` with the
transaction hash and let the client poll; resubmitting can double-apply. The
reasoning is in `idempotency-key-recommendation.md`.

## Implementation sketch

Keep the whole table in one data structure so the convention lives in one place.

```javascript
const HTTP_BY_ERROR = {
  'EphemeralAccount:11': { status: 410, code: 'ACCOUNT_EXPIRED', retryable: false },
  'EphemeralAccount:7': { status: 409, code: 'ACCOUNT_ALREADY_SWEPT', retryable: false },
  'EphemeralAccount:8': { status: 403, code: 'FORBIDDEN', retryable: false },
  // ...one row per reachable (contract, code) pair from the tables above
};

function toHttpError(contract, contractCode) {
  const entry = HTTP_BY_ERROR[`${contract}:${contractCode}`];
  if (!entry) {
    // Unreachable or unknown codes are bugs, not caller errors.
    return { status: 500, code: 'INTERNAL_ERROR', retryable: false };
  }
  return entry;
}
```

Log the raw host error server-side with the `(contract, code)` pair so an
unexpected `500` can be traced.

## Undecided

| Question | State |
|---|---|
| `409` or `422` for state-dependent preconditions such as `NoPaymentReceived` | `409` proposed; either is defensible |
| Whether `AccountExpired` should be `410` or `409` | `410` proposed, following the issue that asked for this document |
| Whether the factory's silent per-item failures deserve their own code | Not proposed; the contract gives no reason to report |

## Related Documentation

- [`../examples/error-handling.md`](../examples/error-handling.md): what each
  error means and what to do about it
- [`idempotency-key-recommendation.md`](idempotency-key-recommendation.md): the
  retry rules behind the `202` and `409` choices
- [`type-mapping.md`](type-mapping.md): Soroban types in common SDKs
- [`../runbooks/account-factory-batch-failure.md`](../runbooks/account-factory-batch-failure.md):
  diagnosing a failed batch item
- [`../registry/contract-status.md`](../registry/contract-status.md): the
  deployment status this analysis depends on

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial suggested HTTP status mapping for contract errors |
