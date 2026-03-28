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
9. [Quick Reference Table](#reference-table)

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