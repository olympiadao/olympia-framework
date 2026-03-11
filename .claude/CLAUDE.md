# Claude Code Instructions: olympia-framework

> **For Claude Code only.** GitHub Copilot uses `.github/copilot-instructions.md` instead.

## Project Context

**Olympia Framework** — the canonical planning and specification repo for Ethereum Classic's Olympia upgrade. Houses the framework document (README.md) and all 11 ECIP implementation specs.

**Repository:** https://github.com/olympiadao/olympia-framework
**Status:** Phase 0 complete (specs drafted). Phase 1 (Stage 2 contracts) next.

This is a **documentation and planning repo**, not a code repo. No build system, no tests, no dependencies. Solidity interfaces appear inline in spec files for reference.

---

## Olympia Overview

11 ECIPs across 5 stages:

| Stage | ECIPs | Type | Status |
|-------|-------|------|--------|
| 1 — Hard Fork | 1111, 1112, 1121 | Consensus | Implemented (3 clients) |
| 2 — CoreDAO | 1113, 1114, 1119 | Contract | Governor rewrite needed |
| 3 — Futarchy | 1117, 1118 | Contract | Prototype deployed |
| 4 — Miner Experimentation | 1115 | Contract | Phase 4 |
| 5 — Protocol Hardcode | 1116, 1122 | Consensus | Deferred |

**Clients:** core-geth (Go), besu-etc (Java), fukuii (Scala) — all with Olympia branches.

**Mordor activation:** block 15,800,850 (~March 28, 2026)
**ETC mainnet target:** ~24,751,337 (~mid-June 2026)

---

## Repo Structure

```
olympia-framework/
├── README.md              # OLYMPIA-FRAMEWORK.md — the framework document
├── LICENSE                # CC0-1.0
├── .gitignore
├── .claude/CLAUDE.md      # This file
├── .github/
│   ├── AGENTS.md
│   └── copilot-instructions.md
└── specs/
    ├── ecip-1111-implementation.md
    ├── ecip-1112-implementation.md
    ├── ecip-1113-implementation.md
    ├── ecip-1114-implementation.md
    ├── ecip-1115-implementation.md
    ├── ecip-1116-implementation.md
    ├── ecip-1117-implementation.md
    ├── ecip-1118-implementation.md
    ├── ecip-1119-implementation.md
    ├── ecip-1121-implementation.md
    └── ecip-1122-implementation.md
```

---

## Related Repos

| Repo | Org | Purpose |
|------|-----|---------|
| olympia-treasury-contract | olympiadao | Treasury Solidity + Foundry tests |
| degov | olympiadao | Governor contracts (OZ-based) |
| core-geth | chris-mercer | Go ETC client (etc + olympia branches) |
| fukuii-client | *(private)* | Scala ETC client (alpha + olympia branches) |
| olympia-governance-contracts | olympiadao | SanctionsOracle, OlympiaMemberNFT, interfaces (OZ v5.6) |
| olympia-futarchy | olympiadao | Futarchy research + streaming disbursements (ECIP-1117, 1118) |
| ECIPs | ethereumclassic | Published ECIP specifications |

---

## Key Addresses

See README.md Deployment Addresses table.

| Contract | Address |
|----------|---------|
| OlympiaTreasury | `0xd6165F3aF4281037bce810621F62B43077Fb0e37` (Mordor + ETC mainnet) |
| Deployer | `0x3b0952fB8eAAC74E56E176102eBA70BAB1C81537` |

---

## Boundaries

### Always Do

- Keep cross-references consistent when editing any spec (ECIP numbers, addresses, stage numbers)
- Update the ECIP Index table in README.md when spec status changes
- Verify ECIP frontmatter (ecip number, requires, supersedes) matches content
- Use CC0-1.0 license for all spec content

### Ask First

- Changing ECIP numbers or stage assignments
- Adding new ECIPs to the framework
- Modifying contract interfaces in specs (affects multiple repos)
- Changing activation block numbers

### Never Do

- Modify client code from this repo (client changes happen in core-geth/besu/fukuii repos)
- Modify contract source from this repo (contract changes happen in treasury/degov repos)
- Commit secrets, private keys, or internal planning docs
- Change the `supersedes` relationship between ECIPs without discussion

---

## Spec Conventions

- Frontmatter uses YAML with fields: ecip, title, status, type, category, requires, supersedes, author, created, updated, license
- Authors: `Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)`
- License: `CC0-1.0`
- Solidity interfaces use `pragma solidity ^0.8.20` and OpenZeppelin v5.6
- All specs end with a Copyright section linking to CC0
