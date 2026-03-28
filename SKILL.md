---
name: solidity-evm
description: >
  Expert-level guidance for writing, testing, auditing, deploying, and optimizing Solidity smart contracts on EVM-compatible chains (Ethereum, Base, Arbitrum, Optimism, Polygon, etc.).
  Use this skill whenever the user is: writing or reviewing Solidity code, building smart contracts (tokens, DeFi, NFTs, DAOs, access control, staking), asking about ERC standards (ERC-20/721/1155/4626/2981), setting up Foundry or Hardhat, writing contract tests or fuzz tests, deploying onchain, working with proxy/upgradeable contracts, asking about gas optimization, debugging transaction reverts or test failures, or auditing contracts for security vulnerabilities.
  Trigger even for casual phrasing like "my contract keeps reverting", "how do I write a staking contract", "what is reentrancy", "what's the best way to deploy", "how do I upgrade my contract", "deploy on Base/Arbitrum/L2", or "bridge tokens cross-chain".
---

# Solidity & EVM Smart Contract Skill

You are an expert Solidity engineer and smart contract security specialist. You write clean, gas-efficient, well-tested, audit-ready contracts.

## How to use this skill

This SKILL.md provides principles, patterns, and rules. For deep reference material, read the relevant file from `references/`:

| File | When to read it |
|------|----------------|
| `references/security.md` | Vulnerability patterns, attack vectors, ABI encoding pitfalls, MEV protection, audit checklists |
| `references/gas-optimization.md` | Storage layout, opcodes, optimization techniques, contract size limits, transient storage |
| `references/patterns.md` | Design patterns: proxy/upgradeable, factory, access control, libraries, interfaces, CREATE2, OZ v5 migration |
| `references/tooling.md` | Foundry & Hardhat setup, test writing, deployment scripts, CI, contract verification, multi-chain deploy |
| `references/erc-standards.md` | ERC-20, ERC-721, ERC-1155, ERC-4626, ERC-2981, ERC-4337, EIP-712, Permit2, token integration checklist |
| `references/l2-crosschain.md` | L2 gotchas, cross-chain messaging, bridge patterns, multi-chain deployment, L2 gas models |
| `references/evm-internals.md` | EVM architecture, data locations, ABI encoding, DELEGATECALL, CREATE/CREATE2, opcodes, bytecode |
| `references/natspec-quality.md` | NatSpec documentation, Solidity style guide, code review checklist, SWC vulnerability IDs |

Read the relevant reference **before** generating code for any non-trivial task. For tasks touching security, proxy patterns, or unfamiliar ERCs, always read the reference.

---

## Core Philosophy

Smart contracts handle real money and are immutable once deployed. Approach every contract with:

1. **Security first** — a functional but exploitable contract is worse than a broken one
2. **Simplicity as defense** — complexity is the enemy of auditability
3. **Tests as specification** — a contract without tests is not production-ready
4. **Gas as a UX metric** — unnecessarily expensive contracts hurt users
5. **Upgradeability is a trade-off** — not a default. Immutable contracts can be fine

---

## Solidity Version & Pragmas

Always use a recent, specific Solidity version. As of 2025, `^0.8.24` or `^0.8.26` are solid choices.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;
```

- Avoid `pragma solidity ^0.8.0` (too broad — misses later safety features)
- Avoid floating pragmas (`>=0.8.0 <0.9.0`) in final contracts; float only in libraries
- `0.8.x+` includes built-in overflow/underflow protection — SafeMath is no longer needed
- Use `0.8.20+` for ERC-7201 namespaced storage (upgradeable contracts)
- Use `0.8.24+` for transient storage (`TSTORE`/`TLOAD`), named mapping parameters, and `using { fn } for Type global`
- Use `0.8.26+` for custom storage layout with user-defined value types

---

## Contract Structure Template

Follow this ordering consistently:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// 1. Imports
import "@openzeppelin/contracts/access/Ownable.sol";

// 2. Errors (custom errors — cheaper than strings)
error Unauthorized(address caller);
error InsufficientBalance(uint256 available, uint256 required);

// 3. Interfaces (if not imported)

// 4. Libraries

// 5. Contract declaration
contract MyContract is Ownable {
    // 6. Type declarations (structs, enums)
    enum Status { Active, Paused, Closed }

    // 7. State variables
    uint256 public constant MAX_SUPPLY = 10_000;     // constant first
    address public immutable treasury;               // then immutable
    uint256 private _totalSupply;                    // then regular state

    // 8. Events
    event Deposited(address indexed user, uint256 amount);

    // 9. Modifiers
    modifier nonZero(uint256 value) {
        if (value == 0) revert InsufficientBalance(0, 1);
        _;
    }

    // 10. Constructor
    constructor(address _treasury) Ownable(msg.sender) {
        treasury = _treasury;
    }

    // 11. Receive / fallback (if needed)
    receive() external payable {}

    // 12. External functions
    // 13. Public functions
    // 14. Internal functions
    // 15. Private functions
    // 16. View/pure functions (external/public/internal)
}
```

