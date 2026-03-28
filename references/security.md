# Solidity Security Reference

## Table of Contents
1. [Top Vulnerability Classes (OWASP for Smart Contracts)](#top-vulnerabilities)
2. [Reentrancy](#reentrancy)
3. [Access Control Failures](#access-control)
4. [Oracle Manipulation](#oracle-manipulation)
5. [Flash Loan Attacks](#flash-loans)
6. [Integer & Arithmetic Issues](#arithmetic)
7. [Front-Running & MEV](#mev)
8. [Denial of Service](#dos)
9. [Signature Replay & Phishing](#signatures)
10. [Storage Collisions (Proxy)](#storage)
11. [ABI Encoding Pitfalls](#abi-encoding)
12. [Advanced MEV Protection](#mev-protection)
13. [Receive vs Fallback Functions](#receive-fallback)
14. [Security Tooling](#tooling)
15. [Pre-Audit Checklist](#checklist)

---

## Top Vulnerability Classes (2025 Context) {#top-vulnerabilities}

In H1 2025 alone, over $2.3B was lost to smart contract exploits. The top categories:

| Rank | Category | Notes |
|------|----------|-------|
| 1 | Access Control | $1.6B+ lost. Missing `onlyOwner`, stolen admin keys |
| 2 | Price Oracle Manipulation | Flash loan-driven oracle attacks |
| 3 | Logic Errors | Incorrect business logic, miscalculated rewards |
| 4 | Reentrancy | Classic and cross-function variants |
| 5 | Input Validation | Missing zero-address checks, unchecked bounds |

---

## Reentrancy {#reentrancy}

### Classic Reentrancy

Attacker contract calls back into the vulnerable contract before state is updated.

**Vulnerable:**
```solidity
mapping(address => uint256) public balances;

function withdraw() external {
    uint256 amount = balances[msg.sender];
    // ❌ External call before state update
    (bool ok,) = msg.sender.call{value: amount}("");
    require(ok);
    balances[msg.sender] = 0;  // too late
}
```

**Secure (CEI pattern):**
```solidity
function withdraw() external {
    uint256 amount = balances[msg.sender];
    balances[msg.sender] = 0;            // EFFECTS first
    (bool ok,) = msg.sender.call{value: amount}("");  // then INTERACTION
    if (!ok) revert TransferFailed();
}
```

**Secure (ReentrancyGuard):**
```solidity
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract Vault is ReentrancyGuard {
    function withdraw() external nonReentrant {
        // even if CEI is violated, guard prevents re-entry
    }
}
```

### Cross-Function Reentrancy

Attacker calls a different function in the same contract during re-entry. Use `nonReentrant` on all mutating functions or use transient storage locks (EIP-1153, Solidity 0.8.24+).

### Read-Only Reentrancy

Attacker re-enters a view function during a state transition to get stale prices. Affects protocols that rely on `balanceOf` or price reads during callbacks (e.g., ERC-777, ERC-1363).

**Defense:** Add reentrancy guards to critical view paths too, or use snapshot-based price readings.

---

## Access Control Failures {#access-control}

### Missing Function Modifiers

```solidity
// ❌ Anyone can call initialize
function initialize(address owner) external {
    _owner = owner;
}

// ✅ Use OpenZeppelin's Initializable
function initialize(address owner) external initializer {
    _owner = owner;
}
```

### Role Management (OpenZeppelin AccessControl)

```solidity
import "@openzeppelin/contracts/access/AccessControl.sol";

contract Protocol is AccessControl {
    bytes32 public constant OPERATOR_ROLE = keccak256("OPERATOR_ROLE");
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");

    constructor(address admin) {
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        // Do NOT grant OPERATOR_ROLE here by default — explicit grants only
    }

    function pause() external onlyRole(PAUSER_ROLE) { ... }
    function executeAction() external onlyRole(OPERATOR_ROLE) { ... }
}
```

### Two-Step Ownership Transfer

Never transfer ownership in a single step — wrong address = lost ownership forever.

```solidity
// OpenZeppelin's Ownable2Step is the safe pattern
import "@openzeppelin/contracts/access/Ownable2Step.sol";

// transferOwnership() proposes, new owner must call acceptOwnership()
```

---

## Oracle Manipulation {#oracle-manipulation}

### Spot Price Manipulation

Attackers use flash loans to move an AMM spot price within one transaction.

**Never use:**
```solidity
// ❌ Easily manipulated by flash loans
uint256 price = IUniswapV2Pair(pair).price0CumulativeLast(); // wrong
(uint112 reserve0, uint112 reserve1,) = pair.getReserves();
uint256 price = reserve1 / reserve0; // instant spot price — manipulable
```

**Use instead:**
- **Chainlink price feeds** — decentralized, delayed by multiple oracles
- **TWAP (Time-Weighted Average Price)** — Uniswap V3's `observe()` or V2's cumulative price
- **Staleness checks** — verify oracle data is fresh

```solidity
// Chainlink with staleness check
AggregatorV3Interface feed = AggregatorV3Interface(priceFeed);
(, int256 price, , uint256 updatedAt, ) = feed.latestRoundData();
if (block.timestamp - updatedAt > MAX_STALENESS) revert StalePrice();
if (price <= 0) revert InvalidPrice();
```

---

## Flash Loan Attacks {#flash-loans}

Flash loans allow borrowing unlimited capital with zero collateral within one transaction. Attacks often:
- Manipulate AMM spot prices
- Exploit governance snapshots at same block as proposal
- Drain undercollateralized lending pools

**Defenses:**
1. Use TWAPs not spot prices
2. Governance: require votes from balance at block N-1, not current block
3. Add minimum lock periods for governance tokens
4. Validate liquidity pool reserves haven't changed within the same transaction

---

## Integer & Arithmetic Issues {#arithmetic}

Solidity 0.8.x+ has built-in overflow protection — but:

```solidity
// ❌ unchecked block bypasses overflow protection
function unsafeAdd(uint256 a, uint256 b) external pure returns (uint256) {
    unchecked { return a + b; }  // can overflow silently
}

// Use unchecked ONLY when you've mathematically proven no overflow is possible
// (common in gas optimization — e.g., loop counters that can't exceed array length)
for (uint256 i; i < arr.length;) {
    // do work
    unchecked { ++i; }  // safe: i can never exceed arr.length
}
```

**Precision loss from integer division:**
```solidity
// ❌ Divides before multiplying — loses precision
uint256 fee = amount / 100 * feeRate;

// ✅ Multiply before dividing — maintains precision
uint256 fee = amount * feeRate / 100;
```

**Scaling for fixed-point math:**
Use basis points (BPS) for percentages: `10_000 BPS = 100%`
```solidity
uint256 public constant FEE_BPS = 300; // 3%
uint256 fee = (amount * FEE_BPS) / 10_000;
```

---

## Front-Running & MEV {#mev}

### Commit-Reveal Scheme

For situations where submission order matters (auctions, games):

```solidity
// Phase 1: commit (hash of value + secret)
function commit(bytes32 commitment) external {
    commitments[msg.sender] = commitment;
    commitBlock[msg.sender] = block.number;
}

// Phase 2: reveal (after commit period)
function reveal(uint256 value, bytes32 secret) external {
    require(block.number > commitBlock[msg.sender] + COMMIT_WINDOW);
    require(commitments[msg.sender] == keccak256(abi.encodePacked(value, secret)));
    // process value
}
```

### EIP-712 Typed Signatures

For off-chain order books and meta-transactions, use EIP-712 to prevent signature replay:

```solidity
import "@openzeppelin/contracts/utils/cryptography/EIP712.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

bytes32 constant ORDER_TYPEHASH = keccak256("Order(address maker,uint256 amount,uint256 nonce)");

function verifyOrder(Order calldata order, bytes calldata sig) internal view {
    bytes32 structHash = keccak256(abi.encode(ORDER_TYPEHASH, order.maker, order.amount, order.nonce));
    bytes32 digest = _hashTypedDataV4(structHash);
    address signer = ECDSA.recover(digest, sig);
    if (signer != order.maker) revert InvalidSignature();
    if (usedNonces[order.maker][order.nonce]) revert NonceAlreadyUsed();
}
```

---

## Denial of Service {#dos}

### Loop DoS

Never iterate over unbounded arrays in critical paths:

```solidity
// ❌ If array grows large, this exceeds block gas limit
function distributeAll() external {
    for (uint i = 0; i < recipients.length; i++) {
        payable(recipients[i]).transfer(shares[i]);
    }
}

// ✅ Pull-over-push — users claim their own share
mapping(address => uint256) public pendingWithdrawals;

function claimShare() external {
    uint256 amount = pendingWithdrawals[msg.sender];
    pendingWithdrawals[msg.sender] = 0;
    (bool ok,) = msg.sender.call{value: amount}("");
    if (!ok) revert TransferFailed();
}
```

### External Call DoS

If one recipient's `receive()` reverts, it blocks all distributions. Always use pull-over-push.

### Gas Griefing

Attacker passes a sub-contract that consumes all forwarded gas. Specify gas limits on critical calls:

```solidity
(bool ok,) = recipient.call{value: amount, gas: 30_000}("");
```

---

## Signature Replay & Phishing {#signatures}

```solidity
// ❌ Raw signature without domain separation — replayable cross-chain/contract
bytes32 hash = keccak256(abi.encodePacked(amount, nonce));
address signer = ECDSA.recover(hash, sig);

// ✅ EIP-712 includes chainId, contract address — not replayable
bytes32 digest = _hashTypedDataV4(structHash);  // from EIP712 base
```

**Never use `tx.origin` for authorization:**
```solidity
// ❌ Phishing: user calls your contract via malicious intermediary
require(tx.origin == owner);  

// ✅ Always check msg.sender
require(msg.sender == owner);
```

---

## Storage Collisions (Proxy Contracts) {#storage}

When using proxies, the proxy and implementation share storage slots. A mismatch = corrupted state.

```solidity
// ❌ If proxy has a variable at slot 0 and implementation also uses slot 0
// Proxy's variable gets overwritten by implementation

// ✅ ERC-1967: use pseudo-random storage slots
// Implementation slot: bytes32(uint256(keccak256('eip1967.proxy.implementation')) - 1)
// OpenZeppelin handles this automatically in UUPSUpgradeable / TransparentUpgradeableProxy
```

**Storage upgrade rules:**
1. Never remove existing state variables
2. Never change variable types
3. Never reorder variables
4. Only APPEND new variables
5. Use OpenZeppelin Upgrades Plugin to validate

---

## ABI Encoding Pitfalls {#abi-encoding}

### `abi.encodePacked` Hash Collisions

```solidity
// ❌ DANGEROUS — abi.encodePacked with multiple dynamic types creates collisions
bytes32 hash1 = keccak256(abi.encodePacked("ab", "c"));   // same hash!
bytes32 hash2 = keccak256(abi.encodePacked("a", "bc"));   // same hash!
// Both produce keccak256("abc")

// ❌ Real exploit vector — user can craft colliding inputs
function verify(string calldata name, string calldata id) external {
    bytes32 hash = keccak256(abi.encodePacked(name, id));  // collisions possible
}

// ✅ Use abi.encode — adds length prefixes, no collisions
bytes32 hash1 = keccak256(abi.encode("ab", "c"));   // different hash
bytes32 hash2 = keccak256(abi.encode("a", "bc"));   // different hash

// ✅ Or use a separator
bytes32 hash = keccak256(abi.encodePacked(name, "|", id));
```

### Encoding Function Comparison

| Function | Padding | Dynamic Types | Use Case |
|----------|---------|---------------|----------|
| `abi.encode` | ABI-standard (32-byte) | Length-prefixed, safe | Default choice, cross-contract calls |
| `abi.encodePacked` | Tight (no padding) | NO length prefix, collision risk | Gas-optimized hashing of fixed-size types only |
| `abi.encodeWithSelector` | ABI + 4-byte selector | Same as encode | Low-level calls with selector |
| `abi.encodeCall` | Type-safe + ABI | Compiler-checked | **Preferred** for low-level calls (Solidity 0.8.11+) |

### Type-Safe `abi.encodeCall` (Preferred)

```solidity
// ❌ No type checking — silent bugs if signature changes
bytes memory data = abi.encodeWithSelector(
    IVault.deposit.selector,
    amount,
    receiver  // what if these args are swapped? No compile error
);

// ✅ Compiler-checked — fails to compile if types don't match
bytes memory data = abi.encodeCall(
    IVault.deposit,
    (amount, receiver)  // compiler verifies types match the function signature
);

(bool ok, ) = vault.call(data);
```

### Low-Level Call Return Data Decoding

```solidity
// When a low-level call fails, decode the revert reason
function _safeCall(address target, bytes memory data) internal returns (bytes memory) {
    (bool success, bytes memory returnData) = target.call(data);

    if (!success) {
        // If there's return data, bubble up the revert reason
        if (returnData.length > 0) {
            assembly {
                let size := mload(returnData)
                revert(add(32, returnData), size)
            }
        }
        revert CallFailed();
    }
    return returnData;
}

// Decode specific return types
(bool ok, bytes memory result) = target.call(abi.encodeCall(IERC20.balanceOf, (user)));
if (ok && result.length >= 32) {
    uint256 balance = abi.decode(result, (uint256));
}
```

### try/catch for External Calls

```solidity
// Graceful handling of external call failures
function safeGetPrice(AggregatorV3Interface feed) internal view returns (uint256) {
    try feed.latestRoundData() returns (
        uint80 roundId, int256 answer, uint256, uint256 updatedAt, uint80 answeredInRound
    ) {
        if (answer <= 0) revert InvalidPrice();
        if (updatedAt == 0 || answeredInRound < roundId) revert StaleRound();
        return uint256(answer);
    } catch {
        revert OracleCallFailed();
    }
}

// try/catch with different error types
try externalContract.riskyFunction() returns (uint256 result) {
    // success path
} catch Error(string memory reason) {
    // require() or revert("reason") failure
    emit FailedWithReason(reason);
} catch Panic(uint errorCode) {
    // assert() failure or arithmetic error
    emit FailedWithPanic(errorCode);
} catch (bytes memory lowLevelData) {
    // custom error or other failure
    emit FailedWithData(lowLevelData);
}
```

---

## Advanced MEV Protection {#mev-protection}

Beyond commit-reveal schemes:

### Flashbots Protect / Private Mempools

```solidity
// No contract-level change needed — use private RPC endpoints:
// Flashbots Protect: https://rpc.flashbots.net
// MEV Blocker: https://rpc.mevblocker.io
// These route transactions through private channels, hiding them from MEV searchers

// In foundry.toml:
// [rpc_endpoints]
// mainnet_private = "https://rpc.flashbots.net"
```

### Slippage Protection (DEX Integrations)

```solidity
// ❌ No slippage protection — sandwich attackable
function swap(address tokenIn, address tokenOut, uint256 amountIn) external {
    router.exactInputSingle(params);  // any output accepted
}

// ✅ User-specified minimum output
function swap(
    address tokenIn,
    address tokenOut,
    uint256 amountIn,
    uint256 amountOutMinimum,  // user sets acceptable minimum
    uint256 deadline           // user sets expiry
) external {
    if (block.timestamp > deadline) revert DeadlineExpired();

    ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
        tokenIn: tokenIn,
        tokenOut: tokenOut,
        fee: 3000,
        recipient: msg.sender,
        deadline: deadline,
        amountIn: amountIn,
        amountOutMinimum: amountOutMinimum,  // sandwich protection
        sqrtPriceLimitX96: 0
    });
    router.exactInputSingle(params);
}
```

### Deadline Pattern

```solidity
// Always include deadlines for time-sensitive operations
modifier checkDeadline(uint256 deadline) {
    if (block.timestamp > deadline) revert TransactionTooOld();
    _;
}

function deposit(uint256 amount, uint256 deadline) external checkDeadline(deadline) {
    // ... deposit logic
}
```

### MEV Protection Summary

| Technique | Protects Against | Implementation |
|-----------|-----------------|----------------|
| Commit-reveal | Front-running | Contract-level (2 transactions) |
| Flashbots Protect | Sandwich, front-run | RPC endpoint (no code change) |
| Slippage bounds | Sandwich attacks | `amountOutMinimum` parameter |
| Deadline | Stale transaction execution | `deadline` parameter |
| Private mempool | All public mempool MEV | RPC endpoint (no code change) |
| Batch auctions | DEX front-running | Protocol design (e.g., CoW Protocol) |

---

## Receive vs Fallback Functions {#receive-fallback}

```solidity
contract EthReceiver {
    // receive() — called when msg.data is EMPTY and ETH is sent
    // Must be external payable, no arguments, no return
    receive() external payable {
        emit Received(msg.sender, msg.value);
    }

    // fallback() — called when no function matches OR msg.data is non-empty
    // Can optionally be payable
    fallback() external payable {
        // Called when:
        // 1. Function selector doesn't match any function
        // 2. msg.data is non-empty and no receive() exists
        emit FallbackCalled(msg.sender, msg.value, msg.data);
    }
}
```

### Dispatch Logic

```
                     msg.data is empty?
                     /              \
                   YES              NO
                   /                  \
           receive() exists?      fallback() exists?
            /        \              /          \
          YES        NO           YES          NO
          /           \           /              \
      receive()    fallback()  fallback()      REVERT
                   exists?
                  /       \
                YES       NO
                /           \
           fallback()     REVERT
```

### Security Considerations

```solidity
// ❌ Empty receive — contract silently accepts ETH it may not handle
receive() external payable {}

// ✅ Explicit about who can send ETH
receive() external payable {
    if (msg.sender != address(weth) && msg.sender != address(router)) {
        revert UnexpectedETH();
    }
}

// ❌ Dangerous fallback — can mask bugs (calls to non-existent functions succeed)
fallback() external payable {}

// ✅ Fallback for proxy pattern only
fallback() external payable {
    _delegate(implementation);
}
```

---

## Security Tooling {#tooling}

| Tool | Type | Use For |
|------|------|---------|
| **Slither** | Static analysis | ~40+ detectors, fast, CI-friendly |
| **Mythril** | Symbolic execution | Finds deeper logic bugs |
| **Echidna** | Fuzzer | Property-based testing |
| **Foundry fuzz** | Fuzzer | Integrated into test suite |
| **Tenderly** | Runtime monitoring | Transaction simulation, alerts |
| **OpenZeppelin Defender** | Ops security | Relayers, autotasks, monitoring |
| **Certora Prover** | Formal verification | Mathematical correctness proofs |

**Run Slither:**
```bash
pip install slither-analyzer
slither . --config-file slither.config.json
```

**Slither config example:**
```json
{
  "filter_paths": "node_modules,lib",
  "detectors_to_exclude": "naming-convention"
}
```

---

## Pre-Deployment Security Checklist {#checklist}

### Access Control
- [ ] All privileged functions have access modifiers
- [ ] Two-step ownership transfer (Ownable2Step)
- [ ] Role hierarchy is minimal and documented
- [ ] No `tx.origin` for authorization

### Reentrancy
- [ ] All external calls follow CEI pattern
- [ ] `nonReentrant` on complex flows with multiple external calls
- [ ] Pull-over-push used for ETH distributions

### Input Validation
- [ ] `address != address(0)` checks in constructors and setters
- [ ] Amount > 0 where required
- [ ] Array lengths validated
- [ ] Overflow-prone math uses safe arithmetic

### Oracle Safety
- [ ] No spot AMM prices for pricing
- [ ] TWAP or Chainlink used
- [ ] Staleness checks on price feeds
- [ ] Fallback oracle or circuit breaker exists

### Upgradeability (if applicable)
- [ ] `_disableInitializers()` in implementation constructor
- [ ] No state variable removals or reorders across upgrades
- [ ] `_authorizeUpgrade` properly restricted
- [ ] Storage layout validated by OZ Upgrades Plugin

### Testing
- [ ] Unit tests for all critical paths
- [ ] Integration tests for multi-contract scenarios
- [ ] Fuzz tests for financial math functions
- [ ] Invariant tests (total supply, balance conservation)
- [ ] Gas snapshot baseline established