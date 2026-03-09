---
ecip: 1113
title: Olympia DAO Governance Framework — Implementation Spec
status: Implementation
type: Standards Track
category: ECBP
requires: 1112
author: Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)
created: 2025-07-04
updated: 2026-03-08
license: CC0-1.0
---

# ECIP-1113: Olympia DAO Governance Framework — Implementation Spec

## Overview

ECIP-1113 defines the on-chain governance framework for Olympia DAO — the contract suite that authorizes and schedules execution of approved actions affecting the Olympia Treasury (ECIP-1112). The DAO provides a modular Governor, voting mechanism, timelock executor, and sanctions integration. The DAO's authorized execution path is the only route through which Treasury withdrawals can occur.

Governance-layer upgrades follow the Olympia Improvement Proposal (OIP) process. Treasury disbursements are handled through the ECFP workflow (ECIP-1114).

Olympia launches **without a governance token**. Any future tokenization proposal must proceed through OIP.

## Amendments from Draft Spec

### 1. Voting Module — IOlympiaVotingModule Interface

**Draft spec language:** "Sybil-resistant, ETC-domain-restricted, one-address-one-vote model" with external modules for uniqueness/eligibility.

**Implementation:** `IOlympiaVotingModule` interface with `NFTVotingModuleAdapter` wrapping the deployed `OlympiaMemberNFT`.

**Rationale:** The draft spec describes the _what_ (sybil resistance, domain restriction, one-vote) but not the _how_. The IOlympiaVotingModule interface provides a clean abstraction that the Governor reads via `_getVotes()` override. Phase 1 uses NFT-based voting (simple, already deployed). Sybil resistance and domain restriction layers are added incrementally via OIP without changing the Governor.

```solidity
interface IOlympiaVotingModule {
    function votingPower(address voter, uint256 snapshotBlock) external view returns (uint256);
    function isEligible(address voter, uint256 snapshotBlock) external view returns (bool);
}
```

### 2. Executor Module — OlympiaExecutor with Sanctions Check

**Draft spec language:** "sole authorized caller of Treasury withdrawal functions" that "receives operations from the Timelock."

**Implementation:** `OlympiaExecutor` — a thin contract that sits between Timelock and Treasury, holds `WITHDRAWER_ROLE` on the Treasury, and checks `SanctionsOracle` (ECIP-1119) before every withdrawal.

**Rationale:** The draft spec's Executor is a passthrough. The implementation adds the sanctions check as a mandatory pre-execution gate (ECIP-1119 requirement). This is the single point where sanctions enforcement occurs at the execution layer — complementing early checks at `propose()` and mid-lifecycle `cancelIfSanctioned()`.

### 3. Governor — Custom _getVotes() Override

**Draft spec language:** "Governor module SHALL implement governance behavior functionally equivalent to the audited OpenZeppelin Governor model."

**Implementation:** Extends OZ `Governor`, `GovernorSettings`, `GovernorCountingSimple`, `GovernorTimelockControl`. Custom `_getVotes()` override reads from `IOlympiaVotingModule` instead of standard `GovernorVotes(IVotes)`.

**Rationale:** Standard `GovernorVotes` expects an `IVotes` token. Our voting model is module-based, not token-based. The `_getVotes()` override is the minimal change needed to bridge the OZ Governor to our custom voting interface.

### 4. Self-Upgrade via OIP

**Draft spec language:** `updateGovernanceParams()` and `updateVotingModule()` via OIP process.

**Implementation:** Governor exposes `updateGovernanceParams()` and `updateVotingModule()` functions gated by `onlyGovernance` modifier. These execute through the standard Governor → Timelock → Executor pipeline.

### 5. Sanctions Integration (ECIP-1119)

**Draft spec:** Not present in original ECIP-1113; sanctions defined separately in ECIP-1119.

**Implementation:** Three-layer defense integrated into the Governor and Executor:
1. **`propose()` early check** — parses calldatas, extracts recipient, checks `SanctionsOracle.isSanctioned()`. Reverts if sanctioned.
2. **`cancelIfSanctioned(uint256 proposalId)`** — permissionless, callable anytime during proposal lifecycle.
3. **`OlympiaExecutor.executeTreasury()` final gate** — atomic revert before `Treasury.withdraw()`.

## Governance Architecture

```
Governor ──→ Timelock ──→ OlympiaExecutor ──→ Treasury.withdraw()
   ↑              ↑              ↑
IOlympiaVotingModule    SanctionsOracle (ECIP-1119)
   ↑
NFTVotingModuleAdapter
   ↑
OlympiaMemberNFT
```

## Contract Interfaces

