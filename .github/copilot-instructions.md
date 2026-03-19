# GitHub Copilot Instructions: olympia-framework

> **Important:** GitHub Copilot only reads this file and project code.

## Project

> **Demo v0.1** — Not Olympia ECIP spec compliant. Deployed for fast iterative development to build project scaffolding.

Olympia Framework — specification and planning repo for Ethereum Classic's Olympia upgrade. 11 ECIPs across 5 stages: basefee redirect, treasury governance, futarchy, miner distribution, EVM compatibility.

This is a documentation repo. No build system, no tests, no runtime dependencies. ECIP specs are published at https://ecips.ethereumclassic.org.

## Tech Stack

| Technology | Version | Purpose |
|------------|---------|---------|
| Solidity | ^0.8.20 | Contract interfaces (inline in specs) |
| OpenZeppelin | v5.6 | Referenced contract library |
| Foundry | Latest | Referenced test framework |
| Markdown | — | Spec authoring |

## Key Rules

1. Keep ECIP cross-references consistent across all spec files
2. Solidity uses `pragma solidity ^0.8.20` and OpenZeppelin v5.6
3. All content is CC0-1.0 licensed
4. ECIP-1122 supersedes ECIP-1120
5. Contract addresses are Mordor testnet (chainId 63) unless noted

## Structure

```
README.md                 # Framework overview document
```

## Don't

- Modify client or contract source (separate repos)
- Commit secrets or keys
- Change ECIP numbers without discussion
- Break cross-references between specs

## Response Style

- Technical precision with correct EIP/ECIP numbers
- Reference specific spec files
- No pleasantries
