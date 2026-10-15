# Local Mock or Live Testnet? Choosing How to Test Your Integration

## Purpose

There is no ready-made local entry point for exercising Bridgelet's contracts
from an integrator's own code. A team writing an integration has to decide
between **building a local mock of the contracts' behaviour** (fast, offline) and
**always testing against the live testnet deployment** (faithful, slow, shared).

This page lays out the trade-off so the choice is made on purpose. It also says
what a mock must reproduce to be worth having, because several contract
behaviours are surprising enough that a naive mock would make tests pass that
fail for real.

## Status

**Reasoned from the contract source and existing documents; not observed.** No
contract is recorded as deployed (`../registry/contract-status.md`), so the
live-testnet side of the comparison is described from the harness notes and not
from experience.

## Scope

This is about testing **your application's** use of the contracts. Testing the
contracts themselves is a separate job, done in-process with `cargo test`.

| Layer | Tool | Who it is for |
|---|---|---|
| Contract logic | `cargo test` with the Soroban test environment (`scripts/test.sh`) | People changing the contracts |
| Your code against a mock of the contracts | A mock you write | Integrators, for fast feedback |
| Your code against a real deployment | Live testnet | Integrators, for fidelity |

The repository's own `docs/testing.md` also describes starting a local sandbox
container. Another document in this tree says the repository has no local
sandbox. These disagree and neither has been verified here, so this page does
not depend on a sandbox existing.

## The trade-off

