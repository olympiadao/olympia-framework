---
ecip: 1117
title: Olympia Futarchy Governance Framework — Implementation Spec
status: Implementation
type: Meta
requires: 1112, 1116
replaces: 1113
author: Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)
created: 2025-12-14
updated: 2026-03-08
license: CC0-1.0
---

# ECIP-1117: Olympia Futarchy Governance Framework — Implementation Spec

## Overview

ECIP-1117 defines an application-layer governance system for allocating funds from the Olympia Treasury using futarchy-style decision-making via conditional prediction markets. For each funding proposal, the system creates paired markets that estimate protocol welfare under "approve" versus "reject" outcomes, and authorizes Treasury disbursements only when market-derived signals satisfy predefined approval thresholds.

This is the **public market governance platform** (Stage 3) — for contentious or experimental proposals outside mandatory core/critical infrastructure scope. Built to amplify on-chain transactions — positive flywheel inside the governance system itself.

**Stage:** Phase 3 — Futarchy DAO Pipeline. CoreDAO (Stage 2) codes a governance proposal to redirect funds into the Futarchy system.

## Architecture

```
Welfare Metric Registry
        ↓
Proposal Registry (permissionless submission + collateral bond)
        ↓
Conditional Market Factory (paired approve/reject markets)
        ↓
Market Operation & Privacy Layer (encrypted submissions, LMSR AMM)
        ↓
Oracle & Resolution System (multi-stage: reporting → challenge → escalation)
        ↓
Settlement & Execution Engine → SanctionsOracle (ECIP-1119) → Treasury.withdraw()
        ↓
Minority Exit Mechanism (proportionate withdrawal for dissenters)
```

## Key Components

### Welfare Metric Registry
On-chain registry of governance-approved metrics (security, usage, financial sustainability). Supports versioning — metric changes apply only to new proposals.

### Conditional Market Factory
Creates paired prediction markets per proposal:
- **Approval Market:** Expected welfare conditional on proposal execution
- **Rejection Market:** Expected welfare conditional on proposal rejection

Markets use LMSR (Logarithmic Market Scoring Rule) AMM for continuous price availability and bounded loss.

### Privacy and Anti-Collusion
- Encrypted position submission during trading period
- Revocable commitments (prevents credible vote-buying)
- Aggregate transparency post-settlement
- All mechanisms operate at contract layer — no consensus changes

### Oracle and Resolution
Multi-stage resolution: designated reporting → challenge period → escalation. Deterministic inputs to approval checks.

## Deployments

### Mordor (Current — prototype)

| Contract | Address | Status |
|----------|---------|--------|
| OlympiaFutarchyGovernor | `0xEc4AA90c812a997EA0Aa5BDc1A5777B75fB2db54` | Deployed |
| LMSRMarketMaker | Deployed alongside FutarchyGovernor | Deployed |
| WelfareMetricOracleAdapter | Deployed alongside FutarchyGovernor | Deployed |

**Note:** These are prototype deployments. Production deployment occurs in Phase 3 after CoreDAO (Stage 2) is operational.

## Implementation Status

| Component | Status | Notes |
|-----------|--------|-------|
| OlympiaFutarchyGovernor | Deployed (prototype) | Needs integration with SanctionsOracle |
| LMSRMarketMaker | Deployed (prototype) | Paired conditional markets functional |
| WelfareMetricOracleAdapter | Deployed (prototype) | Oracle integration |
| Conditional Market Factory | Deployed (prototype) | Deterministic market creation |
| Privacy Layer | Not implemented | Phase 3 enhancement |
| Minority Exit Mechanism | Not implemented | Phase 3 enhancement |
| SanctionsOracle integration | Not implemented | Required before production |

## Economic Model

- **Zero explicit trading fees** — markets funded through Treasury (basefee accumulation), not per-trade extraction
- **Collateral bonds** for proposal submission (spam deterrence)
- **LMSR AMM** provides bounded loss for Treasury-provided liquidity
- Treasury capital is recyclable across markets (recovered at resolution)

## Relationship to CoreDAO (ECIP-1113)

ECIP-1117 replaces ECIP-1113 for the **futarchy governance path**. However, in the staged rollout:

1. **Stage 2:** CoreDAO (ECIP-1113) activates first — handles non-contentious monthly expenses, core development, critical infrastructure
2. **Stage 3:** CoreDAO codes a governance proposal to redirect funds into Futarchy
3. Both governance paths share the same SanctionsOracle (ECIP-1119) and Executor pattern

CoreDAO handles mandatory operational needs. Futarchy handles everything else — contentious, experimental, or public-facing proposals.

## Sanctions Integration

Same SanctionsOracle, same Executor pattern as CoreDAO:
- FutarchyGovernor routes through FutarchyExecutor
- FutarchyExecutor checks SanctionsOracle before Treasury.withdraw()
- Same three-layer defense (propose check + cancel + execute check)

## Test Suite (Existing)

The Olympia Futarchy research repo at `/media/dev/2tb/dev/olympia-futarchy/prediction-dao-research/` has 1,345 tests covering:
- Paired conditional market creation and settlement
- LMSR pricing and bounded loss
- Oracle resolution mechanics
- Welfare metric registry operations

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)
