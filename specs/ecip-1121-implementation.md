---
ecip: 1121
title: Olympia Execution Client Specification Alignment — Implementation Spec
status: Implementation
type: Meta
author: Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)
created: 2025-12-14
updated: 2026-03-10
license: CC0-1.0
---

# ECIP-1121: Olympia Execution Client Specification Alignment — Implementation Spec

## Overview

ECIP-1121 defines the execution-layer specification scope for the Olympia hard fork — enumerating Ethereum EIPs that are compatible with Ethereum Classic's Proof-of-Work consensus and suitable for coordinated inclusion. This is independent of fee-market governance, treasury design, or governance model selection.

These EIPs are activated alongside ECIP-1111 (EIP-1559 + EIP-3198) in the same Olympia fork.

## Activation

| Network | Block Height | Estimated Date | Status |
|---------|-------------|----------------|--------|
| Mordor Testnet | 15,800,850 | ~March 28, 2026 | Pending |
| ETC Mainnet | ~24,751,337 | ~mid-June 2026 | Pending |

## Included EIPs

### Gas Accounting and State Access

| EIP | Description | Opcode/Feature |
|-----|-------------|----------------|
| EIP-7702 | Set EOA account code | EOA code delegation |
| EIP-7623 | Increase calldata cost | Gas repricing |
| EIP-7825 | Transaction gas limit cap | Safety limit |
| EIP-7883 | MODEXP gas cost increase | Precompile repricing |
| EIP-7935 | Default gas limit configuration | Client config |

### EVM Safety and Forward Compatibility

| EIP | Description | Opcode/Feature |
|-----|-------------|----------------|
| EIP-7934 | RLP execution block size limit | Block size safety |
| EIP-6780 | SELFDESTRUCT restriction | Opcode modification |
| ~~EIP-7642~~ | ~~History expiry and simplified receipts~~ | **REMOVING — see amendment below** |
| EIP-7910 | eth_config JSON-RPC method | RPC interface |

### Cryptographic and Precompile Enhancements

| EIP | Description | Opcode/Feature |
|-----|-------------|----------------|
| EIP-2537 | BLS12-381 curve precompile | New precompile |
| EIP-7951 | secp256r1 curve precompile | New precompile |

### Execution Context Optimizations

| EIP | Description | Opcode/Feature |
|-----|-------------|----------------|
| EIP-5656 | MCOPY opcode | Opcode `0x5E` — memory copy |
| EIP-2935 | Historical block hashes in state | Block hash access |
| EIP-1153 | Transient storage opcodes | `TSTORE` / `TLOAD` |

## Client Implementation Status

| Client | Branch | Language | Status | Tests |
|--------|--------|----------|--------|-------|
| core-geth v1.12.21 | `olympia` (`b1c759dcc`) | Go 1.24 | ✅ All EIPs | ALL PASS |
| besu-etc v26.3 | `olympia` (`52dc37b5bf`) | Java 21 | ✅ All EIPs | ALL PASS |
| fukuii v0.1.240 | `olympia` (`126c1fd5c`) | Scala 3.3 | ✅ All EIPs | ALL PASS (2,308 tests) |

## Deferred Specifications

Deferred because they depend on blob-based data availability (not implemented on ETC):

| EIP | Reason |
|-----|--------|
| EIP-4844 | Shard blob transactions |
| EIP-7516 | BLOBBASEFEE opcode |
| EIP-7691 | Blob throughput parameters |
| EIP-7840 | Blob schedule in execution-layer config |
| EIP-7892 | Blob parameter-only hardforks |
| EIP-7918 | Blob base fee bounded by execution cost |

## Explicit Exclusions

Excluded because they depend on Proof-of-Stake / Beacon Chain:

| EIP | Reason |
|-----|--------|
| EIP-4788 | Requires Beacon Chain state |
| EIP-7002 | Execution-layer triggered validator exits |
| EIP-7685 | Execution-layer request framework for PoS |
| EIP-6110 | Validator deposit handling |
| EIP-7917 | Deterministic proposer lookahead |

## Amendment: EIP-7642 Removal (ACTION REQUIRED)

**EIP-7642 (History expiry and simplified receipts) must be removed from ECIP-1121 before Mordor activation.** This EIP introduces incompatibilities with ETC's state management assumptions around receipt availability and historical data access. Removal must be coordinated across all 3 client implementations before block 15,800,850.

**Status:** Pending removal. All 3 client branches must remove EIP-7642 handling and update fork configuration before Mordor activation (~March 28, 2026).

## Relationship to ECIP-1111

ECIP-1121 EIPs are independent of ECIP-1111's fee-market changes. They share the same activation block but have no functional dependency on basefee, Treasury, or governance.

**ECIP-1111 provides:** EIP-1559 (dynamic basefee, Type-2 transactions) + EIP-3198 (BASEFEE opcode `0x48`).

**ECIP-1121 provides:** Everything else in the Olympia EVM upgrade.

## Verification

```bash
# Verify MCOPY (EIP-5656) — opcode 0x5E
cast call --create "0x5E..." --rpc-url $MORDOR_RPC_URL

# Verify TSTORE/TLOAD (EIP-1153) — opcodes 0x5C/0x5D
cast call --create "0x5C5D..." --rpc-url $MORDOR_RPC_URL

# Verify BLS12-381 precompile (EIP-2537)
cast call 0x0000000000000000000000000000000000000010 <input> --rpc-url $MORDOR_RPC_URL

# Cross-client verification: all 3 clients must produce identical results
```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)
