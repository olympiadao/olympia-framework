# Olympia Framework

> **Demo v0.1** — Not Olympia ECIP spec compliant. Deployed for fast iterative development to build project scaffolding. Not for production use. See `demo_v0.2` for the spec-aligned deployment.

**A Staged Governance and Funding System for Ethereum Classic**

Authors: Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)
Date: March 2026
License: CC0-1.0

---

## What Is Olympia?

Olympia is a framework for sustainable protocol funding on Ethereum Classic. It redirects a small portion of transaction fees (the EIP-1559 basefee) into an on-chain Treasury, then builds governance layers on top to decide how those funds are spent.

The framework is designed as a **staged rollout** — each stage addresses a specific concern and unlocks the next. Nothing is optional; each layer builds on the operational reality of the previous one.

**Core thesis:** ETC's block rewards decline every 5 million blocks (ECIP-1017). Fee revenue is currently negligible. As block rewards drop, the network needs a mechanism to fund core development, critical infrastructure, and security incentives. Olympia provides that mechanism without changing the emission schedule or monetary policy.

---

## The Problem

1. **Block reward decline.** Era 4 (current): 2.048 ETC/block. Era 5: 1.6384 ETC/block. Era 6: 1.31072 ETC/block. Each era is a 20% reduction.

2. **No protocol-level funding.** ETC relies entirely on external donations (ETC Coop, Grayscale trust) for development, infrastructure, and tooling. These are voluntary and fragile.

3. **Fee revenue is negligible.** At current activity (~5 txs/block), tips represent 0.01% of miner income. The basefee is effectively zero. But if usage grows post-Olympia (EVM compatibility, new tx types), this changes.

4. **No governance mechanism.** Even if funds accumulate, there's no on-chain process to allocate them transparently.

Olympia solves all four by building from the bottom up: first accumulate, then govern, then experiment, then harden.

---

## Five Stages

```
┌───────────────────────────────────────────────────────┐
│  STAGE 1: OLYMPIA HARD FORK                           │
│  Consensus layer — basefee redirect + EVM upgrade     │
│  ECIPs: 1111, 1112, 1121                              │
│  Mordor: block TBD                                    │
│  Mainnet: block TBD                                   │
└───────────────┬───────────────────────────────────────┘
                │ Treasury accumulates basefee + donations
                │ No withdrawals — iron out governance details
                ▼
┌───────────────────────────────────────────────────────┐
│  STAGE 2: CoreDAO GOVERNANCE                          │
│  Contract layer — traditional DAO pipeline            │
│  ECIPs: 1113, 1114, 1119                              │
│  First withdrawals enabled                            │
└───────────────┬───────────────────────────────────────┘
                │ Core needs met, redirect to public platform
                ▼
┌───────────────────────────────────────────────────────┐
│  STAGE 3: FUTARCHY DAO                                │
│  Contract layer — prediction market governance        │
│  ECIPs: 1117, 1118                                    │
│  Open platform for experimental proposals             │
└───────────────┬───────────────────────────────────────┘
                │ Empirical data from fee market
                ▼
┌───────────────────────────────────────────────────────┐
│  STAGE 4: MINER DISTRIBUTION EXPERIMENTATION          │
│  Contract layer — L-curve smoothing                   │
│  ECIP: 1115                                           │
│  Test curves, amounts, strategies                     │
└───────────────┬───────────────────────────────────────┘
                │ Proven parameters → protocol embedding
                ▼
┌───────────────────────────────────────────────────────┐
│  STAGE 5: PROTOCOL HARDCODE                           │
│  Consensus layer — second hard fork                   │
│  ECIPs: 1116, 1122                                    │
│  Embed validated split + distribution at protocol     │
└───────────────────────────────────────────────────────┘
```

---

## Current Status (March 2026)

