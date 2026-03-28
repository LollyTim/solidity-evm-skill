# EVM Internals Reference

Understanding what happens under the hood makes you a dramatically better Solidity developer. Gas optimization, proxy patterns, ABI encoding, and assembly all become intuitive once you understand the EVM model.

## Table of Contents
1. [EVM Architecture Overview](#architecture)
2. [Data Locations: Stack, Memory, Storage, Calldata](#data-locations)
3. [ABI Encoding & Decoding](#abi)
4. [How Function Calls Work](#calls)
5. [DELEGATECALL Deep Dive](#delegatecall)
6. [Contract Creation (CREATE vs CREATE2)](#create)
7. [The Solidity Compiler Output](#compiler)
8. [Key Opcodes Reference](#opcodes)
9. [How Solidity Maps to Bytecode](#mapping)

---

## EVM Architecture Overview {#architecture}

The EVM is a **stack-based**, **deterministic**, **sandboxed** virtual machine. Every node in the network runs the same bytecode and arrives at the same state.

```
┌─────────────────────────────────────────────────┐
│                  EVM Execution Context           │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Stack   │  │  Memory  │  │   Storage    │  │
│  │          │  │          │  │              │  │
│  │ 1024     │  │ Byte     │  │ Persistent   │  │
│  │ slots    │  │ array    │  │ key-value    │  │
│  │ 32 bytes │  │ (temp)   │  │ (permanent)  │  │
│  │ each     │  │          │  │              │  │
│  └──────────┘  └──────────┘  └──────────────┘  │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Calldata │  │   Code   │  │   Logs       │  │
│  │          │  │          │  │ (write-only) │  │
│  │ Input    │  │ Bytecode │  │              │  │
│  │ (immut.) │  │ (immut.) │  │              │  │
│  └──────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────────────────────────────┘
```

**Key properties:**
- **Deterministic** — same input → same output, always, everywhere
- **Isolated** — contracts can only affect each other via explicit calls
- **Word size** — 32 bytes (256 bits). Every stack slot, every storage slot is 32 bytes
- **Big-endian** — most significant byte first

---

## Data Locations: Stack, Memory, Storage, Calldata {#data-locations}

### Stack
- Max 1024 slots, each 32 bytes
- All EVM operations read from and write to the stack
- Extremely cheap: ~2-3 gas per operation
- You can't reference arbitrary stack positions — only TOP items (DUP, SWAP)
- Overflowing 1024 causes `Stack too deep` error

### Memory
- Byte-addressable temporary array, cleared after each call
- Grows as needed — but growth has quadratic cost
- Cheap for small sizes, expensive for large allocations
- Solidity uses memory for: function arguments, return values, dynamic types, `abi.encode`

```
Memory layout (Solidity convention):
0x00 - 0x3f : scratch space for hashing
0x40 - 0x5f : free memory pointer (crucial!)
0x60 - 0x7f : zero slot (never written to)
0x80+       : allocated memory begins here
```

**Free memory pointer (0x40):** Always check this before writing to memory manually. `mload(0x40)` = next free memory address.

```solidity
// In assembly, always respect the free memory pointer
assembly {
    let ptr := mload(0x40)           // get free memory pointer
    mstore(ptr, value)               // write at free memory
    mstore(0x40, add(ptr, 0x20))    // advance pointer by 32 bytes
}
```

### Storage
- Persistent across transactions — this is the blockchain state
- 2^256 slots of 32 bytes each (effectively infinite)
- **Extremely expensive**: 22,100 gas (new write), 5,000 gas (update), 2,100 gas (cold read)
- Slot positions determined by **declaration order**

```solidity
contract Example {
    uint256 a;    // slot 0
    uint256 b;    // slot 1
    mapping(address => uint256) c;  // slot 2 (keys hash to: keccak256(key . slot))
    uint256[] d;  // slot 3 (length stored here; elements at keccak256(3) + index)
}
```

**Storage slot computation:**
```solidity
// Simple variable: uses its declared slot directly
// slot(a) = 0

// Mapping: keccak256(abi.encode(key, slot))
// slot(c[userAddress]) = keccak256(abi.encode(userAddress, 2))

// Dynamic array: length at base slot; element i at keccak256(baseSlot) + i
// slot(d.length) = 3
// slot(d[i]) = keccak256(abi.encode(3)) + i
```

### Calldata
- Read-only input data for external/public functions
- Cheaper than memory — no copy needed
- First 4 bytes = function selector (keccak256 of signature, first 4 bytes)
- Remaining bytes = ABI-encoded arguments

---

## ABI Encoding & Decoding {#abi}

The ABI (Application Binary Interface) defines how function calls and return values are encoded as bytes.

### Function Selector
```solidity
// Selector = first 4 bytes of keccak256 of the canonical function signature
bytes4 selector = bytes4(keccak256("transfer(address,uint256)"));
// = 0xa9059cbb

// In calldata: [selector (4 bytes)][encoded args...]
```

### Encoding Rules

**Static types** (fixed size — encoded in-place):
- `uint256`, `int256`, `bool`, `address`, `bytes32`, fixed-size arrays
- Always padded to 32 bytes

```
transfer(address to, uint256 amount) called with (0xAbCd..., 1000):

Calldata:
0xa9059cbb                                              ← selector
000000000000000000000000AbCd...AbCd...AbCd...AbCd      ← address (padded left to 32 bytes)
00000000000000000000000000000000000000000000000000000000000003E8  ← 1000 in hex
```

**Dynamic types** (variable size — uses offset pointers):
- `string`, `bytes`, dynamic arrays `uint256[]`, `bytes[]`
- Encoded as: offset pointer → length → data

```solidity
// abi.encode vs abi.encodePacked
abi.encode(uint256(1), uint256(2))         // 64 bytes: full 32-byte padding
abi.encodePacked(uint256(1), uint256(2))   // 64 bytes: same here, but for smaller types:
abi.encodePacked(uint8(1), uint8(2))       // 2 bytes: no padding ← hash collision risk!

// NEVER use abi.encodePacked with dynamic types for signatures/hashing
// Use abi.encode instead to avoid hash collisions
bytes32 safe   = keccak256(abi.encode(str1, str2));
bytes32 unsafe = keccak256(abi.encodePacked(str1, str2));  // "a"+"bc" == "ab"+"c"
```

### Decoding in Solidity
```solidity
// Decode ABI-encoded data
(uint256 a, address b) = abi.decode(data, (uint256, address));

// Decode function call (strips selector)
(uint256 amount, address to) = abi.decode(msg.data[4:], (uint256, address));
```

---

## How Function Calls Work {#calls}

### CALL vs STATICCALL vs DELEGATECALL vs CALLCODE

| Opcode | Context (msg.sender, storage) | Use case |
|--------|-------------------------------|----------|
| `CALL` | Caller's sender, callee's storage | Normal external call |
| `STATICCALL` | Caller's sender, callee's storage (read-only) | View function calls |
| `DELEGATECALL` | **Caller's sender AND storage** | Proxy contracts |
| `CALLCODE` | Deprecated — use DELEGATECALL | Never use |

### Internal vs External Calls

```solidity
contract A {
    function internalFn() internal { ... }      // JUMP opcode — no new context
    function externalFn() external { ... }      // requires CALL — new context

    function example() public {
        internalFn();           // cheap: just a JUMP in bytecode
        this.externalFn();      // expensive: full external CALL (2600+ gas)
    }
}
```

### Low-Level Call Semantics

```solidity
// All low-level calls return (bool success, bytes memory returndata)
// They NEVER revert on failure — you MUST check the return value

(bool ok, bytes memory data) = target.call{value: 1 ether, gas: 50000}(
    abi.encodeWithSignature("foo(uint256)", 42)
);
if (!ok) revert CallFailed();

// call       — regular call, can send ETH, can change state
// staticcall — read-only, reverts if callee tries to write state
// delegatecall — executes in caller's context (see below)
```

---

## DELEGATECALL Deep Dive {#delegatecall}

This is the single most important opcode to understand for proxy contracts.

```
Normal CALL:
msg.sender = A
A calls B { B uses B's storage }

DELEGATECALL:
msg.sender = A (preserved!)
A calls B via delegatecall { B's CODE runs in A's STORAGE context }
```

```solidity
// Proxy.sol — stores state, delegates logic
contract Proxy {
    address public implementation;  // slot 0
    uint256 public value;           // slot 1

    fallback() external payable {
        address impl = implementation;
        assembly {
            // Copy calldata to memory
            calldatacopy(0, 0, calldatasize())
            // delegatecall: run impl's code with THIS contract's storage
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            // Copy return data
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}

// Implementation.sol — has the logic, NO state of its own
contract Implementation {
    address public implementation;  // slot 0 — MUST match Proxy layout!
    uint256 public value;           // slot 1 — MUST match Proxy layout!

    function setValue(uint256 newValue) external {
        value = newValue;  // writes to PROXY's slot 1, not Implementation's
    }
}
```

**Storage collision danger:**
```solidity
// Proxy has:    slot 0 = implementation address
// Impl has:     slot 0 = some other variable
// → impl overwrites the implementation address → catastrophic

// Solution: ERC-1967 pseudo-random storage slots
// bytes32(uint256(keccak256('eip1967.proxy.implementation')) - 1)
// = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc
// This slot is astronomically unlikely to collide with normal variable layout
```

---

## Contract Creation (CREATE vs CREATE2) {#create}

### CREATE
```
address = keccak256(rlp([deployer_address, deployer_nonce]))[12:]
```
Address depends on deployer's nonce — unpredictable without knowing current nonce.

### CREATE2
```
address = keccak256(0xff ++ deployer_address ++ salt ++ keccak256(init_code))[12:]
```
Address is **deterministic** — predictable before deployment.

```solidity
// Predict CREATE2 address before deployment
address predicted = address(uint160(uint256(keccak256(abi.encodePacked(
    bytes1(0xff),
    address(this),  // deployer
    salt,           // your chosen bytes32
    keccak256(type(MyContract).creationCode)
)))));

// Deploy at predicted address
MyContract deployed = new MyContract{salt: salt}();
assert(address(deployed) == predicted);
```

**Why CREATE2 matters:**
- Counterfactual instantiation — user can send funds to an address before the contract exists
- Deterministic wallet addresses (e.g., Safe multisigs)
- Clone factories where addresses must be predictable
- State channels — contracts deployed only when dispute arises

---

## The Solidity Compiler Output {#compiler}

When you compile Solidity, you get:

**Bytecode** = creation code + runtime code
- **Creation code**: constructor logic, deploys runtime code, runs once
- **Runtime code**: what actually gets stored on-chain, called on every tx

**ABI** = JSON description of all external/public functions, events, errors

```bash
# With solc
solc --bin --abi MyContract.sol

# With Foundry
forge build
cat out/MyContract.sol/MyContract.json | jq '.bytecode.object'  # creation bytecode
cat out/MyContract.sol/MyContract.json | jq '.deployedBytecode.object'  # runtime bytecode
cat out/MyContract.sol/MyContract.json | jq '.abi'  # ABI
```

**Inspecting bytecode:**
```bash
# Disassemble runtime bytecode
cast disasm $(cat out/MyContract.sol/MyContract.json | jq -r '.deployedBytecode.object')

# Get function selector
cast sig "transfer(address,uint256)"    # 0xa9059cbb

# Decode a transaction's calldata
cast calldata-decode "transfer(address,uint256)" 0xa9059cbb000...
```

---

## Key Opcodes Reference {#opcodes}

### Stack Operations
| Opcode | Gas | Description |
|--------|-----|-------------|
| PUSH1..PUSH32 | 3 | Push N bytes onto stack |
| POP | 2 | Remove top of stack |
| DUP1..DUP16 | 3 | Duplicate stack item |
| SWAP1..SWAP16 | 3 | Swap stack items |

### Arithmetic
| Opcode | Gas | Description |
|--------|-----|-------------|
| ADD, SUB, MUL | 3 | Basic arithmetic |
| DIV, MOD | 5 | Division/modulo |
| EXP | 10+ | Exponentiation (expensive!) |
| MULMOD, ADDMOD | 8 | Modular arithmetic |

### Memory
| Opcode | Gas | Description |
|--------|-----|-------------|
| MLOAD | 3+ | Load 32 bytes from memory |
| MSTORE | 3+ | Store 32 bytes to memory |
| MSTORE8 | 3+ | Store 1 byte to memory |
| MSIZE | 2 | Get memory size |

### Storage
| Opcode | Gas | Description |
|--------|-----|-------------|
| SLOAD | 2100 (cold) / 100 (warm) | Load from storage |
| SSTORE | 22100 (new) / 5000 (update) | Store to storage |
| TLOAD | 100 | Transient storage load (EIP-1153) |
| TSTORE | 100 | Transient storage store (EIP-1153) |

### Control Flow
| Opcode | Gas | Description |
|--------|-----|-------------|
| JUMP | 8 | Unconditional jump |
| JUMPI | 10 | Conditional jump |
| REVERT | 0 | Revert execution |
| RETURN | 0 | Return from execution |
| STOP | 0 | Stop execution |

### Call Operations
| Opcode | Gas | Description |
|--------|-----|-------------|
| CALL | 2600+ | Call another contract |
| STATICCALL | 2600+ | Read-only call |
| DELEGATECALL | 2600+ | Call in caller's context |
| CREATE | 32000+ | Deploy new contract |
| CREATE2 | 32000+ | Deploy with deterministic address |

### Hashing & Crypto
| Opcode | Gas | Description |
|--------|-----|-------------|
| SHA3 (KECCAK256) | 30 + 6/word | Hash memory range |
| ECRECOVER | 3000 | Recover signer from signature |

---

## How Solidity Maps to Bytecode {#mapping}

### A simple function
```solidity
function add(uint256 a, uint256 b) external pure returns (uint256) {
    return a + b;
}
```

Compiles roughly to:
```
// Function dispatcher (checks selector)
PUSH4 0x771602f7    // add(uint256,uint256) selector
EQ
JUMPI add_fn        // jump if selector matches

// add_fn:
CALLDATALOAD 0x04   // load first arg (after 4-byte selector)
CALLDATALOAD 0x24   // load second arg
ADD                 // add them
PUSH1 0x00
MSTORE              // store result in memory
PUSH1 0x20          // return 32 bytes
PUSH1 0x00
RETURN
```

### Modifiers become inlined code
```solidity
modifier onlyOwner() {
    require(msg.sender == owner);
    _;
}

// The bytecode of onlyOwner is copied INLINE at every call site
// Using an internal function is more efficient for code size
```

### Events are LOG opcodes
```solidity
emit Transfer(from, to, amount);
// Becomes:
// LOG3(data, topic0=keccak256("Transfer(address,address,uint256)"),
//           topic1=from, topic2=to)
// data = abi.encode(amount)
// Topics are indexed params; non-indexed go in data
```

### Mappings don't exist at runtime
```solidity
mapping(address => uint256) balances;
// balances[user] compiles to:
// slot = keccak256(abi.encode(user, 0))  // 0 = mapping's storage slot
// SLOAD(slot)
```

### Understanding this helps you:
- Debug `Stack too deep` errors — split complex functions
- Understand why `this.fn()` is expensive — it's a full external CALL
- Know why internal functions are cheaper than external ones
- Grasp why events can't be read on-chain — they're LOG opcodes, not storage
- Explain why `constant` variables have zero read cost — they're in bytecode