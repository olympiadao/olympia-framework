---
ecip: 1119
title: Olympia Treasury Withdrawal Sanctions Constraint — Implementation Spec
status: Implementation
type: Meta
requires: 1112
author: Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)
created: 2025-12-14
updated: 2026-03-08
license: CC0-1.0
---

# ECIP-1119: Olympia Treasury Withdrawal Sanctions Constraint — Implementation Spec

## Overview

ECIP-1119 establishes a deterministic screening mechanism that applies to all Treasury withdrawals. No funds leave the Olympia Treasury without passing sanctions screening. The constraint operates independently of Treasury custody, governance processes, or consensus behavior. Failed sanctions checks cause atomic transaction reversion.

**Critical:** Sanctions MUST activate with withdrawals. ECIP-1119 must be live before any funds leave the Treasury (Stage 2).

## Amendments from Draft Spec

### 1. Layered Defense Architecture

**Draft spec:** "deterministic pre-execution check on Treasury withdrawals" — single check point.

**Implementation:** Three-layer defense architecture:

1. **Early check at `Governor.propose()`** — reject proposals targeting sanctioned addresses before voting begins. Saves wasted governance cycles.
2. **`cancelIfSanctioned(uint256 proposalId)`** — permissionless, callable by anyone at any time during proposal lifecycle. Handles mid-lifecycle sanctions changes (address sanctioned after proposal submission).
3. **Final gate at `OlympiaExecutor.executeTreasury()`** — atomic revert immediately before `Treasury.withdraw()`. This is the security invariant — the one check that can never be bypassed.

**Rationale:** A single check at execution time is sufficient for security but insufficient for efficiency. Early checks save wasted governance cycles. The permissionless cancel handles the case where an address becomes sanctioned between proposal submission and execution. The execution-time check is the hard invariant.

### 2. SanctionsOracle Interface

**Draft spec:** "deterministic, read-only interface" without specific function signatures.

**Implementation:** Concrete `ISanctionsOracle` interface with governance-managed blocklist.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

interface ISanctionsOracle {
    event AddressAdded(address indexed account);
    event AddressRemoved(address indexed account);

    function isSanctioned(address account) external view returns (bool);
}
```

### 3. Governance-Managed Blocklist

**Draft spec:** References "oracle aggregation" and multi-oracle models.

**Implementation:** Phase 1 uses a simple governance-managed blocklist. Multi-oracle aggregation is a future enhancement via OIP.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {ISanctionsOracle} from "./interfaces/ISanctionsOracle.sol";
import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";

contract SanctionsOracle is ISanctionsOracle, AccessControl {
    bytes32 public constant MANAGER_ROLE = keccak256("MANAGER_ROLE");

    mapping(address => bool) private _sanctioned;

    error AlreadySanctioned(address account);
    error NotSanctioned(address account);
    error ZeroAddress();

    constructor(address admin) {
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        _grantRole(MANAGER_ROLE, admin);
    }

    function isSanctioned(address account) external view returns (bool) {
        return _sanctioned[account];
    }

    function addAddress(address account) external onlyRole(MANAGER_ROLE) {
        if (account == address(0)) revert ZeroAddress();
        if (_sanctioned[account]) revert AlreadySanctioned(account);
        _sanctioned[account] = true;
        emit AddressAdded(account);
    }

    function removeAddress(address account) external onlyRole(MANAGER_ROLE) {
        if (account == address(0)) revert ZeroAddress();
        if (!_sanctioned[account]) revert NotSanctioned(account);
        _sanctioned[account] = false;
        emit AddressRemoved(account);
    }
}
```

## Layered Defense Architecture

```
┌─────────────────────────────────────────────────────────┐
│ Layer 1: Governor.propose()                             │
│   Parse calldatas → extract recipient → check oracle    │
│   Revert if sanctioned → saves wasted governance cycles │
├─────────────────────────────────────────────────────────┤
│ Layer 2: cancelIfSanctioned(proposalId)                 │
│   Permissionless, callable anytime during lifecycle     │
│   Handles mid-lifecycle sanctions changes               │
├─────────────────────────────────────────────────────────┤
│ Layer 3: OlympiaExecutor.executeTreasury()              │
│   Final gate — atomic revert before Treasury.withdraw() │
│   THE SECURITY INVARIANT — can never be bypassed        │
└─────────────────────────────────────────────────────────┘
```

## Applicability

This constraint applies uniformly to all Olympia Treasury withdrawals across:
- CoreDAO governance pipeline (ECIP-1113, ECIP-1114) — Stage 2
- Futarchy-based governance (ECIP-1117, ECIP-1118) — Stage 3

Both governance paths route through an Executor that checks the same SanctionsOracle.

## Failure Modes

| Scenario | Behavior |
|----------|----------|
| Sanctioned recipient at `propose()` | Revert — proposal never created |
| Address sanctioned mid-lifecycle | `cancelIfSanctioned()` cancels proposal |
| Sanctioned at execution time | Executor reverts — funds stay in Treasury |
| Oracle unavailable/reverts | Execution reverts — fail-closed behavior |
| Zero address | Handled by Treasury's own zero-address check |

## Deployments

| Network | Contract | Address | Status |
|---------|----------|---------|--------|
| Mordor | SanctionsOracle | TBD | Phase 2A complete (built, 14 tests) |

## Non-Goals

This ECIP explicitly does not:
- Guarantee compliance with any specific legal regime
- Mandate particular sanctions lists or data providers
- Modify Treasury custody, governance, or consensus rules
- Restrict governance participation (only withdrawal recipients)
- Introduce identity requirements or discretionary overrides

## Test Suite Requirements

### Phase 2A (14 tests — Complete)

- Add/remove happy path + events (AddressAdded, AddressRemoved)
- isSanctioned state transitions (false → true → false)
- Custom error reverts: AlreadySanctioned, NotSanctioned, ZeroAddress
- Access control: only MANAGER_ROLE can add/remove, admin can grant MANAGER_ROLE
- Edge cases: non-sanctioned returns false, duplicate add reverts, remove non-sanctioned reverts

### Phase 2B (Pending — Governor + Executor integration)

- Sanctions at propose: reverts when recipient is sanctioned
- Sanctions at cancel: `cancelIfSanctioned()` works for anyone
- Sanctions at execute: Executor reverts, funds remain in Treasury
- Edge cases: oracle reverts → fail-closed

## Verification

```bash
# Check if address is sanctioned
cast call <oracle> "isSanctioned(address)" <addr> --rpc-url $MORDOR_RPC_URL

# Verify oracle is connected to Governor
cast call <governor> "sanctionsOracle()" --rpc-url $MORDOR_RPC_URL

# Verify oracle is connected to Executor
cast call <executor> "sanctionsOracle()" --rpc-url $MORDOR_RPC_URL

# Test propose with sanctioned address (should revert)
cast send <governor> "propose(address[],uint256[],bytes[],string)" ... --rpc-url $MORDOR_RPC_URL
```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)