```
Stage 1: COMPLETE ████████████████████ — 3 clients, treasury deployed
Stage 2: DEPLOYED ████████████████░░░░ — All 7 contracts on Mordor, E2E tested
Stage 3: RESEARCH ████░░░░░░░░░░░░░░░░ — 1,345 tests, contracts pending
Stage 4: DEFERRED ░░░░░░░░░░░░░░░░░░░░ — Requires fee market data
Stage 5: DEFERRED ░░░░░░░░░░░░░░░░░░░░ — Requires Stage 4 data
```

**Completed:**
- Hard fork consensus code in 3 ETC clients (core-geth, besu, fukuii) — all tests pass
- Treasury deployed on Mordor + ETC mainnet at `0xd6165...Fb0e37`
- Full governance pipeline deployed on Mordor (7 contracts, 87 tests)
- E2E governance lifecycle tested (ECFP-001 submitted, voted, queued, executed)
- Governance dApp in active development (olympia-app)
- Futarchy research complete (1,345 tests)
- Brand assets, landing pages (olympiadao.org, olympiatreasury.org) live

**In progress:**
- Next step: Mordor Testnet Deployment
- Governance dApp polish and deployment

**Next milestones:**
- Mordor hard fork activation (block TBD)
- Additional governance lifecycle tests (ECFP-002, 003, 004)
- ETC mainnet activation (block TBD)

---

## Stage 1: Olympia Hard Fork

**ECIPs:** 1111, 1112, 1121
**Type:** Consensus layer (hard fork)
**Activation:** Mordor block TBD, ETC mainnet block TBD

Stage 1 does two things: activates EIP-1559 (redirecting basefee to a Treasury instead of burning it) and brings ETC's EVM up to parity with Ethereum's latest opcodes and precompiles.

### ECIP-1111: EIP-1559 + EIP-3198

EIP-1559 introduces dynamic basefee pricing and Type-2 transactions. On Ethereum, the basefee is burned. On ETC, 100% of the basefee is credited to the Treasury address during block finalization via consensus-layer state credit.

**Impact on miners:** At current activity levels (~5 txs/block), the basefee adds ~1 gwei per transaction — less than 0.01% of miner income. Priority fees (tips) remain 100% payable to miners. Block rewards are unchanged.

```
Block Finalization:
  baseFeeAmount = block.gasUsed × block.baseFee
  state.AddBalance(TreasuryAddress, baseFeeAmount)    ← ETC deviation
  // Ethereum burns this; ETC redirects to Treasury
```

EIP-3198 provides the `BASEFEE` opcode (0x48) so smart contracts can read the current basefee.

### ECIP-1112: Olympia Treasury

An immutable vault contract deployed via CREATE2 at a deterministic address. The Treasury:

- Receives basefee via consensus state credit
- Also accepts voluntary donations (ETC Coop, Grayscale, third parties)
- Permits withdrawals only through role-gated access control (`WITHDRAWER_ROLE`)
- Contains zero governance logic — it is a minimal vault

The Treasury uses OpenZeppelin's `AccessControlDefaultAdminRules` (v5.6) for staged governance activation:

```
Phase 1: Bootstrap
  DEFAULT_ADMIN_ROLE → deployer EOA
  WITHDRAWER_ROLE → deployer EOA
  Treasury accumulates, no withdrawals

Phase 2: CoreDAO Activation
  WITHDRAWER_ROLE → OlympiaExecutor
  DEFAULT_ADMIN_ROLE → CoreDAO governance (2-step transfer with delay)

Phase 3: Futarchy Integration
  Additional WITHDRAWER_ROLE → FutarchyExecutor

Phase 4: Admin Renouncement
  Admin renounced → immutable vault, no further role changes
```

**Deployment:** Demo v0.1 deployed at `0xd6165F3aF4281037bce810621F62B43077Fb0e37` on both Mordor and ETC mainnet via CREATE2 (salt: `keccak256("OLYMPIA_DEMO_V0_1")`). Client branches updated with the Treasury address.

### ECIP-1121: EVM Compatibility Sprint

13 EIPs activated alongside ECIP-1111 in the same fork. These are independent of the fee market — they bring ETC's EVM to parity with Ethereum's latest capabilities. (EIP-7642 excluded per published ECIP — not applicable to PoW chain.)