---

## The Golden Rules of Solidity Security

### 1. Checks-Effects-Interactions (CEI)

This is the most important pattern. Always validate → update state → make external calls.

```solidity
// ❌ WRONG — vulnerable to reentrancy
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount);
    (bool ok,) = msg.sender.call{value: amount}("");   // external call FIRST
    require(ok);
    balances[msg.sender] -= amount;                     // state update LAST
}

// ✅ CORRECT — CEI pattern
function withdraw(uint256 amount) external {
    // CHECKS
    if (balances[msg.sender] < amount) revert InsufficientBalance(balances[msg.sender], amount);
    // EFFECTS
    balances[msg.sender] -= amount;
    // INTERACTIONS
    (bool ok,) = msg.sender.call{value: amount}("");
    if (!ok) revert TransferFailed();
}
```

### 2. ReentrancyGuard for complex flows

When CEI alone isn't enough (multi-contract interactions, ERC-777, etc.), add a guard:

```solidity
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract Vault is ReentrancyGuard {
    function withdraw(uint256 amount) external nonReentrant {
        // ...
    }
}
```

### 3. Access Control

Use explicit, layered access control. Prefer OpenZeppelin's battle-tested contracts.

```solidity
// Simple ownership
import "@openzeppelin/contracts/access/Ownable.sol";

// Role-based (preferred for multi-role systems)
import "@openzeppelin/contracts/access/AccessControl.sol";

bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");

function mint(address to, uint256 amount) external onlyRole(MINTER_ROLE) {
    _mint(to, amount);
}
```

### 4. Custom Errors (not strings)

Custom errors save ~50% gas vs `require("string")` and are more informative.

```solidity
// ❌ Old way — expensive, loses data
require(amount > 0, "Amount must be positive");

// ✅ New way — cheap, structured, debuggable
error AmountMustBePositive();
error InvalidAmount(uint256 provided, uint256 minimum);

if (amount == 0) revert AmountMustBePositive();
if (amount < minAmount) revert InvalidAmount(amount, minAmount);
```

### 5. Immutability where possible

Prefer `constant` and `immutable` over regular state variables for values that don't change.

```solidity
uint256 public constant FEE_BPS = 300;      // compile-time constant (FREE to read)
address public immutable factory;            // set once at deploy (cheap to read)
uint256 public totalSupply;                  // regular storage (expensive: ~2100 gas SLOAD)
```

---

## Storage Layout & Slot Packing

The EVM stores state in 32-byte (256-bit) slots. Understand this or waste gas.

```solidity
// ❌ Wastes 3 full slots (3 × 32 bytes = 96 bytes)
contract Wasteful {
    uint256 a;  // slot 0
    uint8 b;    // slot 1  ← wastes 31 bytes
    bool c;     // slot 2  ← wastes 31 bytes
}

// ✅ Packs into 1 slot (saves ~40k deployment gas, ~5k per write)
contract Packed {
    uint8 b;    // slot 0, bytes 0-0
    bool c;     // slot 0, bytes 1-1
    uint240 a;  // slot 0, bytes 2-31  ← fills the slot
}
```

Pack variables that are often read/written together. Separate variables accessed independently.

---

## Events: Best Practices

```solidity
// Always index address fields and key identifiers (up to 3 indexed params)
event Transfer(address indexed from, address indexed to, uint256 value);
event OrderFilled(bytes32 indexed orderId, address indexed maker, uint256 price);

// Emit events for ALL state changes — they're the contract's audit log
// and the only way offchain indexers (like The Graph) can track state
```

---

## Sending ETH

Never use `transfer()` or `send()` — they have hardcoded 2300 gas stipend and can break with EIP changes.

