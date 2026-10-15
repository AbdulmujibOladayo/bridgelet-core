# Runbook: "Contract Not Found" Style RPC Errors

## Overview

Every invocation against a Bridgelet contract starts with the RPC resolving a
contract ID to a live ledger entry. When that resolution fails, the failure
happens **before any contract code runs** — there is no `Error(Contract, #N)`,
no event, and nothing to correlate on-chain. The typical shapes:

- `stellar contract invoke` aborts with an RPC error such as
  `Invalid contract ID` or a contract-entry lookup that returns nothing;
- `simulateTransaction` fails with an infrastructure/host error naming the
  contract rather than a contract error;
- `getLedgerEntries` for key `contract:<ID>` returns a null entry.

This runbook covers the three real causes, in the order they should be
checked:

1. **Wrong or stale contract ID** — the ID was valid once (or never was): it
   was copied from old artifacts, or the testnet was reset since.
2. **Wrong network configured** — the ID belongs to another network
   entirely.
3. **The contract was never deployed** — notably `account_factory`, which has
   no testnet deployment, so any remembered or locally-built ID for it will
   always fail this way.

## Symptom

- Invocation or simulation fails at the RPC layer with a contract-resolution
  error; the CLI never prints a contract error code.
- `get_info` on an address that "used to work" fails the same way after a
  redeploy or reset.
- Only one particular call fails this way while others against a different ID
  succeed — the ID is the variable, not the network or the CLI.

## Prerequisites

- The exact contract ID string being used
- Soroban RPC access: `https://soroban-testnet.stellar.org`
- Canonical deployment records:
  - `testnet/registry/contract-status.md`
  - `testnet/registry/last-verified.json`
  - `deployment-artifacts/contract-ids.txt`
  - `testnet/docs/network-config.md`
- A funded testnet identity for simulations

## Diagnosis

### Step 1: Validate the ID string itself

A Soroban contract address is a `C...` strkey. A malformed copy fails even
before the RPC is contacted:

```bash
# The CLI rejects invalid strkeys during parsing
stellar contract invoke \
  --simulate-only \
  --id "$CONTRACT_ID" \
  --network testnet \
  --source "$READER" \
  -- \
  get_info
```

An immediate parse/argument error (not an RPC round-trip failure) means the
string is truncated, contains whitespace, or is a different address type.
Note the related mix-ups:

| Prefix | Type | Invokable as a contract? |
|---|---|---|
| `C...` | Soroban contract address | yes |
| `G...` | classic account | no — this is where Bridgelet ephemeral accounts hold their *classic* XLM balances, not where their code lives |
| `S...` | secret/seed | never — and it should not be in any log |

### Step 2: Ask the network whether the contract instance exists

```bash
# Sanity check: the RPC is alive and its ledger is advancing
curl -fsS https://soroban-testnet.stellar.org \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getLatestLedger"}'
```

The authoritative existence check is an RPC `getLedgerEntries` call whose key
is the base64 XDR of `ScLedgerKey::Contract` for the ID. A present entry
includes the contract instance (or code); a null entry means nothing is
recorded at that address on this network. The exact key-encoding procedure is
owned by `testnet/tooling/contract-id-lookup-spec.md`. As a quick local
cross-check, `stellar contract ls` lists the contract IDs your CLI already
knows about — useful for spotting a profile mix-up, but not proof of
on-chain existence.

- **Entry present**: the contract exists. A "not found"-style failure here is
  a client/config issue (wrong RPC URL in one of several configured profiles)
  — compare `stellar network show` against `testnet/docs/network-config.md`.
- **Entry absent**: the ID resolves to nothing on this network. Continue to
  Steps 3–5.

### Step 3: Distinguish "not found" from "not initialized"

An existing-but-never-initialized `EphemeralAccount` instance answers
differently, and the difference matters:

```bash
stellar contract invoke \
  --simulate-only \
  --id "$CONTRACT_ID" \
  --network testnet \
  --source "$READER" \
  -- \
  get_info
```

| Result | Meaning |
|---|---|
| `Error(Contract, #2)` — `NotInitialized` | The instance **exists**; `initialize()` was never called. Not this runbook — complete initialization per `testnet/docs/getting-started.md` |
| RPC/host error before any contract code runs | The ID does not resolve — continue |

### Step 4: Rule out the wrong network

An ID is only meaningful on the network it was deployed to. A mainnet or
localnet `C...` string is a valid strkey on testnet — it simply has no ledger
entry there, producing exactly the "not found" symptom.

Check what the client is actually pointed at:

```bash
stellar network show testnet
echo "$STELLAR_NETWORK_PASSPHRASE"
```

Compare against `testnet/docs/network-config.md`:
passphrase `Test SDF Network ; September 2015`, RPC
`https://soroban-testnet.stellar.org`. If the ID came from a mainnet or
localnet environment, the fix is to use the testnet ID — and if you also
observed `tx_bad_auth` on submissions, see
`wrong-network-passphrase.md`; the two failures are the two halves of the same
config mix-up.

### Step 5: Check whether the ID is stale — resets and copied artifacts

