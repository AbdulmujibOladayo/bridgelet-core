# Runbook: Insufficient Reserve Errors

## Overview

Two completely different things in this project are called a "reserve", and
conflating them is the single most common misdiagnosis when a testnet
transaction fails near an account's balance boundary:

1. **Stellar's minimum base reserve** — a network rule enforced by the
   protocol. Every funded classic account must hold at least
   `base_reserve × (number of ledger entries)`; the current base reserve is
   **0.5 XLM**. Violations surface as classic operation result codes before any
   Bridgelet contract code runs.
2. **Bridgelet's internal reserve tracking** — bookkeeping inside
   `EphemeralAccount`. At `initialize()` the contract records a fixed internal
   reserve of `BASE_RESERVE_STROOPS = 1_000_000_000` (1 XLM, defined in
   `contracts/ephemeral_account/src/lib.rs`) and later "reclaims" it to the
   sweep/expire destination as a tracked transfer with a `reserve` event. This
   value is **not enforced by the network**, and it can never cause a network
   reserve error.

This runbook teaches you to recognize which one you are looking at, and how to
resolve each.

## Symptom

A transaction fails with a balance-related error, or an operator notices that
an account's on-chain balance does not match what the project's reserve
trackers report. The two families of symptoms:

**Network-level (real, protocol-enforced):**

- Classic operation result codes: `op_low_reserve`, `op_underfunded`,
  `op_line_full`, or transaction-level `tx_insufficient_balance`.
- Soroban flows failing at submission with `INSUFFICIENT_FEE`-class outcomes
  when the source cannot cover resource fees (rent) for a contract instance.

**Project-internal (bookkeeping only):**

- `reclaim_reserve()` returning `0`, or a `reserve` event with
  `amount: 0` / `fully_reclaimed: true`.
- `Error(Contract, #4)` (`InvalidAmount`) from the reclaim path — the only
  error the internal tracker can produce, and only if tracked values are
  negative.

## The Two Concepts Side by Side

| | Network base reserve | `EphemeralAccount` internal reserve |
|---|---|---|
| Defined by | Stellar protocol | `BASE_RESERVE_STROOPS` in `contracts/ephemeral_account/src/lib.rs` |
| Value | 0.5 XLM per ledger entry (network-wide) | 1 XLM (1,000,000,000 stroops), hardcoded |
| Enforced by | The network, at submission | Nobody — it is read-modify-write bookkeeping in contract storage |
| Failure surface | `op_low_reserve`, `op_underfunded`, `tx_insufficient_balance`, fee failures | At most `InvalidAmount` (contract error code 4); normally just `0` reclaimed |
| Fixed after deploy? | Moves only when the network changes the base reserve | Hardcoded at compile time; not read from `ReserveContract` (which is not deployed — see `testnet/registry/reserve-contract-status.md`) |
| Inspecting it | Horizon `accounts/{id}` balances and subentries | `get_reserve_remaining`, `get_reserve_available`, `is_reserve_reclaimed`, `get_last_reserve_event`, `get_reserve_reclaim_event_count` |
| What to do | Fund the account | Nothing is broken; it is expected lifecycle output |

> `testnet/registry/contract-status.md` documents the wiring gap explicitly:
> `ReserveContract` is not deployed on testnet, and `EphemeralAccount` uses
> its own hardcoded constant instead of reading from it. Do not expect any
> `ReserveContract` error in a testnet failure — there is no such call path.

## Prerequisites

- A funded testnet identity for queries (`stellar keys generate --global
  testnet-investigator`)
- Contract IDs from `testnet/registry/contract-status.md`
- Access to Horizon (classic balances) and Soroban RPC (contract calls)

## Diagnosis

### Step 1: Classify the error layer first

Decode the failure before reading any balance:

```bash
stellar transaction get <TX_HASH> --network testnet --output json
```

| What the result shows | Layer | Go to |
|---|---|---|
| Operation code `op_low_reserve` / `op_underfunded` / `op_line_full`, or `tx_insufficient_balance` | Network | Step 2 |
| `Error(Contract, #4)` from a `reclaim_reserve`/`sweep`/`expire` call | Internal tracker | Step 4 |
| `Error(Contract, #N)` with any other N | Not a reserve problem at all | `testnet/examples/error-handling.md` |
| Soroban resource-fee rejection at submission | Network (rent/fee) | Step 3 |

The network never mentions "reserve" in a contract error, and the internal
tracker never produces a classic operation code. The layer is unambiguous once
you look at the result XDR.

### Step 2: Network-level — inspect the account's real balances

```bash
curl -fsS "https://horizon-testnet.stellar.org/accounts/<G_ADDRESS>" |
  jq '{balances, subentry_count, signers}'
```

Each trustline, offer, signer, or data entry consumes one base reserve. An
account at exactly the minimum cannot accept an operation that adds another
entry — that is `op_low_reserve`. The fix is ordinary: send the account more
XLM (Friendbot for brand-new addresses, see `testnet/faucet/README.md`), or
remove the entry the account cannot afford.

In Bridgelet flows the typical network-reserve situations are:

| Flow | Reserve pressure |
|---|---|
| Funding a brand-new address for the first time | The payment must leave the destination able to hold its own ledger entry (≥ 0.5 XLM plus any entries created) |
| Recording multi-asset payments via Horizon transfers | Each new trustline/asset entry on the counterparty adds reserve the payer must cover |
| Deploying/initializing contract instances | Source pays rent as a resource fee; underfunded source → fee rejection, not `op_low_reserve` |

