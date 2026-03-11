---
ecip: 1112
title: Olympia Treasury Contract — Implementation Spec
status: Implementation
type: Standards Track
category: Core
requires: 1111
author: Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)
created: 2025-07-04
updated: 2026-03-08
license: CC0-1.0
---

# ECIP-1112: Olympia Treasury Contract — Implementation Spec

## Overview

The Olympia Treasury is a deterministic, immutable smart contract that receives all basefee revenue redirected under ECIP-1111. It functions as a governance-isolated vault — it receives funds via consensus-layer state credit and permits withdrawals only through role-gated access control. The Treasury also accepts voluntary third-party donations (e.g., from Bitmain, Grayscale, ETC Coop).

No governance logic, proposal processing, or allocation rules are implemented within this contract. Governance and execution are specified independently in ECIP-1113, ECIP-1114, and ECIP-1119.

## Amendments from Draft Spec

The following changes reflect implementation decisions validated against OpenZeppelin best practices and industry standards (Compound, Aave, Uniswap, Arbitrum):

### 1. Access Control Pattern

**Draft spec language:** "single authorized caller address" / "single, authorized execution entry point"

**Implementation:** OpenZeppelin `AccessControlDefaultAdminRules` (v5.6) with `WITHDRAWER_ROLE`.

**Rationale:** The AccessControl pattern is the industry standard for staged governance activation. `DEFAULT_ADMIN_ROLE` is held by the deployer during bootstrap, transferred to CoreDAO governance via 2-step process, then renounced. After renouncement, the contract converges to a single-executor vault where only `WITHDRAWER_ROLE` holders can call `withdraw()`.

`AccessControlDefaultAdminRules` (OZ v5.6) adds critical safety:
- 2-step admin transfer with configurable delay (e.g., 3 days)
- Prevents accidental admin transfer to wrong address
- 2-step renouncement process
- `beginDefaultAdminTransfer()` → wait delay → `acceptDefaultAdminTransfer()`

### 2. Interface

**Draft spec:** `ITreasury.release(address, uint256)`

**Implementation:** `withdraw(address payable to, uint256 amount)`

**Rationale:** `withdraw()` is the established convention for vault contracts. The `payable` qualifier on the `to` parameter is required for native ETC transfers.

### 3. Events

**Draft spec:** `FundsReleased`, `Received`

**Implementation:**
- `Withdrawal(address indexed to, uint256 amount)` — emitted on successful withdrawal
- `Received(address indexed from, uint256 amount)` — emitted on `receive()` for voluntary donations

**Note:** Basefee deposits arrive via consensus-layer state credit, not through `receive()`. The `Received` event tracks only voluntary donations.

### 4. Out-of-Scope Clarification

**Draft spec:** Listed "role delegation or admin key models" as prohibited.

**Implementation:** The AccessControl pattern is used for staged governance activation. Post-renouncement, the contract is operationally equivalent to a single-executor vault. The prohibition on role delegation is replaced by the staged governance lifecycle described below.

## Staged Governance Lifecycle

```
Phase 1: Bootstrap (current)
  DEFAULT_ADMIN_ROLE → deployer EOA
  WITHDRAWER_ROLE → deployer EOA
  Treasury accumulates basefee + donations, no withdrawals

Phase 2: CoreDAO Activation (Stage 2)
  WITHDRAWER_ROLE → OlympiaExecutor (ECIP-1113)
  DEFAULT_ADMIN_ROLE → CoreDAO governance (via 2-step transfer)
  Withdrawals enabled through Governor → Timelock → Executor pipeline

Phase 3: Futarchy Integration (Stage 3)
  Additional WITHDRAWER_ROLE → FutarchyExecutor (ECIP-1117)
  CoreDAO codes redirect to Futarchy via governance proposal

Phase 4: Admin Renouncement
  beginDefaultAdminTransfer(address(0)) → wait delay → renounceRole()
  Contract converges to immutable single-executor vault
  No further role changes possible
```

## Contract Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {AccessControlDefaultAdminRules} from
    "@openzeppelin/contracts/access/extensions/AccessControlDefaultAdminRules.sol";

