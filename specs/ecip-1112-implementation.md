---
ecip: 1112
title: Olympia Treasury Contract — Implementation Spec
status: Implementation
type: Standards Track
category: Core
requires: 1111
author: Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)
created: 2025-07-04
updated: 2026-03-18
license: CC0-1.0
---

# ECIP-1112: Olympia Treasury Contract — Implementation Spec

## Overview

The Olympia Treasury is an immutable smart contract that receives all basefee revenue redirected under [ECIP-1111](https://ecips.ethereumclassic.org/ECIPs/ecip-1111). It functions as a governance-isolated vault — it receives funds via consensus-layer state credit and permits withdrawals only through a single authorized caller (the `immutable executor`). The Treasury also accepts voluntary third-party donations (e.g., from Bitmain, Grayscale, ETC Coop).

No governance logic, proposal processing, or allocation rules are implemented within this contract. Governance and execution are specified independently in [ECIP-1113](https://ecips.ethereumclassic.org/ECIPs/ecip-1113), [ECIP-1114](https://ecips.ethereumclassic.org/ECIPs/ecip-1114), and [ECIP-1119](https://ecips.ethereumclassic.org/ECIPs/ecip-1119).

## Amendments from Draft Spec

The following changes reflect implementation decisions. The Treasury is pure Solidity with no external dependencies — 49 lines, zero attack surface.

### 1. Access Control Pattern

**Draft spec language:** "single authorized caller address" / "single, authorized execution entry point"

**Implementation:** `immutable executor` — a single address set at construction time, never changeable.

**Rationale:** The immutable executor pattern is the simplest possible access control. No roles, no admin, no staged lifecycle. The executor address is pre-computed via CREATE2 before the Treasury is deployed. Once set, it cannot be changed. This eliminates the entire class of admin-key vulnerabilities.

### 2. Interface

**Draft spec:** `ITreasury.release(address, uint256)`

**Implementation:** `withdraw(address payable to, uint256 amount)`

**Rationale:** `withdraw()` is the established convention for vault contracts. The `payable` qualifier on the `to` parameter is required for native ETC transfers.

### 3. Events and Errors

**Draft spec:** `FundsReleased`, `Received`

**Implementation:**
- `Withdrawal(address indexed to, uint256 amount)` — emitted on successful withdrawal
- `Received(address indexed from, uint256 amount)` — emitted on `receive()` for voluntary donations
- Custom errors: `Unauthorized`, `ZeroAddress`, `InsufficientBalance`, `TransferFailed`

**Note:** Basefee deposits arrive via consensus-layer state credit, not through `receive()`. The `Received` event tracks only voluntary donations. Custom errors replace `require()` strings for gas efficiency.

### 4. Out-of-Scope

The draft spec listed "role delegation or admin key models" as prohibited. The implementation honors this — there are no roles, no admin, no delegation. The executor is a single immutable address.

## Architecture

```
Treasury (CREATE, immutable executor)
    ↓ only executor can call withdraw()
OlympiaExecutor (CREATE2, governance pipeline)
    ↓ authorized by Timelock
OlympiaGovernor → TimelockController → Executor → Treasury.withdraw()
```

The Treasury is immutable from deployment. No admin transfer, no renouncement, no staged lifecycle. The executor address is fixed at construction time.

The governance pipeline activates when the OlympiaExecutor contract is deployed at the pre-computed CREATE2 address. Before deployment, `withdraw()` reverts with `Unauthorized` (no code exists at the executor address, so no one can call it).

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {IOlympiaTreasury} from "./interfaces/IOlympiaTreasury.sol";

/// @title OlympiaTreasury
/// @notice ECIP-1112 compliant immutable treasury vault.
///         Receives basefee revenue via consensus-layer state credit (ECIP-1111)
///         and voluntary donations via receive(). Only the pre-computed executor
///         (OlympiaExecutor deployed via CREATE2) can withdraw funds.
/// @dev No OpenZeppelin. No roles. No admin. No upgrade path. Pure Solidity.
contract OlympiaTreasury is IOlympiaTreasury {
    address public immutable executor;

    error Unauthorized();
    error ZeroAddress();
    error InsufficientBalance();
    error TransferFailed();

    constructor(address _executor) {
        if (_executor == address(0)) revert ZeroAddress();
        executor = _executor;
    }

    function withdraw(address payable to, uint256 amount) external {
        if (msg.sender != executor) revert Unauthorized();
        if (to == address(0)) revert ZeroAddress();
        if (address(this).balance < amount) revert InsufficientBalance();

        emit Withdrawal(to, amount);

        (bool success,) = to.call{value: amount}("");
        if (!success) revert TransferFailed();
    }

    receive() external payable {
        emit Received(msg.sender, msg.value);
    }
}
```

## Deployments

Demo v0.2 deployment. Treasury deployed via CREATE (nonce-based), executor pre-computed via CREATE2.

| Network | Address | Deployment | Deployer | Status |
|---------|---------|------------|----------|--------|
| Mordor | `0x035b2e3c189B772e52F4C3DA6c45c84A3bB871bf` | CREATE (nonce 0) | `0x7C3311F29e318617fed0833E68D6522948AaE995` | Demo deployed |
| ETC Mainnet | `0x035b2e3c189B772e52F4C3DA6c45c84A3bB871bf` | CREATE (nonce 0) | `0x7C3311F29e318617fed0833E68D6522948AaE995` | Demo deployed |

All 3 client olympia branches have been updated:
- core-geth: `params/config_mordor.go` + `params/config_classic.go` → `OlympiaTreasuryAddress`
- besu-etc: `config/mordor.json` + `config/classic.json` → `olympiaTreasuryAddress`
- fukuii: `mordor-chain.conf` + `etc-chain.conf` + `gorgoroth-chain.conf` → `treasury-address`

## Deploy Script

Demo v0.2 uses CREATE for the Treasury and pre-computes the executor address via CREATE2:

```solidity
contract DeployScript is Script {
    // Pre-computed OlympiaExecutor CREATE2 address (OZ 5.1 bytecode).
    // The executor contract does not exist yet — governance deploys it later.
    address constant EXECUTOR = 0x64624f74F77639CbA268a6c8bEDC2778B707eF9a;

    function run() public {
        // Treasury uses CREATE (nonce-based), not CREATE2.
        // This breaks the circular dependency with Executor:
        // - Treasury address = f(deployer, nonce) — no dependency on constructor args
        // - Executor uses CREATE2 with Treasury address as constructor arg
        vm.startBroadcast();
        OlympiaTreasury treasury = new OlympiaTreasury(EXECUTOR);
        vm.stopBroadcast();
    }
}
```

## Deterministic Deployment — The Bootstrap Problem

Treasury and Executor have a circular dependency: Treasury needs the executor address at construction, and Executor needs the treasury address at construction. This is resolved by using different deployment mechanisms:

1. **Treasury** uses CREATE (nonce-based): `address = keccak256(rlp(deployer, nonce))`. Predictable if deployer and nonce are known (fresh EOA, nonce 0).
2. **Executor** uses CREATE2 (salt-based): `address = keccak256(0xff, deployer, salt, bytecodeHash)`. Predictable from bytecode + salt. Treasury address is a constructor arg baked into the bytecode hash.

Both addresses are pre-computed by `PrecomputeAddresses.s.sol` in the governance repo before either contract is deployed.

## Consensus-Layer Funding

Upon activation of [ECIP-1111](https://ecips.ethereumclassic.org/ECIPs/ecip-1111):
- The full basefee-derived amount from each transaction is credited to the Treasury address during block finalization
- This is a state credit, not a transaction — no `receive()` call, no event emission
- The Treasury balance increases deterministically with each block

## Security Properties

1. **Immutable:** No proxy, no delegatecall, no callcode, no upgrade mechanism
2. **Pure Solidity:** No external dependencies — zero supply chain attack surface
3. **Minimal bytecode:** < 1 KB compiled, fully auditable
4. **Single entry point:** Only `executor` can call `withdraw()`, set once at construction
5. **No admin:** No roles, no admin transfer, no renouncement — immutable from deployment
6. **No SELFDESTRUCT:** Contract cannot be destroyed
7. **Custom errors:** Gas-efficient reverts (`Unauthorized`, `ZeroAddress`, `InsufficientBalance`, `TransferFailed`)

## Verification

```bash
# Run tests (33 tests)
forge test -vv

# Deploy
forge script script/Deploy.s.sol --rpc-url $MORDOR_RPC_URL --broadcast --legacy

# Verify executor is correctly set
cast call 0x035b2e3c189B772e52F4C3DA6c45c84A3bB871bf "executor()" --rpc-url $MORDOR_RPC_URL
# Expected: 0x64624f74F77639CbA268a6c8bEDC2778B707eF9a

# Verify Treasury balance
cast balance 0x035b2e3c189B772e52F4C3DA6c45c84A3bB871bf --rpc-url $MORDOR_RPC_URL

# Verify bytecode deployed
cast code 0x035b2e3c189B772e52F4C3DA6c45c84A3bB871bf --rpc-url $MORDOR_RPC_URL
```

## Production Notes

Demo v0.2 uses pure Solidity with an immutable executor. Production deployment uses the same architecture — recompiled against OZ 5.6.0 (Cancun EVM) for the governance contracts. Treasury remains pure Solidity (identical bytecode). All production addresses differ: the Treasury CREATE address changes because production uses a different deployer EOA (fresh, nonce 0). The Executor CREATE2 address changes because its initcode includes the new Treasury address and the new TimelockController address (OZ 5.6 bytecode) as constructor args. Both addresses are recomputed together by `PrecomputeAddresses.s.sol` in the governance repo before either contract is deployed.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)
