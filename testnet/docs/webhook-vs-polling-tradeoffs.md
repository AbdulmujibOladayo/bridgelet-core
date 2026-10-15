# Polling vs a Webhook Relay for Testnet Events

## Purpose

An integration usually needs to know when something happened on a contract: an
account was created, a payment was recorded, a sweep finished, an account
expired. There are two broad ways to find out. You can **poll** the network for
events yourself, or you can run a **relay** (an indexer that watches the network
and pushes events to your service as webhooks).

This page compares the two for a **testnet** integration and recommends where a
small integrator should start. The short answer is: poll first, and move to a
relay only when a specific need below forces it.

## Status

**Reasoned from the event documentation in this tree; not observed.** No
contracts are recorded as deployed (`../registry/contract-status.md`), so none
of this has been run against a live deployment. Rate limits and event retention
on the public RPC are not stated in this repository and are marked below as
needing a check.

## The important fact: there is no native webhook

Soroban RPC and Horizon expose events for you to **pull** (and Horizon can also
stream them to a client that holds a connection open). Neither calls your URL
when an event occurs. So "webhook" here always means **a relay that you or a
vendor run**, which polls or streams on your behalf and delivers over HTTP.

That makes the real question: *do I run the polling loop inside my own service,
or in a separate relay that calls my service?* The loop exists either way.

## The options

| Option | What it is | Moving parts |
|---|---|---|
| **A. One-off CLI poll** | `stellar contract events` run by hand or from a test | None |
| **B. Polling loop in your service** | A scheduled job that reads events from a stored ledger cursor | One job, one cursor |
| **C. Streaming client** | A long-lived connection to Horizon's event stream | One connection, reconnect logic |
| **D. Relay with webhooks** | A separate indexer that stores events and POSTs them to you | Indexer, database, delivery queue, receiver endpoint |

Examples of A and C are in
[`../examples/monitoring-a-single-account-lifecycle.md`](../examples/monitoring-a-single-account-lifecycle.md).

## Comparison

| | A. CLI poll | B. Polling loop | C. Streaming | D. Relay + webhooks |
|---|---|---|---|---|
| Setup effort | Minutes | An afternoon | A day | Days, plus ongoing upkeep |
| Infrastructure | None | Your service only | Your service only | Indexer, database, queue, public endpoint |
| Latency | Manual | Your poll interval | Near real time | Near real time, plus delivery |
| Survives your service restarting | n/a | Yes, from the stored cursor | Only if you also store a cursor | Yes, the relay keeps its own |
| Catches up after downtime | Only within the RPC retention window | Only within the RPC retention window | Same | Yes, from its own store |
| Duplicate events | n/a | You deduplicate | You deduplicate | At-least-once delivery, so you still deduplicate |
| Public-endpoint rate limits | Easy to hit if looped | Your interval controls it | One connection | The relay absorbs it |
| Handles a testnet reset | Manual | You must detect it | You must detect it | The relay must detect it |
| Right for | Debugging, a first look | Most testnet integrations | Dashboards | Many consumers, or history beyond retention |

## Why polling is the right default on testnet

- **Testnet is small and quiet.** The volume of Bridgelet events is low, so a
  modest interval is enough and a pushed event buys little.
- **You get to control the load.** The public endpoint is rate-limited
  (`../faucet/README.md` describes the same for Friendbot; check the RPC's own
  limits before choosing an interval). A relay does not remove that limit, it
  just moves the loop elsewhere.
- **A relay is another thing to reset.** Testnet is wiped periodically
  (`../policies/reset-policy.md`). A relay with a database has to notice the wipe
  and discard stale data, which is extra work for an environment that is
  disposable by design.
- **Less to secure.** A webhook receiver is a public endpoint that must verify
  who is calling it. A polling job makes only outbound calls.

## What a correct polling loop needs

These are the parts that are easy to get wrong, and they apply to B, C and D
alike.

| Requirement | Why |
|---|---|
| **Cursor is a ledger sequence, stored durably** | A ledger sequence is a total order over finalised state. A timestamp is not a safe cursor. See [`../tooling/event-tail-spec.md`](../tooling/event-tail-spec.md) |
| **Deduplicate on `(ledger, tx_index, event_index)`** | Ranges overlap on retry, and delivery may repeat |
| **Query both contracts** | A sweep emits on `EphemeralAccount` and on `SweepController`; one subscription sees half |
| **Detect a reset** | If the network head is **lower** than your cursor, testnet was reset. Discard the cursor and re-verify the contract IDs |
| **Handle the retention window** | A start ledger older than the RPC retains cannot be read. Treat it as a fatal gap, not a retry |
| **Back off on errors** | A tight retry loop on a rate-limited endpoint makes the outage longer |
| **Reconcile with state** | Events are facts, not state. `get_info` is the authority; use the stream to know when to look |

A skeleton:

```javascript
async function pollOnce(rpc, contractId, cursorStore) {
  const head = await rpc.getLatestLedger();
  const cursor = await cursorStore.get(); // a ledger sequence

  if (cursor !== null && head.sequence < cursor) {
    // Testnet was reset: the ledger went backwards.
    await cursorStore.clear();
    throw new Error('Testnet reset detected: re-verify contract IDs');
  }

  const start = cursor === null ? head.sequence : cursor + 1;
  const events = await rpc.getEvents({
    startLedger: start,
    filters: [{ type: 'contract', contractIds: [contractId] }],
  });

  for (const event of events.events) {
    const key = `${event.ledger}:${event.txIndex}:${event.eventIndex}`;
    if (await seen(key)) continue; // deduplicate
    await handle(event);
    await markSeen(key);
  }

  await cursorStore.set(events.latestLedger);
}
```

Treat it as structure, not code to paste: the exact field names depend on your
SDK version, and nothing here has been run against a live deployment. Topic
symbols and payloads are in [`../registry/event-topics.md`](../registry/event-topics.md).

## When to move to a relay

Move to option D when one of these is true, and not before:

| Trigger | Why polling stops being enough |
|---|---|
| Several services need the same events | One shared store beats each service polling the same range |
| You need history older than the RPC keeps | Only your own store can answer that |
| You need push delivery into a system that cannot poll | Some platforms only accept inbound HTTP |
| Poll volume is hitting rate limits | A single relay can fan out |

If you do build or buy a relay, design the receiver for **at-least-once**
delivery: verify the sender, return quickly, deduplicate on the same event key,
and process asynchronously. Never trust a webhook body as proof that an event
happened; confirm important transitions with `get_info`.

## Undecided

| Question | State |
|---|---|
| The public RPC's request rate limit | Not stated in this repository; check before choosing an interval |
| How long the RPC retains events | Not stated here; the spec treats it as a configurable window |
| Whether this project should offer a reference relay | Not proposed; `../tooling/` holds specifications only |

## Related Documentation

- [`../examples/monitoring-a-single-account-lifecycle.md`](../examples/monitoring-a-single-account-lifecycle.md):
  the polling and streaming examples this page builds on
- [`../tooling/event-tail-spec.md`](../tooling/event-tail-spec.md): the cursor,
  deduplication and reset rules in detail
- [`../registry/event-topics.md`](../registry/event-topics.md): topics and
  payloads
- [`../runbooks/event-not-appearing-in-horizon.md`](../runbooks/event-not-appearing-in-horizon.md):
  when an expected event is missing
- [`../monitoring/event-watchlist.md`](../monitoring/event-watchlist.md): which
  events are worth watching
- [`../policies/reset-policy.md`](../policies/reset-policy.md): why a cursor can
  become invalid

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial comparison of polling and webhook relays for testnet events |
