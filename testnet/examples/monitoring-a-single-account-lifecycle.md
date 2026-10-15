# Worked Example: Monitoring a Single Account's Lifecycle via Events

## Overview

This example subscribes to the **complete event sequence** for one
`EphemeralAccount` from creation to its terminal state, across both contracts
that emit events for it, and merges those streams into one correctly ordered
history. It is written as a template for a status-tracking dashboard: the
ordering rules and the state machine below are the reusable part; the
transport (Horizon stream, RPC polling, CLI) is interchangeable.

Two properties make this less trivial than "listen for events":

1. **The lifecycle spans two contracts.** The account emits `created`,
   `payment`/`multi_pay`, `swept_mul`/`expired`, and `reserve`; the
   `SweepController` emits the correlated `sweep`. A single-contract
   subscription silently misses the controller half of a sweep.
2. **Events are append-only facts, not state.** The contract's stored state
   (`get_info`) is the authority; the event stream is the most efficient way
   to *observe* transitions, and the two must be reconcilable at any point.

Topic symbols, payload structs, and trigger conditions are specified in
[`event-topics.md`](../registry/event-topics.md); this document assumes that
table and does not duplicate it.

---

## The Two Canonical Sequences

An account's life ends in exactly one of two terminal states, and the event
order within each is fixed by the contract code:

```text
Sweep path (terminal status 2, Swept):
  created ──► payment ──► [multi_pay …] ──► swept_mul ──► reserve
                                                 │
  controller (other contract):                   └──► sweep
  terminal event pair, same workflow

Expiry path (terminal status 3, Expired):
  created ──► [payment …] ──► expired ──► reserve
```

Rules that fall out of `contracts/ephemeral_account/src/lib.rs` and
`contracts/sweep_controller/src/lib.rs`:

| Rule | Consequence for a collector |
|---|---|
| `created` is emitted exactly once, by `initialize()` | The account's birth is unambiguous; there is no re-emission after upgrades |
| `payment` is emitted only for the **first** `record_payment`; every later distinct asset emits `multi_pay` | An account with one asset never emits `multi_pay`; neither sequence is wrong |
| Max 10 recorded assets (`TooManyPayments`) | `payment` + at most 9 `multi_pay` events |
| `swept_mul` and `expired` are mutually exclusive — the state machine permits only one terminal transition | A stream containing both means you are merging two different accounts, or a contract was redeployed at the same ID (verify per [`verify-contract-live.md`](../registry/verify-contract-live.md)) |
| `reserve` is emitted after the terminal transition (`sweep`/`sweep_claim`/`expire`/`recover`), as part of the same transaction | It always sorts after the terminal event within its ledger |
| **Expiry is not automatic.** Crossing `expiry_ledger` emits nothing; `expire()` or `recover()` must be called | A quiet account past its expiry ledger is *expected* — absence of `expired` is not a fault |
| The controller's `sweep` is emitted by `execute_sweep` **or** `claim`, after the account-side state transition succeeds | The sweep path always produces events on both contracts; query both |

---

## Ordering

Order by the tuple `(ledger_sequence, tx_index, event_index)` — never by
arrival time at your collector, and never by wall-clock.

- **Within one contract**, Horizon's `/events` returns records already in
  that order and assigns each a `paging_token`; persist that token as your
  resume cursor.
- **Across the two contracts** there is no shared cursor. Merge by ledger
  sequence first. Within the same ledger, the account-side `swept_mul`/`expired`
  and the controller-side `sweep` belong to the same transaction, so their
  relative order is observable from the transaction's operation/return
  order — but for dashboard purposes treat them as a *simultaneous pair*
  and key your derived state off the account-side event.
- Ledger sequence and `ReserveReclaimed.sweep_id` are **ledger values**, not
  timestamps (see [`event-watchlist.md`](../monitoring/event-watchlist.md)).
  If your dashboard shows wall-clock time, store the ledger-to-time mapping
  separately.

---

## Step 1: Pick a start ledger before the workflow runs

