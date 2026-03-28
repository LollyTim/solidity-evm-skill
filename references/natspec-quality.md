# NatSpec Documentation & Code Quality Reference

Professional Solidity contracts are fully documented. Auditors read NatSpec. Developers integrating your contract rely on it. Undocumented contracts are harder to audit, harder to integrate, and signal amateur work.

## Table of Contents
1. [NatSpec Format](#natspec)
2. [Complete Contract Documentation Example](#example)
3. [Solidity Style Guide](#style)
4. [Code Review Checklist](#review)
5. [SWC Registry — Vulnerability IDs](#swc)

---

## NatSpec Format {#natspec}

NatSpec (Natural Language Specification) is the official Solidity documentation standard. It compiles into the ABI's `devdoc` and `userdoc` fields and is read by tools like Etherscan, Remix, and audit firms.

### Tags Reference

| Tag | Scope | Purpose |
|-----|-------|---------|
| `@title` | Contract | One-line contract description |
| `@author` | Contract | Author name or contact |
| `@notice` | Contract, Function, Event, Error | User-facing explanation (what it does) |
| `@dev` | Contract, Function, Event, Error | Developer notes (how it works, edge cases) |
| `@param` | Function, Event, Error | Describe a parameter |
| `@return` | Function | Describe return value(s) |
| `@inheritdoc` | Function | Inherit docs from interface/base |
| `@custom:...` | Anything | Custom tags (e.g., `@custom:security-contact`) |

### When to use `///` vs `/** */`

```solidity
// Single-line NatSpec — use /// for single tags
/// @notice Returns the token balance of an account
function balanceOf(address account) external view returns (uint256);

// Multi-line NatSpec — use /** */ for multiple tags
/**
 * @notice Transfers tokens from the caller to a recipient
 * @dev Emits a {Transfer} event. Reverts if balance is insufficient.
 * @param to The recipient address
 * @param amount The number of tokens to transfer
 * @return success True if the transfer succeeded
 */
function transfer(address to, uint256 amount) external returns (bool success);
```

---

## Complete Contract Documentation Example {#example}

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable2Step.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title StakingVault
 * @author LollyTech <dev@lollytech.xyz>
 * @notice A staking contract that distributes ERC-20 rewards to token stakers
 * @dev Implements a reward-per-token accumulator pattern.
 *      Rewards accrue continuously and are claimed on withdraw or explicit claim.
 *      Security contact: security@lollytech.xyz
 * @custom:security-contact security@lollytech.xyz
 */
contract StakingVault is Ownable2Step, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // =========================================================================
    // Errors
    // =========================================================================

    /// @notice Thrown when attempting to stake zero tokens
    error ZeroAmount();

    /// @notice Thrown when a user has no pending rewards to claim
    error NoRewardPending();

    /// @notice Thrown when the new reward rate would overflow
    /// @param rate The invalid rate that was provided
    error InvalidRewardRate(uint256 rate);

    // =========================================================================
    // Events
    // =========================================================================

    /**
     * @notice Emitted when a user stakes tokens
     * @param user The address of the staker
     * @param amount The number of tokens staked
     */
    event Staked(address indexed user, uint256 amount);

    /**
     * @notice Emitted when a user withdraws staked tokens
     * @param user The address of the withdrawer
     * @param amount The number of tokens withdrawn
     */
    event Withdrawn(address indexed user, uint256 amount);

    /**
     * @notice Emitted when a user claims their pending rewards
     * @param user The address of the claimer
     * @param reward The amount of reward tokens claimed
     */
    event RewardClaimed(address indexed user, uint256 reward);

    /**
     * @notice Emitted when the owner sets a new reward rate
     * @param oldRate The previous reward rate (tokens per second)
     * @param newRate The new reward rate (tokens per second)
     */
    event RewardRateUpdated(uint256 oldRate, uint256 newRate);

    // =========================================================================
    // State Variables
    // =========================================================================

    /// @notice The ERC-20 token accepted for staking
    IERC20 public immutable stakingToken;

    /// @notice The ERC-20 token distributed as rewards
    IERC20 public immutable rewardToken;

    /// @notice Reward tokens emitted per second across all stakers
    uint256 public rewardRate;

    /// @notice Timestamp of the last global reward update
    uint256 public lastUpdateTime;

    /// @notice Accumulated reward per token, scaled by 1e18
    /// @dev Increases monotonically as rewards accrue
    uint256 public rewardPerTokenStored;

    /// @notice Total tokens currently staked across all users
    uint256 public totalStaked;

    /// @dev Snapshot of rewardPerTokenStored at the time of last user action
    mapping(address => uint256) private _userRewardPerTokenPaid;

    /// @dev Pending (unclaimed) rewards for each user
    mapping(address => uint256) private _pendingRewards;

    /// @notice Amount staked by each user
    mapping(address => uint256) public stakedBalance;

    // =========================================================================
    // Constructor
    // =========================================================================

    /**
     * @notice Deploys the staking vault
     * @param _stakingToken Address of the token users will stake
     * @param _rewardToken Address of the token distributed as rewards
     * @param initialOwner Address that will own the contract
     */
    constructor(
        IERC20 _stakingToken,
        IERC20 _rewardToken,
        address initialOwner
    ) Ownable(initialOwner) {
        if (address(_stakingToken) == address(0)) revert ZeroAmount();
        if (address(_rewardToken) == address(0)) revert ZeroAmount();
        stakingToken = _stakingToken;
        rewardToken = _rewardToken;
    }

    // =========================================================================
    // External Functions
    // =========================================================================

    /**
     * @notice Stake tokens to begin earning rewards
     * @dev Caller must have approved this contract to spend `amount` staking tokens.
     *      Rewards are snapshotted before the balance change.
     * @param amount The number of staking tokens to deposit
     */
    function stake(uint256 amount) external nonReentrant updateReward(msg.sender) {
        if (amount == 0) revert ZeroAmount();
        totalStaked += amount;
        stakedBalance[msg.sender] += amount;
        stakingToken.safeTransferFrom(msg.sender, address(this), amount);
        emit Staked(msg.sender, amount);
    }

    /**
     * @notice Withdraw staked tokens
     * @dev Pending rewards are preserved and not automatically claimed.
     *      Call {claimReward} separately to receive them.
     * @param amount The number of staking tokens to withdraw
     */
    function withdraw(uint256 amount) external nonReentrant updateReward(msg.sender) {
        if (amount == 0) revert ZeroAmount();
        totalStaked -= amount;
        stakedBalance[msg.sender] -= amount;
        stakingToken.safeTransfer(msg.sender, amount);
        emit Withdrawn(msg.sender, amount);
    }

    /**
     * @notice Claim all pending rewards
     * @dev Reverts if the caller has no pending rewards.
     *      Safe against reentrancy via the {nonReentrant} modifier.
     */
    function claimReward() external nonReentrant updateReward(msg.sender) {
        uint256 reward = _pendingRewards[msg.sender];
        if (reward == 0) revert NoRewardPending();
        _pendingRewards[msg.sender] = 0;
        rewardToken.safeTransfer(msg.sender, reward);
        emit RewardClaimed(msg.sender, reward);
    }

    // =========================================================================
    // View Functions
    // =========================================================================

    /**
     * @notice Returns the current reward per token, including unaccounted accrual
     * @return The reward per staked token, scaled by 1e18
     */
    function rewardPerToken() public view returns (uint256) {
        if (totalStaked == 0) return rewardPerTokenStored;
        return rewardPerTokenStored +
            (rewardRate * (block.timestamp - lastUpdateTime) * 1e18) / totalStaked;
    }

    /**
     * @notice Returns the total pending (unclaimed) rewards for an account
     * @param account The address to query
     * @return The amount of reward tokens the account can claim
     */
    function earned(address account) public view returns (uint256) {
        return (stakedBalance[account] *
            (rewardPerToken() - _userRewardPerTokenPaid[account])) / 1e18
            + _pendingRewards[account];
    }

    // =========================================================================
    // Owner Functions
    // =========================================================================

    /**
     * @notice Update the reward emission rate
     * @dev The reward token contract must have sufficient balance to sustain
     *      the new rate. Existing rewards are settled before changing the rate.
     * @param newRate New reward tokens emitted per second (18 decimals scale)
     */
    function setRewardRate(uint256 newRate) external onlyOwner updateReward(address(0)) {
        emit RewardRateUpdated(rewardRate, newRate);
        rewardRate = newRate;
    }

    // =========================================================================
    // Modifiers
    // =========================================================================

    /**
     * @dev Settles global and per-user reward state before any balance change.
     *      Must be applied to all functions that read or modify staked balances.
     * @param account The user whose reward state should be updated (address(0) = global only)
     */
    modifier updateReward(address account) {
        rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = block.timestamp;
        if (account != address(0)) {
            _pendingRewards[account] = earned(account);
            _userRewardPerTokenPaid[account] = rewardPerTokenStored;
        }
        _;
    }
}
```

---

## Solidity Style Guide {#style}

Follow the [official Solidity style guide](https://docs.soliditylang.org/en/latest/style-guide.html) plus these team-standard conventions:

### Naming Conventions

```solidity
// Contracts: PascalCase
contract StakingVault { }

// Functions and variables: camelCase
function getBalance() external view returns (uint256 balance) { }
uint256 public totalSupply;

// Constants and immutables: SCREAMING_SNAKE_CASE
uint256 public constant MAX_SUPPLY = 10_000;
address public immutable FACTORY;

// Private/internal storage: underscore prefix
uint256 private _locked;
mapping(address => uint256) internal _balances;

// Function parameters: _underscorePrefix (optional but common)
function transfer(address _to, uint256 _amount) external { }

// Events: PascalCase
event Transfer(address indexed from, address indexed to, uint256 value);

// Errors: PascalCase, descriptive
error InsufficientBalance(uint256 available, uint256 required);

// Interfaces: IPascalCase prefix
interface IStakingVault { }

// Abstract contracts: AbstractPascalCase (common pattern)
abstract contract ERC20Base { }
```

### Function Order (within a contract)

```solidity
contract Example {
    // 1. Type declarations (structs, enums)
    // 2. State variables (constants → immutables → regular)
    // 3. Events
    // 4. Errors
    // 5. Modifiers
    // 6. Constructor
    // 7. receive() / fallback()
    // 8. External functions
    // 9. Public functions
    // 10. Internal functions
    // 11. Private functions
    // 12. External/public view and pure functions
    // 13. Internal/private view and pure functions
}
```

### Number Formatting

```solidity
// Use underscores for readability
uint256 constant MAX_SUPPLY = 10_000_000;       // not 10000000
uint256 constant YEAR_IN_SECONDS = 365 days;    // Solidity time units
uint256 constant FEE = 0.003 ether;             // Solidity ether units (= 3e15 wei)
uint256 constant BPS = 10_000;                  // basis points
```

### Import Organization

```solidity
// 1. OpenZeppelin / external dependencies
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
// 2. Local interfaces
import "./interfaces/IVault.sol";
// 3. Local libraries
import "./libraries/Math.sol";
// 4. Local contracts
import "./base/BaseVault.sol";
```

---

## Code Review Checklist {#review}

Use this before submitting code for audit or merging PRs.

### Logic & Correctness
- [ ] Every code path has been considered (zero values, max values, empty arrays)
- [ ] Arithmetic is either protected by 0.8.x or explicitly `unchecked` with justification
- [ ] No divide-before-multiply precision loss
- [ ] State machines transition correctly — no invalid state reachable
- [ ] Return values from external calls are checked
- [ ] Functions revert with descriptive custom errors (not generic reverts)

### Documentation
- [ ] Every `external` and `public` function has `@notice` and `@dev` NatSpec
- [ ] All `@param` tags are present and accurate
- [ ] All `@return` tags are present
- [ ] All events are documented with `@notice` and `@param`
- [ ] All custom errors are documented with `@notice`
- [ ] Complex math or invariants are explained in `@dev` comments
- [ ] `@custom:security-contact` is set on the contract

### Security
- [ ] CEI pattern followed in all external-call functions
- [ ] `nonReentrant` on complex flows
- [ ] All privileged functions have access modifiers
- [ ] No `tx.origin` for auth
- [ ] No spot AMM prices for pricing
- [ ] Constructor validates all inputs (no zero addresses where critical)
- [ ] If upgradeable: `_disableInitializers()` in implementation constructor

### Gas
- [ ] Storage variables cached in memory when read multiple times
- [ ] Related small variables are slot-packed
- [ ] `constant`/`immutable` used for non-changing values
- [ ] `calldata` used instead of `memory` for read-only external function params
- [ ] Custom errors used instead of require strings

### Testing
- [ ] All critical functions have unit tests
- [ ] All revert conditions are tested
- [ ] All events are tested
- [ ] Fuzz tests for arithmetic-heavy functions
- [ ] Invariant tests for conservation properties (supply, balances)
- [ ] Tests run against a mainnet fork for integrations

---

## SWC Registry — Vulnerability IDs {#swc}

The Smart Contract Weakness Classification (SWC) Registry is the industry standard for referencing vulnerability types in audit reports. Knowing these by ID signals professionalism.

| SWC | Name | Key Concern |
|-----|------|-------------|
| SWC-100 | Function Default Visibility | Functions default to public — always specify visibility |
| SWC-101 | Integer Overflow/Underflow | Use Solidity 0.8.x or SafeMath |
| SWC-102 | Outdated Compiler | Use recent Solidity version |
| SWC-103 | Floating Pragma | Pin compiler version in contracts |
| SWC-104 | Unchecked Return Value | Always check `(bool ok,)` from low-level calls |
| SWC-105 | Unprotected Ether Withdrawal | Access control on withdrawal functions |
| SWC-106 | Unprotected SELFDESTRUCT | Never expose selfdestruct without auth |
| SWC-107 | Reentrancy | CEI pattern + ReentrancyGuard |
| SWC-108 | State Variable Default Visibility | Always specify `private`/`internal`/`public` |
| SWC-109 | Uninitialized Storage Pointer | Solidity 0.8+ largely eliminates this |
| SWC-110 | Assert Violation | Use `assert` for invariants only, `require` for inputs |
| SWC-111 | Use of Deprecated Functions | No `suicide()`, `sha3()`, `throw` |
| SWC-112 | Delegatecall to Untrusted Callee | Never delegatecall to user-supplied address |
| SWC-113 | DoS with Failed Call | Pull-over-push pattern |
| SWC-114 | Transaction Order Dependence | Commit-reveal for sensitive operations |
| SWC-115 | Authorization Through tx.origin | Use `msg.sender` for auth |
| SWC-116 | Block values as Proxy for Time | Use `block.timestamp`, not `block.number` for time |
| SWC-120 | Weak Sources of Randomness | Use Chainlink VRF, not block values |
| SWC-122 | Lack of Proper Signature Verification | Verify signer, not just signature existence |
| SWC-123 | Requirement Violation | Validate all invariants |
| SWC-124 | Write to Arbitrary Storage | Never write to user-supplied storage slot |
| SWC-125 | Incorrect Inheritance Order | Understand C3 linearization |
| SWC-127 | Arbitrary Jump with Function Type | Avoid function pointers from untrusted input |
| SWC-128 | DoS With Block Gas Limit | Cap loop iterations, use pull-over-push |
| SWC-129 | Typographical Error | Review for `=+` vs `+=`, `||` vs `&&` |
| SWC-131 | Presence of Unused Variables | Remove dead code |
| SWC-132 | Unexpected Ether Balance | Don't rely on `address(this).balance` for logic |
| SWC-134 | Message Call with Hardcoded Gas | Don't hardcode `.gas(2300)` |
| SWC-135 | Code With No Effects | Remove dead code and no-op operations |
| SWC-136 | Unencrypted Private Data On-Chain | Never store secrets in private variables |