# Testing Against Contracts That May Be Upgraded Mid-Run

## Purpose

`EphemeralAccount::upgrade(new_wasm_hash)` replaces a contract's code in place.
The contract ID does not change, nothing is emitted, and the next call simply
runs different code. If that happens while your test suite is running, tests
that passed a minute ago start failing for reasons that look like your own bugs.

This page describes how a consuming project's test suite should **pin the
expected WASM hash, check it at startup, and fail fast with a clear message**,
instead of producing confusing downstream failures.

## Status

**Reasoned from the contract source and the upgrade runbook; not observed.** No
contract is recorded as deployed (`../registry/contract-status.md`) and no
upgrade has been recorded (`../registry/upgrade-history.md`). The CLI commands
below are those named in `../runbooks/upgrade-verification.md`; support varies by
CLI version and none has been run against a live deployment.

## What can change underneath you

| Event | Contract ID | State | Signal you get | Covered here |
|---|---|---|---|---|
| In-place `upgrade()` | Same | Kept | **None.** No event is emitted | Yes |
| Redeploy | **New** | **Destroyed** | Old ID stops resolving | Partly; see below |
| SDF testnet reset | Gone | Destroyed | Ledger sequence goes backwards | Partly; see below |

Only `EphemeralAccount` has an `upgrade()` function in the current source.
`SweepController`, `ReserveContract` and `AccountFactory` have none. An upgrade
is gated by the `admin` stored at `initialize()`, so it affects instances whose
admin key was used, and it changes only the instance it is called on, not the
others.

The danger is the silence: `upgrade()` emits no event and has no timelock, so
the only evidence is the transaction history and the contract's current code
hash.

## The check: pin a hash, verify at startup

### 1. Record the hash you tested against

When you first get the suite green against a verified deployment, record the
deployed WASM hash and keep it in version control, next to the contract ID:

```json
{
  "network": "testnet",
  "contracts": {
    "ephemeralAccount": {
      "id": "<EPHEMERAL_ACCOUNT_ID>",
      "wasmHash": "<64-hex-character-hash>"
    }
  }
}
```

A hash is public information, so committing it is safe. Never put a secret key,
seed or credential in this file.

### 2. Read the live hash

```bash
stellar contract info hash \
  --contract-id <EPHEMERAL_ACCOUNT_ID> \
  --network testnet
```

`contract info hash` is not available in every CLI version. If yours lacks it,
use the SDK or RPC to read the contract's executable from its ledger entry. Do
not guess a ledger key, and do not substitute the hash of a local build: that is
the hash of what you *built*, not of what is *deployed*.

### 3. Compare, and stop on a mismatch

```bash
#!/usr/bin/env bash
set -euo pipefail

EXPECTED="$(jq -r '.contracts.ephemeralAccount.wasmHash' pinned-contracts.json)"
CONTRACT_ID="$(jq -r '.contracts.ephemeralAccount.id' pinned-contracts.json)"

ACTUAL="$(stellar contract info hash --contract-id "$CONTRACT_ID" --network testnet)"

if [ "$ACTUAL" != "$EXPECTED" ]; then
  cat >&2 <<MSG
Contract code changed: this test run is invalid.
  contract: $CONTRACT_ID
  pinned:   $EXPECTED
  deployed: $ACTUAL
The contract was probably upgraded in place. Review the change, run the
upgrade-verification checks, and update pinned-contracts.json in a reviewed
commit. Do not re-run the suite until you have.
MSG
  exit 3
fi
```

Run this **before** the first test and make a non-zero exit stop the whole run.
Use a distinct exit code so CI can report "contract changed" separately from
"tests failed".

## Failing clearly

A good failure message answers four questions without anyone opening a log:

| Question | Answer in the message |
|---|---|
| What changed? | The contract ID, the pinned hash and the deployed hash |
| Is it my code? | No: the dependency changed |
| What should I do? | Review the upgrade, then update the pin deliberately |
| Is the run usable? | No: results are void, do not read them |

Report it as an **environment failure**, not a test failure, so a flaky-test
dashboard does not absorb it.

## Do not auto-update the pin

The tempting shortcut is to have the suite overwrite the pin when it differs.
That turns the check into a no-op: it would accept a malicious or accidental
upgrade as the new normal. Updating the pin should be a human decision recorded
in a commit, after comparing the new interface and behaviour with what your tests
assume.

## Upgrades during a run

A startup check proves the code was right when the run began. It does not prove
nothing changed since. For a long suite:

| Technique | Cost | Catches |
|---|---|---|
| Re-check the hash at the end of the run | One read | An upgrade at any point during the run, reported after the fact |
| Re-check between test files | One read per file | An upgrade within one file's window |
| Re-check on any unexpected failure | One read, on failure only | Distinguishes "contract changed" from "real bug" at the moment it matters |

The third is the cheapest useful one: when a test fails in a way you do not
recognise, read the hash before you spend an hour debugging. If it differs, the
failure is explained.

If a run does span an upgrade, **discard its results**. Tests before and after
ran against different code, so neither half is trustworthy on its own.

## Pair the hash with an interface check

A hash tells you the code changed; it does not tell you whether the change
matters to you. Where your CLI supports it, also record the deployed interface:

```bash
stellar contract info interface \
  --contract-id <EPHEMERAL_ACCOUNT_ID> \
  --network testnet
```

Compare it against the interface your suite was written for. An unchanged
interface with a changed hash can still mean changed behaviour, so keep the hash
check; a changed interface is a stronger signal still.

## Redeploys and resets

These look different from an upgrade, and the same startup step should handle
them:

- **A redeploy** gives new contract IDs. The old ID stops resolving, and the
  read above fails rather than returning a different hash. Treat "contract not
  found" at startup as its own clear message (see
  [`../runbooks/contract-not-found-error.md`](../runbooks/contract-not-found-error.md)).
- **An SDF reset** wipes everything. The network's latest ledger is lower than
  one you recorded. Store the ledger you last ran against and compare.

See [`../policies/reset-policy.md`](../policies/reset-policy.md) for what each
destroys.

## What not to do

| Mistake | Why it hurts |
|---|---|
| Pin only the contract ID | It does not change on an upgrade, so the check passes while the code differs |
| Compare against a locally built WASM's hash | That is your build, not the deployment |
| Subscribe to an "upgraded" event | None is emitted |
| Overwrite the pin automatically | It accepts any change as correct |
| Keep running after a mismatch | Every later result is about unknown code |

## Undecided

| Question | State |
|---|---|
| Whether `contract info hash` is available in the pinned CLI version | Needs checking against the version in use |
| Whether this project should publish a machine-readable pin file | Not proposed; the registry holds the hashes in Markdown |
| How quickly an upgrade is announced | No notice commitment exists; see the reset policy |

## Related Documentation

- [`../runbooks/upgrade-verification.md`](../runbooks/upgrade-verification.md):
  verifying an upgrade, including the code-hash commands used here
- [`../registry/upgrade-history.md`](../registry/upgrade-history.md): the log of
  recorded upgrades, currently empty
- [`../registry/wasm-hash-reference.md`](../registry/wasm-hash-reference.md): the
  hash each contract is recorded as running
- [`../examples/integration-test-harness-notes.md`](../examples/integration-test-harness-notes.md):
  the rest of a live-testnet suite's design
- [`../security/admin-key-hygiene.md`](../security/admin-key-hygiene.md): who can
  call `upgrade()`, and why it is silent
- [`../policies/reset-policy.md`](../policies/reset-policy.md): redeploys and
  resets

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial guidance for pinning and checking the WASM hash in a test suite |
