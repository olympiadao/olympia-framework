---
ecip: 1113
title: Olympia DAO Governance Framework — Implementation Spec
status: Implementation
type: Standards Track
category: ECBP
requires: 1112
author: Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)
created: 2025-07-04
updated: 2026-03-18
license: CC0-1.0
---

# ECIP-1113: Olympia DAO Governance Framework — Implementation Spec

## Overview

ECIP-1113 defines the on-chain governance framework for Olympia DAO — the contract suite that authorizes and schedules execution of approved actions affecting the Olympia Treasury ([ECIP-1112](https://ecips.ethereumclassic.org/ECIPs/ecip-1112)). The DAO provides a modular Governor, voting mechanism, timelock executor, and sanctions integration. The DAO's authorized execution path is the only route through which Treasury withdrawals can occur.

Governance-layer upgrades follow the Olympia Improvement Proposal (OIP) process. Treasury disbursements are handled through the ECFP workflow ([ECIP-1114](https://ecips.ethereumclassic.org/ECIPs/ecip-1114)).

Olympia launches **without a governance token**. Any future tokenization proposal must proceed through OIP.

## Amendments from Draft Spec

### 1. Voting Module — Soulbound NFT + Standard OZ GovernorVotes

**Draft spec language:** "Sybil-resistant, ETC-domain-restricted, one-address-one-vote model" with external modules for uniqueness/eligibility.

**Implementation (Demo v0.2):** Soulbound `OlympiaMemberNFT` (ERC721 + ERC721Votes + ERC5192) read directly by standard OZ `GovernorVotes` + `GovernorVotesQuorumFraction`. One soulbound NFT = one vote. KYC/BrightID/Gitcoin Passport-verified accounts receive a non-transferable NFT via `MINTER_ROLE`. No custom `_getVotes()` override, no adapter contract.

**Forward-looking:** The `IOlympiaVotingModule` interface is included as a spec artifact for future governance-gated module swaps. It is not wired into the Demo v0.2 Governor.

```solidity
// Forward-looking interface (not used in Demo v0.2)
interface IOlympiaVotingModule {
    function votingPower(address account, uint256 timepoint) external view returns (uint256);
    function isEligible(address account, uint256 timepoint) external view returns (bool);
}
```

**OlympiaMemberNFT key properties:**
- Soulbound: `_update()` blocks transfers (allows mint/burn only)
- ERC5192: `locked()` always returns `true`
- Auto-delegate: `_delegate(to, to)` on mint — votes active immediately
- Block number clock mode (OZ default) — no `clock()` or `CLOCK_MODE()` override
- MINTER_ROLE gates issuance (identity verification at application layer)

### 2. Executor Module — OlympiaExecutor with Sanctions Check

**Draft spec language:** "sole authorized caller of Treasury withdrawal functions" that "receives operations from the Timelock."

**Implementation:** `OlympiaExecutor` — a thin contract that sits between Timelock and Treasury. It is the Treasury's immutable authorized caller and checks `SanctionsOracle` ([ECIP-1119](https://ecips.ethereumclassic.org/ECIPs/ecip-1119)) before every withdrawal.

**Rationale:** The draft spec's Executor is a passthrough. The implementation adds the sanctions check as a mandatory pre-execution gate ([ECIP-1119](https://ecips.ethereumclassic.org/ECIPs/ecip-1119) requirement). This is the single point where sanctions enforcement occurs at the execution layer — complementing early checks at `propose()` and mid-lifecycle `cancelIfSanctioned()`.

### 3. Governor — Standard OZ GovernorVotes (Demo v0.2)

**Draft spec language:** "Governor module SHALL implement governance behavior functionally equivalent to the audited OpenZeppelin Governor model."

**Implementation (Demo v0.2):** Extends OZ `Governor`, `GovernorSettings`, `GovernorCountingSimple`, `GovernorVotes`, `GovernorVotesQuorumFraction`, `GovernorTimelockControl`, `GovernorPreventLateQuorum`, `GovernorStorage`. Standard `GovernorVotes` reads `getPastVotes()` directly from the soulbound `OlympiaMemberNFT` (which implements `ERC721Votes`). No custom `_getVotes()` override needed. Bravo-style quorum (only "For" votes count toward quorum). Demo v0.2 additions: `GovernorPreventLateQuorum` (extends voting if quorum reached late), `GovernorStorage` (on-chain proposal storage), custom errors (`SanctionedRecipient`, `NoSanctionedRecipients`), assembly calldata decoding in sanctions helpers.

**Rationale:** The soulbound `OlympiaMemberNFT` already implements `IVotes` via `ERC721Votes`, so standard `GovernorVotes` works directly. This is the simplest, most battle-tested approach. The `IOlympiaVotingModule` abstraction layer can be introduced later via OIP when module swapping is needed.

### 4. Self-Upgrade via OIP

**Draft spec language:** `updateGovernanceParams()` and `updateVotingModule()` via OIP process.

**Implementation:** Governor exposes `updateSanctionsOracle()` and governance parameter setters gated by `onlyGovernance` modifier. These execute through the standard Governor → Timelock → Executor pipeline. Voting module swap (`updateVotingModule()`) is deferred to a future release when the `IOlympiaVotingModule` interface is wired in.

### 5. Sanctions Integration (ECIP-1119)

**Draft spec:** Not present in original ECIP-1113; sanctions defined separately in ECIP-1119.

**Implementation:** Three-layer defense integrated into the Governor and Executor:
1. **`propose()` early check** — parses calldatas, extracts recipient, checks `SanctionsOracle.isSanctioned()`. Reverts if sanctioned.
2. **`cancelIfSanctioned(uint256 proposalId)`** — permissionless, callable anytime during proposal lifecycle.
3. **`OlympiaExecutor.executeTreasury()` final gate** — atomic revert before `Treasury.withdraw()`.

## Governance Architecture

```
OlympiaGovernor ──→ Timelock ──→ OlympiaExecutor ──→ Treasury.withdraw()
       ↑                                ↑
GovernorVotes (OZ)              SanctionsOracle (ECIP-1119)
       ↑
OlympiaMemberNFT (soulbound ERC721Votes)
```

## Contract Interfaces

### IOlympiaVotingModule (Forward-Looking Spec Artifact)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

/// @title IOlympiaVotingModule
/// @notice Modular voting power interface for OlympiaGovernor (ECIP-1113)
/// @dev Demo v0.2 uses standard OZ GovernorVotes with soulbound
///      OlympiaMemberNFT. This interface defines the swappable voting module
///      pattern for governance-gated upgrades via OIP in future releases.
interface IOlympiaVotingModule {
    function votingPower(address account, uint256 timepoint) external view returns (uint256);
    function isEligible(address account, uint256 timepoint) external view returns (bool);
}
```

### OlympiaMemberNFT (Phase 2A — Complete)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {ERC721} from "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import {ERC721Enumerable} from "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import {ERC721Votes} from "@openzeppelin/contracts/token/ERC721/extensions/ERC721Votes.sol";
import {EIP712} from "@openzeppelin/contracts/utils/cryptography/EIP712.sol";
import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";
import {IERC5192} from "./interfaces/IERC5192.sol";

contract OlympiaMemberNFT is ERC721, ERC721Enumerable, ERC721Votes, IERC5192, AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    uint256 private _nextTokenId;

    error SoulboundTransferBlocked();

    constructor(address admin) ERC721("Olympia Member", "OLYMPIA") EIP712("Olympia Member", "1") {
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        _grantRole(MINTER_ROLE, admin);
    }

    function safeMint(address to) external onlyRole(MINTER_ROLE) {
        uint256 tokenId = _nextTokenId++;
        _safeMint(to, tokenId);
    }

    function locked(uint256 tokenId) external view returns (bool) {
        _requireOwned(tokenId);
        return true;
    }

    function _update(address to, uint256 tokenId, address auth)
        internal override(ERC721, ERC721Enumerable, ERC721Votes) returns (address)
    {
        address from = super._update(to, tokenId, auth);
        if (from != address(0) && to != address(0)) revert SoulboundTransferBlocked();
        if (from == address(0) && to != address(0)) {
            _delegate(to, to);  // Auto-delegate on mint
            emit Locked(tokenId);
        }
        return from;
    }

    function _increaseBalance(address account, uint128 value)
        internal override(ERC721, ERC721Enumerable, ERC721Votes)
    { super._increaseBalance(account, value); }

    function supportsInterface(bytes4 interfaceId)
        public view override(ERC721, ERC721Enumerable, AccessControl) returns (bool)
    { return interfaceId == type(IERC5192).interfaceId || super.supportsInterface(interfaceId); }
}
```

### OlympiaGovernor (Phase 2B — Complete)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {Governor} from "@openzeppelin/contracts/governance/Governor.sol";
import {GovernorSettings} from "@openzeppelin/contracts/governance/extensions/GovernorSettings.sol";
import {GovernorCountingSimple} from "@openzeppelin/contracts/governance/extensions/GovernorCountingSimple.sol";
import {GovernorVotes} from "@openzeppelin/contracts/governance/extensions/GovernorVotes.sol";
import {GovernorVotesQuorumFraction} from "@openzeppelin/contracts/governance/extensions/GovernorVotesQuorumFraction.sol";
import {GovernorTimelockControl} from "@openzeppelin/contracts/governance/extensions/GovernorTimelockControl.sol";
import {IVotes} from "@openzeppelin/contracts/governance/utils/IVotes.sol";

contract OlympiaGovernor is
    Governor,
    GovernorSettings,
    GovernorCountingSimple,
    GovernorVotes,
    GovernorVotesQuorumFraction,
    GovernorTimelockControl
{
    ISanctionsOracle public sanctionsOracle;

    constructor(
        string memory name,
        IVotes _token,                     // OlympiaMemberNFT (soulbound ERC721Votes)
        ISanctionsOracle _sanctionsOracle,
        TimelockController _timelock,
        uint48 votingDelay_,
        uint32 votingPeriod_,
        uint256 quorumPercent_             // e.g., 4 = 4% of NFT holders
    )
        Governor(name)
        GovernorSettings(votingDelay_, votingPeriod_, 0)
        GovernorVotes(_token)
        GovernorVotesQuorumFraction(quorumPercent_)
        GovernorTimelockControl(_timelock)
    {
        sanctionsOracle = _sanctionsOracle;
    }

    // --- Sanctions integration (ECIP-1119) ---

    function propose(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        string memory description
    ) public override returns (uint256) {
        // Layer 1: Early sanctions check — extract recipients from calldatas
        for (uint256 i = 0; i < targets.length; i++) {
            if (calldatas[i].length >= 36) {
                address recipient = abi.decode(_slice(calldatas[i], 4, 36), (address));
                require(!sanctionsOracle.isSanctioned(recipient), "OlympiaGovernor: sanctioned recipient");
            }
        }
        return super.propose(targets, values, calldatas, description);
    }

    function cancelIfSanctioned(uint256 proposalId) external {
        // Layer 2: Permissionless — anyone can cancel a proposal targeting a sanctioned address
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

    function updateSanctionsOracle(ISanctionsOracle newOracle) external onlyGovernance {
        sanctionsOracle = newOracle;
    }
}
```

### OlympiaExecutor (Phase 2B — Complete)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

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

Demo v0.2 contracts deployed on both Mordor and ETC mainnet via CREATE2 (salt: `keccak256("OLYMPIA_DEMO_V0_2")`). Deployer: `0x7C3311F29e318617fed0833E68D6522948AaE995`.

| Contract | Address (Mordor + ETC) | Status |
|----------|----------------------|--------|
| SanctionsOracle | `0xfF2B8D7937D908D81C72D20AC99302EE6ACc2709` | ✅ Deployed |
| OlympiaMemberNFT | `0x73e78d3a3470396325b975FcAFA8105A89A9E672` | ✅ Deployed |
| OlympiaGovernor | `0xB85dbc899472756470EF4033b9637ff8fa2FD23D` | ✅ Deployed |
| OlympiaExecutor | `0x64624f74F77639CbA268a6c8bEDC2778B707eF9a` | ✅ Deployed |
| TimelockController | `0xA5839b3e9445f7eE7AffdBC796DC0601f9b976C2` | ✅ Deployed |
| ECFPRegistry | `0xFB4De5674a6b9a301d16876795a74f3bdacfa722` | ✅ Deployed |

## Gap Analysis

| Spec Requirement | Status | Notes |
|-----------------|--------|-------|
| SanctionsOracle ([ECIP-1119](https://ecips.ethereumclassic.org/ECIPs/ecip-1119)) | ✅ Complete | MANAGER_ROLE, deployed |
| OlympiaMemberNFT (soulbound) | ✅ Complete | ERC721Votes + ERC5192, deployed |
| IOlympiaVotingModule interface | ✅ Complete | Spec artifact (not wired in Demo v0.2) |
| IERC5192 interface | ✅ Complete | Soulbound standard |
| OlympiaGovernor | ✅ Complete | GovernorVotes + GovernorVotesQuorumFraction + GovernorPreventLateQuorum + GovernorStorage |
| OlympiaExecutor | ✅ Complete | Sanctions gate, immutable Treasury caller |
| TimelockController | ✅ Complete | Standard OZ timelock |
| ECFPRegistry | ✅ Complete | [ECIP-1114](https://ecips.ethereumclassic.org/ECIPs/ecip-1114) hash-bound proposals |
| Sanctions on `propose()` | ✅ Complete | Layer 1 early check |
| `cancelIfSanctioned()` | ✅ Complete | Layer 2 permissionless cancel |
| `updateSanctionsOracle()` via OIP | ✅ Complete | `onlyGovernance` setter |

## Staged Governance Lifecycle

```
Phase 1: Bootstrap
  Treasury deployed with immutable executor (pre-computed CREATE2 address)
  Treasury accumulates basefee + donations
  No withdrawals — executor contract not yet deployed (no code at that address)

Phase 2: CoreDAO Activation (Stage 2) ✅ Complete
  OlympiaGovernor deployed with sanctions integration
  OlympiaExecutor deployed at pre-computed CREATE2 address → governance activates
  Timelock configured: Governor=PROPOSER+CANCELLER, Executor=EXECUTOR
  Withdrawals enabled through Governor → Timelock → Executor → Treasury pipeline

Phase 3: Futarchy Integration (Stage 3)
  FutarchyExecutor deployed as additional authorized caller
  CoreDAO activates Futarchy as a child DAO via governance proposal; both paths share the Treasury
```

**Note:** Demo v0.2 Treasury is immutable from deployment — no admin, no roles, no staged admin transfer. The executor is fixed at construction time.

## OpenZeppelin Compatibility

### Phase 2A (Complete)

| Component | Version | Contract |
|-----------|---------|----------|
| ERC721 | v5.1.0 | Base NFT for OlympiaMemberNFT |
| ERC721Enumerable | v5.1.0 | totalSupply(), tokenByIndex() |
| ERC721Votes | v5.1.0 | Delegation + getPastVotes() |
| EIP712 | v5.1.0 | Structured data signing (required by ERC721Votes) |
| AccessControl | v5.1.0 | MINTER_ROLE (NFT), MANAGER_ROLE (Oracle) |

### Phase 2B (Complete)

| Component | Version | Contract |
|-----------|---------|----------|
| Governor | v5.1.0 | Proposal lifecycle |
| GovernorSettings | v5.1.0 | Voting delay, period, proposal threshold |
| GovernorCountingSimple | v5.1.0 | For/Against/Abstain (Bravo-style) |
| GovernorVotes | v5.1.0 | Reads soulbound NFT votes directly |
| GovernorVotesQuorumFraction | v5.1.0 | Quorum = % of NFT holders |
| GovernorTimelockControl | v5.1.0 | Timelock integration |
| GovernorPreventLateQuorum | v5.1.0 | Extends voting if quorum reached late |
| GovernorStorage | v5.1.0 | On-chain proposal storage |
| TimelockController | v5.1.0 | Delay queue |

## Deployment Sequence

```
Phase 2A (Complete — olympia-governance-contracts repo):
1. ✅ Build SanctionsOracle (ECIP-1119) — 14 tests
2. ✅ Build OlympiaMemberNFT (soulbound ERC721Votes + ERC5192) — 19 tests
3. ✅ Build IOlympiaVotingModule interface (spec artifact)

Phase 2B (Complete — deployed Mordor + ETC):
4. ✅ Deploy SanctionsOracle → 0xfF2B...2709
5. ✅ Deploy OlympiaMemberNFT → 0x73e7...9672
6. ✅ Deploy TimelockController → 0xA583...f9C2
7. ✅ Deploy OlympiaExecutor (treasury + timelock + sanctionsOracle) → 0x6462...eF9a
8. ✅ Deploy OlympiaGovernor (GovernorVotes reads OlympiaMemberNFT directly) → 0xB85d...F23D
9. ✅ Deploy ECFPRegistry → 0xFB4D...a722
10. ✅ Configure Timelock: Governor=PROPOSER+CANCELLER, Executor=EXECUTOR
11. ✅ Executor is Treasury's immutable authorized caller (set at construction)
12. ✅ E2E lifecycle tested: ECFP-001 submitted, voted, queued, executed
```

## Test Suite (106 tests — Complete)

### Phase 2A

**SanctionsOracle (14 tests):**
- Add/remove happy path + events
- isSanctioned state transitions
- Revert: duplicate add, remove non-sanctioned, zero address
- Access control: only MANAGER_ROLE, admin can grant

**OlympiaMemberNFT (19 tests):**
- Mint: assigns token, emits Transfer + Locked
- Auto-delegate: getVotes(recipient) == 1 after mint
- Soulbound: transfer reverts (SoulboundTransferBlocked)
- getPastVotes: snapshot correctness with vm.roll()
- Multiple mints: totalSupply increments, enumeration works
- Locked: locked(tokenId) returns true
- supportsInterface: ERC721, ERC721Enumerable, IERC5192, AccessControl
- Access control: only MINTER_ROLE can mint

### Phase 2B (Complete)

**OlympiaGovernor + OlympiaExecutor + TimelockController + ECFPRegistry (73 tests):**
- Full lifecycle: deposit → propose → vote → queue → timelock → execute → receive
- Sanctions: early rejection on propose, mid-lifecycle cancel, execution-time revert
- Governance upgrades: `updateSanctionsOracle()` via OIP
- Access control: unauthorized calls revert at every layer
- ECFPRegistry: submit, draft lifecycle, review period, input validation
- Edge cases: zero-value, duplicate execution, cancelled proposals, quorum failures
- Late quorum extension (GovernorPreventLateQuorum)
- Proposal storage and retrieval (GovernorStorage)

## Verification

```bash
# Run tests (106 tests)
cd olympia-governance-contracts && forge test -vv

# Verify governance parameters
cast call 0xB85dbc899472756470EF4033b9637ff8fa2FD23D "votingDelay()" --rpc-url $MORDOR_RPC_URL
cast call 0xB85dbc899472756470EF4033b9637ff8fa2FD23D "votingPeriod()" --rpc-url $MORDOR_RPC_URL

# Verify GovernorVotes reads OlympiaMemberNFT
cast call 0xB85dbc899472756470EF4033b9637ff8fa2FD23D "token()" --rpc-url $MORDOR_RPC_URL
# Expected: 0x73e78d3a3470396325b975FcAFA8105A89A9E672

# Verify Executor
cast call 0x64624f74F77639CbA268a6c8bEDC2778B707eF9a "treasury()" --rpc-url $MORDOR_RPC_URL
# Expected: 0x035b2e3c189B772e52F4C3DA6c45c84A3bB871bf
cast call 0x64624f74F77639CbA268a6c8bEDC2778B707eF9a "timelock()" --rpc-url $MORDOR_RPC_URL
# Expected: 0xA5839b3e9445f7eE7AffdBC796DC0601f9b976C2

# Verify Sanctions integration
cast call 0xB85dbc899472756470EF4033b9637ff8fa2FD23D "sanctionsOracle()" --rpc-url $MORDOR_RPC_URL
# Expected: 0xfF2B8D7937D908D81C72D20AC99302EE6ACc2709
```

## Production Notes

Demo v0.2 deployed with OpenZeppelin v5.1.0 (Shanghai EVM compatible). Production deployment targets OZ v5.6.0 (requires Cancun opcodes via Olympia fork). All production addresses differ: governance CREATE2 addresses change (OZ 5.6 bytecode), and the Treasury CREATE address changes (different deployer EOA, nonce 0). The Executor CREATE2 address is affected by both — its constructor args include the Treasury address and TimelockController address, both of which change. `PrecomputeAddresses.s.sol` recomputes the full address set before deployment.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)