```bash
export NETWORK=testnet
export RPC_URL="https://soroban-testnet.stellar.org"
export READER="<funded-testnet-identity>"

export EPHEMERAL_ACCOUNT_ID="<verified-ephemeral-account-id>"
export SWEEP_CONTROLLER_ID="<verified-sweep-controller-id>"

export START_LEDGER="$(curl -fsS "$RPC_URL" \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getLatestLedger"}' |
  jq -r '.result.sequence')"
echo "start ledger: $START_LEDGER"
```

Recording the start ledger first is what makes "no events yet" a meaningful
observation later — the same rule
[`event-watchlist.md`](../monitoring/event-watchlist.md) applies to alerting.

## Step 2: Run the lifecycle

Nothing monitor-specific here — use the existing walkthroughs and keep the
transactions' hashes:

- create + initialize: [`README.md`](README.md) ("Create and initialize"),
  or [`expiry-and-recovery.md`](expiry-and-recovery.md) for a short-expiry
  account;
- pay: "Pay and record" in [`README.md`](README.md);
- terminal: [`signed-sweep-walkthrough.md`](signed-sweep-walkthrough.md) or
  [`gas-free-claim-walkthrough.md`](gas-free-claim-walkthrough.md) (sweep
  path), or [`expiry-and-recovery.md`](expiry-and-recovery.md) (expiry path).

## Step 3: Collect the merged stream

### Option A: CLI polling (smallest moving parts)

```bash
stellar contract events \
  --contract-id "$EPHEMERAL_ACCOUNT_ID" \
  --start-ledger "$START_LEDGER" \
  --filter '{"topics":[["created"],["payment"],["multi_pay"],["swept_mul"],["expired"],["reserve"]]}' \
  --network "$NETWORK"

# same range, the controller half:
stellar contract events \
  --contract-id "$SWEEP_CONTROLLER_ID" \
  --start-ledger "$START_LEDGER" \
  --filter '{"topics":[["sweep"]]}' \
  --network "$NETWORK"
```

### Option B: Horizon streaming (the dashboard shape)

Horizon's event stream is per contract, so run one stream per contract and
merge on ledger. Sketch (same shape as the indexer example in
[`event-topics.md`](../registry/event-topics.md)):

```javascript
import { Horizon } from '@stellar/stellar-sdk';

const horizon = new Horizon.Server('https://horizon-testnet.stellar.org');
const TOPICS = ['created', 'payment', 'multi_pay', 'swept_mul', 'expired', 'reserve'];

function reduce(accountId, state, event) {
  // event.topic[0] is the event-name symbol; event.ledger is the sequence.
  switch (event.topic[0]) {
    case 'created':   return { ...state, status: 'Active',  bornIn: event.ledger };
    case 'payment':
    case 'multi_pay': return { ...state, status: 'PaymentReceived' };
    case 'swept_mul': return { ...state, status: 'Swept',  to: event.value.destination };
    case 'expired':   return { ...state, status: 'Expired', to: event.value.recovery_address };
    case 'reserve':   return { ...state, lastReserve: event.value };  // informational
  }
}

// One stream per contract; merge by event.ledger in your store.
horizon.events().forContract(accountId).cursor(startCursor).stream({
  onmessage: (e) => dispatch('account', e),
  onerror: (err) => logError(err),
});
horizon.events().forContract(controllerId).cursor(startCursor).stream({
  onmessage: (e) => dispatch('controller', e),
  onerror: (err) => logError(err),
});
```

Treat the sketch as structure, not code to paste unchanged: decode payloads
per the structs in [`event-topics.md`](../registry/event-topics.md), and
persist `paging_token` per stream so each resumes independently.

## Step 4: Reconcile against state — always

Any dashboard built on this stream needs a reconciliation path, because three
things are true of events that they are not true of state:

1. **They lag.** Horizon ingests asynchronously; a "missing" terminal event
   seconds after a `SUCCESS` transaction is usually indexing lag. See
   [`event-not-appearing-in-horizon.md`](../runbooks/event-not-appearing-in-horizon.md).
2. **They can be missed across a collector gap.** RPC `getEvents` only
   retains a recent ledger window; a collector that was down for hours cannot
   replay from `START_LEDGER` against the RPC.
