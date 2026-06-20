# From Zero to Pro: Cryptocurrency & Smart Contract Development in Solidity (EVM)

> A comprehensive, project-based course that takes developers from **zero blockchain experience** to **professional-level smart contract engineering**.

[![Solidity](https://img.shields.io/badge/Solidity-^0.8.24-363636?logo=solidity)](https://soliditylang.org/)
[![Foundry](https://img.shields.io/badge/Built%20with-Foundry-f0b238?logo=data:image/svg+xml;base64,)](https://getfoundry.sh/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)

---

## Who Is This For?

- Developers who know basic programming (any language) but have **zero blockchain experience**
- Engineers looking to transition into Web3 / smart contract roles
- Anyone preparing for senior smart contract engineering positions

## What You'll Build

By the end of this course you will have shipped:

- **A yield-bearing DeFi vault** (ERC-4626) with deposit/withdrawal, share minting/burning, fee logic, and emergency pause
- **Comprehensive Foundry test suite** — unit, integration, fuzz, and invariant tests
- **A React front-end** with wallet connection (MetaMask / WalletConnect)
- **Testnet deployment** with verified source code on Etherscan
- **A written case study** documenting architecture and security

## Tech Stack

| Tool | Purpose |
|------|---------|
| **Solidity** `^0.8.24` | Smart contract language |
| **Foundry** (forge, cast, anvil) | Build, test, deploy framework |
| **OpenZeppelin Contracts** | Battle-tested contract library |
| **ethers.js / wagmi** | Front-end blockchain interaction |
| **React + TypeScript** | User interface |

---

## Course Structure

### Part 1 — Blockchain & Cryptography Foundations

| # | Module | Topics |
|---|--------|--------|
| 1 | [Blockchain Fundamentals](./course/01-blockchain-fundamentals.md) | Distributed ledgers, consensus (PoW vs PoS), blocks, smart contracts |
| 2 | [Cryptography](./course/02-cryptography.md) | keccak256, ECDSA signatures, Merkle trees |
| 3 | [Wallets, Keys & Transactions](./course/03-wallets-keys-transactions.md) | EOAs, nonces, gas mechanics, mempool |
| 4 | [Dev Environment Setup](./course/04-dev-environment-setup.md) | Node.js, Foundry, VS Code, Alchemy/Infura |

### Part 2 — Solidity Programming

| # | Module | Topics |
|---|--------|--------|
| 5 | [Solidity Basics](./course/05-solidity-basics.md) | Data types, mappings, structs, storage vs memory |
| 6 | [Functions, Modifiers, Events & Errors](./course/06-functions-modifiers-events-errors.md) | Visibility, custom errors, event emission |
| 7 | [Inheritance, Interfaces & Abstract Contracts](./course/07-inheritance-interfaces-abstract.md) | C3 linearisation, OOP in Solidity |
| 8 | [Libraries & OpenZeppelin](./course/08-libraries-openzeppelin.md) | `using for`, installing OZ, access control |

### Part 3 — Testing, Tooling & Deployment

| # | Module | Topics |
|---|--------|--------|
| 9 | [Testing with Foundry](./course/09-testing-foundry.md) | Unit tests, fuzz tests, invariant tests, cheatcodes |
| 10 | [Deploying & Verifying](./course/10-deploying-verifying.md) | Testnet deployment, Etherscan verification, scripts |
| 11 | [Upgradeable Contracts](./course/11-upgradeable-contracts.md) | UUPS proxies, storage collisions, OZ upgrades |
| 12 | [Gas Optimisation](./course/12-gas-optimisation.md) | Storage packing, memory vs calldata, assembly |

### Part 4 — Token Standards & DeFi Primitives

| # | Module | Topics |
|---|--------|--------|
| 13 | [ERC-20 Tokens](./course/13-erc20-tokens.md) | Token implementation, ICO crowdsale, vesting |
| 14 | [ERC-721 & ERC-1155 NFTs](./course/14-erc721-erc1155-nfts.md) | NFT standards, marketplace, royalties |
| 15 | [DeFi Foundations](./course/15-defi-foundations.md) | AMMs, constant product formula, liquidity pools |
| 16 | [Building a Simple DEX](./course/16-building-simple-dex.md) | Factory, pair, router — swap and liquidity |
| 17 | [Lending & Borrowing](./course/17-lending-borrowing.md) | Collateralised lending, interest rates, liquidation |

### Part 5 — L2s, Sidechains & Advanced Patterns

| # | Module | Topics |
|---|--------|--------|
| 18 | [Layer-2 Scaling](./course/18-layer2-scaling.md) | Optimistic rollups, ZK rollups, Arbitrum, Optimism |
| 19 | [Sidechains & Alt-EVMs](./course/19-sidechains-alt-evms.md) | Polygon, BNB Chain, Avalanche, multi-chain deploy |

### Part 6 — Security & Auditing

| # | Module | Topics |
|---|--------|--------|
| 20 | [Common Vulnerabilities](./course/20-common-vulnerabilities.md) | Re-entrancy, front-running, oracle manipulation |
| 21 | [Formal Verification & Static Analysis](./course/21-formal-verification-static-analysis.md) | Slither, Aderyn, Certora Prover |
| 22 | [Audit Readiness](./course/22-audit-readiness.md) | Pre-audit checklist, NatSpec, threat models |

### Part 7 — Capstone & Professional Growth

| # | Module | Topics |
|---|--------|--------|
| 23 | [Capstone: Full-Stack DeFi Protocol](./course/23-capstone-defi-protocol.md) | ERC-4626 vault + tests + React front-end + deployment |

### Bonus

| # | Module | Topics |
|---|--------|--------|
| 24 | [Blockchain in Rust / Solana](./course/24-bonus-rust-solana.md) | Rust basics, Solana accounts, PDAs, Anchor |

### Appendix

| Document | Description |
|----------|-------------|
| [EVM Job Landscape & Skills Roadmap](./course/appendix-evm-job-landscape.md) | Career paths, salary ranges, portfolio guide, interview prep |

---

## Getting Started

### Prerequisites

- Basic programming experience (any language)
- A computer with macOS, Linux, or Windows (WSL)

### Quick Start

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/blockchain-course.git
cd blockchain-course

# Start with Module 1
cat course/01-blockchain-fundamentals.md

# Or open in VS Code
code course/
```

### Setting Up Your Dev Environment (Module 4)

```bash
# Install Foundry
curl -L https://foundry.paradigm.xyz | bash
foundryup

# Create your first project
forge init hello_foundry
cd hello_foundry

# Build and test
forge build
forge test
```

---

## How to Use This Course

1. **Follow modules sequentially** — each builds on the previous one
2. **Type every line of code** — don't copy-paste; muscle memory matters
3. **Complete every checkpoint** before moving on
4. **Fork this repo** and add your own solutions alongside the examples
5. **Keep a learning journal** — write down "aha" moments and blockers

## Each Module Includes

- **Learning objectives** — what you'll be able to do
- **Concept explanations** — with real-world analogies
- **Mermaid diagrams** — architecture, transaction flows, inheritance
- **Complete, runnable code** — Solidity, Foundry tests, scripts
- **Step-by-step build instructions** — follow along
- **Checkpoint tasks** — validate your understanding
- **Common pitfalls** — avoid beginner mistakes
- **Further reading** — go deeper on each topic

---

## Contributing

Found a typo, outdated code, or want to add an exercise? PRs are welcome!

1. Fork this repository
2. Create a feature branch (`git checkout -b fix/module-5-typo`)
3. Commit your changes (`git commit -m 'Fix typo in module 5'`)
4. Push to the branch (`git push origin fix/module-5-typo`)
5. Open a Pull Request

---

## License

This course is released under the [MIT License](./LICENSE).

---

## Acknowledgments

- [Foundry](https://getfoundry.sh/) — Fastest Solidity development framework
- [OpenZeppelin](https://www.openzeppelin.com/) — Industry-standard smart contract library
- [Ethereum Foundation](https://ethereum.org/) — Documentation and educational resources
- [Cyfrin](https://www.cyfrin.io/) — Security education and tooling (Aderyn)
- [Patrick Collins](https://github.com/PatrickAlphaC) — Inspiration for blockchain education

---

**Ready to start?** → [Module 1: Blockchain Fundamentals](./course/01-blockchain-fundamentals.md)