**Key additions:**
- **MCOPY** (EIP-5656): Efficient memory-to-memory copy
- **Transient storage** (EIP-1153): `TSTORE`/`TLOAD` — ephemeral within a transaction
- **BLS12-381** (EIP-2537): Cryptographic precompile for zero-knowledge proofs
- **EOA code delegation** (EIP-7702): Set code on externally owned accounts
- **secp256r1** (EIP-7951): WebAuthn/passkey support via precompile
- **Historical block hashes** (EIP-2935): Block hashes persisted in state
- **SELFDESTRUCT restriction** (EIP-6780): Forward compatibility
- **Gas and safety EIPs** (7623, 7825, 7883, 7934, 7935): Calldata cost, gas cap, block size

**Excluded:** All blob/data availability EIPs (ETC has no blob layer), all PoS/Beacon Chain EIPs.

**Client status:** All 3 clients (core-geth, besu, fukuii) have complete implementations. All tests pass.

---

## Stage 2: CoreDAO Governance

**ECIPs:** 1113, 1114, 1119
**Type:** Contract layer (no fork required)

Stage 2 transitions the Treasury from "accumulate only" to "functional withdrawals." This is the traditional DAO pipeline — not concerned with decentralization off the bat, focused on funding core development and critical infrastructure transparently.

### ECIP-1113: Olympia DAO Governance Framework

The governance pipeline:

```
OlympiaGovernor ──→ Timelock ──→ OlympiaExecutor ──→ Treasury.withdraw()
       ↑                                ↑
GovernorVotes (OZ)              SanctionsOracle (ECIP-1119)
       ↑
OlympiaMemberNFT (soulbound ERC721Votes)
```

**Key design decisions:**

1. **No governance token.** Olympia launches with soulbound NFT-based membership voting — no token distribution, no bootstrapping problem. Any future tokenization must go through an OIP (Olympia Improvement Proposal). One soulbound NFT = one vote. KYC/BrightID/Gitcoin Passport-verified accounts receive a non-transferable NFT via `MINTER_ROLE`.

2. **Standard OZ voting (Demo v0.1).** The Governor uses OZ `GovernorVotes` + `GovernorVotesQuorumFraction`, reading `getPastVotes()` directly from the soulbound `OlympiaMemberNFT`. No custom `_getVotes()` override, no adapter contract. The `IOlympiaVotingModule` interface is included as a forward-looking spec artifact for governance-gated module swaps in future releases.

3. **Soulbound enforcement.** `OlympiaMemberNFT` blocks transfers in `_update()` (allows mint/burn only). Implements ERC5192 `locked()` interface. Auto-delegates on mint via `_delegate(to, to)` so votes are active immediately. Uses OZ default block number clock mode.

4. **OlympiaExecutor.** A thin contract sitting between Timelock and Treasury. Holds `WITHDRAWER_ROLE`, checks the SanctionsOracle before every `Treasury.withdraw()` call. Immutable references to treasury, timelock, and sanctions oracle.

5. **Self-upgrade via OIP.** The Governor exposes `updateSanctionsOracle()` and governance parameter setters — all gated by `onlyGovernance`. The system can evolve without redeployment.

### ECIP-1114: ECFP (Funding Proposals)

The Ethereum Classic Funding Proposal (ECFP) process. Permissionless submission via an on-chain `ECFPRegistry`:

```
hashId = keccak256(ecfpId, recipient, amount, metadataCID, chainId)
```

Lifecycle: Submit → Draft → Active (voting) → Approved → Queued (timelock) → Executed → Completed.

Hash-bound integrity ensures the proposal that was approved is the exact proposal that executes. The ECFPRegistry handles this at the application layer — the Treasury itself remains a minimal vault.

### ECIP-1119: Sanctions Constraint

**Non-negotiable:** Sanctions MUST activate with withdrawals. No funds leave the Treasury without screening.

Three-layer defense:

