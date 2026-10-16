# Checking Your Contract-ID Config Is Not Stale

After a testnet reset or redeploy, cached contract IDs point at contracts that no longer exist. Run this check before a test session, or on a schedule.

## 1. Compare with the published record

```bash
curl -s https://raw.githubusercontent.com/bridgelet-org/bridgelet-core/main/deployments/testnet.json
```

Check that `contracts.sweepController` and `contracts.ephemeralAccount` match your local config.

## 2. Confirm the contract answers

`SweepController` is initialized by the deploy script, so a read-only call is a reliable probe:

```bash
stellar contract invoke --id <SWEEP_CONTROLLER_ID> --network testnet \
  --source <ANY_FUNDED_KEY> -- get_nonce
```

Expected: a `u64` (for example `0`). An error saying the contract or its code cannot be found means the ID is stale.

## 3. If it is stale

1. Pull the latest `deployments/testnet.json`.
2. Update your local config and re-run the probe.
3. If the IDs are unchanged and the probe still fails, follow `testnet/runbooks/contract-not-found-error.md`.