3. **They prove transitions, not balances.** `swept_mul` proves the
   account-side state path ran — not that the destination received funds
   (same caveat as [`event-watchlist.md`](../monitoring/event-watchlist.md)).

The reconciliation read for one account is:

```bash
stellar contract invoke \
  --simulate-only \
  --id "$EPHEMERAL_ACCOUNT_ID" \
  --network "$NETWORK" \
  --source "$READER" \
  -- \
  get_info
```

`get_info` returns the stored `status`, `expiry_ledger`, payments, and
`swept_to` — exactly the fields the reducer above derives from events. The
derived and stored values must agree; if they do not, trust `get_info` and
treat the discrepancy as a collector defect, not a chain one.

### Status mapping (the state machine your reducer implements)

| `AccountStatus` | Value | Reached after event | Terminal |
|---|---:|---|---|
| `Active` | 0 | `created` | no |
| `PaymentReceived` | 1 | `payment` (first recorded asset) | no |
| `Swept` | 2 | `swept_mul` (+ controller `sweep`) | yes |
| `Expired` | 3 | `expired` (via `expire()` or `recover()`) | yes |

Transitions are strictly forward: `0 → 1 → 2` or `0 → 1 → 3` (or `0 → 3`
for an account that never received a payment). There is no transition out of
2 or 3, and no transition from 0 directly to 2/3 without a payment —
`sweep`/`sweep_claim` reject with `NoPaymentReceived` (10) first.

---

## Edge Cases to Build In

| Edge case | What you will see | Handling |
|---|---|---|
| Single-asset account | `payment` then `swept_mul`; no `multi_pay`, ever | Do not require `multi_pay` in sequence validation |
| Expiry ledger passed, nobody called `expire()`/`recover()` | Silence; `is_expired()` is already `true` | The stream is correctly quiet; surface "expirable" from state, not from events |
| Concurrent sweeps on the shared controller | Another account's `sweep` interleaves in the controller stream | Filter controller events by `SweepCompleted.ephemeral_account`; the nonce moves globally regardless |
| Duplicate submissions | At most one terminal transition commits; retries fail on state (`AlreadySwept` 7 / `NoPaymentReceived` 10) and emit nothing | Ignore missing duplicate events — they were never emitted |
| Testnet reset | All history for the ID vanishes | Segment storage by reset epoch ([`reset-policy.md`](../policies/reset-policy.md)); an old ID will never emit again |
| Redeploy at a verified-fresh ID | Clean stream resumes at `created` | Anchor subscriptions on WASM-hash/registry verification ([`verify-contract-live.md`](../registry/verify-contract-live.md)) |

---

## Dashboard Template

Per account, the minimum useful record derived from this stream:

```text
account_id, creator, born_ledger, expiry_ledger,
status (derived), status_source ("events" | "get_info"),
payments: [{asset, amount, recorded_in_ledger}],
terminal: {kind: "sweep" | "expiry", ledger, destination},
reserve: {fully_reclaimed, remaining},
last_event_cursor, last_reconciled_ledger
```

Alert-worthy conditions are rare by design — per
[`event-watchlist.md`](../monitoring/event-watchlist.md), alert only when a
*known, committed* operation's event has not arrived within the collector's
deadline, never on mere idleness.

---

## Related Documents

- [`event-topics.md`](../registry/event-topics.md) — canonical topics and payload structs
- [`event-watchlist.md`](../monitoring/event-watchlist.md) — absence policy and alert preconditions
- [`event-not-appearing-in-horizon.md`](../runbooks/event-not-appearing-in-horizon.md) — a committed event that is not visible yet
- [`README.md`](README.md) — create/pay/sweep walkthroughs this template monitors
- [`expiry-and-recovery.md`](expiry-and-recovery.md) — driving the expiry path to completion
- [`checking-account-status.md`](checking-account-status.md) — the `get_info` reconciliation read
- [`reading-contract-storage.md`](reading-contract-storage.md) — an independent view when you distrust both events and getters
- [`explorer-links.md`](../monitoring/explorer-links.md) — links for recorded contract IDs
- [`reset-policy.md`](../policies/reset-policy.md) — why pre-reset history disappears

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-10-15 | Initial single-account lifecycle monitoring example |