| Layer | Where | Purpose |
|-------|-------|---------|
| 1 | `Governor.propose()` | Early rejection — saves wasted governance cycles |
| 2 | `cancelIfSanctioned(proposalId)` | Permissionless mid-lifecycle cancel — anyone can call |
| 3 | `Executor.executeTreasury()` | Final gate — atomic revert, the security invariant |

The `SanctionsOracle` is a governance-managed blocklist: `addAddress(address)` and `removeAddress(address)` via `MANAGER_ROLE`. Read-only `isSanctioned(address)` is called by Governor and Executor. Custom errors (`AlreadySanctioned`, `NotSanctioned`, `ZeroAddress`) for gas-efficient reverts. Fail-closed: if the oracle reverts, execution reverts.

---

## Stage 3: Futarchy DAO

**ECIPs:** 1117, 1118
**Type:** Contract layer

Once CoreDAO handles mandatory operational needs (Stage 2), Stage 3 opens a platform for contentious or experimental proposals. Built to amplify on-chain transactions — a positive flywheel inside the governance system itself.

### ECIP-1117: Futarchy Governance

Prediction market governance using paired conditional markets. For each proposal:

1. Two markets are created: "expected welfare if approved" vs. "expected welfare if rejected"
2. Market prices aggregate information from participants
3. If the approval market's price exceeds the rejection market's price by a threshold, the proposal passes
4. Execution routes through a FutarchyExecutor → SanctionsOracle → Treasury.withdraw()

**Key components:**
- **Welfare Metric Registry** — governance-approved metrics (security, usage, sustainability)
- **Conditional Market Factory** — deterministic paired market creation
- **LMSR AMM** — Logarithmic Market Scoring Rule for continuous pricing and bounded loss
- **Privacy layer** — encrypted position submission, revocable commitments (anti-collusion)
- **Minority exit** — proportionate withdrawal for dissenters

The research repo has 1,345 tests.

### ECIP-1118: Streaming Disbursements

Milestone-gated streaming for approved proposals. Funds release incrementally, not lump-sum.

**Allocation categories:** Market Liquidity, Participant Incentives, Infrastructure Operations, Strategic Reserves.

**Proposal classes:** Infrastructure Service, Market Liquidity & Incentive, Development & Tooling, Exceptional/Emergency.

**Safety mechanisms:**
- Milestone verification before each tranche release
- Governance-authorized clawback (can terminate streams, reclaim unearned funds)
- Cannot retroactively invalidate funds earned under verified milestones
- Sustainability constraint: disbursements should not persistently exceed inflows

---

## Stage 4: Miner Distribution Experimentation

**ECIP:** 1115
**Type:** Contract layer

Miners still have block rewards and fees are currently negligible. This stage matters most once the fee market is relevant — after Olympia activates and Type-2 transactions generate real basefee data.

### ECIP-1115: L-Curve Smoothing

A governance-controlled mechanism to reshape the timing of basefee-derived miner incentives:

```
R_k = f · B_k                          (smoothed portion)
A(k + j) = R_k · L(j)    for j = 1…N   (payout schedule)
```

Where:
- `B_k` is the basefee revenue for block `k`
- `f` is the governance-selected allocation fraction (0 ≤ f ≤ 1)
- `L(j)` is the weighting function across a window of `N` blocks
- All parameters adjustable via OIP without a hard fork

The `SmoothingModule` computes intended allocations. The `MinerRewardModule` maps outputs to miner addresses and prepares Treasury withdrawal calls. Neither module holds funds or modifies Treasury balances.

**Goal:** Empirical data. What smoothing curves, fee amounts, and strategies best stabilize the security budget across block reward decline cycles? Trust measurements, not opinions.

---

## Stage 5: Protocol Hardcode

**ECIPs:** 1116, 1122
**Type:** Consensus layer (second hard fork)

Only proceeds after Stage 4 produces empirical data validating which parameters are appropriate for protocol embedding. Too risky to hardcode without ironing out variables first.

