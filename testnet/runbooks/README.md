# Testnet Runbooks Index

## Overview

This directory contains operational runbooks for diagnosing and resolving issues with Bridgelet contracts on Stellar Testnet. Each runbook covers a specific failure scenario with step-by-step investigation procedures.

## Runbook Index

| Runbook | Scenario | Related Contracts |
|---|---|---|
| [`account-factory-batch-failure.md`](account-factory-batch-failure.md) | `AccountFactory::batch_initialize` partial failure | AccountFactory, EphemeralAccount |
| [`multi-payment-edge-cases.md`](multi-payment-edge-cases.md) | Repeated `record_payment` for same asset | EphemeralAccount |
| [`failed-sweep-signature.md`](failed-sweep-signature.md) | `SweepController::execute_sweep` signature rejection | SweepController, EphemeralAccount |
| [`stuck-ephemeral-account.md`](stuck-ephemeral-account.md) | EphemeralAccount won't sweep or expire | EphemeralAccount, SweepController |
| [`wrong-network-passphrase.md`](wrong-network-passphrase.md) | Transaction rejected with `tx_bad_auth`; signed for the wrong network | Network layer (any contract) |
| [`insufficient-reserve-errors.md`](insufficient-reserve-errors.md) | Reserve-related failures: network base reserve vs. internal reserve tracking | EphemeralAccount, network layer |
| [`event-not-appearing-in-horizon.md`](event-not-appearing-in-horizon.md) | Successful transaction but expected contract event missing from Horizon | EphemeralAccount, SweepController |
| [`contract-not-found-error.md`](contract-not-found-error.md) | RPC cannot resolve a contract ID (stale ID, wrong network, never deployed) | Any contract |

## Quick Start

1. **Identify the symptom** from the table above
2. **Open the corresponding runbook**
3. **Follow the investigation steps** in order
4. **Escalate** if root cause not found

## Common Prerequisites

All runbooks assume:
- Testnet identity with XLM for queries (`stellar keys generate --global testnet-investigator`)
- Stellar CLI configured for testnet
- Contract IDs from `testnet/registry/contract-status.md`
- Access to Soroban RPC: `https://soroban-testnet.stellar.org`

## Runbook Template

Each runbook follows this structure:
1. **Symptom** - What the user observes
2. **Prerequisites** - Required access/tools
3. **Investigation Steps** - Ordered diagnostic procedures
4. **Common Causes** - Known failure modes
5. **Resolution** - Fix procedures
6. **Escalation** - When to contact core team

## Contributing

To add a new runbook:
1. Create `testnet/runbooks/<name>.md` following the template
2. Add entry to this index table
3. Cross-reference in `testnet/registry/contract-status.md`

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-24 | Initial runbook index |