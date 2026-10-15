# Finding Which Source Version Is Live on Testnet

## Purpose

An integrator sees a contract ID and needs to answer: *what code is that, and
which commit of this repository does it correspond to?* Without an answer, a
behaviour difference between the documentation and the deployment cannot be
told apart from a bug.

This page describes **how to correlate a live testnet contract ID back to
source**, step by step, and how much to trust each step. It does not change or
replace the deployment record, and it does not claim a particular version is
live.

## Status

**A process, not a result.** No contract is recorded as deployed
(`../registry/contract-status.md`), and the identifiers in
`deployments/testnet.json` are unverified. The git history shown below was read
from this repository; nothing was read from the chain.

The simple recipe, "check the contract ID against `deployments/testnet.json`,
then run `git blame` on that file", is a useful starting point and a weak
conclusion. [Why git blame is only an upper bound](#why-git-blame-is-only-an-upper-bound)
explains why, and the steps below add the evidence that makes it reliable.

## Evidence, from strongest to weakest

| # | Evidence | Strength | What it proves |
|---|---|---|---|
| 1 | The deployed WASM hash equals the hash of a build of a specific commit | Strong | That the live code is the code at that commit, when the build reproduces |
| 2 | A changelog entry names a deployed commit and a WASM hash, and the live hash matches | Strong | The declared commit, confirmed by the chain |
| 3 | `wasm-hash-reference.md` maps the live hash to a commit | Medium | The same, if the table is populated (it is currently `TBD`) |
| 4 | The commit that last changed `deployments/testnet.json` | Weak | A commit **no earlier than** the deployment, see below |
| 5 | The `deployedAt` timestamp, matched to commit dates | Weak | A rough window only |

Always lead with evidence 1 or 2. Use 4 and 5 only to pick a candidate commit to
test.

## Procedure

### 1. Confirm the ID is the one you think it is

Check the ID in your hand against the places that publish it, and treat any
disagreement as a finding, not a detail:

| Source | Role |
|---|---|
| `../registry/contract-status.md` | The canonical status table |
| `network-config.md` | The published IDs |
| `deployments/testnet.json` | Written by the deploy script |
| `deployment-artifacts/contract-ids.txt` | Committed by hand |
| `changelog.md` | Dated entries, with a deployed commit |

Then confirm the ID resolves on the network using
[`../registry/verify-contract-live.md`](../registry/verify-contract-live.md). An
ID that does not resolve has no version to find: testnet may have been reset or
the contracts redeployed.

### 2. Read the deployed WASM hash

```bash
stellar contract info hash \
  --contract-id <CONTRACT_ID> \
  --network testnet
```

This is the identity of the code actually running. Not every CLI version has this
subcommand; if yours does not, read the contract's executable from its ledger
entry through the SDK or RPC. Do not substitute the hash of a local build: that
is what you built, not what is deployed.

### 3. Look the hash up

Check, in order:

1. `changelog.md`, for an entry that lists this hash and a deployed commit.
2. `../registry/wasm-hash-reference.md`, for a row mapping the hash to a commit.
3. `deployments/testnet.json`, whose `config.ephemeralAccountWasmHash` records the
   hash the deploy script used for `EphemeralAccount`.

A match in any of these names a candidate commit. Continue to step 4 to confirm
it.

### 4. Confirm by rebuilding

```bash
git checkout <CANDIDATE_COMMIT>
./scripts/build.sh
```

Compute the hash of the resulting WASM for the contract you are checking and
compare it with step 2. The hash is the SHA-256 of the exact bytes that were
uploaded, so the build must produce identical bytes.

A **match** is strong evidence. A **mismatch is not proof of a different
source**: a build is only reproducible with the same toolchain. The versions that
matter are recorded in
[`../security/dependency-audit-notes.md`](../security/dependency-audit-notes.md)
(`soroban-sdk` 22.0.0, `stellar-cli` 23.4.1) and include the Rust compiler and any
optimisation step. If the hash does not match, say so and try the adjacent
commits before concluding anything.

### 5. Narrow the candidates with git history

When no document names a commit, the history of the deployment record gives a
shortlist:

```bash
git log --format='%h %ad %s' --date=short -- deployments/testnet.json
```

Each commit that changed the file is a point at which the recorded deployment
changed. Start your rebuild at the commit that first contains the ID you care
about, then try the commits just before it, because the code may predate the
commit that recorded it (see the next section).

### 6. Check for in-place upgrades

`EphemeralAccount::upgrade()` changes the code **without changing the ID**, and
emits no event. A hash that differs from the one in the changelog may therefore
mean an upgrade, not a different deployment. Consult
[`../registry/upgrade-history.md`](../registry/upgrade-history.md) and
[`../runbooks/upgrade-verification.md`](../runbooks/upgrade-verification.md).

## Why git blame is only an upper bound

`git blame` or `git log` on `deployments/testnet.json` tells you which commit
last **recorded** a deployment. It does not tell you which commit was
**deployed**:

- The deploy script writes the file from whatever working tree it ran in. That
  tree may have had uncommitted changes, or may be older than the commit that
  later saved the file.
- The file records `deployedAt` and the contract IDs but **not the commit SHA**.
  The reset policy lists adding it as a requirement the script does not yet meet.
- A commit can change the file without redeploying everything (see the example).

So the commit you find is a bound, not an answer: the deployed code is that
commit or work that came before it, never later work. That makes it a good
starting candidate for step 4, and nothing more.

## Worked example: reading this repository's own history

This is a reading of the repository's git history and files. It says nothing
about what is on the chain, and it is included to show how records can
disagree.

```bash
git log --format='%h %ad %s' --date=short -- deployments/testnet.json
```

The history of that file in this repository shows four commits, including:

| Commit | Date | Message |
|---|---|---|
| `741aec2` | 2026-07-12 | feat: contracts deployed |
| `e7efd55` | 2026-10-11 | fix: redeployment of contracts except ephemral |

Read against the other records:

| Record | `EphemeralAccount` | `SweepController` |
|---|---|---|
| `changelog.md`, entry 2026-07-12 (commit `741aec2`) | `CB7Z22TX...` | `CBEU4X5M...` |
| `deployments/testnet.json` and `contract-ids.txt` (current) | `CB7Z22TX...` | `CAJB4RRM...` |

What the evidence supports:

- The `EphemeralAccount` ID is the same in both records, and so is its WASM hash
  (`5e667ea0...`), which fits the commit message "except ephemeral".
- Commit `e7efd55` changed `deployments/testnet.json`, `contract-ids.txt` and
  `scripts/deploy-testnet.sh` together. The commit that records the deployment is
  therefore also a commit that changed the deploy tooling, which is exactly why
  it is a bound and not proof of the deployed code.
- The other three IDs differ between the changelog and the current record. The
  changelog entry was not updated for the later redeployment.
- No `testnet-v*` tag exists in the repository, so there is no tag to resolve
  (`../policies/versioning-scheme.md` describes the scheme but none has been
  applied).

What it does **not** establish: which set of IDs, if either, is live. That
needs steps 1 and 2 against the network.

## Recording what you find

If you complete the correlation, record it where the next person will look:

| Where | What to add |
|---|---|
| `../registry/wasm-hash-reference.md` | The hash, contract ID, commit and date |
| `../registry/last-verified.json` | A fresh timestamp, only after verifying on-chain |
| `changelog.md` | A dated entry if you found an unrecorded redeployment |

Do not write a version into these files from the git history alone. Record only
what the chain and a reproducing build have confirmed.

## Undecided

| Question | State |
|---|---|
| Whether the deploy script should record the commit SHA | Not changed here; `scripts/` is out of scope |
| Whether deployments should be tagged `testnet-v1.x` | Described in the versioning scheme; not applied |
| Whether builds are reproducible across machines | Not tested |

## Related Documentation

- [`changelog.md`](changelog.md): declared deployments and their commits
- [`network-config.md`](network-config.md): the published IDs
- [`../registry/wasm-hash-reference.md`](../registry/wasm-hash-reference.md): the
  hash-to-commit table
- [`../registry/verify-contract-live.md`](../registry/verify-contract-live.md):
  confirming an ID resolves
- [`../registry/upgrade-history.md`](../registry/upgrade-history.md): in-place
  upgrades
- [`../policies/versioning-scheme.md`](../policies/versioning-scheme.md): the
  deployment tag scheme
- [`../policies/reset-policy.md`](../policies/reset-policy.md): why the commit
  SHA is missing from the record
- [`../security/dependency-audit-notes.md`](../security/dependency-audit-notes.md):
  toolchain versions needed to reproduce a build

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial procedure for correlating a testnet contract ID to a source version |
