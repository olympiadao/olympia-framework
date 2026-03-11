---
ecip: 1114
title: Ethereum Classic Funding Proposal Process — Implementation Spec
status: Implementation
type: Standards Track
category: ECBP
requires: 1113
author: Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)
created: 2025-07-04
updated: 2026-03-08
license: CC0-1.0
---

# ECIP-1114: Ethereum Classic Funding Proposal Process — Implementation Spec

## Overview

ECIP-1114 defines the Ethereum Classic Funding Proposal (ECFP) process — the exclusive, standardized, and permissionless mechanism for requesting and executing funding from the Olympia Treasury (ECIP-1112). ECFPs are application-layer funding proposals submitted by any participant, reviewed and voted on through Olympia DAO governance (ECIP-1113), and executed only via the Governor → Timelock → Executor pipeline.

This process governs the allocation of basefee revenue redirected under ECIP-1111.

## Amendments from Draft Spec

### 1. Hash-Bound Execution Model

**Draft spec:** Treasury validates disbursements via hash-bound model using `(ecfpId, recipient, amount, metadataCID, chainid)`.

**Implementation:** Hash-bound validation lives in `ECFPRegistry`, not in the Treasury contract. The Treasury remains a minimal vault with `withdraw(address payable, uint256)`. The ECFPRegistry provides the hash-bound integrity layer at the application level.

**Rationale:** Keeping the Treasury minimal and immutable (ECIP-1112) means hash-bound logic should not be embedded in it. The ECFPRegistry validates proposal integrity before the Governor encodes the withdrawal call.

### 2. ECFPRegistry Contract

**Draft spec:** Describes proposal lifecycle and data model but not a specific registry contract.

**Implementation:** On-chain `ECFPRegistry` contract for permissionless proposal submission with hash-bound identifiers.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract ECFPRegistry {
    struct Proposal {
        bytes32 ecfpId;
        address recipient;
        uint256 amount;
        bytes32 metadataCID;
        address proposer;
        uint256 timestamp;
        ProposalStatus status;
    }

    enum ProposalStatus { Draft, Active, Approved, Rejected, Executed, Expired }

    mapping(bytes32 => Proposal) public proposals;

    event ProposalSubmitted(bytes32 indexed hashId, bytes32 ecfpId, address recipient, uint256 amount, bytes32 metadataCID);
    event ProposalActivated(bytes32 indexed hashId);
    event ProposalQueued(bytes32 indexed hashId);
    event ProposalExecuted(bytes32 indexed hashId, address recipient, uint256 amount, uint256 timestamp);

    function submit(
        bytes32 ecfpId,
        address recipient,
        uint256 amount,
        bytes32 metadataCID
    ) external returns (bytes32 hashId) {
        hashId = keccak256(abi.encodePacked(ecfpId, recipient, amount, metadataCID, block.chainid));
        require(proposals[hashId].timestamp == 0, "ECFPRegistry: duplicate");

        proposals[hashId] = Proposal({
            ecfpId: ecfpId,
            recipient: recipient,
            amount: amount,
            metadataCID: metadataCID,
            proposer: msg.sender,
            timestamp: block.timestamp,
            status: ProposalStatus.Draft
        });

        emit ProposalSubmitted(hashId, ecfpId, recipient, amount, metadataCID);
    }

    function computeHashId(
        bytes32 ecfpId,
        address recipient,
        uint256 amount,
        bytes32 metadataCID
    ) external view returns (bytes32) {
        return keccak256(abi.encodePacked(ecfpId, recipient, amount, metadataCID, block.chainid));
    }
}
```

### 3. Execution Path

**Draft spec:** Executor forwards opaque `{targets, values, calldatas}` payload to Treasury.

**Implementation:** The execution path is: ECFPRegistry (proposal data) → Governor (propose/vote) → Timelock (queue/delay) → OlympiaExecutor (sanctions check + `Treasury.withdraw()`). The ECFP hash-bound identifier is validated at the application layer by the ECFPRegistry before the Governor proposal is created. The Executor receives only `(recipient, amount)` — the Treasury's `withdraw()` interface.

## ECFP Lifecycle

```
1. Submit     → ECFPRegistry.submit(ecfpId, recipient, amount, metadataCID)
2. Draft      → Open for discussion; not yet eligible for voting
3. Active     → Governor proposal created referencing hash-bound ID
4. Voting     → Snapshot-block voting via IOlympiaVotingModule
5. Approved   → Queued in Timelock
6. Executed   → Timelock → OlympiaExecutor → Treasury.withdraw()
7. Completed  → Funds disbursed, ProposalExecuted event emitted
```

## ECFP Data Model

| Field | Type | Description |
|-------|------|-------------|
| `ecfpId` | `bytes32` | Unique proposal identifier |
| `recipient` | `address` | Address to receive funds |
| `amount` | `uint256` | Requested funding amount (wei) |
| `metadataCID` | `bytes32` | Content-addressed hash (IPFS/Arweave) binding off-chain metadata |
| `chainid` | `uint256` | Chain ID for replay protection |
| `hashId` | `bytes32` | `keccak256(ecfpId, recipient, amount, metadataCID, chainid)` |

## Proposal Statuses

| Status | Description |
|--------|-------------|
| `Draft` | Submitted; open for discussion; not yet eligible for voting |
| `Active` | Undergoing governance voting |
| `Approved` | Passed quorum and threshold; queued in Timelock |
| `Rejected` | Failed quorum or approval threshold |
| `Executed` | Treasury disbursement successfully completed |
| `Expired` | Did not activate or queue within validity window |

## Deployments

| Network | Contract | Address | Status |
|---------|----------|---------|--------|
| Mordor | ECFPRegistry | TBD | Pending deployment (Phase 2) |

## Implementation Status

ECFPRegistry is Phase 2 of the development plan. Phase 1 (CoreDAO pipeline) must be operational before ECFP proposals can flow through the system.

## Relationship to Other ECIPs

- **ECIP-1112:** Treasury contract — receives `withdraw()` calls. Does not interpret ECFP data.
- **ECIP-1113:** Governor → Timelock → Executor pipeline — executes approved ECFPs.
- **ECIP-1119:** Sanctions constraint — checked at propose(), cancelIfSanctioned(), and execute().

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)
