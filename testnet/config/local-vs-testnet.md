# Local Sandbox vs Real Testnet

`scripts/` has no local-sandbox entry point (there is no `test-local.sh`), so a local network is an unsupported, do-it-yourself setup. This page lists what changes if you try one.

| Setting | Real testnet | Local sandbox (stellar quickstart) |
|---|---|---|
| Network passphrase | `Test SDF Network ; September 2015` | `Standalone Network ; February 2017` |
| Soroban RPC URL | `https://soroban-testnet.stellar.org` | `http://localhost:8000/rpc` |
| Horizon URL | `https://horizon-testnet.stellar.org` | `http://localhost:8000` |
| Funding | Friendbot at `friendbot.stellar.org` | Local friendbot at `http://localhost:8000/friendbot` |
| Contract IDs | In `deployments/testnet.json` | None. You must deploy everything yourself |
| Ledger state | Shared, periodically reset by SDF | Yours alone, lost when the container stops |

## Practical differences

- Nothing in `deployments/testnet.json` or `testnet/registry/` applies to a local network.
- A transaction signed with one passphrase is rejected on the other. See `testnet/runbooks/wrong-network-passphrase.md`.
- A sandbox may close ledgers at a different rate than testnet, so expiry timings from `testnet/config/expiry-ledger-testing.md` will not match.
- Results on a sandbox do not prove behavior on testnet. Re-run key flows on testnet before relying on them.