| | Local mock | Live testnet |
|---|---|---|
| Speed | Milliseconds | 5 to 15 seconds per state-changing step |
| Determinism | Total | Shared state, other users, network variance |
| Works offline | Yes | No |
| Cost and limits | None | Friendbot and RPC rate limits |
| Fidelity | Only as good as the mock | The real thing |
| Catches contract drift | **No**, unless you update the mock | Yes |
| Catches nonce contention | No | **Only here** |
| Catches wrong network config | No | Yes |
| Maintenance | You own the mock | None, but the environment changes under you |
| Failure cause is clear | Yes | Often not (could be someone else's activity) |

Neither dominates. A mock buys speed and determinism and pays for it in
fidelity; live testnet is the reverse.

## What a mock must get right

The danger of a mock is not that it is simple, it is that it is **plausible and
wrong**. These behaviours are the ones a straightforward implementation gets
wrong. Each is taken from the source and the existing error documentation.

| Behaviour | A naive mock does | The contracts actually do |
|---|---|---|
| A bad sweep signature | Returns a named error | **Traps** with no error code (`ed25519_verify` aborts) |
| `EphemeralAccount::sweep()` signature | Validates it | **Ignores it**; authorisation is the stored controller's `require_auth()` |
| Nonce | One per account | **One per controller**, advanced by every successful sweep |
| Error codes | Globally unique | Scoped to the contract; the same number means different things |
| Second payment, same asset | Adds to the first | Rejected with `DuplicateAsset` (13) |
| Payments per account | Unlimited | At most 10 distinct assets, then `TooManyPayments` |
| `record_payment` | Moves or checks funds | Only stores an entry |
| Expiry | Expires on its own | Nothing happens at `expiry_ledger`; someone must call `expire()` |
| `batch_initialize` failure | Returns a reason | Returns `success: false` and no reason |
| Destination change | Always allowed | Rejected once the nonce is above zero |
| Time | Wall clock | Ledger sequence; roughly five seconds per ledger |

A mock that gets these right is a small state machine. One that does not will
teach your code to handle errors that never occur and miss the ones that do.

### A minimal shape

```typescript
type Status = 'Active' | 'PaymentReceived' | 'Swept' | 'Expired';

class MockEphemeralAccount {
  status: Status = 'Active';
  payments = new Map<string, bigint>(); // one entry per asset, max 10

  constructor(readonly expiryLedger: number, private clock: { ledger: number }) {}

  recordPayment(asset: string, amount: bigint) {
    if (amount <= 0n) throw contractError('EphemeralAccount', 4);   // InvalidAmount
    if (this.payments.has(asset)) throw contractError('EphemeralAccount', 13); // DuplicateAsset
    if (this.payments.size >= 10) throw contractError('EphemeralAccount', 14); // TooManyPayments
    this.payments.set(asset, amount);
    if (this.status === 'Active') this.status = 'PaymentReceived';
  }

  isExpired() {
    return this.clock.ledger > this.expiryLedger; // true does NOT change status
  }
}
```

`contractError` should carry the `(contract, code)` pair, never a bare integer.
Model a rejected signature as a thrown trap that has no code, so your error
handling is forced to deal with it.

## Keeping a mock honest

A mock drifts the moment a contract changes. Three habits keep it useful:

1. **Run the same scenarios against both.** Write each test as a scenario that
   can target the mock or the live deployment. When a scenario passes on the mock
   and fails live, the mock is wrong and is corrected.
2. **Pin what the mock was written against.** Record the contract version and
   WASM hash it models. See
   [`testing-against-upgraded-contracts.md`](testing-against-upgraded-contracts.md).
3. **Keep it thin.** Mock the contract's state machine and error codes, not the
   network. The more of Stellar you reimplement, the more there is to get wrong.

## What never to mock

| Thing | Why |
|---|---|
| The sweep signature bytes | The digest is `SHA256(destination || nonce || controller_id)` and the encoding is easy to get subtly wrong. Test signing against the real signer tool |
| The global nonce under contention | The behaviour is a property of the shared deployment |
| Network passphrase and endpoint config | A wrong value fails only against the real network |

## A practical split

For most teams a hybrid beats either extreme:

| Suite | Runs | Target | Purpose |
|---|---|---|---|
| Unit and component tests | Every commit | Mock | Fast feedback on your logic and error handling |
| Contract-parity scenarios | On demand, and before releases | Mock **and** live | Detect mock drift |
| Smoke test | Nightly or before release | Live testnet | Prove the deployment is usable |

Keep the live suite **small**. It exists to confirm that the shared deployment
works and that your configuration is right, not to re-test your logic slowly.
The harness design for that suite is in
[`../examples/integration-test-harness-notes.md`](../examples/integration-test-harness-notes.md).

## When to choose which

| Situation | Lean toward |
|---|---|
| Early development, changing code constantly | Mock |
| Testing error handling for every contract error | Mock (you can force each one) |
| Verifying you can actually create, pay and sweep | Live testnet |
| CI that must be fast and never flaky | Mock for gating, live as a non-blocking check |
| Anything involving signatures or nonces | Live testnet |
| No contracts deployed yet | Mock is the only option |

The last row is the current situation: with no deployment recorded, a mock is
the only thing available, which makes keeping it honest more important, not
less.

## Undecided

| Question | State |
|---|---|
| Whether this project should publish a reference mock | Not proposed; `../tooling/` holds specifications only |
| Whether a local sandbox container is a supported option | `docs/testing.md` describes one; its status is unverified here |
| How closely a mock should model ledger and fee behaviour | Left to the integrator |

## Related Documentation

- [`../examples/integration-test-harness-notes.md`](../examples/integration-test-harness-notes.md):
  designing a live-testnet suite
- [`testing-against-upgraded-contracts.md`](testing-against-upgraded-contracts.md):
  pinning what a mock or a suite was written against
- [`../examples/error-handling.md`](../examples/error-handling.md): the error
  behaviour a mock must reproduce
- [`../examples/idempotency-considerations.md`](../examples/idempotency-considerations.md):
  nonce and replay behaviour
- [`../../docs/testing.md`](../../docs/testing.md): testing the contracts
  themselves
- [`../registry/contract-status.md`](../registry/contract-status.md): whether
  there is anything live to test against

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial trade-off guide for mock versus live-testnet testing |
