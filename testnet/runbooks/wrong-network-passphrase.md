# Runbook: Transaction Signed With the Wrong Network Passphrase

## Overview

Every Stellar transaction signature covers the network it was signed for. The
network passphrase is mixed into the transaction hash as
`network_id = SHA256(passphrase)`, and each signature is computed over
`SHA256(network_id || envelope_xdr)`. A signature produced against one
passphrase is therefore cryptographically invalid on any other network — even
when the same key, account, and operation are used.

This is one of the most common copy-paste mistakes when switching between
testnet and mainnet configs: a `.env` file, an SDK `networkPassphrase`
argument, or a `stellar network add` entry carries the wrong value, and every
submission from that environment fails.

Testnet passphrase (must match exactly, including the spaces around `;`):

```text
Test SDF Network ; September 2015
```

Mainnet:

```text
Public Global Stellar Network ; September 2015
```

Source of truth: `testnet/docs/network-config.md`.

## Symptom

- Transactions are rejected at submission with **no contract error code**.
- Horizon returns HTTP 400 with result code `tx_bad_auth`.
- Soroban RPC `sendTransaction` returns status `ERROR` with an `errorResultXdr`
  that decodes to `tx_bad_auth`; a later `getTransaction` shows `FAILED` with
  the same code.
- Simulation succeeds, submission fails — consistently, for every transaction
  from the affected environment.

## Why It Is Not a Contract-Level Authorization Failure

