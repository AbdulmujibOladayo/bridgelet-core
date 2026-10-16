# Soroban RPC Fallback for Testnet

The primary endpoint is `https://soroban-testnet.stellar.org` (SDF-hosted). If it is down, a fallback provider can keep read-only work going.

## Candidate provider

| Provider | Endpoint | Status |
|---|---|---|
| Gateway.fm public testnet RPC | `https://soroban-rpc.testnet.stellar.gateway.fm` | Candidate, **not yet approved** |

Do not rely on a candidate until a maintainer has run the checks below. Public endpoints can change or rate-limit without notice.

## Verify before use

```bash
URL=https://soroban-rpc.testnet.stellar.gateway.fm
curl -s -X POST $URL -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getNetwork"}'
```

- `passphrase` must be exactly `Test SDF Network ; September 2015`.
- `getHealth` returns healthy and `getLatestLedger` is close to the primary's ledger.
- A read-only `get_nonce` call on the SweepController returns the same value as on the primary.

## Using it

Pass it with `--rpc-url` and the testnet passphrase, or set `STELLAR_SOROBAN_RPC_URL`. Do not auto-replay in-flight transactions after failing over.

See `testnet/runbooks/rpc-outage.md` for the fallback policy.
