# solidity-evm-skill

> The most comprehensive Solidity & EVM smart contract skill for AI agents.

Install it once. Your AI agent becomes a senior Solidity engineer.

```bash
npx skills add LollyTim/solidity-evm-skill
```

---

## What This Skill Does

This skill gives AI agents (Claude, Cursor, and any skills-compatible agent) deep, production-grade knowledge for building smart contracts on Ethereum and EVM-compatible chains.

It covers everything from writing your first token to deploying a gas-optimized, audited, upgradeable DeFi protocol — with real code patterns, not generic advice.

---

## Coverage

### 🔒 Security (`references/security.md`)
- Top vulnerability classes with 2025 loss data ($2.3B+ stolen H1 2025)
- Reentrancy (classic, cross-function, read-only variants)
- Access control failures — the #1 exploit category
- Oracle manipulation & flash loan attack patterns
- Front-running, commit-reveal, EIP-712 signature replay defense
- Proxy storage collision bugs
- Security tooling: Slither, Mythril, Echidna, Tenderly, Certora
- Full pre-deployment audit checklist

### ⛽ Gas Optimization (`references/gas-optimization.md`)
- EVM opcode cost table (SSTORE, SLOAD, CALL, MLOAD, etc.)
- Storage slot packing — pack structs into single slots
- Caching storage reads in memory (eliminate redundant SLOADs)
- `constant` and `immutable` — zero-cost reads
- `calldata` vs `memory` — eliminate unnecessary copies
- Custom errors vs require strings (~50% gas savings)
- Bitmap patterns for boolean flags (256 flags per slot)
- Unchecked arithmetic, loop optimization, mapping vs array
- Compiler optimizer settings (`runs`, `viaIR`)
- Clone factories (EIP-1167) — 350k gas per deploy saved
- Measurement: Foundry gas reports and snapshots

### 🏗️ Design Patterns (`references/patterns.md`)
- **Proxy & Upgradeable**: UUPS, Transparent, Beacon, Diamond (EIP-2535)
- **Factory Pattern**: standard + clone factory with CREATE2
- **Access Control**: Ownable2Step, AccessControl, Pausable, circuit breakers
- **Pull-over-Push**: safe ETH/token distribution patterns
- **State Machine**: enum-based phase management
- **Staking**: complete reward accumulator implementation
- **Vesting**: cliff + linear vesting with pull model
- **ERC-4626 Vault**: tokenized yield-bearing vaults
- **Governance**: Governor + TimelockController

### 🛠️ Tooling (`references/tooling.md`)
- **Foundry** — full setup, `foundry.toml`, all forge/cast/anvil/chisel commands
- **Hardhat** — TypeScript config, Ignition deployments, test patterns
- Unit tests, fuzz tests, invariant tests with handler contracts
- Foundry cheatcodes: `vm.prank`, `vm.expectRevert`, `vm.warp`, `vm.roll`, `deal`
- Deployment scripts with broadcast & verification
- GitHub Actions CI/CD pipeline
- Mainnet forking for integration tests
- RPC providers and active testnets (2025)

### 📋 ERC Standards (`references/erc-standards.md`)
- **ERC-20**: SafeERC20, Permit (EIP-2612), capped, burnable, non-standard token handling
- **ERC-721**: NFTs, Merkle whitelist minting, on-chain metadata
- **ERC-1155**: Multi-token standard, batch operations
- **ERC-4626**: Tokenized vault standard — the DeFi building block
- **ERC-2981**: NFT royalties
- **EIP-712**: Typed structured data signing, order books, meta-transactions
- **ERC-4337**: Account abstraction, UserOperation, Paymaster
- **ERC-165**: Interface detection
- Transient storage (EIP-1153, Solidity 0.8.24+)

### 🔬 EVM Internals (`references/evm-internals.md`)
- EVM architecture: stack, memory, storage, calldata, code, logs
- Memory layout and the free memory pointer (0x40)
- Storage slot computation for variables, mappings, and dynamic arrays
- ABI encoding and decoding — static vs dynamic types
- `CALL` vs `STATICCALL` vs `DELEGATECALL` vs `CALLCODE`
- `DELEGATECALL` deep dive — how proxies actually work at the opcode level
- `CREATE` vs `CREATE2` — address computation and deterministic deployment
- Solidity compiler output: bytecode, ABI, creation vs runtime code
- Key opcodes with gas costs
- How Solidity maps to bytecode (function dispatch, modifiers, events, mappings)

### 🌐 L2 & Multichain (`references/l2-multichain.md`)
- L2 architecture: Optimistic rollups vs ZK rollups vs Sidechains
- **Arbitrum**: ArbSys precompile, block.number quirks, gas pricing
- **Base & Optimism**: OP Stack, L1Block precompile, `block.timestamp` reliability
- **zkSync Era**: CREATE2 differences, keccak256 cost, native AA
- `block.timestamp` and `block.number` behavior on each chain — critical gotchas
- L2 gas pricing: L1 calldata cost + L2 execution cost
- EIP-4844 blobs and their impact on L2 fees
- Precompile reference by chain
- Cross-chain messaging: native bridges, LayerZero, Chainlink CCIP
- Sequencer risks and escape hatches
- Multi-chain deployment with Foundry + deterministic CREATE2 addresses
- Chain ID reference table (mainnet + testnets)

### 📝 NatSpec & Code Quality (`references/natspec-quality.md`)
- Complete NatSpec tag reference (`@title`, `@notice`, `@dev`, `@param`, `@return`, etc.)
- Full documented contract example (StakingVault with all tags)
- Solidity style guide: naming conventions, function ordering, import organization
- Pre-PR code review checklist (logic, documentation, security, gas, testing)
- SWC Registry — all 30+ vulnerability IDs used in professional audit reports

---

## Install

```bash
npx skills add LollyTim/solidity-evm-skill
```

Works with Claude Code, Cursor (2.4+), and any [skills-compatible](https://skills.sh) agent.

---

## File Structure

```
solidity-evm-skill/
├── SKILL.md                        ← Core skill (always loaded)
└── references/
    ├── security.md                 ← Vulnerabilities & audit
    ├── gas-optimization.md         ← Gas techniques
    ← patterns.md                  ← Design patterns
    ├── tooling.md                  ← Foundry & Hardhat
    ├── erc-standards.md            ← Token standards
    ├── evm-internals.md            ← EVM under the hood
    ├── l2-multichain.md            ← L2 & cross-chain
    └── natspec-quality.md          ← Docs & code quality
```

---

## Built by

[LollyTim](https://github.com/LollyTim) Building on Hyperliquid.

---

## License

MIT