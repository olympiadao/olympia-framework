---
ecip: 1111
title: Olympia EVM and Protocol Upgrades — Implementation Spec
status: Implementation
type: Standards Track
category: Core
requires: 1112
author: Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)
created: 2025-07-04
updated: 2026-03-08
license: CC0-1.0
---

# ECIP-1111: Olympia EVM and Protocol Upgrades — Implementation Spec

## Overview

ECIP-1111 introduces EIP-1559 and EIP-3198 to Ethereum Classic. Unlike Ethereum Mainnet where the basefee is burned, ETC redirects 100% of the basefee to the Olympia Treasury (ECIP-1112) via consensus-layer state credit. Priority fees (tips) remain payable directly to block producers.

This adds ~1 gwei per transaction — less than 0.01% of miner income at current activity levels.

## Included EIPs

| EIP | Description | Opcode/Feature |
|-----|-------------|----------------|
| EIP-1559 | Fee Market Change | Dynamic basefee, Type-2 transaction format, `maxFeePerGas` + `maxPriorityFeePerGas` |
| EIP-3198 | BASEFEE Opcode | Opcode `0x48` — returns current block's basefee |

## Activation

| Network | Block Height | Estimated Date | Status |
|---------|-------------|----------------|--------|
| Mordor Testnet | 15,800,850 | ~March 28, 2026 | Pending |
| ETC Mainnet | ~24,751,337 | ~mid-June 2026 | Pending |

## ETC-Specific Deviation from Ethereum

The sole deviation from canonical EIP-1559: the basefee is **not burned**. Instead, the aggregate basefee for each block is credited to the Olympia Treasury address at the consensus layer during block finalization.

```
// Pseudocode — consensus-layer basefee handling
func finalizeBlock(block, state) {
    baseFeeAmount = block.gasUsed * block.baseFee
    state.AddBalance(OlympiaTreasuryAddress, baseFeeAmount)
    // Ethereum burns this; ETC redirects to Treasury
}
```

## Treasury Address

Deployed via CREATE2 with a single salt for deterministic addressing across both chains. See ECIP-1112 implementation spec.

| Network | Address | Derivation |
|---------|---------|------------|
| Mordor | `0xd6165F3aF4281037bce810621F62B43077Fb0e37` | CREATE2 with salt `keccak256("OLYMPIA_DEMO_V0_1")` |
| ETC Mainnet | `0xd6165F3aF4281037bce810621F62B43077Fb0e37` | CREATE2 with salt `keccak256("OLYMPIA_DEMO_V0_1")` |

## Client Implementation Status

| Client | Branch | Language | EIP-1559 | EIP-3198 | Tests |
|--------|--------|----------|----------|----------|-------|
| core-geth v1.12.21 | `olympia` (`b1c759dcc`) | Go 1.24 | ✅ | ✅ | ALL PASS |
| besu-etc v26.3 | `olympia` (`52dc37b5bf`) | Java 21 | ✅ | ✅ | ALL PASS |
| fukuii v0.1.240 | `olympia` (`126c1fd5c`) | Scala 3.3 | ✅ | ✅ | ALL PASS (2,308 tests) |

### Key Implementation Details

**core-geth:**
- `params/config_mordor.go` → `OlympiaTreasuryAddress` constant
- `consensus/ethash/consensus.go` → basefee credit in `Finalize()`
- Gas limit convergence: 8M → 60M via ±1/1024 per block (2,055 blocks)

**besu-etc:**
- Mordor genesis config → treasury address
- `MainnetProtocolSchedule` → Olympia milestone configuration
- EIP-1559 transaction processing in `MainnetTransactionProcessor`

**fukuii:**
- Mordor chain config → treasury address
- `BlockRewardCalculator` → basefee redirect logic
- SNAP sync support for state trie with treasury balance

## Gas Limit Convergence

Olympia increases the target gas limit from 8M to 60M. The EIP-1559 mechanism adjusts the gas limit by ±1/1024 per block:

- **Convergence period:** ~2,055 blocks (~8.5 hours at 15s block time)
- **Formula:** `newGasLimit = parentGasLimit ± floor(parentGasLimit / 1024)`

## Transaction Types After Olympia

| Type | Name | Supported | Fee Handling |
|------|------|-----------|-------------|
| Type-0 | Legacy | ✅ | `gasPrice` → basefee to Treasury, remainder to miner |
| Type-1 | Access List (EIP-2930) | ✅ | Same as Type-0 |
| Type-2 | EIP-1559 | ✅ (new) | `basefee` to Treasury, `priorityFee` to miner |

## Relationship to Other ECIPs

- **ECIP-1112:** Defines the Treasury contract that receives the basefee
- **ECIP-1121:** Additional EVM opcodes activated in the same fork (independent of basefee)
- **ECIP-1116:** Future proposal to split basefee 5%/95% between Treasury and miners (Phase 5, second fork)

## Verification

```bash
# Verify basefee is being redirected to Treasury after activation
cast balance <treasury_address> --rpc-url $MORDOR_RPC_URL

# Verify BASEFEE opcode works
cast call --create "0x48600052602060006000f3" --rpc-url $MORDOR_RPC_URL

# Cross-client verification: all 3 clients must produce identical Treasury balance
```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)