```solidity
// ❌ Legacy — breaks if recipient is a contract with complex receive()
payable(to).transfer(amount);

// ✅ Modern — always check return value
(bool success,) = payable(to).call{value: amount}("");
if (!success) revert ETHTransferFailed();
```

---

## Key Anti-Patterns to Avoid

| Anti-pattern | Problem | Fix |
|---|---|---|
| `tx.origin` for auth | Phishing attacks | Use `msg.sender` |
| Floating pragma in contracts | Unpredictable compiler | Pin the version |
| Unbounded loops | DoS via gas exhaustion | Cap loops or use pull-over-push |
| `block.timestamp` for precision | Miner manipulation (±15s) | Use block numbers or Chainlink |
| Divide-before-multiply | Precision loss | Multiply first, divide last |
| Missing zero-address checks | Locked funds/ownership | Validate `address != address(0)` |
| Unchecked low-level call return | Silent failures | Always check `success` |
| `delegatecall` to untrusted contracts | Storage corruption | Never delegate to user input |

---

## Interacting With External Contracts

Treat every external call as a potential attack vector.

```solidity
// Use interfaces — never call raw addresses
IERC20 token = IERC20(tokenAddress);

// Validate return values
bool success = token.transfer(to, amount);
if (!success) revert TransferFailed();

// For tokens that don't return bool (USDT etc.), use SafeERC20
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
using SafeERC20 for IERC20;
token.safeTransfer(to, amount);  // handles non-standard tokens
```

---

## Assembly (Yul) — When and When Not

**When to use:**
- High-frequency tight loops where gas savings are significant
- Memory layout manipulation (e.g., custom storage patterns in proxies)
- Bit manipulation operations
- Reading returndata or custom slot access

**When NOT to use:**
- Anywhere Solidity's high-level constructs are sufficient
- Security-critical code paths (assembly bypasses all safety checks)
- Code that will be maintained by others without Yul knowledge

Always comment assembly blocks extensively.

---

## NatSpec Documentation

Every public/external function and every contract MUST have NatSpec in production code. NatSpec powers `forge doc`, Etherscan display, and is critical for auditors.

```solidity
/// @title Vault — ETH deposit and withdrawal contract
/// @author MyTeam
/// @notice User-facing description of the contract
/// @dev Implementation notes for developers
contract Vault {
    /// @notice Deposit ETH into the vault
    /// @dev Emits {Deposited} and updates user balance
    /// @param receiver The address to credit the deposit to
    /// @return shares The number of vault shares minted
    function deposit(address receiver) external payable returns (uint256 shares) {
        // ...
    }

    /// @notice Withdraw ETH from the vault
    /// @param amount The amount of ETH to withdraw
    /// @custom:security nonReentrant
    function withdraw(uint256 amount) external {
        // ...
    }
}
```

**Tags:** `@title`, `@author`, `@notice` (user-facing), `@dev` (developer), `@param`, `@return`, `@inheritdoc`, `@custom:*`

---

## ABI Encoding Safety

Use `abi.encodeCall` (Solidity 0.8.11+) for type-safe low-level calls. Avoid `abi.encodePacked` with multiple dynamic types — it creates hash collisions.

```solidity
// ❌ Hash collision — abi.encodePacked("ab", "c") == abi.encodePacked("a", "bc")
bytes32 hash = keccak256(abi.encodePacked(name, symbol));

// ✅ Safe — abi.encode adds length prefixes
bytes32 hash = keccak256(abi.encode(name, symbol));

// ❌ No type checking on args
bytes memory data = abi.encodeWithSelector(IVault.deposit.selector, amount);

// ✅ Compiler checks arg types match the function signature
bytes memory data = abi.encodeCall(IVault.deposit, (amount));
```

See `references/security.md` → ABI Encoding Pitfalls for full patterns.

---

## Receive & Fallback Functions

```solidity
// receive() — called when msg.data is EMPTY and ETH is sent
receive() external payable {
    emit Received(msg.sender, msg.value);
}

// fallback() — called when no function selector matches, or msg.data non-empty with no receive()
fallback() external payable {
    _delegate(implementation);  // proxy pattern
}
```

**Key rules:**
- Never leave `receive()` empty (`receive() external payable {}`) unless intentional — it silently swallows ETH
- Never leave `fallback()` empty unless it's a proxy — it masks bugs by silently accepting calls to non-existent functions
- If a contract should NOT receive ETH, omit both `receive()` and `fallback()` (or make them revert)

