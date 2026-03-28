# Solidity Tooling Reference

## Table of Contents
1. [Foundry — Complete Guide](#foundry)
2. [Hardhat — Complete Guide](#hardhat)
3. [Which to Choose](#comparison)
4. [Testing Patterns](#testing)
5. [Deployment & Scripts](#deployment)
6. [CI/CD Integration](#ci)
7. [Debugging & Inspection](#debugging)
8. [RPC Providers & Testnets](#rpc)
9. [Contract Verification](#verification)
10. [Testing Best Practices & Edge Cases](#testing-edge-cases)
11. [Multi-Chain Deployment](#multi-chain)

---

## Foundry — Complete Guide {#foundry}

Foundry is the industry-leading Rust-based toolchain. As of 2025, 51%+ of Solidity developers use it.

### Installation

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup  # install/update to latest
```

### Project Setup

```bash
forge init my-project
cd my-project
```

**Structure:**
```
my-project/
├── foundry.toml        # config
├── src/                # contracts
├── test/               # .t.sol test files
├── script/             # .s.sol deployment scripts
└── lib/                # git submodule dependencies
```

**Install dependencies:**
```bash
forge install OpenZeppelin/openzeppelin-contracts
forge install OpenZeppelin/openzeppelin-contracts-upgradeable
forge install foundry-rs/forge-std          # testing utilities (usually auto-installed)
```

**foundry.toml:**
```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
remappings = [
    "@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/",
    "@openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts/",
]
optimizer = true
optimizer_runs = 200
via_ir = false  # set true for aggressive optimization (slower compile)

[profile.ci]
fuzz = { runs = 1000 }
invariant = { runs = 256 }

[rpc_endpoints]
mainnet = "${MAINNET_RPC}"
sepolia = "${SEPOLIA_RPC}"
base = "${BASE_RPC}"
arbitrum = "${ARBITRUM_RPC}"
```

### Common Forge Commands

```bash
forge build                          # compile contracts
forge test                           # run all tests
forge test -vvv                      # verbose: show traces
forge test --match-test testDeposit  # run specific test
forge test --match-contract Vault    # run tests in VaultTest
forge test --gas-report              # show gas usage per function
forge snapshot                       # write .gas-snapshot
forge snapshot --diff                # diff against last snapshot
forge coverage                       # generate coverage report
forge fmt                            # auto-format contracts
forge doc                            # generate docs from NatSpec
```

### Cast — Onchain Interaction

```bash
# Read contract state
cast call 0x6B175... "totalSupply()(uint256)" --rpc-url mainnet

# Send transaction
cast send 0x... "transfer(address,uint256)" 0xRecipient 1000000000000000000 \
    --private-key $PRIVATE_KEY --rpc-url sepolia

# Decode calldata
cast calldata-decode "transfer(address,uint256)" 0xa9059cbb...

# Convert units
cast --to-unit 1000000000000000000 ether  # → 1.000000000000000000
cast --to-wei 1 ether                     # → 1000000000000000000

# Get block info
cast block latest --rpc-url mainnet

# Compute function selector
cast sig "transfer(address,uint256)"      # → 0xa9059cbb

# Compute storage slot for mapping
cast index address 0x1234... 0            # slot for mapping at slot 0
```

### Anvil — Local Node

```bash
anvil                                        # start local chain (default: 8545)
anvil --fork-url $MAINNET_RPC               # fork mainnet
anvil --fork-url $MAINNET_RPC --block-number 19000000  # fork at specific block
anvil --accounts 10 --balance 1000          # 10 funded accounts
```

### Chisel — Solidity REPL

```bash
chisel
# then type Solidity directly:
uint256 x = 1 ether;
x * 2
```

---

## Writing Foundry Tests {#foundry-tests}

```solidity
// test/Vault.t.sol
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../src/Vault.sol";

contract VaultTest is Test {
    Vault vault;
    address alice = makeAddr("alice");    // deterministic test address
    address bob   = makeAddr("bob");

    // Runs before each test
    function setUp() public {
        vault = new Vault();
        vm.deal(alice, 10 ether);         // give alice ETH
        vm.deal(bob, 10 ether);
    }

    // Unit test
    function test_Deposit() public {
        vm.prank(alice);                   // next call from alice
        vault.deposit{value: 1 ether}();
        assertEq(vault.balanceOf(alice), 1 ether);
    }

    // Expect revert
    function test_RevertWhen_DepositZero() public {
        vm.prank(alice);
        vm.expectRevert(Vault.ZeroAmount.selector);
        vault.deposit{value: 0}();
    }

    // Expect emit
    function test_EmitsDepositEvent() public {
        vm.expectEmit(true, false, false, true);  // (topic1 indexed, topic2, topic3, data)
        emit Vault.Deposited(alice, 1 ether);
        vm.prank(alice);
        vault.deposit{value: 1 ether}();
    }

    // Fuzz test — Foundry generates random `amount` values
    function testFuzz_Deposit(uint256 amount) public {
        amount = bound(amount, 1, 10 ether);       // constrain to valid range
        vm.deal(alice, amount);
        vm.prank(alice);
        vault.deposit{value: amount}();
        assertEq(vault.balanceOf(alice), amount);
    }

    // Impersonate any address
    function test_WithImpersonation() public {
        vm.startPrank(alice);
        vault.deposit{value: 1 ether}();
        vault.approve(bob, 1 ether);
        vm.stopPrank();

        vm.prank(bob);
        vault.withdraw(alice, 0.5 ether);
    }

    // Time manipulation
    function test_Vesting() public {
        vm.warp(block.timestamp + 30 days);   // advance time
        vm.roll(block.number + 100);           // advance blocks
    }
}
```

### Invariant Tests

Property that must ALWAYS hold, no matter what sequence of calls:

```solidity
contract VaultInvariantTest is Test {
    Vault vault;
    VaultHandler handler;

    function setUp() public {
        vault = new Vault();
        handler = new VaultHandler(vault);
        targetContract(address(handler));  // fuzzer calls handler's functions
    }

    // This invariant must hold after any sequence of actions
    function invariant_TotalSupplyEqualsBalances() public {
        assertEq(vault.totalSupply(), handler.sumBalances());
    }

    function invariant_ContractBalanceGTEDeposits() public {
        assertGe(address(vault).balance, vault.totalSupply());
    }
}

contract VaultHandler is Test {
    Vault vault;
    address[] public actors;

    constructor(Vault _vault) { vault = _vault; }

    function deposit(uint256 amount) external {
        amount = bound(amount, 1, 100 ether);
        vm.deal(msg.sender, amount);
        vm.prank(msg.sender);
        vault.deposit{value: amount}();
        actors.push(msg.sender);
    }

    function sumBalances() external view returns (uint256 total) {
        for (uint i; i < actors.length; i++) {
            total += vault.balanceOf(actors[i]);
        }
    }
}
```

---

## Hardhat — Complete Guide {#hardhat}

Node.js-based, great for TypeScript/JS teams and projects needing rich scripting.

### Installation

```bash
mkdir my-project && cd my-project
npm init -y
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox
npx hardhat init  # select "TypeScript project"
```

**hardhat.config.ts:**
```typescript
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";

const config: HardhatUserConfig = {
  solidity: {
    version: "0.8.24",
    settings: {
      optimizer: { enabled: true, runs: 200 },
    },
  },
  networks: {
    mainnet: {
      url: process.env.MAINNET_RPC || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    },
    sepolia: {
      url: process.env.SEPOLIA_RPC || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    },
  },
  etherscan: {
    apiKey: process.env.ETHERSCAN_API_KEY,
  },
  gasReporter: {
    enabled: process.env.REPORT_GAS !== undefined,
    currency: "USD",
  },
};
export default config;
```

### Hardhat Tests (TypeScript)

```typescript
import { ethers } from "hardhat";
import { expect } from "chai";
import { Vault, Vault__factory } from "../typechain-types";

describe("Vault", function () {
  let vault: Vault;
  let alice: Signer, bob: Signer;

  beforeEach(async function () {
    [, alice, bob] = await ethers.getSigners();
    const VaultFactory = await ethers.getContractFactory("Vault") as Vault__factory;
    vault = await VaultFactory.deploy();
    await vault.waitForDeployment();
  });

  it("Should deposit ETH", async function () {
    await vault.connect(alice).deposit({ value: ethers.parseEther("1") });
    expect(await vault.balanceOf(alice.address)).to.equal(ethers.parseEther("1"));
  });

  it("Should revert on zero deposit", async function () {
    await expect(vault.connect(alice).deposit({ value: 0 }))
      .to.be.revertedWithCustomError(vault, "ZeroAmount");
  });

  it("Should emit Deposited event", async function () {
    await expect(vault.connect(alice).deposit({ value: ethers.parseEther("1") }))
      .to.emit(vault, "Deposited")
      .withArgs(alice.address, ethers.parseEther("1"));
  });

  it("Should fork mainnet and test with real DAI", async function () {
    // This test requires: npx hardhat test --network hardhat (with mainnet fork config)
    const daiAddress = "0x6B175474E89094C44Da98b954EedeAC495271d0F";
    const dai = await ethers.getContractAt("IERC20", daiAddress);
    const whale = await ethers.getImpersonatedSigner("0xWhaleAddress...");
    // ... interact with fork state
  });
});
```

### Hardhat + Ignition (Deployment)

```typescript
// ignition/modules/Vault.ts
import { buildModule } from "@nomicfoundation/hardhat-ignition/modules";

const VaultModule = buildModule("VaultModule", (m) => {
  const treasury = m.getParameter("treasury", "0x...");
  const vault = m.contract("Vault", [treasury]);
  return { vault };
});

export default VaultModule;
```

```bash
npx hardhat ignition deploy ignition/modules/Vault.ts --network sepolia
npx hardhat ignition deploy ignition/modules/Vault.ts --network mainnet --verify
```

---

## Which to Choose {#comparison}

| Situation | Use |
|-----------|-----|
| New project, Solidity-only team | **Foundry** |
| TypeScript-heavy team, complex scripts | **Hardhat** |
| Fastest compile/test cycles | **Foundry** |
| Rich plugin ecosystem needed | **Hardhat** |
| Advanced fuzz/invariant testing | **Foundry** |
| Team knows JS but not Rust | **Hardhat** |
| Both in same project | Both (hardhat-foundry plugin) |

As of 2025 Solidity survey: Foundry 51.1%, Hardhat 32.9%.

---

## Deployment & Scripts {#deployment}

### Foundry Deployment Script

```solidity
// script/Deploy.s.sol
pragma solidity ^0.8.24;

import "forge-std/Script.sol";
import "../src/Vault.sol";

contract DeployVault is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        address treasury = vm.envAddress("TREASURY_ADDRESS");

        vm.startBroadcast(deployerPrivateKey);

        Vault vault = new Vault(treasury);
        console.log("Vault deployed to:", address(vault));

        vm.stopBroadcast();
    }
}
```

```bash
# Dry run (no broadcast)
forge script script/Deploy.s.sol --rpc-url sepolia -vvvv

# Broadcast to network
forge script script/Deploy.s.sol --rpc-url sepolia --broadcast --verify

# Verify existing deployment
forge verify-contract 0x... src/Vault.sol:Vault --chain sepolia
```

### Environment Setup

```bash
# .env
PRIVATE_KEY=0x...
MAINNET_RPC=https://mainnet.infura.io/v3/YOUR_KEY
SEPOLIA_RPC=https://sepolia.infura.io/v3/YOUR_KEY
BASE_RPC=https://mainnet.base.org
ETHERSCAN_API_KEY=...
BASESCAN_API_KEY=...
```

---

## CI/CD Integration {#ci}

### GitHub Actions (Foundry)

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          submodules: recursive  # important for forge libs

      - uses: foundry-rs/foundry-toolchain@v1
        with:
          version: nightly

      - name: Install dependencies
        run: forge install

      - name: Build
        run: forge build --sizes

      - name: Test
        run: forge test -vvv

      - name: Snapshot check
        run: forge snapshot --check  # fail if gas increases unexpectedly

      - name: Coverage
        run: forge coverage --report lcov

      - name: Slither
        uses: crytic/slither-action@v0.3.0
```

---

## Debugging & Inspection {#debugging}

### Foundry Traces

```bash
# -v = test name, -vv = logs, -vvv = call traces, -vvvv = all traces
forge test -vvvv --match-test test_FailingTest
```

### console.log in Contracts

```solidity
import "forge-std/console.sol";

function deposit(uint256 amount) external {
    console.log("Depositing", amount, "from", msg.sender);
    console.log("Balance before:", balances[msg.sender]);
}
```

### Storage Inspection

```bash
# Read any storage slot
cast storage 0xContractAddress 0  # slot 0
cast storage 0xContractAddress $(cast index address 0xUser 1)  # slot for mapping[user] at slot 1
```

---

## RPC Providers & Testnets {#rpc}

### Major RPC Providers (2025)

| Provider | Free Tier | Notes |
|----------|-----------|-------|
| Alchemy | Yes | Best DX, reliable |
| Infura | Yes | Established, good uptime |
| QuickNode | Yes | Fast, many chains |
| Tenderly | Yes | Best for debugging |
| Chainstack | Yes | Multi-chain support |

### Active Testnets (2025)

| Testnet | Chain ID | Faucet |
|---------|---------|--------|
| Sepolia (ETH) | 11155111 | sepoliafaucet.com |
| Holesky (ETH, staking) | 17000 | cloud.google.com/application/web3 |
| Base Sepolia | 84532 | faucet.quicknode.com |
| Arbitrum Sepolia | 421614 | faucets.chain.link |
| Optimism Sepolia | 11155420 | app.optimism.io/faucet |

**Note:** Goerli and Mumbai are deprecated and should not be used.

---

## Contract Verification {#verification}

### Foundry Verification

```bash
# Verify on Etherscan
forge verify-contract 0xDeployedAddress src/MyContract.sol:MyContract \
    --chain sepolia \
    --etherscan-api-key $ETHERSCAN_API_KEY

# With constructor args
forge verify-contract 0xDeployedAddress src/MyContract.sol:MyContract \
    --chain mainnet \
    --constructor-args $(cast abi-encode "constructor(address,uint256)" 0xTreasury 1000) \
    --etherscan-api-key $ETHERSCAN_API_KEY

# Verify proxy implementation
forge verify-contract 0xImplAddress src/MyContract.sol:MyContract \
    --chain mainnet \
    --etherscan-api-key $ETHERSCAN_API_KEY
# Then on Etherscan UI: "Is this a proxy?" → verify proxy

# Auto-verify during deployment
forge script script/Deploy.s.sol --rpc-url sepolia --broadcast --verify
```

### Hardhat Verification

```bash
npx hardhat verify --network sepolia 0xDeployedAddress "constructorArg1" "constructorArg2"

# For proxy contracts
npx hardhat verify --network mainnet 0xImplementationAddress
```

### Sourcify (decentralized verification)

- Alternative to Etherscan — fully open-source

```bash
forge verify-contract 0xAddress src/Contract.sol:Contract \
    --verifier sourcify --chain-id 1
```

### Multi-file vs Flattened

- Foundry/Hardhat submit standard JSON input (preferred — preserves imports)
- Flattened: `forge flatten src/Contract.sol > Flat.sol` (last resort)
- Common issue: mismatched compiler settings — ensure exact same version, optimizer settings, viaIR flag

### Verification Troubleshooting

| Problem | Fix |
|---------|-----|
| "ByteCode does not match" | Check compiler version, optimizer runs, viaIR flag |
| Constructor args wrong | Use `cast abi-encode` to get exact encoding |
| Proxy not showing as proxy | Use Etherscan "More Options → Is this a proxy?" |
| Libraries not linked | Pass `--libraries` flag with deployed library addresses |

---

## Testing Best Practices & Edge Cases {#testing-edge-cases}

### Edge Cases Every Test Suite Should Cover

```solidity
// Boundary values
function test_MaxUint256() public {
    vm.expectRevert();  // or handle gracefully
    vault.deposit(type(uint256).max);
}

function test_ZeroAmount() public {
    vm.expectRevert(Vault.ZeroAmount.selector);
    vault.deposit(0);
}

function test_ZeroAddress() public {
    vm.expectRevert(Vault.ZeroAddress.selector);
    vault.transfer(address(0), 100);
}

// Empty arrays
function test_EmptyArray() public {
    uint256[] memory empty = new uint256[](0);
    vault.batchDeposit(empty);  // should handle gracefully
}

// Same-block interactions
function test_DepositAndWithdrawSameBlock() public {
    vault.deposit{value: 1 ether}();
    vault.withdraw(1 ether);  // some protocols restrict this
}

// Reentrancy attempt
function test_ReentrancyProtection() public {
    ReentrantAttacker attacker = new ReentrantAttacker(vault);
    vm.deal(address(attacker), 10 ether);
    vm.expectRevert();  // should revert on re-entry
    attacker.attack();
}

// Overflow in multiplication before division
function testFuzz_NoOverflowInFeeCalc(uint256 amount, uint256 feeBps) public {
    amount = bound(amount, 0, type(uint128).max);
    feeBps = bound(feeBps, 0, 10_000);
    uint256 fee = (amount * feeBps) / 10_000;
    assertLe(fee, amount);
}
```

### Fork Testing Patterns

```solidity
// Test against real mainnet state
function test_SwapOnUniswap() public {
    vm.createSelectFork("mainnet", 19_000_000);  // pin block for reproducibility

    ISwapRouter router = ISwapRouter(0xE592427A0AEce92De3Edee1F18E0157C05861564);
    // ... test real swap
}

// Multi-fork testing
function test_CrossChainState() public {
    uint256 mainnetFork = vm.createFork("mainnet");
    uint256 arbitrumFork = vm.createFork("arbitrum");

    vm.selectFork(mainnetFork);
    uint256 l1Balance = token.balanceOf(bridge);

    vm.selectFork(arbitrumFork);
    uint256 l2Supply = token.totalSupply();

    assertEq(l1Balance, l2Supply);  // bridge invariant
}
```

### Test Organization Convention

```
test/
├── unit/              # isolated function tests
│   ├── Vault.deposit.t.sol
│   └── Vault.withdraw.t.sol
├── integration/       # multi-contract interactions
│   └── VaultRouter.t.sol
├── fuzz/              # property-based tests
│   └── Vault.fuzz.t.sol
├── invariant/         # stateful invariant tests
│   ├── handlers/
│   │   └── VaultHandler.sol
│   └── Vault.invariant.t.sol
└── fork/              # mainnet fork tests
    └── VaultMainnet.t.sol
```

---

## Multi-Chain Deployment {#multi-chain}

### Foundry Multi-Chain Script

```solidity
// script/DeployMultiChain.s.sol
pragma solidity ^0.8.24;

import "forge-std/Script.sol";
import "../src/MyContract.sol";

contract DeployMultiChain is Script {
    struct ChainConfig {
        string rpcAlias;
        address treasury;
        uint256 feeBps;
    }

    function run() external {
        // Deploy to current --rpc-url chain
        uint256 deployerKey = vm.envUint("PRIVATE_KEY");
        address treasury = vm.envAddress("TREASURY");

        vm.startBroadcast(deployerKey);
        MyContract c = new MyContract(treasury);
        console.log("Deployed to:", address(c));
        vm.stopBroadcast();
    }
}
```

```bash
# Deploy to multiple chains sequentially
forge script script/Deploy.s.sol --rpc-url sepolia --broadcast --verify
forge script script/Deploy.s.sol --rpc-url base-sepolia --broadcast --verify
forge script script/Deploy.s.sol --rpc-url arbitrum-sepolia --broadcast --verify

# Same address via CREATE2 (use a deterministic deployer)
forge script script/DeployCreate2.s.sol --rpc-url sepolia --broadcast
forge script script/DeployCreate2.s.sol --rpc-url base --broadcast
# Both deploy to the same address if salt and initcode match
```

### Chain-Specific Configuration

```toml
# foundry.toml
[rpc_endpoints]
mainnet = "${MAINNET_RPC}"
sepolia = "${SEPOLIA_RPC}"
base = "${BASE_RPC}"
base-sepolia = "${BASE_SEPOLIA_RPC}"
arbitrum = "${ARBITRUM_RPC}"
arbitrum-sepolia = "${ARBITRUM_SEPOLIA_RPC}"
optimism = "${OPTIMISM_RPC}"

[etherscan]
mainnet = { key = "${ETHERSCAN_API_KEY}" }
sepolia = { key = "${ETHERSCAN_API_KEY}" }
base = { key = "${BASESCAN_API_KEY}", url = "https://api.basescan.org/api" }
arbitrum = { key = "${ARBISCAN_API_KEY}", url = "https://api.arbiscan.io/api" }
optimism = { key = "${OPTIMISTIC_API_KEY}", url = "https://api-optimistic.etherscan.io/api" }
```

### Block Explorer APIs

| Chain | Explorer | API Base URL |
|-------|----------|-------------|
| Ethereum | Etherscan | api.etherscan.io |
| Base | Basescan | api.basescan.org |
| Arbitrum | Arbiscan | api.arbiscan.io |
| Optimism | Optimistic Etherscan | api-optimistic.etherscan.io |
| Polygon | Polygonscan | api.polygonscan.com |