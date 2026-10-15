# Runbook: Contract Event Not Appearing in Horizon

## Overview

A transaction is visibly successful — it shows `SUCCESS` in an explorer, or
`getTransaction` returns a final result — yet the contract event you expected
does not appear when you query Horizon's events endpoint (or Soroban RPC's
`getEvents`). This runbook separates the four reasons this happens, in the
order they should be checked:

1. **Indexing lag** — Horizon ingests contract events asynchronously after
   ledger close; the event is not lost, just not indexed yet.
2. **Wrong topic filter** — the event exists, but your filter does not match
   the on-chain topic encoding.
3. **Wrong contract queried** — the event was emitted by a different contract
   than the one you are watching.
4. **No event was ever emitted** — the transaction succeeded but its success
   path does not emit the event you assumed it would.

Only after those are excluded is an event genuinely "missing".

## Symptom

- `GET https://horizon-testnet.stellar.org/events?contract_id=<ID>` returns an
  empty `data` array (or no entry matching your expected topic).
- An explorer shows the transaction as successful.
- Your indexer/desired workflow is stuck waiting for a terminal event
  (`swept_mul`, `sweep`, `expired`, `reserve`, …) that never arrives.

## Prerequisites

- The transaction hash of the "successful" transaction
- The contract ID(s) you expect the event from
- Event topic reference: `testnet/registry/event-topics.md`
- Access to both Soroban RPC and Horizon (they are independent pipelines —
  see `testnet/runbooks/rpc-outage.md`)

## Diagnosis

### Step 1: Prove the transaction really succeeded, and record its ledger

```bash
curl -fsS https://soroban-testnet.stellar.org \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getTransaction",
       "params":{"hash":"<TX_HASH>"}}' | jq '.result.status, .result.ledger'
```

Record the ledger sequence. Everything below is anchored to it. If the status
is not `SUCCESS`, stop — you do not have an event-visibility problem, you have
a failed transaction; route to `testnet/examples/error-handling.md`.

### Step 2: Check indexing lag before anything else

Horizon serves contract events from its own ingestion pipeline, which trails
the Soroban RPC by at least one ledger close (roughly five seconds on
testnet, longer under load or during ingestion hiccups).

```bash
# Horizon's most recently ingested ledger
curl -fsS "https://horizon-testnet.stellar.org/ledgers?order=desc&limit=1" |
  jq '.records[0].sequence'
```

If Horizon's latest ingested ledger is **behind** the transaction's ledger,
wait and re-query with the same parameters. Do not change your filter yet —
changing the filter while lagging converts a timing problem into a mystery.

Also confirm you are not querying a cursor that points *past* the event: a
`cursor` taken from a later event (or a `start_ledger` newer than the tx
ledger) skips it permanently in forward order. When in doubt, re-query from
the transaction's own ledger with no cursor.

### Step 3: Verify the topic filter against the canonical encoding

Topic filters are exact matches against XDR-encoded `ScVal`s, not free-text
searches. The first topic is always an `ScSymbol` carrying the **event
name** — which is frequently *not* the payload struct name:

| You are looking for | Correct topic[0] | Common wrong guesses |
|---|---|---|
| `AccountCreated` payload | `created` | `AccountCreated`, `account_created` |
| `PaymentReceived` payload | `payment` | `PaymentReceived`, `multi_pay` (that is the *second* payment) |
| `MultiPaymentReceived` payload | `multi_pay` | `MultiPaymentReceived`, `payment` |
| `SweepExecutedMulti` payload | `swept_mul` | `sweep`, `SweepExecutedMulti` |
| `AccountExpired` payload | `expired` | `AccountExpired`, `expire` |
| `ReserveReclaimed` payload | `reserve` | `ReserveReclaimed`, `reserve_reclaimed` |
| `SweepCompleted` payload (controller) | `sweep` | `SweepCompleted`, `swept_mul` |
| `DestinationAuthorized` payload | `dest_auth` | `DestinationAuthorized` |
| `DestinationUpdated` payload | `dest_upd` | `DestinationUpdated`, `dest_update` |

Symbols are case-sensitive and short — several were truncated by Soroban's
9-character `symbol_short!` limit (`swept_mul`, `dest_upd`, `dest_auth`). The
authoritative table is `testnet/registry/event-topics.md`; if your filter was
built from a Rust struct name, it will never match.

Correct shapes:

```bash
# Horizon — one topic
curl -fsS "https://horizon-testnet.stellar.org/events?contract_id=<ID>&type=contract&order=asc&start_ledger=<START>"

# Filtered, via the RPC the CLI uses
stellar contract events \
  --contract-id "$EPHEMERAL_ACCOUNT_ID" \
  --start-ledger "$START_LEDGER" \
  --filter '{"topics": [["swept_mul"],["reserve"]]}' \
  --network testnet
```

A filter list like `[["swept_mul"],["reserve"]]` means "topic[0] is
`swept_mul` **or** `reserve`". Nested arrays of length > 1 (e.g.
`[["payment","multi_pay"]]`) constrain topic[0] and topic[1] *together* — a
frequent cause of empty results when the intent was "either event".

### Step 4: Confirm you are watching the right contract

Events are scoped to the contract that emitted them, and Bridgelet splits a
single logical flow across two contracts:

| Event topic | Emitted by |
|---|---|
| `created`, `payment`, `multi_pay`, `swept_mul`, `expired`, `reserve` | the `EphemeralAccount` instance |
| `sweep`, `dest_auth`, `dest_upd` | the `SweepController` deployment |

The classic miss: watching only the ephemeral account for a `claim()` flow and
waiting for a `sweep` event that was emitted on the **controller**
(`gas-free-claim-walkthrough.md` documents this split). Query both contracts
over the same ledger range and correlate.

Also verify the contract ID itself — a stale ID after a testnet reset answers
empty for everything. Route to `contract-not-found-error.md` if
`getLedgerEntries` shows no contract instance.

### Step 5: Verify that this success path emits the event at all

A `SUCCESS` status proves the transaction executed; it does not prove any
particular event was emitted. Per `testnet/registry/event-topics.md` and the
contract sources:

| Call | Events actually emitted |
|---|---|
| First `record_payment` on an account | `payment` (not `multi_pay`) |
| Any later `record_payment` | `multi_pay` (not `payment`) |
| `sweep` / `sweep_claim` on the account | `swept_mul`, then `reserve` |
| `execute_sweep` / `claim` on the controller | `sweep` (on the controller only) |
| `expire` / `recover` | `expired`, then `reserve` |
| `update_authorized_destination` | `dest_upd` |
| `initialize` of the controller with a locked destination | `dest_auth` |
| `AccountFactory` operations | **none** — it emits no events; account creation is tracked via each child's `created` event |

If the table shows no event for your call, nothing is broken — you were
waiting for an event the code never emits.

### Step 6: Cross-check with RPC `getEvents` and with state

Horizon and the Soroban RPC are independent consumers. If RPC `getEvents`
returns the event but Horizon does not, the lag or gap is on Horizon's side —
keep polling Horizon; do not re-run the transaction.

Then confirm the *state change* independently of any event plumbing:

```bash
stellar contract invoke \
  --simulate-only \
  --id "$EPHEMERAL_ACCOUNT_ID" \
  --network "$NETWORK" \
  --source "$READER" \
  -- \
  get_info
```

The internal reserve event can also be read back from storage via
`get_last_reserve_event` and `get_reserve_reclaim_event_count` — useful for
`reserve` events specifically, because those getters are the contract's own
record of what it emitted.

## Common Causes

| Cause | Signature | Fix |
|---|---|---|
| Indexing lag | Empty now, populated minutes later with identical query | Wait; poll with same cursor/params |
| Filter uses struct name | Always empty, no matter how long you wait | Use the topic symbols in Step 3 |
| Nested-array filter semantics | Always empty | Use one inner array per alternative |
| Watching the wrong contract | Other contract's query shows it | Query both contracts (Step 4) |
| `start_ledger`/`cursor` past the event | Empty in forward order | Re-query from the tx ledger without cursor |
| RPC `getEvents` retention exceeded | Error or empty for old ranges | Use Horizon for history; RPC retains only a recent window |
| Success path emits nothing | Event never appears anywhere | Step 5 table — expected behavior |
| Testnet reset | Everything before the reset vanishes | Redeploy and re-record IDs (`testnet/policies/reset-policy.md`) |

## Resolution

- Lag: no action; the event arrives. Alerting should tolerate minutes of
  Horizon lag before paging.
- Wrong filter/contract/range: correct the query per the steps above; no
  on-chain action needed.
- Genuinely absent event: do **not** replay the transaction hoping for a
  second emission — many state machines forbid the second call
  (`AlreadySwept` (7), `DuplicateAsset` (13), …). Verify state via
  `get_info` and treat the workflow as complete if state is terminal.
- Suspected true loss (state changed but its event never appears anywhere
  after lag, retention, and filter issues are excluded): capture the tx hash,
  the queries tried, and both endpoints' results, then escalate.

## Escalation

Capture: tx hash, both contract IDs, exact filter JSON, start ledger/cursor,
Horizon latest-ingested ledger, RPC `getEvents` output, and the corresponding
`get_info` state read. Escalate per `testnet/runbooks/on-call-testnet.md`.

## Prevention Checklist

- [ ] Topic filters are generated from `testnet/registry/event-topics.md`,
      never from Rust type names
- [ ] Indexers subscribe to **both** the account and controller contracts for
      every flow
- [ ] Resume logic stores the Horizon `paging_token`, not a timestamp
- [ ] Alerts distinguish "event missing" from "state not terminal" — the
      state read is the source of truth
- [ ] Dashboards tolerate indexing lag of several ledger closes

## Related Documentation

- `testnet/registry/event-topics.md` — canonical topics and payload structs
- `testnet/examples/monitoring-a-single-account-lifecycle.md` — ordering the
  full event sequence for one account
- `testnet/monitoring/event-watchlist.md` — which events to watch for
  alerting
- `testnet/examples/reading-contract-storage.md` — independent state reads
  when you distrust the event pipeline
- `testnet/runbooks/rpc-outage.md` — endpoint outages masquerading as missing
  events
- `testnet/runbooks/contract-not-found-error.md` — stale IDs that answer
  empty for every query
- `testnet/policies/reset-policy.md` — why pre-reset history disappears

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-10-15 | Initial event-not-appearing-in-Horizon runbook |