This is the distinction this runbook exists to make. A wrong-passphrase
failure and a contract auth failure look superficially similar ("my
authorization was rejected") but live at completely different layers:

| | Wrong network passphrase | Contract-level auth failure |
|---|---|---|
| Failing layer | Network envelope signature verification | Contract code |
| Result code | `tx_bad_auth` (transaction-level) | `Error(Contract, #N)` or a host trap |
| Contract code executed? | **No** — rejected before any invocation runs | Yes |
| Simulation outcome | **Succeeds** (simulation does not verify signatures) | May fail with the contract error |
| Typical Bridgelet example | any call to any contract | `UnauthorizedDestination` (13), `AuthorizationFailed` (3), or an `ed25519_verify` trap in `execute_sweep` |

Key consequences:

- You will **never** see `Error(Contract, #N)` for this failure. If your
  failure carries a contract error code, the passphrase was fine — go to the
  runbook for that error instead (`failed-sweep-signature.md`,
  `nonce-desync.md`, etc.).
- `UnauthorizedDestination` (controller code 13) and the `ed25519_verify` trap
  in the sweep digest are **not** caused by a wrong network passphrase. The
  sweep digest (`SHA256(destination ‖ nonce ‖ controller_id)` — see
  `docs/SIGNATURE_FORMAT.md`) does not contain the passphrase at all; it is
  the *outer transaction* signature that breaks.
- A contract ID from another network does not produce this error either — it
  produces a "contract not found" style failure. See
  `contract-not-found-error.md`.

## Prerequisites

- The exact transaction (or at least its environment config) that failed
- Access to the Soroban RPC and Horizon endpoints you submitted against
- A funded testnet identity for test submissions
- Network values from `testnet/docs/network-config.md`

## Diagnosis

### Step 1: Confirm the failure is transaction-level

```bash
stellar transaction get <TX_HASH> --network testnet --output json
# or via RPC:
curl -fsS https://soroban-testnet.stellar.org \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getTransaction",
       "params":{"hash":"<TX_HASH>"}}'
```

Decode `errorResultXdr`. If the transaction-level code is `tx_bad_auth` (or
`tx_insufficient_fee` variants never apply here), continue. If you see an
operation-level Soroban diagnostic event carrying `Error(Contract, #N)`, stop:
the passphrase is not your problem.

### Step 2: Reproduce with simulation

```bash
stellar contract invoke \
  --simulate-only \
  --id "$SWEEP_CONTROLLER_ID" \
  --network testnet \
  --source testnet-investigator \
  -- \
  get_nonce
```

If simulation succeeds while real submission of the equivalent signed
transaction fails with `tx_bad_auth`, that combination is the fingerprint of a
passphrase mismatch. Simulation skips signature verification, which is exactly
why it cannot catch this.

### Step 3: Inspect the configured passphrase

```bash
# What does the CLI think "testnet" is?
stellar network show testnet
```

For SDK/config-driven flows, print the value actually in use:

```bash
echo "$STELLAR_NETWORK_PASSPHRASE"
cat .env 2>/dev/null | grep -i passphrase
```

Compare character-by-character with
`testnet/docs/network-config.md`. Common defects:

| Defect | Example |
|---|---|
| Mainnet passphrase in a testnet config | `Public Global Stellar Network ; September 2015` |
| Missing space before `;` | `Test SDF Network; September 2015` |
| Missing space after `;` | `Test SDF Network ;September 2015` |
| Trailing whitespace/newline in a config value | `Test SDF Network ; September 2015 ` |
| Typo'd network name | `Test SDF Network ; September 2016` |

The passphrase must match **exactly**. There is no prefix or fuzzy matching.

### Step 4: Check whether the signing and sending environments differ

A frequent shape of this bug is a CI matrix or a shared signing service where
one leg signs with mainnet config while another submits to the testnet RPC:

- transaction built and signed with `--network mainnet` (or an SDK
  `TransactionBuilder` with `Networks.PASSPHRASE`), then broadcast to
  `https://soroban-testnet.stellar.org`;
- signer tool configured once with mainnet values while relayers run testnet;
- a locally re-signed envelope attached to a submission path that assumes the
  original signature applies to the target network.

Signatures are **not portable between networks**. There is no re-broadcast
fix; the transaction must be rebuilt and re-signed with the target network's
passphrase.

### Step 5: Rule out the adjacent failure — source account missing on target
network

If the same keypair was never funded on testnet, the network rejects with
`tx_no_account` rather than `tx_bad_auth`, because the source account does not
exist. That is a different fix (fund via Friendbot, see
`testnet/faucet/README.md`), not a passphrase problem.

```bash
stellar keys fund <YOUR_TESTNET_ADDRESS> --network testnet
```

## Common Causes

| Cause | Where it hides |
|---|---|
| `.env` copied from a mainnet setup | `STELLAR_NETWORK_PASSPHRASE` |
| SDK hardcodes `Networks.PASSPHRASE` (mainnet) in a testnet build | transaction builder code |
| `stellar network add` entry with a subtly wrong passphrase | `~/.config/stellar/` CLI config |
| Signer runs in a different environment than the relayer | `tools/sweep-signer` config vs. relayer config |
| Whitespace/newline in a config file value | dotenv files, YAML/JSON configs |

## Resolution

1. Correct the passphrase to `Test SDF Network ; September 2015` everywhere it
   is configured (CLI network entry, `.env`, SDK constant, signer config).
2. Rebuild and re-sign the transaction. Do not attempt to reuse the old
   signature.
3. Submit and confirm: `getTransaction` returns `SUCCESS`, and the hash the
   CLI printed matches what the network recorded.
4. Re-run the simulation-vs-submission check from Step 2 as a regression
   check for the whole environment, not just the one transaction.

## Prevention Checklist

- [ ] Single source of truth for the passphrase —
      `testnet/docs/network-config.md` — referenced, not copied, by each
      environment
- [ ] CI asserts the configured passphrase equals the expected literal before
      running
- [ ] Signer and relayer configs are generated from the same file
- [ ] A smoke test that simulates **and** submits one no-op read before any
      change window (simulation alone proves nothing about signatures)
- [ ] Never log or cache signed envelopes across networks

## Escalation

If the passphrase is confirmed correct, the transaction was signed after any
config fix, and `tx_bad_auth` still appears for a funded account, capture the
`errorResultXdr`, the passphrase string (it is public), and the transaction
envelope, and escalate per `testnet/runbooks/on-call-testnet.md`.

## Related Documentation

- `testnet/docs/network-config.md` — canonical passphrase and endpoints
- `testnet/runbooks/contract-not-found-error.md` — the other direction of the
  same config mix-up (IDs from the wrong network)
- `testnet/runbooks/failed-sweep-signature.md` — sweep-digest signature
  failures (a different layer; produce traps, not `tx_bad_auth`)
- `testnet/faucet/README.md` — funding a testnet account (`tx_no_account`)
- `docs/SIGNATURE_FORMAT.md` — what the sweep digest does and does not
  contain

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-10-15 | Initial wrong-network-passphrase runbook |