### IOlympiaVotingModule

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IOlympiaVotingModule {
    /// @notice Returns the voting power of `voter` at `snapshotBlock`.
    /// @dev Phase 1: 1 if NFT holder, 0 otherwise. Future phases may return weighted values.
    function votingPower(address voter, uint256 snapshotBlock) external view returns (uint256);

    /// @notice Returns whether `voter` is eligible to participate at `snapshotBlock`.
    function isEligible(address voter, uint256 snapshotBlock) external view returns (bool);
}
```

### NFTVotingModuleAdapter

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {IOlympiaVotingModule} from "./IOlympiaVotingModule.sol";
import {OlympiaMemberNFT} from "./OlympiaMemberNFT.sol";

contract NFTVotingModuleAdapter is IOlympiaVotingModule {
    OlympiaMemberNFT public immutable nft;

    constructor(address _nft) {
        nft = OlympiaMemberNFT(_nft);
    }

    function votingPower(address voter, uint256 snapshotBlock) external view returns (uint256) {
        return nft.getPastVotes(voter, snapshotBlock) > 0 ? 1 : 0;
    }

    function isEligible(address voter, uint256 snapshotBlock) external view returns (bool) {
        return nft.getPastVotes(voter, snapshotBlock) > 0;
    }
}
```

### OlympiaGovernor (Key Overrides)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Governor} from "@openzeppelin/contracts/governance/Governor.sol";
import {GovernorSettings} from "@openzeppelin/contracts/governance/extensions/GovernorSettings.sol";
import {GovernorCountingSimple} from "@openzeppelin/contracts/governance/extensions/GovernorCountingSimple.sol";
import {GovernorTimelockControl} from "@openzeppelin/contracts/governance/extensions/GovernorTimelockControl.sol";

contract OlympiaGovernor is Governor, GovernorSettings, GovernorCountingSimple, GovernorTimelockControl {
    IOlympiaVotingModule public votingModule;
    ISanctionsOracle public sanctionsOracle;

    constructor(
        string memory name,
        IOlympiaVotingModule _votingModule,
        ISanctionsOracle _sanctionsOracle,
        TimelockController _timelock,
        uint48 votingDelay_,
        uint32 votingPeriod_,
        uint256 quorum_
    )
        Governor(name)
        GovernorSettings(votingDelay_, votingPeriod_, 0)
        GovernorTimelockControl(_timelock)
    {
        votingModule = _votingModule;
        sanctionsOracle = _sanctionsOracle;
    }

    // --- IOlympiaVotingModule integration ---

    function _getVotes(address account, uint256 timepoint, bytes memory)
        internal view override returns (uint256)
    {
        return votingModule.votingPower(account, timepoint);
    }

    // --- Sanctions integration (ECIP-1119) ---

    function propose(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        string memory description
    ) public override returns (uint256) {
        // Early sanctions check — extract recipients from calldatas
        for (uint256 i = 0; i < targets.length; i++) {
            if (calldatas[i].length >= 36) {
                address recipient = abi.decode(_slice(calldatas[i], 4, 36), (address));
                require(!sanctionsOracle.isSanctioned(recipient), "OlympiaGovernor: sanctioned recipient");
            }
        }
        return super.propose(targets, values, calldatas, description);
    }

    function cancelIfSanctioned(uint256 proposalId) external {
        // Permissionless — anyone can cancel a proposal targeting a sanctioned address
        (address[] memory targets,, bytes[] memory calldatas,) = proposalDetails(proposalId);
        for (uint256 i = 0; i < targets.length; i++) {
            if (calldatas[i].length >= 36) {
                address recipient = abi.decode(_slice(calldatas[i], 4, 36), (address));
                if (sanctionsOracle.isSanctioned(recipient)) {
                    _cancel(targets, new uint256[](targets.length), calldatas, keccak256(bytes("")));
                    return;
                }
            }
        }
        revert("OlympiaGovernor: no sanctioned recipients");
    }

    // --- OIP self-upgrade ---

    function updateVotingModule(IOlympiaVotingModule newModule) external onlyGovernance {
        votingModule = newModule;
    }

    function updateSanctionsOracle(ISanctionsOracle newOracle) external onlyGovernance {
        sanctionsOracle = newOracle;
    }
}
```

### OlympiaExecutor

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ISanctionsOracle} from "./ISanctionsOracle.sol";

interface ITreasury {
    function withdraw(address payable to, uint256 amount) external;
}

contract OlympiaExecutor {
    address public immutable treasury;
    address public immutable timelock;
    ISanctionsOracle public immutable sanctionsOracle;

    event TreasuryExecution(address indexed recipient, uint256 amount);

    constructor(address _treasury, address _timelock, address _sanctionsOracle) {
        treasury = _treasury;
        timelock = _timelock;
        sanctionsOracle = ISanctionsOracle(_sanctionsOracle);
    }

    function executeTreasury(address payable recipient, uint256 amount) external {
        require(msg.sender == timelock, "OlympiaExecutor: only timelock");
        require(!sanctionsOracle.isSanctioned(recipient), "OlympiaExecutor: sanctioned");
        ITreasury(treasury).withdraw(recipient, amount);
        emit TreasuryExecution(recipient, amount);
    }
}
```