### ECIP-1116: Basefee Split

Embeds a fixed split ratio at consensus. Proposed: 5% basefee → Treasury, 95% → miners. But the exact ratio comes from Stage 4 data.

```
TreasuryPortion      = floor(TotalBaseFee × 5 / 100)
BlockProducerPortion = TotalBaseFee − TreasuryPortion
```

### ECIP-1122: Protocol-Native Miner Distribution

Supersedes ECIP-1120 (istora, original author). Embeds the miner distribution curve at consensus — the pattern that Stage 4 experimentation proved optimal. Paired with ECIP-1116 in the same second hard fork.

---

## ECIP Index

| ECIP | Title | Stage | Type | Status |
|------|-------|-------|------|--------|
| 1111 | EIP-1559 + EIP-3198 | 1 | Consensus | Implemented (3 clients) |
| 1112 | Treasury Contract | 1 | Contract | Deployed (Mordor + ETC mainnet) |
| 1113 | CoreDAO Governance Framework | 2 | Contract | Deployed on Mordor (Demo v0.1) |
| 1114 | ECFP Funding Proposals | 2 | Contract | Deployed on Mordor (Demo v0.1) |
| 1115 | L-Curve Smoothing | 4 | Contract | Phase 4 |
| 1116 | Basefee Split (5%/95%) | 5 | Consensus | Deferred |
| 1117 | Futarchy DAO | 3 | Contract | Research complete |
| 1118 | Streaming Disbursements | 3 | Contract | Phase 3 |
| 1119 | Sanctions Constraint | 2 | Contract | Deployed on Mordor (Demo v0.1) |
| 1121 | EVM Compatibility Sprint | 1 | Consensus | Implemented (3 clients) |
| 1122 | Protocol-Native Miner Distribution | 5 | Consensus | Deferred |

---

## Client Status

Three independent ETC client implementations, all with complete Olympia support:

| Client | Language | Branch | Commit | Tests |
|--------|----------|--------|--------|-------|
| core-geth v1.12.21 | Go 1.24 | `olympia` | `b1c759dcc` | ALL PASS |
| besu-etc v26.3 | Java 21 | `olympia` | `52dc37b5bf` | ALL PASS |
| fukuii v0.1.240 | Scala 3.3 LTS | `olympia` | `126c1fd5c` | ALL PASS (2,308) |

Cross-client verification completed via six-client audit (March 2026). All consensus-critical bugs found and fixed. All 3 clients produce identical Treasury balances. Docker images built and smoke-tested for all 3 clients (pre-olympia + olympia tags).

---

## Contract Stack

All Demo v0.1 contracts use **Solidity 0.8.28**, **OpenZeppelin v5.6.0**, and **Foundry** for build/test/deploy.

## Deployment Addresses

All Demo v0.1 contracts deployed via CREATE2 (salt: `keccak256("OLYMPIA_DEMO_V0_1")`).

| Contract | Phase | Mordor | ETC Mainnet |
|----------|-------|--------|-------------|
| OlympiaTreasury | 1 ✅ | `0xd6165F3aF4281037bce810621F62B43077Fb0e37` | `0xd6165F3aF4281037bce810621F62B43077Fb0e37` |
| OlympiaGovernor | 2B ✅ | `0xEdbD61F1cE825CF939beBB422F8C914a69826dDA` | TBD |
| OlympiaExecutor | 2B ✅ | `0x94d4f74dDdE715Ed195B597A3434713690B14e97` | TBD |
| TimelockController | 2B ✅ | `0x1E0fADee5540a77012f1944fcce58677fC087f6e` | TBD |
| ECFPRegistry | 2B ✅ | `0xcB532fe70299D53Cc81B5F6365f56A108784d05d` | TBD |
| SanctionsOracle | 2A ✅ | `0xEeeb33c8b7C936bD8e72A859a3e1F9cc8A26f3B4` | TBD |
| OlympiaMemberNFT | 2A ✅ | `0x720676EBfe45DECfC43c8E9870C64413a2480EE0` | TBD |
| Deployer | — | `0x3b0952fB8eAAC74E56E176102eBA70BAB1C81537` | — |

