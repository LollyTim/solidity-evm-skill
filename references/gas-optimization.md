# Gas Optimization Reference

## Table of Contents
1. [EVM Gas Fundamentals](#fundamentals)
2. [Storage Optimization](#storage)
3. [Variable & Type Choices](#variables)
4. [Function & Call Optimization](#functions)
5. [Loop Optimization](#loops)
6. [Deployment Optimization](#deployment)
7. [Yul Assembly Patterns](#yul)
8. [Measurement Tools](#measurement)
9. [Contract Size Limit (24KB)](#contract-size)
10. [Transient Storage Patterns (EIP-1153)](#transient-storage)
11. [Events as Cheap Storage](#events-as-storage)
12. [Quick Reference Table](#reference-table)

---

## EVM Gas Fundamentals {#fundamentals}

Understanding where gas comes from is prerequisite to optimizing it.

### Key OPCODE Costs (post-Berlin/Merge)

| Operation | Gas Cost | Notes |
|-----------|----------|-------|
| SSTORE (new, nonzero) | ~22,100 | Writing new non-zero storage slot |
| SSTORE (modify) | ~5,000 | Updating existing non-zero slot |
| SLOAD (cold) | 2,100 | First read of a storage slot in a tx |
| SLOAD (warm) | 100 | Subsequent reads of same slot in a tx |
| MLOAD / MSTORE | 3 | Memory read/write (+ memory expansion) |
| CALL (cold) | 2,600 | Calling a contract for first time in tx |
| CALL (warm) | 100 | Calling already-accessed contract |
| LOG (per byte) | 8 | Event data |
| Zero byte in calldata | 4 | Lower for calldata zeroes |
| Non-zero byte in calldata | 16 | Higher for calldata non-zeroes |

**Key insight:** Storage is 100-200× more expensive than memory. Cache storage reads.

---

## Storage Optimization {#storage}

### Cache Storage Variables in Memory

```solidity
// ❌ 3 SLOADs in a loop = 3 × 2100 = 6300 gas per iteration (cold)
function badLoop() external {
    for (uint i; i < count; i++) {   // SLOADs `count` each iteration
        total += values[i];
    }
}

// ✅ 1 SLOAD, rest are MLOADs (~3 gas each)
function goodLoop() external {
    uint256 _count = count;          // cache in memory
    uint256 _total;                  // accumulate in memory
    for (uint i; i < _count; i++) {
        _total += values[i];
    }
    total = _total;                  // 1 SSTORE at end
}
```

### Slot Packing

Pack related small variables together to occupy a single 32-byte slot:

```solidity
// ❌ 3 storage slots (3 × 32 bytes)
contract Wasteful {
    uint256 lastUpdated;   // slot 0
    uint8 status;          // slot 1 (wastes 31 bytes!)
    bool active;           // slot 2 (wastes 31 bytes!)
}

// ✅ 1 storage slot — saves ~40k deployment gas, ~10k per state change
contract Packed {
    uint8 status;          // slot 0, byte 0
    bool active;           // slot 0, byte 1
    uint240 lastUpdated;   // slot 0, bytes 2-31 (fills slot)
}
```

**Packing structs in mappings:**
```solidity
struct UserInfo {
    uint128 balance;    // 16 bytes
    uint64  lockEnd;    // 8 bytes
    uint32  tier;       // 4 bytes
    bool    active;     // 1 byte
    // Padding fills to 32 bytes — all in ONE slot!
}
mapping(address => UserInfo) public users;
```

### Zero-to-Nonzero is Expensive

Writing `0 → nonzero` costs ~22,100 gas. Writing `nonzero → nonzero` costs ~5,000.

```solidity
// Prefer 1 as default/empty value to avoid costly zero-to-nonzero writes
// For example, use 1 = inactive, 2 = active (instead of 0 = inactive)
mapping(address => uint256) private _status; // 1 = default (inactive), 2 = active
```

---

## Variable & Type Choices {#variables}

### `constant` and `immutable`

These are FREE to read — no SLOAD at all:

```solidity
uint256 public constant MAX_SUPPLY = 10_000;   // bytecode constant — 0 gas to read
address public immutable owner;                 // set at deploy, then bytecode — 0 gas to read
uint256 public mutableValue;                    // storage — 2100 gas (cold) to read
```

### Use `uint256` as the Default

Counterintuitively, `uint256` is often cheaper than smaller types, because the EVM pads smaller types to 32 bytes during computation:

```solidity
// uint8 used in computation requires masking/padding operations
function bad(uint8 a, uint8 b) external pure returns (uint8) {
    return a + b;  // EVM pads to uint256, adds, truncates back
}

// uint256 maps directly to the EVM word size
function good(uint256 a, uint256 b) external pure returns (uint256) {
    return a + b;  // no conversion needed
}
```

Exception: use smaller types inside structs for packing benefits.

### `calldata` vs `memory`

```solidity
// ❌ Copies calldata to memory — extra gas
function processArray(uint256[] memory data) external pure { ... }

// ✅ Reads directly from calldata — cheaper for read-only params
function processArray(uint256[] calldata data) external pure { ... }
```

### `bytes32` vs `string`

```solidity
// ❌ Dynamic string — expensive storage, complex hashing
string public name = "MyProtocol";

// ✅ Fixed bytes — cheaper reads, direct comparison
bytes32 public constant NAME = "MyProtocol";
```

---

## Function & Call Optimization {#functions}

### External vs Public

```solidity
// external: cheaper when called from outside — no memory copy of args
function externalFn(uint256[] calldata data) external { ... }

// public: callable internally AND externally — args copied to memory
function publicFn(uint256[] memory data) public { ... }

// Use external over public for functions only called externally
```

### Modifiers vs Internal Functions

Modifiers inline their code — repeated use inflates bytecode:

```solidity
// ❌ Modifier — code inlined at every use site, bloats bytecode
modifier onlyOwner() {
    require(msg.sender == owner, "Not owner");
    _;
}

// ✅ Internal function — called once, referenced everywhere
function _checkOwner() internal view {
    if (msg.sender != owner) revert Unauthorized();
}
function criticalA() external { _checkOwner(); ... }
function criticalB() external { _checkOwner(); ... }
```

Rule of thumb: if modifier is used 3+ times, use internal function.

### Batching Calls

```solidity
// ❌ Three separate transactions = 3× base transaction fee (21,000 each)
approve(spender, 100);
transfer(to1, 50);
transfer(to2, 50);

// ✅ Multicall — 1 transaction, 1× base fee
function multicall(bytes[] calldata calls) external returns (bytes[] memory results) {
    results = new bytes[](calls.length);
    for (uint256 i; i < calls.length;) {
        (bool ok, bytes memory result) = address(this).delegatecall(calls[i]);
        if (!ok) revert CallFailed(i);
        results[i] = result;
        unchecked { ++i; }
    }
}
```

### Custom Errors vs Require Strings

```solidity
// ❌ String stored in bytecode — ~20 bytes × 4 gas = ~80 gas + storage
require(amount > 0, "Amount must be positive");

// ✅ Custom error — 4 bytes (selector only) + any parameters
error AmountMustBePositive();
if (amount == 0) revert AmountMustBePositive();
// Can encode dynamic data at essentially zero overhead
error InvalidAmount(uint256 provided);
if (amount < min) revert InvalidAmount(amount);
```

---

## Loop Optimization {#loops}

### Pre-increment vs Post-increment

```solidity
// ❌ Post-increment: creates temporary copy, increments, returns copy
i++

// ✅ Pre-increment: increments and returns directly
++i
```

### `unchecked` for Safe Loop Counters

When you can prove a counter won't overflow (e.g., bounded by array length):

```solidity
// ❌ Overflow check on every iteration (unnecessary for array bounds)
for (uint256 i = 0; i < arr.length; i++) { ... }

// ✅ unchecked saves ~30 gas per iteration
uint256 len = arr.length;  // cache length too
for (uint256 i; i < len;) {
    // do work
    unchecked { ++i; }
}
```

### Mappings Over Arrays for Lookups

```solidity
// ❌ O(n) lookup — must iterate entire array
address[] public whitelist;
function isWhitelisted(address user) external view returns (bool) {
    for (uint i; i < whitelist.length; i++) {
        if (whitelist[i] == user) return true;
    }
    return false;
}

// ✅ O(1) lookup — constant gas regardless of size
mapping(address => bool) public whitelist;
function isWhitelisted(address user) external view returns (bool) {
    return whitelist[user];
}
```

### Bitmaps for Boolean Flags

```solidity
// ❌ 256 separate booleans = 256 storage slots!
mapping(uint256 => bool) public claimed;

// ✅ Bitmap: 256 booleans in ONE storage slot
mapping(uint256 => uint256) private _claimedBitmap;

function setClaimed(uint256 tokenId) internal {
    _claimedBitmap[tokenId / 256] |= (1 << (tokenId % 256));
}

function isClaimed(uint256 tokenId) internal view returns (bool) {
    return (_claimedBitmap[tokenId / 256] >> (tokenId % 256)) & 1 == 1;
}
```

---

## Deployment Optimization {#deployment}

### Compiler Optimizer Settings

```javascript
// hardhat.config.ts
module.exports = {
  solidity: {
    version: "0.8.24",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200,       // 200 = balance deploy cost vs call cost
                         // Low runs (1-10): cheaper to deploy, costly to call
                         // High runs (10000+): cheaper to call, costly to deploy
      },
      viaIR: true,       // Intermediate Representation: more aggressive optimization
    }
  }
};
```

```toml
# foundry.toml
[profile.default]
optimizer = true
optimizer_runs = 200
via_ir = true
```

### Clone Factories (EIP-1167)

Deploying many instances of the same contract? Use minimal proxies:

```solidity
import "@openzeppelin/contracts/proxy/Clones.sol";

contract VaultFactory {
    address public immutable implementation;

    constructor() {
        implementation = address(new Vault());
    }

    function createVault(address owner) external returns (address) {
        address vault = Clones.clone(implementation);
        IVault(vault).initialize(owner);  // init clone
        return vault;
    }
}
// Clone deployment: ~45k gas vs ~400k+ for full contract deployment
```

---

## Yul Assembly Patterns {#yul}

Only use when critical path optimization justifies complexity.

### Efficient Storage Read

```solidity
// Read from a specific storage slot (useful in proxies)
function _getImplementation() internal view returns (address impl) {
    bytes32 slot = _IMPLEMENTATION_SLOT;
    assembly {
        impl := sload(slot)
    }
}
```

### Memory Copy

```solidity
// Copy calldata to memory manually (sometimes faster than Solidity's copy)
assembly {
    let ptr := mload(0x40)           // free memory pointer
    calldatacopy(ptr, 0, calldatasize())
    mstore(0x40, add(ptr, calldatasize()))
}
```

### Short-Circuit Revert (no return data)

```solidity
// Gas-efficient revert for hot paths (saves ~200 gas vs require)
assembly {
    if iszero(condition) {
        revert(0, 0)
    }
}
```

---

## Measurement Tools {#measurement}

### Foundry Gas Reports

```bash
# Run with gas report
forge test --gas-report

# Create a gas snapshot
forge snapshot

# Compare against snapshot
forge snapshot --diff
```

### Hardhat Gas Reporter

```bash
npm install hardhat-gas-reporter
```

```javascript
// hardhat.config.ts
gasReporter: {
  enabled: process.env.REPORT_GAS !== undefined,
  currency: "USD",
  coinmarketcap: process.env.COINMARKETCAP_API_KEY,
}
```

---

## Contract Size Limit (24KB) {#contract-size}

The Spurious Dragon hard fork introduced a maximum deployed bytecode size of **24,576 bytes** (24 KB) per contract. Any contract exceeding this limit will fail to deploy.

### Checking Contract Size

```bash
# Foundry — prints a table of contract sizes
forge build --sizes

# Hardhat — compile with size output
npx hardhat compile
# or add `contractSizer` plugin for detailed reports:
# npm install hardhat-contract-sizer
```

### Common Causes of Bytecode Bloat

| Cause | Why It Bloats | Fix |
|-------|--------------|-----|
| Long `require` strings | Each string is stored in bytecode | Use custom errors |
| Modifier inlining | Modifier code is copied to every call site | Use internal functions instead |
| Large ABI / many functions | Each external function adds dispatch logic | Split into facets or extensions |
| Dead code & unused imports | Compiler may not fully strip them | Remove manually |
| Heavy inheritance chains | Inherited functions compiled into child | Flatten or split |

### Solutions

**1. Split into multiple contracts with delegation:**

```solidity
// ❌ One fat contract that exceeds 24KB
contract MonolithicProtocol {
    // ... hundreds of functions ...
}

// ✅ Base contract with core logic, extensions for the rest
contract ProtocolCore {
    address public immutable extensions;

    constructor(address _extensions) {
        extensions = _extensions;
    }

    function deposit(uint256 amount) external { /* core logic */ }
    function withdraw(uint256 amount) external { /* core logic */ }

    // Delegate non-core operations
    fallback() external payable {
        address ext = extensions;
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), ext, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}

contract ProtocolExtensions {
    function claimRewards(address user) external { /* ... */ }
    function updateSettings(bytes calldata config) external { /* ... */ }
    // ... additional functions that don't fit in core
}
```

**2. Use external libraries to offload bytecode:**

```solidity
// Library functions called via DELEGATECALL — their bytecode lives separately
library MathLib {
    function complexCalculation(uint256 a, uint256 b) external pure returns (uint256) {
        // ... heavy logic here doesn't count toward caller's size
    }
}

contract MyContract {
    using MathLib for uint256;
    // MathLib.complexCalculation is an external call — not inlined
}
```

**3. Compiler settings:**

```toml
# foundry.toml
[profile.default]
optimizer = true
optimizer_runs = 200    # lower runs = smaller bytecode
via_ir = true           # can reduce size 10-30%
```

**4. Internal functions instead of modifiers:**

```solidity
// ❌ Modifier inlines at every use — if used 10 times, 10 copies in bytecode
modifier onlyAdmin() {
    require(msg.sender == admin, "Not admin");
    _;
}

// ✅ Internal function — one copy in bytecode, called via JUMP
function _onlyAdmin() internal view {
    if (msg.sender != admin) revert NotAdmin();
}
```

**5. Diamond pattern (EIP-2535) — last resort for very large protocols:**

When splitting into two contracts is not enough, the Diamond pattern allows unlimited facets (logic contracts) behind a single proxy address.

---

## Transient Storage Patterns (EIP-1153) {#transient-storage}

Transient storage provides **storage that is automatically cleared at the end of each transaction**. It uses the `TSTORE` and `TLOAD` opcodes introduced in the Cancun/Dencun upgrade.

### Cost Comparison

| Operation | Gas Cost | Notes |
|-----------|----------|-------|
| TSTORE | 100 | Write to transient slot |
| TLOAD | 100 | Read from transient slot |
| SSTORE (new, nonzero) | ~22,100 | Write to regular storage (cold, zero→nonzero) |
| SSTORE (modify) | ~5,000 | Write to regular storage (warm) |
| SLOAD (cold) | 2,100 | Read from regular storage (cold) |

### Solidity Syntax (0.8.24+)

```solidity
// Declare a transient variable — cleared automatically after each transaction
uint256 transient private _reentrancyStatus;
```

### Use Case 1: Cheap Reentrancy Lock

```solidity
// ❌ Traditional reentrancy guard — costs ~5,000+ gas per lock/unlock (SSTORE)
contract ExpensiveGuard {
    uint256 private _status = 1; // NOT_ENTERED

    modifier nonReentrant() {
        require(_status != 2, "Reentrant");
        _status = 2;    // SSTORE: ~5,000 gas
        _;
        _status = 1;    // SSTORE: ~2,900 gas (refund applies)
    }
}

// ✅ Transient reentrancy guard — costs ~200 gas total (2 × TSTORE at 100 each)
contract CheapGuard {
    uint256 transient private _status;

    modifier nonReentrant() {
        if (_status != 0) revert ReentrancyDetected();
        _status = 1;    // TSTORE: 100 gas — auto-cleared after tx
        _;
        _status = 0;    // TSTORE: 100 gas
    }
}
// Savings: ~7,700 gas per guarded call
```

### Use Case 2: Callback Validation

```solidity
contract FlashLender {
    address transient private _currentBorrower;

    function flashLoan(address borrower, uint256 amount) external {
        _currentBorrower = borrower;       // TSTORE: set flag before callback
        IERC20(token).transfer(borrower, amount);
        IFlashBorrower(borrower).onFlashLoan(amount);

        // Verify repayment
        require(
            IERC20(token).balanceOf(address(this)) >= amount + fee,
            "Not repaid"
        );
        _currentBorrower = address(0);     // TSTORE: clear flag
    }

    // Only callable during an active flash loan from the current borrower
    function verifyCallback() external view {
        if (_currentBorrower != msg.sender) revert InvalidCallback();
    }
}
```

### Use Case 3: Cross-Function Communication

```solidity
contract Vault {
    bool transient private _permitUsed;

    /// @notice Permit + deposit in a single transaction
    function permitAndDeposit(
        uint256 amount,
        uint256 deadline,
        uint8 v, bytes32 r, bytes32 s
    ) external {
        IERC20Permit(token).permit(msg.sender, address(this), amount, deadline, v, r, s);
        _permitUsed = true;    // TSTORE: signal to deposit logic
        _deposit(msg.sender, amount);
        // _permitUsed auto-cleared at end of tx
    }

    function _deposit(address user, uint256 amount) internal {
        if (_permitUsed) {
            // Skip allowance check — permit was just called
        }
        // ... deposit logic
    }
}
```

### Use Case 4: Temporary Approval Patterns

```solidity
contract TokenWithTransientApproval {
    mapping(address => mapping(address => uint256)) transient private _tempAllowance;

    /// @notice Grant a one-transaction approval that auto-expires
    function approveOnce(address spender, uint256 amount) external {
        _tempAllowance[msg.sender][spender] = amount;  // TSTORE
    }

    function transferFrom(address from, address to, uint256 amount) external {
        uint256 tempAllow = _tempAllowance[from][msg.sender];  // TLOAD
        if (tempAllow >= amount) {
            _tempAllowance[from][msg.sender] = tempAllow - amount;
        } else {
            // Fall back to persistent allowance
            _spendAllowance(from, msg.sender, amount);
        }
        _transfer(from, to, amount);
    }
}
```

### When NOT to Use Transient Storage

- **Any data that must persist across transactions** — transient storage is wiped clean after every tx
- **Cross-contract reads in separate transactions** — the data will not be there
- **State that needs to survive `SELFDESTRUCT` within the same tx** — behavior is undefined

### Compatibility

Transient storage requires the **Cancun/Dencun** upgrade. As of 2025:
- **Supported:** Ethereum mainnet, Arbitrum, Optimism, Base, Polygon zkEVM, most major L2s
- **Check before deploying** on newer or niche chains — verify `TSTORE`/`TLOAD` opcode support

---

## Events as Cheap Storage {#events-as-storage}

Events (LOG opcodes) provide a way to record data on-chain at a fraction of the cost of storage, with the trade-off that **event data is not readable by smart contracts** — only by offchain services.

### Cost Comparison

| Method | Gas Cost | Readable On-Chain? |
|--------|----------|--------------------|
| SSTORE (new nonzero) | ~22,100 | Yes |
| SSTORE (modify) | ~5,000 | Yes |
| LOG0 (no topics) | 375 base + 8/byte | No |
| LOG1 (1 topic) | 750 base + 8/byte | No |
| LOG2 (2 topics) | 1,125 base + 8/byte | No |

A typical event with 2 indexed parameters and 64 bytes of data costs roughly **~1,600 gas** — compared to **22,100+** for equivalent storage writes.

### When to Use Events Over Storage

| Use Case | Store On-Chain? | Emit Event? |
|----------|----------------|-------------|
| Active order that contracts must read | Yes | Optional |
| Historical trade record for analytics | No | Yes |
| Audit trail of admin actions | No | Yes |
| Configuration change log | No | Yes |
| User activity history | No | Yes |
| Data needed by frontend only | No | Yes |

### Pattern: Minimal Storage + Rich Events

```solidity
contract OrderBook {
    // On-chain: only store what contracts need to verify
    mapping(bytes32 => bool) public activeOrderHashes;
    uint256 public orderCount;

    event OrderPlaced(
        bytes32 indexed orderHash,
        address indexed maker,
        address indexed tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 amountOut,
        uint256 expiry,
        uint256 timestamp
    );

    event OrderCancelled(bytes32 indexed orderHash, address indexed maker);
    event OrderFilled(bytes32 indexed orderHash, address indexed taker, uint256 fillAmount);

    struct Order {
        address maker;
        address tokenIn;
        address tokenOut;
        uint256 amountIn;
        uint256 amountOut;
        uint256 expiry;
    }

    function placeOrder(Order calldata order) external returns (bytes32 orderHash) {
        orderHash = keccak256(abi.encode(order, orderCount++));

        // ✅ Store only the hash on-chain — 1 SSTORE (~22,100 gas)
        activeOrderHashes[orderHash] = true;

        // ✅ Emit full details as event — ~1,600 gas (vs ~110,000+ to store all fields)
        emit OrderPlaced(
            orderHash,
            order.maker,
            order.tokenIn,
            order.tokenOut,
            order.amountIn,
            order.amountOut,
            order.expiry,
            block.timestamp
        );
    }

    function cancelOrder(Order calldata order, uint256 nonce) external {
        bytes32 orderHash = keccak256(abi.encode(order, nonce));
        require(activeOrderHashes[orderHash], "Not active");
        require(order.maker == msg.sender, "Not maker");

        delete activeOrderHashes[orderHash];    // SSTORE: refund ~4,800 gas
        emit OrderCancelled(orderHash, msg.sender);
    }

    function fillOrder(Order calldata order, uint256 nonce, uint256 fillAmount) external {
        bytes32 orderHash = keccak256(abi.encode(order, nonce));
        require(activeOrderHashes[orderHash], "Not active");

        // ... execute swap logic ...

        delete activeOrderHashes[orderHash];
        emit OrderFilled(orderHash, msg.sender, fillAmount);
    }
}
```

**Gas savings:** storing all 7 order fields would cost ~110,000+ gas (5 storage slots). Storing only the hash + emitting an event costs ~23,700 gas — an **~80% reduction**.

### Offchain Indexing with The Graph

Events emitted on-chain can be indexed by [The Graph](https://thegraph.com/) subgraphs, making them queryable by frontends:

```yaml
# subgraph.yaml — event handler mapping
eventHandlers:
  - event: OrderPlaced(indexed bytes32, indexed address, indexed address, address, uint256, uint256, uint256, uint256)
    handler: handleOrderPlaced
  - event: OrderCancelled(indexed bytes32, indexed address)
    handler: handleOrderCancelled
  - event: OrderFilled(indexed bytes32, indexed address, uint256)
    handler: handleOrderFilled
```

```typescript
// mapping.ts — The Graph handler
export function handleOrderPlaced(event: OrderPlaced): void {
    let order = new OrderEntity(event.params.orderHash.toHex());
    order.maker = event.params.maker;
    order.tokenIn = event.params.tokenIn;
    order.tokenOut = event.params.tokenOut;
    order.amountIn = event.params.amountIn;
    order.amountOut = event.params.amountOut;
    order.expiry = event.params.expiry;
    order.timestamp = event.params.timestamp;
    order.status = "ACTIVE";
    order.save();
}
```

Other indexing services (Dune Analytics, Ponder, Envio) can similarly consume these events for dashboards and analytics.

---

## Quick Reference Table {#reference-table}

| Technique | Gas Saving | Complexity |
|-----------|-----------|------------|
| `constant`/`immutable` over state var | ~2000/read | Low |
| Cache storage in memory | ~2000/extra read | Low |
| Slot packing | ~10k-40k | Low |
| `calldata` instead of `memory` | ~200-500/param | Low |
| Custom errors | ~40-200/error | Low |
| `++i` instead of `i++` | ~5/iteration | Low |
| `unchecked` loop counter | ~30/iteration | Low |
| Mapping over array lookup | O(1) vs O(n) | Low |
| Bitmap for booleans | up to 256× storage | Medium |
| Clone factory | ~350k per deploy | Medium |
| Batch transactions | 21k base fee × n-1 | Medium |
| `via_ir` compiler | 5-30% runtime | Zero |
| Yul assembly | Variable (10-50%) | High |
| Contract splitting | Avoids 24KB limit | Medium |
| Transient storage | ~22,000/lock operation | Low |
| Events over storage | ~5,000-22,000/write | Low |