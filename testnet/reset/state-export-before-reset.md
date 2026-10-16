# Exporting Testnet State Before a Reset

## Purpose

A redeploy destroys all on-chain state: every `EphemeralAccount` instance, every
recorded payment, the `SweepController` nonce, the controller's configured
destination and signer (see `testnet/policies/reset-policy.md`, "What is
destroyed"). Some of that state is worth keeping for **historical and debugging
reference**: the final state of the shared demo accounts, and any
edge-case account states that explain a past bug report.

This document describes a lightweight, read-only export to run **before** the
redeploy, so that context is written down instead of lost.

**This is a record, not a backup.** Nothing exported here can be restored onto
the new deployment. Contract state cannot be re-imported, and the new contracts
have new IDs.

## Status

| Statement | Basis |
|---|---|
| No Bridgelet contracts are recorded as deployed | `testnet/registry/contract-status.md` |
| `testnet/registry/known-test-accounts.md` lists only `TBD` placeholders | That document |
| The procedure below uses only read-only methods | `get_status`, `get_info`, `get_payment_count`, `is_expired`, `get_nonce` |
| Nothing here has been run against a live deployment | There is no live deployment recorded |

Until a deployment exists, a run of this procedure produces an empty export and
says so. That is the correct outcome, not a failure.

## Scope

| In scope | Out of scope |
|---|---|
| Final state of the known demo accounts | Restoring state after the reset |
| Edge-case accounts someone has flagged as worth keeping | Exporting integrators' own test accounts |
| The registry and deployment record as of the reset | Changing `scripts/` or the deploy workflow |
| Where the export is stored and what it must not contain | Mainnet |

## What to export

| Item | Source | Why keep it |
|---|---|---|
| Deployment record | `deployments/testnet.json`, `deployment-artifacts/contract-ids.txt` | Maps old IDs mentioned in issues to a deployment |
| Registry state | `testnet/registry/contract-status.md`, `wasm-hash-reference.md`, `last-verified.json` | Shows what was believed live, and when it was last checked |
| Demo account state | `get_info` for each account in `known-test-accounts.md` | The "final state" the demo accounts reached |
| Edge-case accounts | `get_info` for accounts listed in the request issue | Preserves the interesting state behind a past bug |
| Controller nonce | `get_nonce` on the `SweepController` | Explains nonce-desync reports against the old deployment |

## Procedure

### 1. Confirm the reset is approved

An export for an unapproved reset is harmless but pointless. Follow
`testnet/policies/who-can-request-a-reset.md` first.

### 2. Set up read-only access

```bash
export NETWORK=testnet
export EXPORT_DATE=$(date -u +%Y-%m-%d)
export EXPORT_DIR="testnet/reset/exports/${EXPORT_DATE}"
mkdir -p "${EXPORT_DIR}"
```

Use a throwaway identity with no authority over the contracts, for example
`stellar keys generate --global testnet-export`. Reads do not need the admin
key, and the admin key must not be used for an export.

### 3. Copy the records

```bash
cp deployments/testnet.json "${EXPORT_DIR}/deployments-testnet.json"
cp deployment-artifacts/contract-ids.txt "${EXPORT_DIR}/contract-ids.txt"
cp testnet/registry/last-verified.json "${EXPORT_DIR}/last-verified.json"
```

### 4. Snapshot the accounts

For each account ID, run the read-only calls and keep the raw output:

```bash
ACCOUNT_ID=<ACCOUNT_ID>

stellar contract invoke --id "${ACCOUNT_ID}" --network "${NETWORK}" \
  --source testnet-export -- get_info \
  > "${EXPORT_DIR}/account-${ACCOUNT_ID}-info.json"

stellar contract invoke --id "${ACCOUNT_ID}" --network "${NETWORK}" \
  --source testnet-export -- get_status \
  > "${EXPORT_DIR}/account-${ACCOUNT_ID}-status.txt"
```

If a call fails because the contract no longer exists, record that as the
result. Do not retry in a loop: a missing contract may mean a reset has already
happened (see `testnet/runbooks/rpc-outage.md`, section 6).

### 5. Snapshot the controller

```bash
stellar contract invoke --id <SWEEP_CONTROLLER_ID> --network "${NETWORK}" \
  --source testnet-export -- get_nonce \
  > "${EXPORT_DIR}/sweep-controller-nonce.txt"
```

### 6. Write the manifest

Create `${EXPORT_DIR}/MANIFEST.md` listing what was captured, the ledger
sequence at export time, who ran it, and which items could not be read and why.
An empty export still gets a manifest that says "no deployment recorded".

## What must not be in an export

An export is committed to a public repository, so
`testnet/policies/data-retention.md` applies in full:

| Never include | Why |
|---|---|
| Secret keys (`S...`), signing seeds, `.env` files | A public repository is not private |
| Anything that links an address to a real person or customer | Data-retention rule |
| Signatures belonging to someone else | Same |

Review the directory before committing it. `get_info` output contains
addresses and amounts only; if anything else appears, remove it.

## Where exports live and how long

| Question | Proposal |
|---|---|
| Location | `testnet/reset/exports/<UTC date>/`, committed to the repository |
| Naming | One directory per reset, named by UTC date |
| Retention | Kept; they are small text files. A maintainer may prune old ones |
| Linking | The changelog entry for the redeploy links to the directory |

## After the reset

Add a line to the redeploy's `testnet/docs/changelog.md` entry pointing at the
export directory, so a later reader investigating an old report can find the
state it refers to.

## Undecided

| Question | State |
|---|---|
| Should `testnet/reset/exports/` be committed, or attached to the reset issue instead? | Open; committed is proposed |
| Should the export be automated as a script? | Not proposed; `scripts/` is out of scope per `testnet/tooling/decision-log.md` |
| Which edge-case accounts are worth keeping? | Left to whoever raises the reset request |

## Related Documentation

- [`../policies/reset-policy.md`](../policies/reset-policy.md): what a redeploy
  destroys
- [`../policies/who-can-request-a-reset.md`](../policies/who-can-request-a-reset.md):
  the approval that precedes an export
- [`communication-template.md`](communication-template.md): the announcement
  that accompanies a reset
- [`../registry/known-test-accounts.md`](../registry/known-test-accounts.md): the
  demo accounts whose final state is exported
- [`../registry/verify-contract-live.md`](../registry/verify-contract-live.md):
  the read-only calls used here
- [`../policies/data-retention.md`](../policies/data-retention.md): what an
  export must not contain

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial procedure for exporting testnet state before a reset |