### Step 3: Network-level — fee/rent failures on Soroban calls

A Soroban transaction's resource fee includes ledger rent for the contract
instance and storage it touches. An underfunded source fails at submission
with a fee-class rejection even though every classic balance looks "fine" for
a pure payment. Fund the submitting identity and resubmit; nothing in the
contract state is implicated.

### Step 4: Internal tracker — read the tracked values

```bash
stellar contract invoke \
  --simulate-only \
  --id "$EPHEMERAL_ACCOUNT_ID" \
  --network "$NETWORK" \
  --source "$READER" \
  -- \
  get_reserve_remaining

stellar contract invoke \
  --simulate-only \
  --id "$EPHEMERAL_ACCOUNT_ID" \
  --network "$NETWORK" \
  --source "$READER" \
  -- \
  get_reserve_available

stellar contract invoke \
  --simulate-only \
  --id "$EPHEMERAL_ACCOUNT_ID" \
  --network "$NETWORK" \
  --source "$READER" \
  -- \
  is_reserve_reclaimed
```

Expected lifecycle (all of it normal):

1. `initialize()` calls `init_reserve_tracking` with 1 XLM.
2. Each successful `sweep` / `sweep_claim` / `expire` / `recover` path calls
   `reclaim_reserve_to`, which transfers `min(reserve_available,
   reserve_remaining)` to the destination and emits a `reserve` event
   (`ReserveReclaimed { destination, amount, sweep_id, fully_reclaimed,
   remaining_reserve }` — topic `reserve`, see
   `testnet/registry/event-topics.md`).
3. Once fully reclaimed, `is_reserve_reclaimed` is `true`, later reclaims
   transfer 0 and emit `fully_reclaimed: true, amount: 0`, and
   `reclaim_reserve()` is a safe, repeatable no-op.

A `0` reclaim or a fully-reclaimed flag is therefore **not an error**. It
means the lifecycle already completed. `reclaim_reserve()` on an account that
is still `Active` or `PaymentReceived` fails with `InvalidStatus` (12) — also
by design; there is nothing to reclaim until a terminal state.

### Step 5: Cross-check the event trail when the numbers look wrong

```bash
stellar contract events \
  --contract-id "$EPHEMERAL_ACCOUNT_ID" \
  --start-ledger "$START_LEDGER" \
  --filter '{"topics": [["reserve"]]}' \
  --network testnet
```

The last `reserve` event's `remaining_reserve` must equal the current
`get_reserve_remaining` reading. A mismatch means you are reading a different
account ID, or a stale deployment after a testnet reset — route to
`contract-not-found-error.md`, not this runbook.

## Common Causes

| Cause | Which reserve | Fix |
|---|---|---|
| Payment would push destination below 0.5 XLM + entries | Network | Send more XLM or reduce entries |
| Source lacks balance for payment + its own entries | Network (`op_underfunded`) | Fund the source |
| Soroban rent/resource fee exceeds source balance | Network | Fund the submitting identity |
| Expecting `reserve` reclaim to equal the *network* reserve | Internal | It equals the hardcoded 1 XLM tracking constant; they are unrelated numbers |
| Seeing `amount: 0` reclaim events and filing a bug | Internal | Normal once fully reclaimed; check `is_reserve_reclaimed` |
| `ReserveContract`-related expectations | Neither | `ReserveContract` is not deployed and not wired; see `testnet/registry/reserve-contract-status.md` |

## Resolution

- **Network reserve errors**: fund the implicated account so it clears
  `base_reserve × entries`, remove an unaffordable entry, or split the
  operation. Then re-check with a Horizon account read before resubmitting.
- **Internal tracker observations**: no on-chain action is required. Record
  the `reserve` events if you are building an audit trail; the tracked values
  are already consistent by construction.
- **Never** attempt to "fix" a network `op_low_reserve` by calling
  `reclaim_reserve()` — different layer, no effect on network balances beyond
  what a normal transfer would do, and the account must be in a terminal
  state for that call to succeed at all.

## Prevention Checklist

- [ ] Dashboards label the two reserves distinctly (e.g. "XLM balance vs.
      network min" and "tracked internal reserve")
- [ ] Alerts on `op_low_reserve`/`op_underfunded` route here; alerts on
      `reserve` events with `amount: 0` do not page anyone
- [ ] Multi-asset tests budget 0.5 XLM per expected ledger entry on both
      sides before exercising `record_payment`
- [ ] Issue templates ask for the decoded result XDR, so the layer is stated
      up front

## Escalation

Escalate per `testnet/runbooks/on-call-testnet.md` only if the network
rejection persists for an account whose Horizon balance is demonstrably above
the minimum reserve for its entry count, with the result XDR attached.

## Related Documentation

- `testnet/registry/event-topics.md` — the `reserve` event and
  `ReserveReclaimed` payload
- `testnet/registry/reserve-contract-status.md` — why `ReserveContract` is
  not a factor on testnet
- `testnet/registry/contract-status.md` — the hardcoded-reserve wiring note
- `testnet/examples/multi-asset-sweep.md` — multi-asset flows that stress
  counterparty entries
- `testnet/examples/error-handling.md` — mapping every contract error code
- `testnet/faucet/README.md` — funding testnet accounts
- `testnet/runbooks/expired-account-recovery-check.md` — verifying where
  funds actually are after the terminal state

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-10-15 | Initial insufficient-reserve runbook distinguishing network vs. internal reserve |
