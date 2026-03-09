---
ecip: 1122
title: Olympia Protocol-Native Miner Distribution — Implementation Spec
status: Implementation
type: Standards Track
category: Core
requires: 1111, 1112, 1115, 1116
supersedes: 1120
author: Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)
created: 2026-03-08
updated: 2026-03-08
license: CC0-1.0
---

# ECIP-1122: Olympia Protocol-Native Miner Distribution — Implementation Spec

## Overview

ECIP-1122 defines protocol-native miner distribution for the Olympia framework — embedding a miner distribution curve at the consensus layer that determines how the miner portion of basefee revenue (as split by ECIP-1116) is distributed to block producers.

This ECIP supersedes ECIP-1120 (istora, original author) to align with the Olympia framework's staged governance model and incorporate findings from contract-layer experimentation (ECIP-1115).

**Stage:** Phase 5 (second consensus fork). Paired with ECIP-1116 (basefee split).

## Current Status

**Deferred to Phase 5.** Requires:
1. ECIP-1115 contract-layer experimentation (Phase 4) to produce empirical data
2. ECIP-1116 basefee split activation (Phase 5, same fork)
3. Observable data validating which distribution parameters are appropriate

## Supersession of ECIP-1120

ECIP-1120's original author is istora. ECIP-1122 supersedes ECIP-1120 to:
- Align with the Olympia framework's staged rollout (Stages 1-5)
- Incorporate empirical findings from ECIP-1115 contract-layer experimentation
- Ensure compatibility with the Treasury → Executor → SanctionsOracle pipeline
- Coordinate activation with ECIP-1116 in the same hard fork

The specific distribution parameters will be determined after Phase 4 data is available. ECIP-1122 will be fully specified at that time.

## Relationship to ECIP-1115

ECIP-1115 (Phase 4) provides the **contract-layer experimentation framework**:
- `SmoothingModule`: tests different L-curves, window lengths, allocation fractions
- `MinerRewardModule`: tests different miner identification and payout methods
- All parameters adjustable via OIP without hard fork

ECIP-1122 (Phase 5) **embeds proven parameters at consensus**:
- Only proceeds after ECIP-1115 data validates the model
- Removes governance dependency for distribution mechanics
- Hardcodes the distribution curve that experimentation proved optimal

## Relationship to ECIP-1116

Both ECIP-1116 and ECIP-1122 activate in the same second hard fork:
- **ECIP-1116:** Splits basefee 5%/95% between Treasury and miners
- **ECIP-1122:** Defines how the 95% miner portion is distributed

They are complementary — ECIP-1116 determines the amount, ECIP-1122 determines the distribution pattern.

## Client Implementation Requirements

When activated (Phase 5), all 3 clients must implement identical distribution logic:
- core-geth: `consensus/ethash/consensus.go`
- besu-etc: `MainnetTransactionProcessor`
- fukuii: `BlockRewardCalculator`

## Implementation Status

| Component | Status |
|-----------|--------|
| Distribution curve specification | Pending Phase 4 data |
| Client implementations | Not started |
| Cross-client test vectors | Not started |

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)