### Governance Parameters (Mordor Demo v0.1)

| Parameter | Value |
|-----------|-------|
| Voting Delay | 1 block (~13s) |
| Voting Period | 100 blocks (~22 min) |
| Quorum | 10% of NFT supply |
| Late Quorum Extension | 50 blocks (~11 min) |
| Timelock Delay | 3600s (1 hour) |
| Proposal Threshold | 0 (any NFT holder) |

---

## What's Next

### Phase 2A: Foundation Contracts ✅

```
1. ✅ Deploy OlympiaTreasury (AccessControlDefaultAdminRules, OZ v5.6)
   → 0xd6165F3aF4281037bce810621F62B43077Fb0e37 (Mordor + ETC mainnet)
   → All 3 client olympia branches updated with treasury address

2. ✅ Build + deploy SanctionsOracle + OlympiaMemberNFT
   → olympia-governance-contracts repo (87 tests)
```

### Phase 2B: Governor Pipeline ✅

```
3. ✅ Build + deploy OlympiaGovernor (GovernorVotes + GovernorVotesQuorumFraction)
4. ✅ Build + deploy OlympiaExecutor (treasury + timelock + sanctionsOracle)
5. ✅ Build + deploy TimelockController
6. ✅ Build + deploy ECFPRegistry (ECIP-1114)
```

### Phase 2C: Mordor Deployment ✅

```
7.  ✅ All 7 contracts deployed on Mordor via CREATE2
8.  ✅ Timelock roles configured (Governor = PROPOSER + CANCELLER, Executor = EXECUTOR)
9.  ✅ WITHDRAWER_ROLE granted on Treasury to OlympiaExecutor
10. ✅ All roles verified on-chain
```

### Phase 2D: Governance Lifecycle Testing (in progress)

```
11. ✅ ECFP-001 submitted, voted (2 For votes), queued, executed
12. Remaining: ECFP-002, 003, 004 (deferred for additional testing)
13. Sanctions layer validation (propose/cancel/execute gates)
```

### Phase 3: Mordor Activation + Mainnet

```
14. Mordor hard fork activation (block TBD)
15. Monitor governance pipeline through fork boundary
16. ETC mainnet activation (block TBD)
17. Deploy Stage 2 contracts on ETC mainnet
```

---

## Design Principles

1. **Accumulate first, govern later.** The Treasury starts as a receive-only vault. Governance matures separately. No withdrawals until the governance pipeline is battle-tested.

2. **Contract before consensus.** Experiment with parameters at the contract layer (adjustable via OIP) before embedding them at the consensus layer (requires a hard fork). This applies to fee splits (ECIP-1116) and miner distribution (ECIP-1122).

3. **Layered defense.** Sanctions checking at three points (propose, cancel, execute) ensures no single failure mode bypasses screening. The execution-time check is the security invariant.

4. **Forward-looking modularity.** The `IOlympiaVotingModule` interface is spec'd as a design artifact for future governance-gated voting module swaps. Demo v0.1 uses standard OZ `GovernorVotes` directly. The interface will be wired in when module swapping is needed.

5. **Staged governance lifecycle.** The Treasury admin progresses from deployer EOA → CoreDAO governance → admin renounced. `AccessControlDefaultAdminRules` provides 2-step transfers with mandatory delay at each transition.

6. **Empirical before dogmatic.** Phase 4 (L-curve experimentation) exists because we don't know the right basefee split or miner distribution curve. The answer comes from data, not from committee preference. Phase 5 only proceeds after Phase 4 validates the model.

7. **Miner-first economics.** Olympia does not compete with miners. Block rewards and priority fees are untouched. The basefee redirect adds ~1 gwei/tx at current volumes — negligible relative to the 2.048 ETC block reward. The basefee and tips are additive, not competitive.

---

## Implementation Specs

