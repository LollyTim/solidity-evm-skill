# L2 & Cross-Chain Development Reference

## Table of Contents
1. [L2 Landscape Overview](#l2-landscape)
2. [EVM Equivalence vs Compatibility](#evm-equivalence)
3. [L2-Specific Gotchas](#l2-gotchas)
4. [Cross-Chain Messaging](#cross-chain-messaging)
5. [Bridge Patterns](#bridge-patterns)
6. [Multi-Chain Deployment Strategy](#multi-chain-deployment)
7. [L2 Gas & Fee Models](#l2-gas-fees)
8. [Chain-Specific Quick Reference](#chain-reference)

---

## L2 Landscape Overview {#l2-landscape}

### Major L2 Networks

| Chain | Type | Stack | EVM Level | Mainnet Since | Notes |
|-------|------|-------|-----------|---------------|-------|
| Arbitrum One | Optimistic Rollup | Nitro | EVM Equivalent | Aug 2021 | Largest TVL L2 |
| Optimism | Optimistic Rollup | OP Stack | EVM Equivalent | Jan 2021 | Superchain ecosystem |
| Base | Optimistic Rollup | OP Stack | EVM Equivalent | Aug 2023 | Coinbase L2, no token |
| zkSync Era | ZK Rollup | zkSync | EVM Compatible | Mar 2023 | Uses zksolc compiler |
| Scroll | ZK Rollup | Scroll | EVM Equivalent | Oct 2023 | Bytecode-level EVM |
| Linea | ZK Rollup | Linea | EVM Equivalent | Jul 2023 | ConsenSys L2 |
| Polygon zkEVM | ZK Rollup | Polygon | EVM Equivalent | Mar 2023 | Type 2 zkEVM |
| StarkNet | ZK Rollup (STARK) | StarkNet | Not EVM | Nov 2022 | Cairo lang, not Solidity |

### Rollup Types

**Optimistic Rollups** assume transactions are valid; fraud proofs challenge invalid state.
- 7-day withdrawal window for L2->L1 (dispute period)
- Cheaper to operate, simpler proof system
- Examples: Arbitrum, Optimism, Base

**ZK Rollups** generate cryptographic validity proofs for every batch.
- Minutes to hours for L2->L1 finality (proof generation time)
- Higher computation cost for proof generation, lower on-chain verification cost
- Examples: zkSync Era, Scroll, Polygon zkEVM

### Settlement vs Execution

```
┌─────────────────────────────────────────┐
│  Execution Layer (L2)                   │
│  - Transactions execute here            │
│  - State stored here                    │
│  - Cheap gas, high throughput           │
└──────────────┬──────────────────────────┘
               │  Batch / Proof posted
┌──────────────▼──────────────────────────┐
│  Settlement Layer (L1 - Ethereum)       │
│  - Rollup contract verifies state       │
│  - Fraud proof or validity proof        │
│  - Final source of truth                │
└─────────────────────────────────────────┘
```

---

## EVM Equivalence vs Compatibility {#evm-equivalence}

### Definitions

**EVM Equivalent** (Type 2 zkEVM / Optimistic): Same bytecode, same opcodes, same behavior. Deploy existing Solidity contracts without modification. Arbitrum, Optimism, Base, Scroll, Polygon zkEVM.

**EVM Compatible** (Type 3-4 zkEVM): Supports Solidity but compiles to a different VM. May require code changes, different compiler, or has opcode gaps. zkSync Era is the primary example.

### Opcode Differences by Chain

| Opcode | Ethereum | Arbitrum | Optimism/Base | zkSync Era | Scroll |
|--------|----------|----------|---------------|------------|--------|
| `PUSH0` | Yes (Shanghai+) | Yes | Yes | No | Yes |
| `SELFDESTRUCT` | Deprecated (Dencun) | Deprecated | Deprecated | Not supported | Deprecated |
| `PREVRANDAO` | Yes (post-Merge) | Returns ArbOS value | Returns L2 value | Not supported | Returns L2 value |
| `DIFFICULTY` | Alias for PREVRANDAO | L1 `PREVRANDAO` | Returns 0 pre-Bedrock | Not supported | Returns 0 |
| `COINBASE` | Block proposer | Sequencer addr | Sequencer addr | Bootloader addr | Sequencer addr |
| `BLOCKHASH` | Last 256 blocks | Last 256 L2 blocks | Last 256 L2 blocks | Limited support | Last 256 L2 blocks |

### Precompile Availability

| Precompile | Address | Ethereum | Arbitrum | OP/Base | zkSync Era |
|------------|---------|----------|----------|---------|------------|
| ecRecover | 0x01 | Yes | Yes | Yes | Yes |
| SHA-256 | 0x02 | Yes | Yes | Yes | Yes |
| RIPEMD-160 | 0x03 | Yes | Yes | Yes | No |
| identity | 0x04 | Yes | Yes | Yes | Yes |
| modexp | 0x05 | Yes | Yes | Yes | Yes |
| ecAdd | 0x06 | Yes | Yes | Yes | Yes |
| ecMul | 0x07 | Yes | Yes | Yes | Yes |
| ecPairing | 0x08 | Yes | Yes | Yes | Yes |
| blake2f | 0x09 | Yes | Yes | Yes | No |

### Compiler Differences (zkSync)

zkSync Era uses `zksolc` which compiles Solidity to zkEVM bytecode:

```bash
# Standard Solidity compilation
solc --bin Contract.sol

# zkSync compilation (different bytecode output)
zksolc --bin Contract.sol

# Foundry with zkSync (uses foundry-zksync fork)
forge build --zksync
```

**Key zkSync compiler differences:**
- `CREATE` / `CREATE2` opcodes work differently; deployer contracts must use system contracts
- Contract bytecode hashes differ from mainnet
- Constructor arguments are handled differently
- Inline assembly may not compile or behave differently

---

## L2-Specific Gotchas {#l2-gotchas}

This is the most critical section. Ignoring these differences causes real bugs on L2.

### block.number Behavior

```solidity
// On Ethereum: block.number = current L1 block (~12s per block)
// On Optimism/Base: block.number = L2 block number (~2s per block)
// On Arbitrum: block.number = L1 block number (NOT L2 block)

// WRONG on Arbitrum: this measures L1 blocks, not L2 blocks
function isExpired() external view returns (bool) {
    return block.number > deadline; // Deadline in L1 blocks on Arbitrum!
}

// CORRECT on Arbitrum: use ArbSys precompile for L2 block number
interface ArbSys {
    function arbBlockNumber() external view returns (uint256);
}

function isExpiredArbitrum() external view returns (bool) {
    uint256 l2Block = ArbSys(address(100)).arbBlockNumber();
    return l2Block > deadline;
}
```

**Recommendation:** Use `block.timestamp` instead of `block.number` for time-based logic. It is more consistent across chains.

### block.timestamp on L2

```solidity
// L2 timestamps are set by the sequencer, not by miners/validators.
// They must be >= L1 timestamp of the batch and monotonically increasing.
// BUT: they can drift slightly from real wall-clock time.

// SAFE: timestamps for durations (auctions, vesting, etc.)
// The sequencer cannot set timestamps arbitrarily far in the future.
uint256 public constant AUCTION_DURATION = 1 hours;
uint256 public auctionEnd;

function startAuction() external {
    auctionEnd = block.timestamp + AUCTION_DURATION; // OK on all L2s
}
```

### Address Aliasing in Cross-Chain Messages

When an L1 contract sends a message to L2 (Optimism/Arbitrum), `msg.sender` on L2 is NOT the L1 contract address. It is aliased by adding an offset:

```solidity
// Address aliasing constant (Optimism and Arbitrum both use this)
uint160 constant OFFSET = uint160(0x1111000000000000000000000000000000001111);

// On L2, msg.sender for an L1->L2 call from contract 0xABCD...1234 will be:
// address(uint160(0xABCD...1234) + OFFSET)

// Undoing the alias to verify the original L1 sender:
function undoAlias(address aliased) internal pure returns (address original) {
    unchecked {
        original = address(uint160(aliased) - OFFSET);
    }
}

// Usage in an L2 contract receiving L1 messages:
address public immutable l1Sender;

modifier onlyL1Sender() {
    // msg.sender is the aliased address; undo it to verify
    require(undoAlias(msg.sender) == l1Sender, "Not L1 sender");
    _;
}
```

**Why aliasing exists:** Without it, an L1 contract at address X could impersonate an L2 EOA at address X. Aliasing ensures L1 contracts have distinct L2 identities.

### tx.origin Unreliability

```solidity
// NEVER use tx.origin for auth, but especially not on L2.
// During L1->L2 message execution, tx.origin is the aliased L1 address
// or the sequencer address depending on the chain.

// BAD on any chain, worse on L2:
function withdraw() external {
    require(tx.origin == owner, "Not owner"); // Broken during cross-chain calls
}
```

### EOA Detection is Unreliable

```solidity
// BAD: This check is unreliable on ALL chains (not just L2)
function isEOA(address account) internal view returns (bool) {
    return account.code.length == 0; // WRONG
}

// Fails when:
// 1. Called during a constructor (code.length == 0 for the deploying contract)
// 2. CREATE2 counterfactual addresses (no code yet, but contract planned)
// 3. Account abstraction (ERC-4337 and native AA on zkSync)
// 4. Future Ethereum changes (EIP-7702 allows EOAs to have code)

// If you must distinguish EOAs, check msg.sender == tx.origin (also imperfect).
// Best practice: design contracts to work regardless of caller type.
```

### COINBASE Returns Sequencer Address

```solidity
// On L1: block.coinbase = validator/proposer address
// On L2: block.coinbase = sequencer fee recipient address

// Arbitrum One:  varies per batch
// Optimism/Base: 0x4200000000000000000000000000000000000011 (SequencerFeeVault)

// DO NOT use block.coinbase for randomness or unique identification on L2.
```

### Gas Behavior Differences

```solidity
// gasleft() works on L2 but the gas model differs.
// On Arbitrum: gasleft() returns ArbGas units (not 1:1 with L1 gas).
// On Optimism/Base: gasleft() returns L2 gas (does not account for L1 data fee).

// CAUTION: gas-based loops that work on L1 may behave differently on L2.
// Always test gas-dependent logic on the target L2 testnet.

// Example: this pattern is fragile across chains
function processUntilGasLimit(uint256[] calldata items) external {
    for (uint256 i; i < items.length; i++) {
        if (gasleft() < 50_000) break; // Threshold may be wrong for L2
        _process(items[i]);
    }
}
```

### Contract Size Limits

```solidity
// Ethereum:     24,576 bytes (EIP-170)
// Arbitrum:     24,576 bytes (same)
// Optimism:     24,576 bytes (same)
// zkSync Era:   ~2^16 * 32 bytes per contract, but different bytecode format
//               Practically larger contracts are possible on zkSync

// If you hit size limits, use the same strategies as L1:
// - Split into libraries
// - Use the diamond/proxy pattern
// - Optimize with `--via-ir` and `--optimize`
```

---

## Cross-Chain Messaging {#cross-chain-messaging}

### Optimism / Base: CrossDomainMessenger

The standard way to send messages between L1 and L2 on the OP Stack.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// ============================================================
// L1 -> L2 Message Sender (deployed on Ethereum)
// ============================================================

interface ICrossDomainMessenger {
    function sendMessage(address _target, bytes calldata _message, uint32 _minGasLimit) external payable;
    function xDomainMessageSender() external view returns (address);
}

contract L1Sender {
    // Optimism L1CrossDomainMessenger proxy on Ethereum mainnet
    ICrossDomainMessenger public constant L1_MESSENGER =
        ICrossDomainMessenger(0x25ace71c97B33Cc4729CF772ae268934F7ab5fA1);

    address public immutable l2Target;

    constructor(address _l2Target) {
        l2Target = _l2Target;
    }

    /// @notice Send a greeting to L2. Anyone can call this.
    function sendGreetingToL2(string calldata greeting) external {
        bytes memory message = abi.encodeCall(L2Receiver.receiveGreeting, (greeting));
        // _minGasLimit is the minimum L2 gas for executing the message
        L1_MESSENGER.sendMessage(l2Target, message, 200_000);
    }
}

// ============================================================
// L2 Receiver (deployed on Optimism / Base)
// ============================================================

contract L2Receiver {
    // L2CrossDomainMessenger is at a predeploy address on all OP Stack chains
    ICrossDomainMessenger public constant L2_MESSENGER =
        ICrossDomainMessenger(0x4200000000000000000000000000000000000007);

    address public immutable l1Sender;
    string public lastGreeting;

    constructor(address _l1Sender) {
        l1Sender = _l1Sender;
    }

    /// @notice Called by the messenger when an L1 message arrives.
    function receiveGreeting(string calldata greeting) external {
        // Verify: only the messenger can call this
        require(msg.sender == address(L2_MESSENGER), "Not messenger");
        // Verify: the L1 sender is our trusted contract
        require(L2_MESSENGER.xDomainMessageSender() == l1Sender, "Wrong L1 sender");

        lastGreeting = greeting;
    }
}
```

### Arbitrum: Retryable Tickets

Arbitrum uses "retryable tickets" for L1->L2 messaging. They auto-execute on L2 but can be manually retried if they fail.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// ============================================================
// L1 -> L2 via Arbitrum Retryable Ticket
// ============================================================

interface IInbox {
    function createRetryableTicket(
        address to,
        uint256 l2CallValue,
        uint256 maxSubmissionCost,
        address excessFeeRefundAddress,
        address callValueRefundAddress,
        uint256 gasLimit,
        uint256 maxFeePerGas,
        bytes calldata data
    ) external payable returns (uint256);
}

contract L1ToArbitrumSender {
    // Arbitrum One Inbox on Ethereum mainnet
    IInbox public constant INBOX = IInbox(0x4Dbd4fc535Ac27206064B68FfCf827b0A60BAB3f);

    address public immutable l2Target;

    constructor(address _l2Target) {
        l2Target = _l2Target;
    }

    /// @notice Send a message to L2. Caller must send enough ETH to cover fees.
    /// @param greeting The greeting string to send to L2
    /// @param maxSubmissionCost Max cost for L2 ticket submission
    /// @param gasLimit Gas limit for L2 execution
    /// @param maxFeePerGas Max gas price on L2
    function sendToL2(
        string calldata greeting,
        uint256 maxSubmissionCost,
        uint256 gasLimit,
        uint256 maxFeePerGas
    ) external payable returns (uint256 ticketId) {
        bytes memory data = abi.encodeCall(ArbitrumL2Receiver.receiveGreeting, (greeting));

        ticketId = INBOX.createRetryableTicket{value: msg.value}(
            l2Target,
            0,                    // l2CallValue (no ETH sent to target)
            maxSubmissionCost,
            msg.sender,           // excessFeeRefundAddress
            msg.sender,           // callValueRefundAddress
            gasLimit,
            maxFeePerGas,
            data
        );
    }
}

// ============================================================
// L2 Receiver (deployed on Arbitrum One)
// ============================================================

interface IArbSys {
    function wasMyCallersAddressAliased() external view returns (bool);
    function myCallersAddressWithoutAliasing() external view returns (address);
}

contract ArbitrumL2Receiver {
    IArbSys constant ARB_SYS = IArbSys(address(100));

    address public immutable l1Sender;
    string public lastGreeting;

    constructor(address _l1Sender) {
        l1Sender = _l1Sender;
    }

    function receiveGreeting(string calldata greeting) external {
        // On Arbitrum, verify the L1 sender via ArbSys
        require(ARB_SYS.wasMyCallersAddressAliased(), "Not an L1 call");
        require(
            ARB_SYS.myCallersAddressWithoutAliasing() == l1Sender,
            "Wrong L1 sender"
        );

        lastGreeting = greeting;
    }
}
```

### L2 -> L1 Timing

| Direction | Optimistic Rollup | ZK Rollup |
|-----------|-------------------|-----------|
| L1 -> L2 | ~10-15 minutes | ~10-20 minutes |
| L2 -> L1 | **~7 days** (challenge period) | ~1-4 hours (proof generation) |

**For optimistic rollups:** L2->L1 messages require waiting the full 7-day challenge window. Third-party bridges (Across, Hop) can provide faster exits using liquidity pools.

---

## Bridge Patterns {#bridge-patterns}

### Lock-and-Mint

Used for bridging native L1 tokens to L2. The original token is locked on L1; a synthetic representation is minted on L2.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

/// @notice L1 side: locks tokens and emits an event for the bridge relayer.
contract L1TokenBridge {
    using SafeERC20 for IERC20;

    IERC20 public immutable token;
    address public immutable l2Bridge;

    event TokensLocked(address indexed from, address indexed to, uint256 amount);
    event TokensUnlocked(address indexed to, uint256 amount);

    constructor(IERC20 _token, address _l2Bridge) {
        token = _token;
        l2Bridge = _l2Bridge;
    }

    /// @notice Lock tokens on L1 to mint on L2.
    function bridgeToL2(address l2Recipient, uint256 amount) external {
        token.safeTransferFrom(msg.sender, address(this), amount);
        // In production, this triggers an L1->L2 message via the native bridge
        emit TokensLocked(msg.sender, l2Recipient, amount);
    }

    /// @notice Unlock tokens when they are burned on L2 (called via L2->L1 message).
    function unlockTokens(address recipient, uint256 amount) external {
        // In production, verify this is called by the cross-chain messenger
        // from the trusted l2Bridge address
        token.safeTransfer(recipient, amount);
        emit TokensUnlocked(recipient, amount);
    }
}
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

/// @notice L2 side: mints synthetic tokens when L1 tokens are locked.
contract L2BridgedToken is ERC20 {
    address public immutable bridge;

    modifier onlyBridge() {
        require(msg.sender == bridge, "Only bridge");
        _;
    }

    constructor(address _bridge) ERC20("Bridged Token", "bTKN") {
        bridge = _bridge;
    }

    function mint(address to, uint256 amount) external onlyBridge {
        _mint(to, amount);
    }

    function burn(address from, uint256 amount) external onlyBridge {
        _burn(from, amount);
    }
}
```

### Burn-and-Unlock

The reverse of lock-and-mint. Used to return bridged tokens from L2 back to L1: burn the synthetic token on L2, unlock the original on L1.

### Liquidity Pool Bridge

Used by fast bridges (Across, Hop, Stargate). No lock/mint -- LPs supply liquidity on both sides.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

/// @notice Simplified liquidity pool bridge (one side). LPs deposit tokens;
///         users receive tokens instantly, and the bridge settles later.
contract LiquidityPoolBridge {
    using SafeERC20 for IERC20;

    IERC20 public immutable token;
    address public relayer;

    mapping(address => uint256) public lpDeposits;
    mapping(bytes32 => bool) public processedTransfers;

    event Deposited(address indexed lp, uint256 amount);
    event TransferSent(bytes32 indexed transferId, address indexed to, uint256 amount);
    event TransferFilled(bytes32 indexed transferId, address indexed to, uint256 amount);

    constructor(IERC20 _token, address _relayer) {
        token = _token;
        relayer = _relayer;
    }

    /// @notice LPs deposit liquidity into the pool.
    function deposit(uint256 amount) external {
        token.safeTransferFrom(msg.sender, address(this), amount);
        lpDeposits[msg.sender] += amount;
        emit Deposited(msg.sender, amount);
    }

    /// @notice Relayer fills a transfer from the other chain using pool liquidity.
    function fillTransfer(bytes32 transferId, address to, uint256 amount) external {
        require(msg.sender == relayer, "Only relayer");
        require(!processedTransfers[transferId], "Already processed");

        processedTransfers[transferId] = true;
        token.safeTransfer(to, amount);
        emit TransferFilled(transferId, to, amount);
    }
}
```

### Choosing a Bridge Pattern

| Pattern | Speed | Capital Efficiency | Trust Assumption | Best For |
|---------|-------|--------------------|------------------|----------|
| Lock-and-Mint | Slow (native bridge) | High | L1 security | Canonical token bridging |
| Burn-and-Unlock | Slow (native bridge) | High | L1 security | Returning bridged tokens |
| Liquidity Pool | Fast (minutes) | Lower (needs LPs) | Relayer + LP trust | Fast user transfers |

### Bridge Security Considerations

1. **Verify the sender on the receiving chain** -- always check `xDomainMessageSender()` or equivalent
2. **Never trust `msg.sender` directly** for cross-chain calls; it will be the messenger contract
3. **Handle failed messages** -- retryable tickets (Arbitrum) and message replay (OP Stack) exist for a reason
4. **Rate-limit bridge flows** to contain exploits
5. **Use canonical bridges** for trust-minimized transfers; third-party bridges for speed

---

## Multi-Chain Deployment Strategy {#multi-chain-deployment}

### Deterministic Deployment with CREATE2

Use the keyless CREATE2 deployer (available on most EVM chains at the same address) to get identical contract addresses across chains.

```solidity
// The "Arachnid" CREATE2 deployer -- deployed at the same address on 100+ chains
// Address: 0x4e59b44847b379578588920cA78FbF26c0B4956C
// It is a keyless deployment (Nick's method), so no owner.

// address = keccak256(0xff ++ deployerAddress ++ salt ++ keccak256(initCode))[12:]
// Same salt + same initCode = same address on every chain
```

### Foundry Multi-Chain Deploy Script

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Script, console} from "forge-std/Script.sol";
import {MyToken} from "../src/MyToken.sol";

contract DeployMultiChain is Script {
    // CREATE2 deployer (Arachnid's factory)
    address constant CREATE2_DEPLOYER = 0x4e59b44847b379578588920cA78FbF26c0B4956C;

    struct ChainConfig {
        string rpcUrl;
        address admin;
        uint256 initialSupply;
    }

    function run() external {
        // Read chain-specific config from environment
        ChainConfig memory config = _getConfig();

        uint256 deployerKey = vm.envUint("DEPLOYER_PRIVATE_KEY");
        bytes32 salt = keccak256("MyToken-v1");

        vm.startBroadcast(deployerKey);

        // Deploy with CREATE2 for deterministic address
        bytes memory initCode = abi.encodePacked(
            type(MyToken).creationCode,
            abi.encode(config.admin, config.initialSupply)
        );

        address deployed;
        assembly {
            deployed := create2(0, add(initCode, 0x20), mload(initCode), salt)
        }
        require(deployed != address(0), "CREATE2 failed");

        console.log("Deployed MyToken at:", deployed);

        vm.stopBroadcast();
    }

    function _getConfig() internal view returns (ChainConfig memory) {
        uint256 chainId = block.chainid;

        if (chainId == 1) {
            // Ethereum mainnet
            return ChainConfig({
                rpcUrl: "mainnet",
                admin: 0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B,
                initialSupply: 1_000_000e18
            });
        } else if (chainId == 42161) {
            // Arbitrum One
            return ChainConfig({
                rpcUrl: "arbitrum",
                admin: 0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B,
                initialSupply: 1_000_000e18
            });
        } else if (chainId == 10) {
            // Optimism
            return ChainConfig({
                rpcUrl: "optimism",
                admin: 0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B,
                initialSupply: 500_000e18
            });
        } else {
            revert("Unsupported chain");
        }
    }
}
```

### Running Multi-Chain Deploys

```bash
# Deploy to multiple chains sequentially
forge script script/DeployMultiChain.s.sol --rpc-url $ETH_RPC --broadcast --verify
forge script script/DeployMultiChain.s.sol --rpc-url $ARB_RPC --broadcast --verify
forge script script/DeployMultiChain.s.sol --rpc-url $OP_RPC  --broadcast --verify

# Verify on multiple explorers
forge verify-contract $ADDR src/MyToken.sol:MyToken \
    --chain-id 42161 \
    --etherscan-api-key $ARBISCAN_KEY

forge verify-contract $ADDR src/MyToken.sol:MyToken \
    --chain-id 10 \
    --etherscan-api-key $OPSCAN_KEY
```

### Chain-Aware Configuration Pattern

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// @notice Base contract that provides chain-specific addresses and config.
abstract contract ChainConfig {
    error UnsupportedChain(uint256 chainId);

    function _getWETH() internal view returns (address) {
        uint256 id = block.chainid;
        if (id == 1)     return 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2; // Ethereum
        if (id == 42161) return 0x82aF49447D8a07e3bd95BD0d56f35241523fBab1; // Arbitrum
        if (id == 10)    return 0x4200000000000000000000000000000000000006; // Optimism
        if (id == 8453)  return 0x4200000000000000000000000000000000000006; // Base
        if (id == 324)   return 0x5AEa5775959fBC2557Cc8789bC1bf90A239D9a91; // zkSync Era
        revert UnsupportedChain(id);
    }

    function _getUSDC() internal view returns (address) {
        uint256 id = block.chainid;
        if (id == 1)     return 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
        if (id == 42161) return 0xaf88d065e77c8cC2239327C5EDb3A432268e5831; // native USDC
        if (id == 10)    return 0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85; // native USDC
        if (id == 8453)  return 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913; // native USDC
        revert UnsupportedChain(id);
    }
}
```

---

## L2 Gas & Fee Models {#l2-gas-fees}

### Optimism / Base Fee Model (post-EIP-4844)

Total fee = **L2 execution fee** + **L1 data fee**

```
L2 execution fee = l2GasUsed * l2GasPrice

L1 data fee (pre-Ecotone)  = l1BaseFee * (txDataGas + fixedOverhead) * dynamicOverhead
L1 data fee (post-Ecotone) = baseFeeScalar * l1BaseFee * txCompressedSize
                            + blobBaseFeeScalar * l1BlobBaseFee * txCompressedSize
```

The L1 data fee typically dominates for simple transactions. On-chain access:

```solidity
// OP Stack L1 Block predeploy -- available on Optimism, Base, and all OP Stack chains
interface IL1Block {
    function basefee() external view returns (uint256);   // L1 base fee
    function blobBaseFee() external view returns (uint256); // L1 blob base fee (post-Ecotone)
    function l1FeeOverhead() external view returns (uint256);
    function l1FeeScalar() external view returns (uint256);
}

// L1Block predeploy address on OP Stack chains
IL1Block constant L1_BLOCK = IL1Block(0x4200000000000000000000000000000000000015);

// GasPriceOracle predeploy for estimating L1 data fee
interface IGasPriceOracle {
    function getL1Fee(bytes memory data) external view returns (uint256);
    function getL1GasUsed(bytes memory data) external view returns (uint256);
}

IGasPriceOracle constant GAS_ORACLE = IGasPriceOracle(0x420000000000000000000000000000000000000F);
```

### Arbitrum Fee Model

```
Total fee = L2 gas fee + L1 calldata fee

L2 gas fee = l2GasUsed * l2BaseFee
L1 calldata fee = calldataUnits * l1BaseFeeEstimate

// Arbitrum NodeInterface (not a real contract, handled by the node)
interface NodeInterface {
    function gasEstimateL1Component(
        address to,
        bool contractCreation,
        bytes calldata data
    ) external view returns (uint64 gasEstimateForL1, uint256 baseFee, uint256 l1BaseFeeEstimate);
}

// Available at this address on Arbitrum
// NodeInterface is at 0x00000000000000000000000000000000000000C8
```

### Calldata Optimization for L2

Since L1 data posting is the dominant cost on rollups, minimizing calldata is critical.

```solidity
// Each zero byte in calldata costs 4 gas; each non-zero byte costs 16 gas (on L1).
// On L2 rollups, this L1 cost is passed through to the user.

// OPTIMIZATION 1: Pack data tightly
// BAD: uint256 for small values wastes calldata (leading zeros still posted)
function transferBad(uint256 amount, uint256 tokenId) external { }
// GOOD: use smaller types in calldata
function transferGood(uint128 amount, uint64 tokenId) external { }

// OPTIMIZATION 2: Use compact encoding
// BAD: abi.encode pads everything to 32 bytes
bytes memory data = abi.encode(addr, amount, flag);          // 96 bytes
// GOOD: abi.encodePacked removes padding
bytes memory data = abi.encodePacked(addr, amount, flag);    // 53 bytes

// OPTIMIZATION 3: Prefer calldata over memory for read-only params
function process(bytes calldata data) external { }  // cheaper than bytes memory
```

### EIP-4844 Blob Impact on L2 Fees

Since the Dencun upgrade (March 2024), L2s post data as blobs instead of calldata:

- **Blob fees** are priced separately from L1 execution gas via a blob gas market
- L2 transaction fees dropped **~10-100x** after EIP-4844 adoption
- Blobs are pruned after ~18 days; permanent data availability requires alternative solutions
- All major L2s (Arbitrum, Optimism, Base, Scroll, etc.) now use blobs

---

## Chain-Specific Quick Reference {#chain-reference}

### Network Details

| Chain | Chain ID | Type | `block.number` | RPC Endpoint | Explorer |
|-------|----------|------|-----------------|--------------|----------|
| Ethereum | 1 | L1 | L1 block | `https://eth.llamarpc.com` | etherscan.io |
| Arbitrum One | 42161 | Optimistic | **L1 block number** | `https://arb1.arbitrum.io/rpc` | arbiscan.io |
| Optimism | 10 | Optimistic | L2 block (~2s) | `https://mainnet.optimism.io` | optimistic.etherscan.io |
| Base | 8453 | Optimistic | L2 block (~2s) | `https://mainnet.base.org` | basescan.org |
| zkSync Era | 324 | ZK Rollup | L2 block | `https://mainnet.era.zksync.io` | explorer.zksync.io |
| Polygon zkEVM | 1101 | ZK Rollup | L2 block | `https://zkevm-rpc.com` | zkevm.polygonscan.com |
| Scroll | 534352 | ZK Rollup | L2 block | `https://rpc.scroll.io` | scrollscan.com |
| Linea | 59144 | ZK Rollup | L2 block | `https://rpc.linea.build` | lineascan.build |

### Key Predeploy / System Addresses

| Contract | Arbitrum | Optimism / Base | zkSync Era |
|----------|----------|-----------------|------------|
| L2 Messenger | ArbSys `0x64` | `0x4200000000000000000000000000000000000007` | L1Messenger `0x8008` |
| Fee oracle | NodeInterface `0xC8` | `0x420000000000000000000000000000000000000F` | -- |
| L1 Block info | -- | `0x4200000000000000000000000000000000000015` | -- |
| WETH | `0x82aF49447D8a07e3bd95BD0d56f35241523fBab1` | `0x4200000000000000000000000000000000000006` | `0x5AEa5775959fBC2557Cc8789bC1bf90A239D9a91` |
| Sequencer fee vault | -- | `0x4200000000000000000000000000000000000011` | -- |

### Special Notes per Chain

| Chain | Key Gotcha | Recommendation |
|-------|-----------|----------------|
| Arbitrum | `block.number` returns L1 block | Use `ArbSys(0x64).arbBlockNumber()` for L2 blocks |
| Arbitrum | Custom gas token support | Check if the chain uses ETH or a custom gas token |
| Optimism/Base | L2->L1 takes 7 days | Use third-party bridges for fast exits |
| Optimism/Base | `COINBASE` is SequencerFeeVault | Do not rely on it for proposer identity |
| zkSync Era | Uses `zksolc` compiler | Test compilation separately; some Solidity features unsupported |
| zkSync Era | Native account abstraction | All accounts can have custom validation logic |
| zkSync Era | `CREATE`/`CREATE2` differ | Use `IContractDeployer` system contract for deterministic deploys |
| Polygon zkEVM | Batch finality can be slow | Wait for verified batches before considering tx final |
| Scroll | Relatively new chain | Verify precompile support for your use case |
| Linea | Proof generation latency | L2->L1 finality depends on proof submission cadence |