---

## Low-Level Call Error Handling

Always decode revert reasons from failed low-level calls:

```solidity
(bool ok, bytes memory returnData) = target.call(data);
if (!ok) {
    // Bubble up the revert reason
    if (returnData.length > 0) {
        assembly { revert(add(32, returnData), mload(returnData)) }
    }
    revert CallFailed();
}
```

For external calls that may fail gracefully, use `try/catch`:

```solidity
try oracle.latestRoundData() returns (uint80, int256 price, uint256, uint256 updatedAt, uint80) {
    if (price <= 0) revert InvalidPrice();
    return uint256(price);
} catch {
    revert OracleCallFailed();
}
```

---

## Solidity 0.8.24+ Features

Key features available in modern Solidity that you should use:

| Feature | Version | Use |
|---------|---------|-----|
| Transient storage (`TSTORE`/`TLOAD`) | 0.8.24 | Cheap reentrancy guards, callback flags |
| Named parameters in mappings | 0.8.18 | `mapping(address user => uint256 balance)` |
| User-defined value types | 0.8.8 | `type TokenAmount is uint256;` — type safety |
| `using { fn } for Type global` | 0.8.24 | File-level function attachment |
| `bytes.concat()` | 0.8.4 | Replace `abi.encodePacked` for byte concatenation |
| `abi.encodeCall` | 0.8.11 | Type-safe encoding for low-level calls |

---

## Contract Size Awareness

EVM enforces a 24,576-byte limit on deployed bytecode (Spurious Dragon, EIP-170). Check with `forge build --sizes`. If your contract is approaching the limit:

1. Use custom errors instead of require strings
2. Replace modifiers with internal functions (modifiers inline at every use site)
3. Split into base + extension contracts
4. Enable `viaIR` in compiler settings
5. Use external libraries for heavy logic
6. Last resort: Diamond pattern (EIP-2535)

See `references/gas-optimization.md` → Contract Size Limit for details.

---

## Deployment Checklist

Before deploying any contract to mainnet, verify:

- [ ] Comprehensive test suite passes (unit + integration + fuzz)
- [ ] All functions have correct visibility (`external` vs `public` vs `internal`)
- [ ] Access control is explicitly set on every privileged function
- [ ] Events are emitted for all state changes
- [ ] No hardcoded addresses (except constants like address(0))
- [ ] Constructor validates all inputs (no zero-address, no zero-value where required)
- [ ] Reentrancy guards on functions with external calls
- [ ] `constant`/`immutable` used wherever values don't change
- [ ] Custom errors used instead of string messages
- [ ] NatSpec on all public/external functions
- [ ] Contract size under 24KB (`forge build --sizes`)
- [ ] Slither static analysis run and findings addressed
- [ ] Contract verified on block explorer (Etherscan/Basescan/Arbiscan)
- [ ] If upgradeable: storage layout verified, initializer set, `_disableInitializers()` called
- [ ] If L2: read `references/l2-crosschain.md` for chain-specific gotchas

---

## When to Read Reference Files

- **Writing a token (ERC-20/721/1155)?** → Read `references/erc-standards.md`
- **Setting up Foundry or Hardhat?** → Read `references/tooling.md`
- **Security issue, audit, reentrancy, oracle manipulation, MEV?** → Read `references/security.md`
- **Proxy, UUPS, upgradeable, Diamond, libraries, interfaces?** → Read `references/patterns.md`
- **Trying to reduce gas costs or hitting contract size limit?** → Read `references/gas-optimization.md`
- **Deploying to L2, cross-chain messaging, bridging?** → Read `references/l2-crosschain.md`
- **Deploying to multiple chains, verifying contracts?** → Read `references/tooling.md` (verification & multi-chain sections)
- **Accepting arbitrary ERC-20 tokens (USDT, USDC, fee-on-transfer)?** → Read `references/erc-standards.md` (token integration checklist)
- **Using OpenZeppelin v5 (or migrating from v4)?** → Read `references/patterns.md` (OZ v5 migration section)
- **Need to understand EVM internals, storage slots, DELEGATECALL, opcodes?** → Read `references/evm-internals.md`
- **Writing NatSpec, code review, style guide, SWC IDs?** → Read `references/natspec-quality.md`

When generating a non-trivial contract (>50 lines), always at minimum skim the relevant reference.