---
ecip: 1116
title: Olympia Base Fee Development Funding Mechanism — Implementation Spec
status: Implementation
type: Standards Track
category: Core
requires: 1111, 1112
author: Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)
created: 2025-12-14
updated: 2026-03-08
license: CC0-1.0
---

# ECIP-1116: Olympia Base Fee Development Funding Mechanism — Implementation Spec

## Overview

ECIP-1116 defines a protocol-level development funding mechanism that allocates 5% of the transaction basefee to the Olympia Treasury (ECIP-1112), while allocating 95% of the basefee, 100% of block rewards, and 100% of priority fees (tips) to block producers.

This is a **consensus-layer change** requiring a **second hard fork** after Olympia. It is deliberately deferred to Phase 5.

## Current Status

**Deferred to Phase 5 (second consensus fork).** Too risky to hardcode the split ratio without empirical data from Phase 4 (ECIP-1115 experimentation). The contract-layer experimentation must prove which parameters are appropriate before protocol embedding.

## Activation

| Network | Block Height | Status |
|---------|-------------|--------|
| Mordor Testnet | TBD | Deferred — pending Phase 4 data |
| ETC Mainnet | TBD | Deferred — pending Phase 4 data |

**Prerequisite:** ECIP-1115 observable data must validate the 5%/95% split ratio (or an alternative) before this ECIP proceeds to implementation.

## Base Fee Allocation Semantics

For each block after activation:

```
TreasuryPortion      = floor(TotalBaseFee × 5 / 100)
BlockProducerPortion = TotalBaseFee − TreasuryPortion
```

- Applies to aggregate basefee per block (not per-transaction)
- Integer arithmetic with remainder to block producer
- Fixed ratio — not configurable or governance-adjustable
- No effect on block rewards or priority fees

## Context: Why Phase 5

**Phase 1 (Olympia):** 100% basefee → Treasury. This adds ~1 gwei per transaction — less than 0.01% of miner income at current activity levels. Miners retain 100% of block rewards + 100% of tips.

**Phase 4 (ECIP-1115):** Contract-layer experimentation with smoothing curves, fee amounts, and strategies around block reward decline cycles. Empirical evidence, not opinions.

**Phase 5 (this ECIP):** Only after Phase 4 data proves appropriate parameters. By this point the fee market will be relevant (Olympia has been live, Type-2 transactions generating real basefee data).

The key insight: hardcoding a split ratio at consensus before we have fee-market data is premature. Better to experiment at the contract layer first, then embed proven parameters at protocol level.

## Client Implementation Requirements

When activated, all 3 clients must implement identical allocation arithmetic:
- core-geth: `consensus/ethash/consensus.go` → modify `Finalize()` basefee handling
- besu-etc: `MainnetTransactionProcessor` → split basefee allocation
- fukuii: `BlockRewardCalculator` → split basefee allocation

## Backwards Compatibility

ECIP-1116 requires a coordinated network upgrade (hard fork). Nodes that do not upgrade will diverge at activation. Legacy Type-0 and Type-1 transactions remain unchanged.

## Relationship to Other ECIPs

- **ECIP-1111:** Provides the basefee mechanism. Phase 1 sends 100% to Treasury.
- **ECIP-1112:** Treasury receives the 5% allocation.
- **ECIP-1115:** Contract-layer experimentation that must precede this protocol embedding.
- **ECIP-1122:** Protocol-native miner distribution (supersedes ECIP-1120) — paired with this ECIP in Phase 5.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)
