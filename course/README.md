# From Zero to Pro: Cryptocurrency & Smart Contract Development in Solidity (EVM)

## Course Syllabus

A comprehensive, project-based course that takes developers from **zero blockchain experience** to **professional-level smart contract engineering**, ready for senior roles in Web3.

**Prerequisites:** Basic programming in any language. No blockchain or Solidity experience required.

**Primary Stack:** Solidity · Foundry (forge, cast, anvil) · ethers.js · React

---

## Parts & Modules

### Part 1 · Blockchain & Cryptography Foundations

| # | Module | Learning Objectives |
|---|--------|---------------------|
| 1 | [Blockchain Fundamentals](./01-blockchain-fundamentals.md) | Understand distributed ledgers, consensus mechanisms, blocks, and the difference between Bitcoin and Ethereum |
| 2 | [Cryptographic Building Blocks](./02-cryptography.md) | Master hashing (keccak256), digital signatures (ECDSA), and Merkle trees with hands-on demos |
| 3 | [Wallets, Keys & Transactions](./03-wallets-keys-transactions.md) | Understand EOAs, nonces, gas mechanics, and how transactions flow through the mempool |
| 4 | [Development Environment Setup](./04-dev-environment-setup.md) | Install and configure Node.js, Foundry, VS Code, Git, and connect to testnets via Alchemy/Infura |

### Part 2 · Solidity Programming

| # | Module | Learning Objectives |
|---|--------|---------------------|
| 5 | [Solidity Basics](./05-solidity-basics.md) | Data types, mappings, structs, arrays, and your first smart contract |
| 6 | [Functions, Modifiers, Events & Errors](./06-functions-modifiers-events-errors.md) | Visibility, modifiers, event emission, custom errors, and error handling patterns |
| 7 | [Inheritance, Interfaces & Abstract Contracts](./07-inheritance-interfaces-abstract.md) | Build a simple token using OOP patterns in Solidity |
| 8 | [Libraries & Open-Source Security](./08-libraries-openzeppelin.md) | Use OpenZeppelin contracts, `using for`, and understand library linking |

### Part 3 · Testing, Tooling & Deployment

| # | Module | Learning Objectives |
|---|--------|---------------------|
| 9 | [Testing with Foundry](./09-testing-foundry.md) | Unit tests, fuzz tests, invariant tests, cheatcodes, and fork testing |
| 10 | [Deploying & Verifying Contracts](./10-deploying-verifying.md) | Deploy to Sepolia/Goerli, automate with scripts, verify on Etherscan |
| 11 | [Upgradeable Contracts](./11-upgradeable-contracts.md) | Proxy patterns (UUPS, Transparent), storage collisions, OpenZeppelin upgrades |
| 12 | [Gas Optimisation](./12-gas-optimisation.md) | Storage packing, memory vs calldata, assembly/Yul, gas golfing techniques |

### Part 4 · Token Standards & DeFi Primitives

| # | Module | Learning Objectives |
|---|--------|---------------------|
| 13 | [ERC-20 Tokens](./13-erc20-tokens.md) | Implement a full ERC-20 with ICO crowdsale and vesting schedules |
| 14 | [ERC-721 & ERC-1155 NFTs](./14-erc721-erc1155-nfts.md) | NFT standards, metadata, minting, marketplace integration |
| 15 | [DeFi Foundations](./15-defi-foundations.md) | AMMs, constant product formula, liquidity pools, and yield farming |
| 16 | [Building a Simple DEX](./16-building-simple-dex.md) | Implement swap, add/remove liquidity, fee accrual from scratch |
| 17 | [Lending & Borrowing](./17-lending-borrowing.md) | Collateralised lending, interest rate models, Compound/Aave simplified |

### Part 5 · L2s, Sidechains & Advanced Patterns

| # | Module | Learning Objectives |
|---|--------|---------------------|
| 18 | [Layer-2 Scaling](./18-layer2-scaling.md) | Optimistic rollups, ZK rollups, bridges; deploy to Arbitrum and Optimism |
| 19 | [Sidechains & Alt-EVMs](./19-sidechains-alt-evms.md) | Polygon PoS, BNB Chain, Avalanche C-Chain; multi-chain deployment strategies |

### Part 6 · Security & Auditing

| # | Module | Learning Objectives |
|---|--------|---------------------|
| 20 | [Common Vulnerabilities](./20-common-vulnerabilities.md) | Re-entrancy, front-running, oracle manipulation, access control exploits |
| 21 | [Formal Verification & Static Analysis](./21-formal-verification-static-analysis.md) | Slither, Aderyn, Certora; writing provably correct contracts |
| 22 | [Audit Readiness](./22-audit-readiness.md) | Prepare code for audits, write clear documentation, respond to findings |

### Part 7 · Capstone & Professional Growth

| # | Module | Learning Objectives |
|---|--------|---------------------|
| 23 | [Capstone: Full-Stack DeFi Protocol](./23-capstone-defi-protocol.md) | Build a yield-bearing ERC-4626 vault with front-end, tests, and testnet deployment |

### Bonus

| # | Module | Learning Objectives |
|---|--------|---------------------|
| 24 | [Blockchain in Rust / Solana](./24-bonus-rust-solana.md) | Rust for blockchains, Solana's account model, PDAs, Anchor framework; compare with EVM |

### Appendix

| Document | Description |
|----------|-------------|
| [EVM Job Landscape & Skills Roadmap](./appendix-evm-job-landscape.md) | Career paths, salary ranges, skills checklist, and interview prep for Web3 roles |

---

## Capstone Deliverables

By the end of this course you will have built and shipped:

1. **A yield-bearing DeFi vault** (ERC-4626) with deposit/withdrawal logic, share minting/burning, fee accrual, and emergency pause
2. **Comprehensive Foundry test suite** — unit, integration, fuzz, and invariant tests
3. **A React front-end** with wallet connection (MetaMask / WalletConnect) and real-time vault interaction
4. **Testnet deployment** with verified source code on Etherscan
5. **A written case study** documenting architecture, security considerations, and deployment process

---

## How to Use This Course

1. **Sequential order is recommended** for beginners — each module builds on the previous one
2. **Type every line of code** — don't copy-paste; muscle memory matters
3. **Complete every checkpoint** before moving on
4. **Join a community** — the course references Discord/Telegram groups for Q&A
5. **Keep a learning journal** — write down "aha" moments and blockers

---

## Conventions

- Solidity version: `^0.8.24`
- Foundry profile: `default`
- All code is commented and tested unless stated otherwise
- Gas values shown in examples are illustrative; real costs vary with network conditions

---

*Happy building! 🛠️*
