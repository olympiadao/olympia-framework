# Claude Code Instructions: olympia-framework

> **For Claude Code only.** GitHub Copilot uses `.github/copilot-instructions.md` instead.

## Project Context

**Olympia Framework** — the canonical planning and specification repo for Ethereum Classic's Olympia upgrade. Houses the framework document (README.md). ECIP specs are published at https://ecips.ethereumclassic.org.

**Repository:** https://github.com/olympiadao/olympia-framework
**Status:** Phase 2D in progress (governance lifecycle testing). Stage 2 contracts deployed.

This is a **documentation and planning repo**, not a code repo. No build system, no tests, no dependencies.

---

## Olympia Overview

11 ECIPs across 5 stages:

| Stage | ECIPs | Type | Status |
|-------|-------|------|--------|
| 1 — Hard Fork | 1111, 1112, 1121 | Consensus | Implemented (3 clients) |
| 2 — CoreDAO | 1113, 1114, 1119 | Contract | Deployed (7 contracts, 106 tests) |
| 3 — Futarchy | 1117, 1118 | Contract | Prototype deployed |
| 4 — Miner Experimentation | 1115 | Contract | Phase 4 |
| 5 — Protocol Hardcode | 1116, 1122 | Consensus | Deferred |

**Clients:** core-geth (Go), besu-etc (Java), fukuii (Scala) — all with Olympia branches.

**Next step:** Mordor Testnet Deployment (block TBD)
**ETC mainnet target:** TBD

---

## Repo Structure

```
olympia-framework/
├── README.md              # OLYMPIA-FRAMEWORK.md — the framework document
├── LICENSE                # CC0-1.0
├── .gitignore
├── .claude/CLAUDE.md      # This file
└── .github/
    ├── AGENTS.md
    └── copilot-instructions.md
```

---

## Related Repos

| Repo | Org | Purpose |
|------|-----|---------|
| olympia-treasury-contract | olympiadao | Treasury vault (ECIP-1112) — deployed Mordor + ETC |
| olympia-governance-contracts | olympiadao | Governor, Executor, Timelock, ECFPRegistry, SanctionsOracle, MemberNFT (ECIP-1113, 1114, 1119) — 106 tests |
| olympia-futarchy | olympiadao | Futarchy research + prediction markets (ECIP-1117, 1118) |
| olympia-brand | olympiadao | Logo SVGs, favicons, OG images, design tokens |
| olympiadao-org | olympiadao | olympiadao.org landing page |
| olympiatreasury-org | olympiadao | olympiatreasury.org landing page |
| olympia-app | olympiadao | Governance dApp (placeholder) |
| core-geth | ethereumclassic | Go ETC client (etc + olympia branches) |
| besu | ethereumclassic | Java ETC client (etc + olympia branches) |
| fukuii | ethereumclassic | Scala ETC client (alpha + olympia branches) |
| ECIPs | ethereumclassic | Published ECIP specifications |

---

## Key Addresses

See README.md Deployment Addresses table.

| Contract | Address |
|----------|---------|
| OlympiaTreasury | `0x035b2e3c189B772e52F4C3DA6c45c84A3bB871bf` (Mordor + ETC mainnet) |
| Deployer | `0x7C3311F29e318617fed0833E68D6522948AaE995` |

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
- Modifying contract interfaces (affects multiple repos)
- Changing activation block numbers

### Never Do

- Modify client code from this repo (client changes happen in core-geth/besu/fukuii repos)
- Modify contract source from this repo (contract changes happen in treasury/governance-contracts repos)
- Commit secrets, private keys, or internal planning docs
- Change the `supersedes` relationship between ECIPs without discussion

---

## ECIP Specifications

All 11 Olympia ECIPs are published at https://ecips.ethereumclassic.org/ECIPs/ecip-{number}.

- Authors: `Cody Burns (@realcodywburns), Chris Mercer (@chris-mercer)`
- License: `CC0-1.0`
- Governance contracts use OpenZeppelin v5.1.0 (Shanghai EVM). Treasury is pure Solidity (no OZ dependency).