contract OlympiaTreasury is AccessControlDefaultAdminRules {
    bytes32 public constant WITHDRAWER_ROLE = keccak256("WITHDRAWER_ROLE");

    event Withdrawal(address indexed to, uint256 amount);
    event Received(address indexed from, uint256 amount);

    constructor(uint48 adminTransferDelay, address admin)
        AccessControlDefaultAdminRules(adminTransferDelay, admin)
    {
        _grantRole(WITHDRAWER_ROLE, admin);
    }

    function withdraw(address payable to, uint256 amount)
        external
        onlyRole(WITHDRAWER_ROLE)
    {
        require(to != address(0), "OlympiaTreasury: zero address");
        require(amount <= address(this).balance, "OlympiaTreasury: insufficient balance");
        (bool success,) = to.call{value: amount}("");
        require(success, "OlympiaTreasury: transfer failed");
        emit Withdrawal(to, amount);
    }

    receive() external payable {
        emit Received(msg.sender, msg.value);
    }
}
```

## Deployments

Demo v0.1 deployment uses a single CREATE2 salt for deterministic addressing across both chains.

| Network | Address | Salt | Status |
|---------|---------|------|--------|
| Mordor | `0xd6165F3aF4281037bce810621F62B43077Fb0e37` | `keccak256("OLYMPIA_DEMO_V0_1")` | Demo deployed |
| ETC Mainnet | `0xd6165F3aF4281037bce810621F62B43077Fb0e37` | `keccak256("OLYMPIA_DEMO_V0_1")` | Demo deployed |

All 3 client olympia branches have been updated:
- core-geth: `params/config_mordor.go` + `params/config_classic.go` → `OlympiaTreasuryAddress`
- besu-etc: `config/mordor.json` + `config/classic.json` → `olympiaTreasuryAddress`
- fukuii: `mordor-chain.conf` + `etc-chain.conf` + `gorgoroth-chain.conf` → `treasury-address`

## Deploy Script

```solidity
contract DeployScript is Script {
    function run() public {
        // Versioned salt per network
        bytes32 salt = keccak256(abi.encodePacked(
            "OLYMPIA_TREASURY_",
            block.chainid == 63 ? "MORDOR" : "MAINNET"
        ));

        uint48 adminDelay = 3 days; // governance-appropriate delay

        vm.startBroadcast();
        OlympiaTreasury treasury = new OlympiaTreasury{salt: salt}(
            adminDelay,
            msg.sender
        );
        vm.stopBroadcast();
    }
}
```

## Deterministic Deployment

The Treasury address is derived using CREATE2:

```solidity
address computedAddress = address(uint160(uint256(keccak256(abi.encodePacked(
    bytes1(0xff),
    deployerAddress,
    salt,
    keccak256(type(OlympiaTreasury).creationCode)
)))));
```

The deployer address, salt, and bytecode hash are published prior to deployment. Client implementations treat this address as the canonical recipient of protocol-level basefee revenue.

## Consensus-Layer Funding

Upon activation of ECIP-1111:
- The full basefee-derived amount from each transaction is credited to the Treasury address during block finalization
- This is a state credit, not a transaction — no `receive()` call, no event emission
- The Treasury balance increases deterministically with each block

## OpenZeppelin Compatibility

| Component | Version | Contract |
|-----------|---------|----------|
| AccessControlDefaultAdminRules | v5.6.0 | `@openzeppelin/contracts/access/extensions/AccessControlDefaultAdminRules.sol` |
| IAccessControlDefaultAdminRules | v5.6.0 | Interface for 2-step admin transfer |
| IERC5313 | v5.6.0 | `owner()` compatibility (returns `defaultAdmin()`) |

## Security Properties

1. **Immutable:** No proxy, no delegatecall, no upgrade mechanism
2. **Deterministic:** CREATE2 address identical across chains with same salt
3. **Fail-safe admin transfer:** 2-step process with configurable delay prevents accidental transfers
4. **Role separation:** `WITHDRAWER_ROLE` is independent of `DEFAULT_ADMIN_ROLE`
5. **Convergence to single-executor:** After admin renouncement, no further role changes possible

## Verification

```bash
# Deploy and verify
forge test -vv
forge script script/Deploy.s.sol --rpc-url $MORDOR_RPC_URL --broadcast --legacy

# Verify AccessControlDefaultAdminRules
cast call <treasury> "defaultAdmin()" --rpc-url $MORDOR_RPC_URL
cast call <treasury> "defaultAdminDelay()" --rpc-url $MORDOR_RPC_URL

# Verify WITHDRAWER_ROLE
cast call <treasury> "hasRole(bytes32,address)" $(cast keccak "WITHDRAWER_ROLE") <admin>

# Test 2-step admin transfer
cast send <treasury> "beginDefaultAdminTransfer(address)" <new_admin>
# Wait adminDelay...
cast send <treasury> "acceptDefaultAdminTransfer()" --from <new_admin>
```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)
