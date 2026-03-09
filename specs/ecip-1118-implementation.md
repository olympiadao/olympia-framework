---
ecip: 1118
title: Olympia Futarchy Funding and Disbursement Framework — Implementation Spec
status: Implementation
type: Meta
requires: 1112, 1116, 1117
replaces: 1114
author: Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)
created: 2024-12-14
updated: 2026-03-08
license: CC0-1.0
---

# ECIP-1118: Olympia Futarchy Funding and Disbursement Framework — Implementation Spec

## Overview

ECIP-1118 defines the funding and disbursement framework for Treasury funds used to support prediction market infrastructure and governance-approved proposals under the Futarchy system (ECIP-1117). Funds are categorized into predefined allocation classes and deployed through milestone-gated streaming disbursements.

**Stage:** Phase 3 — Futarchy DAO Pipeline. Activates alongside ECIP-1117.

ECIP-1118 replaces ECIP-1114 for the futarchy governance path. ECIP-1114 remains authoritative for the CoreDAO path (ECIP-1113).

## Treasury Capital Allocation Categories

| Category | Purpose | Constraints |
|----------|---------|-------------|
| **Market Liquidity Provision** | Baseline liquidity for prediction markets | Bounded-loss, recyclable across markets |
| **Market Participant Incentives** | Rewards for liquidity providers, market makers | Objective performance criteria, terminable |
| **Infrastructure Operations** | Data services, compliance inputs, UI, monitoring | Recurring, fixed review cadence |
| **Strategic & Emergency Reserves** | Unexpected disruptions, revenue shortfalls | Stricter access controls, maintained reserves |

Rebalancing between categories requires governance approval (ECIP-1117).

## Funding Proposal Classes

| Class | Scope | Requirements |
|-------|-------|-------------|
| **Infrastructure Service** | Continuous/recurring services for market operation | Service scope, duration, performance indicators, termination conditions |
| **Market Liquidity & Incentive** | Liquidity provisioning, quality improvement | Targeted markets, performance criteria, capital bounds |
| **Development & Tooling** | Software, interfaces, governance tooling | Deliverables, milestones, verification methods, maintenance |
| **Exceptional/Emergency** | Temporary extraordinary funding | Justified exceptional status, heightened scrutiny |

## Disbursement Mechanics

### Streaming Disbursements

Funds released incrementally over time rather than lump-sum:
- Deterministic total funding caps
- Time-based or milestone-based release schedules
- Governance-authorized cancellation/suspension
- Recovery of undistributed funds upon termination

### Milestone Gating

- Milestones defined at proposal approval time
- Objective verification criteria (on-chain data, third-party attestation, governance review)
- Subsequent tranches blocked until milestone verified
- Failure → funding suspension, modification, or termination

### Clawback Authority

Governance retains authority to:
- Terminate active disbursement streams
- Reclaim unearned/undistributed funds
- Return recovered funds to Treasury for future use
- Cannot retroactively invalidate funds earned under verified milestones

## Contract Architecture

### StreamingDisbursement

```solidity
interface IStreamingDisbursement {
    function createStream(
        address recipient,
        uint256 totalAmount,
        uint256 startTime,
        uint256 endTime,
        bytes32[] calldata milestoneHashes
    ) external returns (uint256 streamId);

    function verifyMilestone(uint256 streamId, uint256 milestoneIndex, bytes calldata proof) external;
    function claimable(uint256 streamId) external view returns (uint256);
    function claim(uint256 streamId) external;
    function terminate(uint256 streamId) external;
}
```

### MilestoneVerifier

```solidity
interface IMilestoneVerifier {
    function verify(bytes32 milestoneHash, bytes calldata proof) external view returns (bool);
}
```

### ClawbackController

```solidity
interface IClawbackController {
    function terminate(uint256 streamId) external;
    function reclaim(uint256 streamId) external returns (uint256 recovered);
}
```

## Implementation Status

| Component | Status | Priority |
|-----------|--------|----------|
| IStreamingDisbursement interface | Specified | Phase 3 |
| IMilestoneVerifier interface | Specified | Phase 3 |
| IClawbackController interface | Specified | Phase 3 |
| StreamingDisbursement contract | Not implemented | Phase 3 |
| MilestoneVerifier contract | Not implemented | Phase 3 |
| ClawbackController contract | Not implemented | Phase 3 |

## Sustainability Constraints

- Aggregate disbursements should not exceed Treasury inflows over extended periods
- Persistent imbalances trigger governance review
- Non-critical programs may be paused if sustainability thresholds breached
- Strategic reserves maintained for operational continuity

## Sanctions Integration

Same SanctionsOracle (ECIP-1119) applies to all disbursements:
- Recipient checked at streaming disbursement creation
- Checked again at each claim
- Fail-closed: sanctioned recipients cannot claim

## Backwards Compatibility

ECIP-1118 replaces ECIP-1114 for the futarchy path. Existing proposals under ECIP-1114 may continue under original terms. No changes to Treasury custody, basefee allocation, or governance approval mechanisms.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)
