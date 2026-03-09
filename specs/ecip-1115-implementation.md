---
ecip: 1115
title: Olympia L-Curve Smoothing for Long-Term Network Security — Implementation Spec
status: Implementation
type: Meta
requires: 1111, 1112, 1113, 1114
author: Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)
created: 2025-11-15
updated: 2026-03-08
license: CC0-1.0
---

# ECIP-1115: Olympia L-Curve Smoothing — Implementation Spec

## Overview

ECIP-1115 introduces an optional governance-layer mechanism that applies a deterministic L-curve to a governance-selected portion of Treasury-held basefee revenue across a future window. The mechanism reshapes the timing of basefee-derived incentives to provide a more predictable miner revenue profile, if governance elects to activate it.

Smoothing is entirely optional, operates at the contract layer (not consensus), and can be activated, adjusted, or disabled through the OIP process without a hard fork.

**Stage:** Phase 4 — Miner Distribution Experimentation. This exploration matters most once the fee market is relevant. The goal is empirical data, not opinions.

## Smoothing Model

For block `k`, let `B_k` denote the basefee revenue credited to the Treasury. Governance specifies a fraction `f` (0 ≤ f ≤ 1):

```
R_k = f · B_k                          (smoothed portion)
A(k + j) = R_k · L(j)    for j = 1…N   (payout schedule)
```

Invariants:
- **Normalization:** Σ L(j) = 1
- **Determinism:** `L(j)` must be fully on-chain computable
- **Non-negativity:** L(j) ≥ 0
- **Configurability:** `N`, `f`, and `L(j)` adjustable via OIP only

## Contract Architecture

### SmoothingModule

```solidity
interface ISmoothingModule {
    /// @notice Compute the intended allocation for a given block offset
    function computeAllocation(uint256 blockNumber, uint256 offsetJ) external view returns (uint256);

    /// @notice Get the current smoothing parameters
    function getParams() external view returns (uint256 windowN, uint256 fractionBps, bytes memory curveParams);

    /// @notice Update parameters via governance
    function updateParams(uint256 windowN, uint256 fractionBps, bytes calldata curveParams) external;
}
```

The SmoothingModule:
- Receives governance-approved parameters (N, f, L(j))
- Computes intended allocations from historical `B_k` values
- Produces deterministic outputs for Treasury withdrawal calls
- MUST NOT hold funds, modify Treasury balances, or introduce automatic withdrawal logic

### MinerRewardModule

```solidity
interface IMinerRewardModule {
    /// @notice Map smoothing outputs to miner payout addresses
    function computePayouts(uint256 blockNumber) external view returns (address[] memory recipients, uint256[] memory amounts);

    /// @notice Prepare Treasury withdrawal calls conforming to ECIP-1114
    function prepareWithdrawals(uint256 blockNumber) external view returns (bytes[] memory calldatas);
}
```

The MinerRewardModule:
- Receives intended allocation amounts from SmoothingModule
- Computes miner payout shares per OIP-defined parameters
- Prepares Treasury withdrawal calls conforming to ECIP-1114
- MUST NOT reserve Treasury funds or bypass governance pipeline

## Governance Integration

All parameters are set exclusively through OIP:
- Smoothing window `N`
- Allocation fraction `f`
- Weighting function `L(j)` and its parameters
- Activation/deactivation of smoothing
- Module upgrade/replacement

Execution path: SmoothingModule outputs → ECFP proposal → Governor → Timelock → Executor → Treasury.withdraw()

## Treasury Interaction

The Treasury (ECIP-1112) remains unchanged. Smoothing operates strictly after basefee deposits. The Treasury does not compute smoothing schedules, maintain smoothing state, or reserve funds. All withdrawals go through the standard governance pipeline.

## Implementation Status

| Component | Status | Priority |
|-----------|--------|----------|
| ISmoothingModule interface | Specified | Phase 4 |
| IMinerRewardModule interface | Specified | Phase 4 |
| SmoothingModule contract | Not implemented | Phase 4 |
| MinerRewardModule contract | Not implemented | Phase 4 |

**Rationale for Phase 4:** Miners still have block rewards now, and there are minimal fees presently. This exploration matters most once the fee market is relevant after Olympia activation. The goal is empirical evidence before protocol embedding (ECIP-1116/1122, Phase 5).

## Relationship to ECIP-1116 and ECIP-1122

ECIP-1115 provides the **contract-layer experimentation framework**. If empirical data from Phase 4 proves that smoothing parameters are appropriate for protocol embedding:

- **ECIP-1116** embeds the basefee split ratio at consensus (5%/95%)
- **ECIP-1122** embeds the miner distribution curve at consensus (supersedes ECIP-1120)

Both require a second hard fork (Phase 5) and should only proceed after ECIP-1115 observable data validates the model.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)