## Deployments

Contract addresses will be updated with Mordor and mainnet addresses at deployment.

| Network | Contract | Address | Status |
|---------|----------|---------|--------|
| Mordor | OlympiaGovernor | TBD | Pending deployment |
| Mordor | NFTVotingModuleAdapter | TBD | Pending deployment |
| Mordor | OlympiaExecutor | TBD | Pending deployment |
| Mordor | SanctionsOracle | TBD | Pending deployment (ECIP-1119) |
| Mordor | OlympiaMemberNFT | TBD | Pending deployment |
| Mordor | Timelock | TBD | Pending deployment |

## Gap Analysis

| Spec Requirement | Status | Action |
|-----------------|--------|--------|
| IOlympiaVotingModule interface | Not built | Custom `_getVotes()` override |
| `votingPower()` + `isEligible()` | Not built | NFTVotingModuleAdapter |
| Distinct Executor Module | Not built | OlympiaExecutor |
| `updateGovernanceParams()` via OIP | Not built | `onlyGovernance` setters |
| `updateVotingModule()` via OIP | Not built | `onlyGovernance` setter |
| Sanctions check on `propose()` | Not built | Sanctions integration (ECIP-1119) |
| `cancelIfSanctioned()` | Not built | Permissionless cancel |

## Staged Governance Lifecycle

```
Phase 1: Bootstrap (current)
  Treasury accumulates basefee + donations
  No withdrawals — governance not yet connected to Treasury

Phase 2: CoreDAO Activation (Stage 2)
  OlympiaGovernor deployed with IOlympiaVotingModule + sanctions
  OlympiaExecutor deployed, granted WITHDRAWER_ROLE on Treasury
  Timelock configured: Governor=PROPOSER+CANCELLER, Executor=EXECUTOR
  Withdrawals enabled through Governor → Timelock → Executor pipeline

Phase 3: Futarchy Integration (Stage 3)
  Additional WITHDRAWER_ROLE → FutarchyExecutor (ECIP-1117)
  CoreDAO codes redirect to Futarchy via governance proposal

Phase 4: Governance Maturity
  Admin renouncement on Treasury
  Contract converges to immutable multi-executor vault
```

## OpenZeppelin Compatibility

| Component | Version | Contract |
|-----------|---------|----------|
| Governor | v5.6.0 | `@openzeppelin/contracts/governance/Governor.sol` |
| GovernorSettings | v5.6.0 | Voting delay, period, proposal threshold |
| GovernorCountingSimple | v5.6.0 | For/Against/Abstain counting |
| GovernorTimelockControl | v5.6.0 | Timelock integration |
| TimelockController | v5.6.0 | Reuse existing deployment with updated roles |

## Deployment Sequence

```
1. Deploy SanctionsOracle (ECIP-1119)
2. Deploy NFTVotingModuleAdapter (wrapping OlympiaMemberNFT)
3. Deploy OlympiaTimelock
4. Deploy OlympiaExecutor (treasury + timelock + sanctionsOracle)
5. Deploy OlympiaGovernor (votingModule + timelock + sanctionsOracle)
6. Configure Timelock: Governor=PROPOSER+CANCELLER, Executor=EXECUTOR
7. Grant WITHDRAWER_ROLE on Treasury to OlympiaExecutor
8. Test full governance lifecycle on Mordor
```

## Test Suite Requirements

- Full lifecycle: deposit → propose → vote → queue → timelock → execute → receive
- Sanctions: early rejection on propose, mid-lifecycle cancel, execution-time revert
- Governance upgrades: `updateGovernanceParams()`, `updateVotingModule()`
- Access control: unauthorized calls revert at every layer
- Staged governance: admin → CoreDAO executor → admin renounce
- Edge cases: zero-value, duplicate execution, cancelled proposals, quorum failures

## Verification

```bash
# Verify Governor configuration
cast call <governor> "votingDelay()" --rpc-url $MORDOR_RPC_URL
cast call <governor> "votingPeriod()" --rpc-url $MORDOR_RPC_URL
cast call <governor> "quorum(uint256)" <block> --rpc-url $MORDOR_RPC_URL

# Verify VotingModule
cast call <adapter> "votingPower(address,uint256)" <voter> <block> --rpc-url $MORDOR_RPC_URL

# Verify Executor
cast call <executor> "treasury()" --rpc-url $MORDOR_RPC_URL
cast call <executor> "timelock()" --rpc-url $MORDOR_RPC_URL

# Verify Sanctions integration
cast call <governor> "sanctionsOracle()" --rpc-url $MORDOR_RPC_URL
```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)