Full implementation specifications for all 11 ECIPs are in [`specs/`](specs/):

| ECIP | Spec | Stage |
|------|------|-------|
| 1111 | [ecip-1111-implementation.md](specs/ecip-1111-implementation.md) | 1 — Hard Fork |
| 1112 | [ecip-1112-implementation.md](specs/ecip-1112-implementation.md) | 1 — Hard Fork |
| 1113 | [ecip-1113-implementation.md](specs/ecip-1113-implementation.md) | 2 — CoreDAO |
| 1114 | [ecip-1114-implementation.md](specs/ecip-1114-implementation.md) | 2 — CoreDAO |
| 1115 | [ecip-1115-implementation.md](specs/ecip-1115-implementation.md) | 4 — Miner Experimentation |
| 1116 | [ecip-1116-implementation.md](specs/ecip-1116-implementation.md) | 5 — Protocol Hardcode |
| 1117 | [ecip-1117-implementation.md](specs/ecip-1117-implementation.md) | 3 — Futarchy |
| 1118 | [ecip-1118-implementation.md](specs/ecip-1118-implementation.md) | 3 — Futarchy |
| 1119 | [ecip-1119-implementation.md](specs/ecip-1119-implementation.md) | 2 — CoreDAO |
| 1121 | [ecip-1121-implementation.md](specs/ecip-1121-implementation.md) | 1 — Hard Fork |
| 1122 | [ecip-1122-implementation.md](specs/ecip-1122-implementation.md) | 5 — Protocol Hardcode |

Each spec includes contract interfaces, deployment details, gap analysis against the original draft, amendments with rationale, test requirements, and verification commands.

---

## Repos

### Contract Repos

| Repo | Purpose | Stage | Status |
|------|---------|-------|--------|
| [olympia-treasury-contract](https://github.com/olympiadao/olympia-treasury-contract) | Treasury vault — Solidity + Foundry tests (ECIP-1112) | 1 | Deployed (Mordor + ETC) |
| [olympia-governance-contracts](https://github.com/olympiadao/olympia-governance-contracts) | Governor, Executor, Timelock, ECFPRegistry, SanctionsOracle, MemberNFT (ECIP-1113, 1114, 1119) | 2 | Deployed on Mordor (87 tests) |
| [degov](https://github.com/olympiadao/degov) | Original Governor prototype — superseded by olympia-governance-contracts | 2 | Archived |
| [olympia-futarchy](https://github.com/olympiadao/olympia-futarchy) | Futarchy research + prediction market contracts (ECIP-1117, 1118) | 3 | Research complete |

### Client Repos

| Repo | Purpose | Status |
|------|---------|--------|
| [core-geth](https://github.com/ethereumclassic/core-geth) | Go ETC client (`olympia` branch) | Olympia implemented |
| [besu](https://github.com/ethereumclassic/besu) | Java ETC client (`olympia` branch) | Olympia implemented |
| [fukuii](https://github.com/ethereumclassic/fukuii) | Scala ETC client (`olympia` branch) | Olympia implemented |

### Web & Brand Repos

| Repo | Purpose | Status |
|------|---------|--------|
| [olympia-brand](https://github.com/olympiadao/olympia-brand) | Logo SVGs, favicons, OG images, design tokens | Complete |
| [olympiadao-org](https://github.com/olympiadao/olympiadao-org) | olympiadao.org — Next.js 16 landing page | Complete |
| [olympiatreasury-org](https://github.com/olympiadao/olympiatreasury-org) | olympiatreasury.org — Next.js 16 landing page | Complete |
| [olympia-app](https://github.com/olympiadao/olympia-app) | Governance dApp — proposals, voting, treasury (Next.js 16) | Active development |

### External

| Repo | Org | Purpose |
|------|-----|---------|
| [olympia-framework](https://github.com/olympiadao/olympia-framework) | olympiadao | This repo — specs and framework document |
| [ECIPs](https://github.com/ethereumclassic/ECIPs) | ethereumclassic | Published ECIP specifications |

---

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)
