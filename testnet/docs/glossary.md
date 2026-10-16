# Testnet Glossary

## Purpose

A short glossary of the terms used across Bridgelet's contract interfaces,
events and testnet documentation. It is written for someone arriving cold, such
as a hackathon participant or a partner integrator, who needs the vocabulary
before the API reference makes sense.

Definitions are taken from the contract source. Where a term is a Stellar
concept rather than a Bridgelet one, it says so.

## Status

**Derived from the contract source; not observed on a live deployment.** No
contracts are recorded as deployed (`testnet/registry/contract-status.md`).

## The core idea in four sentences

A **creator** makes a short-lived **ephemeral account** to receive payments. When
a payment has been **recorded**, the account can be **swept** to a destination,
or **claimed** by a recipient. If nobody sweeps it before its **expiry ledger**,
anyone allowed to can **expire** it and the funds go to the **recovery
address**.

## Terms

### Accounts and lifecycle

| Term | Meaning |
|---|---|
| **Ephemeral account** | A short-lived `EphemeralAccount` contract instance that receives and holds payments until they are swept or the account expires. Each one is a separate contract |
| **Creator** | The address that initialised the account. May also trigger recovery after expiry |
| **Status** | `Active` (0), `PaymentReceived` (1), `Swept` (2) or `Expired` (3). The status moves forward and does not go back |
| **Payment (recorded)** | A stored `(asset, amount, timestamp)` entry. Recording it is bookkeeping, not a transfer of funds |
| **Sweep** | Moving an account's recorded payments to a destination and marking the account `Swept`. A one-way transition |
| **Claim** | A gas-free sweep path: the recipient signs an authorisation, a relayer submits and pays the fee. Goes through `SweepController::claim()` |
| **Expire** | Marking an account `Expired` once its expiry ledger has passed. Only valid before a sweep |
| **Recover** | Returning an expired account's funds to the recovery address. May be called by the creator or the recovery address |
| **Recovery address** | Where funds go if the account expires unswept |
| **Destination** | The address a sweep pays out to. The controller can be locked to one authorised destination |

### Time and limits

| Term | Meaning |
|---|---|
| **`expiry_ledger`** | The ledger sequence number after which the account is expired. Set at `initialize()` and must be in the future. Ledgers close roughly every five seconds, so a ledger count is a time estimate, not a clock |
| **Ledger** | One block of the Stellar network, identified by a sequence number |
| **Reserve** | A small amount of XLM that must stay in a Stellar account to keep it alive. The ephemeral account tracks a base reserve to reclaim after a sweep |
| **Stroop** | The smallest unit of XLM: 1 XLM is 10,000,000 stroops. Reserve values are in stroops |
| **Nonce** | A counter on `SweepController` that makes each signed sweep usable once. It is **global** to the controller, and it advances on every successful sweep |

### Contracts and roles

| Term | Meaning |
|---|---|
| **`EphemeralAccount`** | Holds payments and enforces the account lifecycle |
| **`SweepController`** | Verifies sweep authorisation and nonce, and is the recommended entry point for sweeping |
| **`ReserveContract`** | Stores an admin-set base reserve value. Not currently wired into the ephemeral account |
| **`AccountFactory`** | Creates many ephemeral accounts in one transaction. Not part of the supported testnet deployment |
| **Authorized controller** | The address set at `initialize()` that is allowed to drive an account's sweep. Normally the `SweepController` |
| **Authorized signer** | The Ed25519 public key the controller checks sweep signatures against |
| **Admin** | The address allowed to call `upgrade()` on an account, and to configure `ReserveContract` |
| **Relayer** | A party that submits a transaction and pays its fee on someone else's behalf, as in the gas-free claim path |

### Events

Events are published with the event name as the first topic. The full list and
payloads are in [`testnet/registry/event-topics.md`](../registry/event-topics.md).

| Event (topic) | Emitted when | Carries |
|---|---|---|
| **`AccountCreated`** (`created`) | An account is initialised | Creator and expiry ledger |
| **`PaymentReceived`** (`payment`) | The **first** payment is recorded | Amount and asset |
| **`MultiPaymentReceived`** (`multi_pay`) | A further payment, in another asset, is recorded | Asset and amount |
| **`SweepExecutedMulti`** (`swept_mul`) | An account is swept | Destination and the payments swept |
| **`AccountExpired`** (`expired`) | An account expires or is recovered | Recovery address and amounts |
| **`ReserveReclaimed`** (`reserve`) | Reserve is returned after a sweep or expiry | Destination, amount and remaining reserve |
| **`SweepCompleted`** (`sweep`) | `SweepController` finishes a sweep | Account, destination and amount |
| **`DestinationAuthorized`** (`dest_auth`) | The controller's destination is set at initialisation | Destination |
| **`DestinationUpdated`** (`dest_upd`) | The authorised destination changes | Old and new destination |

### Stellar and tooling terms

| Term | Meaning |
|---|---|
| **Testnet** | Stellar's public test network. Free to use, shared, and periodically reset |
| **Friendbot** | The testnet faucet that funds a new account with test XLM |
| **Soroban** | Stellar's smart contract platform, which these contracts run on |
| **Horizon** | Stellar's REST API for accounts and transactions |
| **Soroban RPC** | The JSON-RPC endpoint used to simulate and submit contract calls and to read events |
| **Network passphrase** | A string that identifies the network inside every signature. Testnet's is `Test SDF Network ; September 2015` |
| **`SEP-41`** | The standard Stellar token interface that Soroban tokens implement |
| **Trap** | A hard abort of a transaction with no contract error code. A rejected sweep signature is a trap |
| **Simulation** | A dry run of a contract call that reports cost and result without submitting it |

## Words that sound alike

| Pair | The difference |
|---|---|
| **Sweep** vs **claim** | Both move the funds out. A sweep is signed by the authorised signer; a claim is signed by the recipient and can be relayed |
| **Expire** vs **recover** | Expiry marks the account expired. Recovery returns its funds to the recovery address |
| **Record** vs **transfer** | Recording stores an entry. It does not move any asset |
| **Creator** vs **admin** | The creator made the account; the admin may upgrade it. They are often the same address, but the contract treats them as separate roles |

## Related Documentation

- [`overview.md`](overview.md): which contracts are deployed and callable
- [`known-limitations.md`](known-limitations.md): where the real behaviour differs
  from what the names suggest
- [`../registry/event-topics.md`](../registry/event-topics.md): event topics and
  payloads
- [`../../docs/api-reference.md`](../../docs/api-reference.md): the full contract
  interface
- [`../examples/README.md`](../examples/README.md): walkthroughs using these terms

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial testnet onboarding glossary |