Testnet is reset periodically. After a reset, **every account and contract ID
is gone** until `scripts/deploy-testnet.sh` is re-run and the IDs are
re-recorded (`testnet/docs/network-config.md`, `testnet/policies/reset-policy.md`).
A perfectly correct ID from before the reset now resolves to nothing.

Two project-specific staleness traps:

- **Artifact conflict.** `deployment-artifacts/contract-ids.txt` and
  `deployments/testnet.json` can hold IDs from an older deployment that
  conflicts with newer records. `testnet/examples/README.md` states the rule
  plainly: *a copied ID is not proof of availability* — verify before use.
- **Age.** `testnet/registry/last-verified.json` is a staleness mechanism
  (30-day threshold per `testnet/policies/reset-policy.md`), not proof of
  liveness. An ID whose "last verified" timestamp is older than the threshold
  must be re-verified, not trusted — see
  `testnet/tooling/contract-id-lookup-spec.md` for the lookup procedure.

### Step 6: Confirm the contract was ever deployed at all

Some contracts have **no testnet deployment by design of the current state**,
and no amount of re-verification will find them:

| Contract | Testnet status (see `testnet/registry/contract-status.md`) |
|---|---|
| `EphemeralAccount`, `SweepController` | Deployed per `testnet/docs/network-config.md`; verify live before each use |
| `ReserveContract` | Not deployed; also not wired into `EphemeralAccount` (hardcoded internal reserve) |
| `account_factory` (`AccountFactory`) | **Not deployed** — not built by `scripts/build.sh`, not in CI, not in the deploy script |

So the issue scenario of "calling a contract that was never deployed, e.g.
`account_factory`" is not hypothetical: any `C...` ID for the factory, whether
remembered from a local run or copied from a sibling project, will always
fail with a not-found-style error on testnet. Do not chase it. Instead:

- for multiple ephemeral accounts, follow the deploy-and-initialize loop in
  `testnet/examples/batch-account-creation-mock.md` (the documented stand-in
  for the factory);
- if you genuinely need the factory, the path is to build and deploy it
  yourself per `testnet/registry/account-factory-status.md`, then record the
  new ID in the registry — not to reuse an ID from elsewhere.

## Common Causes

| Cause | Clue | Fix |
|---|---|---|
| Malformed ID string | CLI parse error, no RPC round-trip | Re-copy the full `C...` string; strip whitespace |
| Stale ID after testnet reset | Every lookup empty; registry timestamps pre-date the reset | Redeploy via `scripts/deploy-testnet.sh`; update registry |
| ID copied from conflicting artifacts | Works in one environment, not another | Verify live per `testnet/registry/verify-contract-live.md`; prefer verified registry values |
| ID from another network | Valid strkey, no entry on testnet | Use testnet IDs; see Step 4 and `wrong-network-passphrase.md` |
| Never-deployed contract (`account_factory`, `ReserveContract`) | Fails identically for every ID of that contract | Expected; use documented alternatives (Step 6) |
| Wrong RPC URL in a secondary profile | Entry exists per one client, not another | Diff every configured network profile |

## Resolution

1. Establish which of the three causes applies via Steps 1–6.
2. For stale/reset IDs: redeploy, then update
   `testnet/registry/contract-status.md`, `last-verified.json`, and any
   deployment artifacts in the same change, per the registry's Update
   Process.
3. For never-deployed contracts: switch to the documented stand-in flow; do
   not file the failure as an outage.
4. Re-run the original invocation with `--simulate-only` first; only submit
   for real once simulation returns a result from the contract (success or a
   contract error code — either proves the ID resolves).

## Prevention Checklist

- [ ] Contract IDs are read from a verified registry lookup at startup, not
      hardcoded from docs or old transcripts
- [ ] CI validates IDs against a live `getLedgerEntries` before deployment
      gates
- [ ] Post-reset checklist includes redeploy → re-verify → update registry →
      update consumers
- [ ] Dashboards alert on registry staleness (> 30 days) before integrators
      hit not-found errors
- [ ] Environments pin the network profile once and derive the RPC URL,
      passphrase, and IDs from it

## Escalation

If a **verified-fresh** ID (checked via `testnet/registry/verify-contract-live.md`
within the same session) still fails to resolve, capture the ID, the
`getLatestLedger` output, the `getLedgerEntries` response, and the RPC URL in
use, and escalate per `testnet/runbooks/on-call-testnet.md` — that pattern
points at an RPC-side or deployment-side problem rather than a config error.

## Related Documentation

- `testnet/registry/verify-contract-live.md` — the live verification
  procedure
- `testnet/registry/contract-status.md` — deployment status of every
  contract
- `testnet/registry/account-factory-status.md` — why `account_factory` is
  never found on testnet
- `testnet/tooling/contract-id-lookup-spec.md` — authoritative lookup order
  and staleness thresholds
- `testnet/docs/network-config.md` — endpoints, passphrase, recorded IDs
- `testnet/examples/batch-account-creation-mock.md` — the factory stand-in
- `testnet/runbooks/wrong-network-passphrase.md` — the signature-side half of
  a wrong-network config
- `testnet/policies/reset-policy.md` — what a testnet reset invalidates

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-10-15 | Initial contract-not-found runbook |
