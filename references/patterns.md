# Solidity Design Patterns Reference

## Table of Contents
1. [Proxy & Upgradeable Contracts](#proxy)
2. [Factory Pattern](#factory)
3. [Access Control Patterns](#access-control)
4. [Pull-Over-Push / Withdrawal Pattern](#pull-payment)
5. [State Machine Pattern](#state-machine)
6. [Guard Check Pattern](#guard-check)
7. [Oracle Pattern](#oracle)
8. [Multisig & Governance](#governance)
9. [Diamond / EIP-2535](#diamond)
10. [Common DeFi Patterns](#defi)
11. [OpenZeppelin v5 Migration Notes](#oz-v5)
12. [Library Patterns](#libraries)
13. [Interface & Abstract Contract Patterns](#interfaces)
14. [Deterministic Deployment (CREATE2)](#create2)

---

## Proxy & Upgradeable Contracts {#proxy}

Smart contracts are immutable by default. Proxy patterns allow upgrading logic while keeping state.

### How Proxies Work

```
User → Proxy (state, address) ──delegatecall──→ Implementation (logic only)
```

`delegatecall` executes the implementation's code **in the proxy's storage context**. The implementation writes to the proxy's storage, not its own.

### Proxy Pattern Comparison

| Pattern | Gas (Calls) | Upgrade Auth | Recommendation |
|---------|------------|--------------|----------------|
| Transparent Proxy (TPP) | Higher (loads admin slot every call) | ProxyAdmin contract | Legacy; fine for established protocols |
| UUPS (ERC-1822) | Lower (no per-call admin check) | Logic contract | **Recommended for new projects** |
| Beacon Proxy | Medium | Single beacon upgrades all | Multiple same-logic proxies |
| Diamond (EIP-2535) | Higher (routing overhead) | Facets | Very large/modular contracts |

### UUPS Upgradeable Contract (Recommended)

```solidity
// Implementation contract
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

contract MyProtocolV1 is Initializable, UUPSUpgradeable, OwnableUpgradeable {
    // State variables — layout is sacred, never reorder/remove
    uint256 public totalSupply;
    mapping(address => uint256) public balances;

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();  // prevent implementation contract from being initialized
    }

    function initialize(address initialOwner) external initializer {
        __Ownable_init(initialOwner);
        __UUPSUpgradeable_init();
    }

    // Business logic here...

    function _authorizeUpgrade(address newImpl) internal override onlyOwner {}
}
```

```solidity
// V2 — ONLY append new variables, never modify existing layout
contract MyProtocolV2 is MyProtocolV1 {
    uint256 public feeRate;         // ✅ new variable APPENDED
    // uint256 public totalSupply;  // ❌ NEVER change existing variable types/order
}
```

**Deployment with Foundry:**
```solidity
// script/Deploy.s.sol
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";

contract DeployScript is Script {
    function run() external {
        vm.startBroadcast();
        address proxy = Upgrades.deployUUPSProxy(
            "MyProtocolV1.sol:MyProtocolV1",
            abi.encodeCall(MyProtocolV1.initialize, (msg.sender))
        );
        vm.stopBroadcast();
    }
}
```

**Upgrading:**
```solidity
function upgradeScript() external {
    vm.startBroadcast();
    Upgrades.upgradeProxy(
        proxyAddress,
        "MyProtocolV2.sol:MyProtocolV2",
        ""  // optional reinitialization calldata
    );
    vm.stopBroadcast();
}
```

### Storage Layout Safety Rules

1. **NEVER** remove state variables
2. **NEVER** change variable types
3. **NEVER** reorder existing variables
4. **ALWAYS** append new variables at the end
5. Use **ERC-7201 namespaced storage** for complex cases

```solidity
// ERC-7201 namespaced storage (avoids collisions in complex inheritance)
/// @custom:storage-location erc7201:myprotocol.main
struct MainStorage {
    uint256 totalSupply;
    mapping(address => uint256) balances;
}

bytes32 private constant MAIN_STORAGE_LOCATION =
    keccak256(abi.encode(uint256(keccak256("myprotocol.main")) - 1)) & ~bytes32(uint256(0xff));

function _getMainStorage() private pure returns (MainStorage storage $) {
    assembly { $.slot := MAIN_STORAGE_LOCATION }
}
```

---

## Factory Pattern {#factory}

Deploy multiple contract instances efficiently.

### Standard Factory

```solidity
contract VaultFactory {
    address[] public vaults;
    mapping(address => address) public ownerToVault;

    event VaultCreated(address indexed owner, address indexed vault);

    function createVault() external returns (address vault) {
        if (ownerToVault[msg.sender] != address(0)) revert VaultExists();
        vault = address(new Vault(msg.sender));  // deploy with owner
        vaults.push(vault);
        ownerToVault[msg.sender] = vault;
        emit VaultCreated(msg.sender, vault);
    }
}
```

### Clone Factory (EIP-1167 — Gas Efficient)

```solidity
import "@openzeppelin/contracts/proxy/Clones.sol";

contract VaultFactory {
    address public immutable implementation;

    constructor() {
        implementation = address(new Vault());
    }

    // Deterministic address (CREATE2) — address predictable before deploy
    function createVaultDeterministic(bytes32 salt) external returns (address vault) {
        vault = Clones.cloneDeterministic(implementation, salt);
        IVault(vault).initialize(msg.sender);
        emit VaultCreated(msg.sender, vault);
    }

    // Predict address before deploy
    function predictVaultAddress(bytes32 salt) external view returns (address) {
        return Clones.predictDeterministicAddress(implementation, salt);
    }
}
// Savings: ~45k gas vs ~400k+ gas for full contract deploy
```

---

## Access Control Patterns {#access-control}

### Ownable (Single Owner)

```solidity
import "@openzeppelin/contracts/access/Ownable2Step.sol";  // 2-step is safer

contract MyContract is Ownable2Step {
    constructor(address initialOwner) Ownable(initialOwner) {}

    function sensitiveOperation() external onlyOwner { ... }
}
// transferOwnership() proposes; new owner must call acceptOwnership()
```

### Role-Based Access Control (AccessControl)

```solidity
import "@openzeppelin/contracts/access/AccessControl.sol";

contract Protocol is AccessControl {
    bytes32 public constant OPERATOR_ROLE = keccak256("OPERATOR_ROLE");
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");
    bytes32 public constant UPGRADER_ROLE = keccak256("UPGRADER_ROLE");

    constructor(address admin) {
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        // Note: admin can grant/revoke all roles
    }

    function pause() external onlyRole(PAUSER_ROLE) { ... }
    function unpause() external onlyRole(PAUSER_ROLE) { ... }
    function executeAction() external onlyRole(OPERATOR_ROLE) { ... }
}
```

### Pausable

```solidity
import "@openzeppelin/contracts/utils/Pausable.sol";

contract Protocol is Ownable, Pausable {
    function deposit(uint256 amount) external whenNotPaused { ... }
    function withdraw(uint256 amount) external { ... }  // allow withdrawal even when paused

    function pause() external onlyOwner { _pause(); }
    function unpause() external onlyOwner { _unpause(); }
}
```

### Emergency Circuit Breaker (Custom)

```solidity
contract EmergencyStop {
    bool public stopped = false;
    address public guardian;

    modifier stopInEmergency() {
        if (stopped) revert EmergencyActive();
        _;
    }

    modifier onlyInEmergency() {
        if (!stopped) revert NotInEmergency();
        _;
    }

    function triggerEmergency() external {
        if (msg.sender != guardian) revert Unauthorized();
        stopped = true;
        emit EmergencyTriggered(msg.sender);
    }
}
```

---

## Pull-Over-Push / Withdrawal Pattern {#pull-payment}

Never push funds — let users pull their own:

```solidity
// ❌ PUSH — dangerous: one bad recipient breaks all, DoS risk
function distributeRewards() external {
    for (uint i; i < users.length; i++) {
        payable(users[i]).transfer(rewards[users[i]]);  // if one reverts, ALL revert
    }
}

// ✅ PULL — each user claims independently
mapping(address => uint256) public pendingRewards;

function accrueReward(address user, uint256 amount) internal {
    pendingRewards[user] += amount;
    emit RewardAccrued(user, amount);
}

function claimReward() external nonReentrant {
    uint256 amount = pendingRewards[msg.sender];
    if (amount == 0) revert NoRewardPending();
    pendingRewards[msg.sender] = 0;  // EFFECTS first
    (bool ok,) = msg.sender.call{value: amount}("");  // then INTERACTION
    if (!ok) revert TransferFailed();
    emit RewardClaimed(msg.sender, amount);
}
```

---

## State Machine Pattern {#state-machine}

Model contracts with distinct phases using enums:

```solidity
enum AuctionState { NotStarted, Active, Ended, Claimed }

contract Auction {
    AuctionState public state;
    uint256 public endTime;

    modifier inState(AuctionState required) {
        if (state != required) revert InvalidState(state, required);
        _;
    }

    function startAuction(uint256 duration) external onlyOwner inState(AuctionState.NotStarted) {
        endTime = block.timestamp + duration;
        state = AuctionState.Active;
        emit AuctionStarted(endTime);
    }

    function bid() external payable inState(AuctionState.Active) {
        if (block.timestamp >= endTime) revert AuctionExpired();
        // ... bidding logic
    }

    function endAuction() external inState(AuctionState.Active) {
        if (block.timestamp < endTime) revert AuctionNotOver();
        state = AuctionState.Ended;
        emit AuctionEnded(highestBidder, highestBid);
    }
}
```

---

## Guard Check Pattern {#guard-check}

Structure validation with `require`/`revert` for clarity:

```solidity
// Separate validation into a dedicated modifier or function
modifier validTransfer(address to, uint256 amount) {
    if (to == address(0)) revert ZeroAddress();
    if (amount == 0) revert ZeroAmount();
    if (balances[msg.sender] < amount) revert InsufficientBalance(balances[msg.sender], amount);
    _;
}

function transfer(address to, uint256 amount) external validTransfer(to, amount) {
    balances[msg.sender] -= amount;
    balances[to] += amount;
    emit Transfer(msg.sender, to, amount);
}
```

---

## Oracle Pattern {#oracle}

### Chainlink Price Feed

```solidity
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract PriceConsumer {
    AggregatorV3Interface public immutable priceFeed;
    uint256 public constant STALENESS_THRESHOLD = 1 hours;

    constructor(address _priceFeed) {
        priceFeed = AggregatorV3Interface(_priceFeed);
    }

    function getPrice() public view returns (uint256 price, uint8 decimals) {
        (
            uint80 roundId,
            int256 answer,
            ,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = priceFeed.latestRoundData();

        // Validate freshness and validity
        if (updatedAt == 0 || answeredInRound < roundId) revert IncompleteRound();
        if (block.timestamp - updatedAt > STALENESS_THRESHOLD) revert StalePrice(updatedAt);
        if (answer <= 0) revert InvalidPrice(answer);

        return (uint256(answer), priceFeed.decimals());
    }
}
```

---

## Multisig & Governance {#governance}

### Timelock Controller

```solidity
import "@openzeppelin/contracts/governance/TimelockController.sol";

// Minimum delay before any admin action executes (e.g., 48 hours)
// Gives users time to exit if they disagree
uint256 minDelay = 48 hours;
address[] memory proposers = [multisig];
address[] memory executors = [multisig];
TimelockController timelock = new TimelockController(minDelay, proposers, executors, admin);
```

### Governor (On-chain Voting)

```solidity
import "@openzeppelin/contracts/governance/Governor.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorSettings.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorCountingSimple.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorVotes.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorTimelockControl.sol";

contract ProtocolGovernor is Governor, GovernorSettings, GovernorCountingSimple, GovernorVotes, GovernorTimelockControl {
    constructor(IVotes _token, TimelockController _timelock)
        Governor("ProtocolGovernor")
        GovernorSettings(7200 /* voting delay */, 50400 /* voting period */, 1e18 /* proposal threshold */)
        GovernorVotes(_token)
        GovernorTimelockControl(_timelock)
    {}
    // ... override resolution functions
}
```

---

## Diamond / EIP-2535 {#diamond}

For contracts that need more functionality than the 24KB max contract size allows, or that need very modular upgradeability.

**Use when:**
- Contract exceeds 24KB size limit
- You need per-function upgradeability granularity
- Protocol has many feature modules that evolve independently

**Key concepts:**
- `Diamond` — proxy contract; delegates to facets
- `Facets` — implementation contracts (each can be upgraded independently)
- `DiamondCut` — upgrade mechanism; adds/replaces/removes function selectors
- `DiamondLoupe` — introspection; see what facets are registered

```solidity
// Each facet handles a subset of functions
contract TokenFacet {
    function transfer(address to, uint256 amount) external { ... }
    function approve(address spender, uint256 amount) external { ... }
}

contract StakingFacet {
    function stake(uint256 amount) external { ... }
    function unstake(uint256 amount) external { ... }
}
```

**Warning:** Diamonds are complex. Unless you truly need per-facet upgradeability, UUPS is simpler and safer.

---

## Common DeFi Patterns {#defi}

### ERC-4626 Tokenized Vault (Yield-Bearing Token)

```solidity
import "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";

contract YieldVault is ERC4626 {
    constructor(IERC20 asset) ERC4626(asset) ERC20("Vault Share", "vSHARE") {}

    function totalAssets() public view override returns (uint256) {
        return IERC20(asset()).balanceOf(address(this)) + _pendingYield();
    }

    // User deposits asset → receives shares
    // deposit(assets, receiver) → shares
    // withdraw(assets, receiver, owner) → burns shares

    function _pendingYield() internal view returns (uint256) {
        // calculate accrued yield from strategy
    }
}
```

### Staking Contract

```solidity
contract StakingRewards is ReentrancyGuard, Pausable, Ownable {
    IERC20 public immutable stakingToken;
    IERC20 public immutable rewardToken;

    uint256 public rewardRate;
    uint256 public lastUpdateTime;
    uint256 public rewardPerTokenStored;

    mapping(address => uint256) public userRewardPerTokenPaid;
    mapping(address => uint256) public rewards;
    mapping(address => uint256) public balances;
    uint256 public totalSupply;

    modifier updateReward(address account) {
        rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = block.timestamp;
        if (account != address(0)) {
            rewards[account] = earned(account);
            userRewardPerTokenPaid[account] = rewardPerTokenStored;
        }
        _;
    }

    function rewardPerToken() public view returns (uint256) {
        if (totalSupply == 0) return rewardPerTokenStored;
        return rewardPerTokenStored +
            (rewardRate * (block.timestamp - lastUpdateTime) * 1e18) / totalSupply;
    }

    function earned(address account) public view returns (uint256) {
        return (balances[account] * (rewardPerToken() - userRewardPerTokenPaid[account])) / 1e18
            + rewards[account];
    }

    function stake(uint256 amount) external nonReentrant whenNotPaused updateReward(msg.sender) {
        if (amount == 0) revert ZeroAmount();
        totalSupply += amount;
        balances[msg.sender] += amount;
        stakingToken.safeTransferFrom(msg.sender, address(this), amount);
        emit Staked(msg.sender, amount);
    }

    function getReward() external nonReentrant updateReward(msg.sender) {
        uint256 reward = rewards[msg.sender];
        if (reward == 0) revert NoReward();
        rewards[msg.sender] = 0;
        rewardToken.safeTransfer(msg.sender, reward);
        emit RewardPaid(msg.sender, reward);
    }
}
```

### Vesting Contract

```solidity
contract VestingVault {
    struct VestingSchedule {
        uint64  startTime;
        uint64  cliffDuration;
        uint64  vestingDuration;
        uint128 totalAmount;
        uint128 released;
    }

    mapping(address => VestingSchedule) public schedules;

    function releasable(address beneficiary) public view returns (uint256) {
        VestingSchedule memory s = schedules[beneficiary];
        if (block.timestamp < s.startTime + s.cliffDuration) return 0;
        uint256 elapsed = block.timestamp - s.startTime;
        uint256 vested = (uint256(s.totalAmount) * elapsed) / s.vestingDuration;
        if (vested > s.totalAmount) vested = s.totalAmount;
        return vested - s.released;
    }

    function release() external {
        uint256 amount = releasable(msg.sender);
        if (amount == 0) revert NothingToRelease();
        schedules[msg.sender].released += uint128(amount);
        token.safeTransfer(msg.sender, amount);
        emit Released(msg.sender, amount);
    }
}

---

## OpenZeppelin v5 Migration Notes {#oz-v5}

OpenZeppelin Contracts v5 (released alongside Solidity 0.8.20+) introduced significant breaking changes. This section covers the most common migration pitfalls.

### Constructor-Based Initialization

In OZ v4, `Ownable` defaulted to `msg.sender`. In v5, you **must** pass the initial owner explicitly:

```solidity
// OZ v4 — implicit owner
contract MyToken is ERC20, Ownable {
    constructor() ERC20("Token", "TKN") {}
    // msg.sender is automatically the owner
}

// OZ v5 — explicit owner required
contract MyToken is ERC20, Ownable {
    constructor(address initialOwner)
        ERC20("Token", "TKN")
        Ownable(initialOwner)
    {}
}
```

### Role Setup Changes

`_setupRole` was removed. Use `_grantRole` directly in constructors:

```solidity
// OZ v4
constructor() {
    _setupRole(DEFAULT_ADMIN_ROLE, msg.sender);
    _setupRole(MINTER_ROLE, msg.sender);
}

// OZ v5
constructor(address admin) {
    _grantRole(DEFAULT_ADMIN_ROLE, admin);
    _grantRole(MINTER_ROLE, admin);
}
```

### ERC20 Minting

`ERC20._mint` is no longer exposed as a public/external override target. Use the internal `_mint` in your constructor or privileged functions:

```solidity
// OZ v5 — internal _mint in constructor
contract MyToken is ERC20, Ownable {
    constructor(address initialOwner)
        ERC20("Token", "TKN")
        Ownable(initialOwner)
    {
        _mint(initialOwner, 1_000_000 * 10 ** decimals());
    }

    function mint(address to, uint256 amount) external onlyOwner {
        _mint(to, amount);
    }
}
```

### Token Transfer Hooks: `_update` Replaces `_beforeTokenTransfer` / `_afterTokenTransfer`

OZ v5 consolidates the two hooks into a single `_update` function:

```solidity
// OZ v4
function _beforeTokenTransfer(address from, address to, uint256 amount) internal override {
    super._beforeTokenTransfer(from, to, amount);
    require(!paused(), "token transfer while paused");
}

// OZ v5 — use _update instead
function _update(address from, address to, uint256 value) internal override {
    if (paused()) revert EnforcedPause();
    super._update(from, to, value);
}
```

For ERC721, `_update` also replaces `_beforeTokenTransfer` and `_afterTokenTransfer`. The `_safeMint` safety checks are now handled internally:

```solidity
// OZ v5 ERC721 — override _update for custom logic
contract MyNFT is ERC721, ERC721Enumerable, Ownable {
    constructor(address initialOwner)
        ERC721("MyNFT", "MNFT")
        Ownable(initialOwner)
    {}

    function _update(address to, uint256 tokenId, address auth)
        internal
        override(ERC721, ERC721Enumerable)
        returns (address)
    {
        return super._update(to, tokenId, auth);
    }

    function _increaseBalance(address account, uint128 value)
        internal
        override(ERC721, ERC721Enumerable)
    {
        super._increaseBalance(account, value);
    }

    function supportsInterface(bytes4 interfaceId)
        public
        view
        override(ERC721, ERC721Enumerable)
        returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
}
```

### Counters Removed

`Counters.sol` was removed entirely. Use a plain `uint256`:

```solidity
// OZ v4
using Counters for Counters.Counter;
Counters.Counter private _tokenIds;

function mint() external {
    _tokenIds.increment();
    uint256 newId = _tokenIds.current();
    _safeMint(msg.sender, newId);
}

// OZ v5 — plain uint256
uint256 private _nextTokenId;

function mint() external {
    uint256 tokenId = _nextTokenId++;
    _safeMint(msg.sender, tokenId);
}
```

### Other Removals and Changes

- **`Address.isContract()`** — Removed because it is unreliable during construction (returns `false` for constructors). If you need this check, use `address(x).code.length > 0` with the understanding that it also returns `false` during construction.
- **`ReentrancyGuard`** — On Solidity 0.8.24+ with EVM version `cancun`, OZ v5 uses **transient storage** (`tstore`/`tload`) by default, reducing gas cost from ~5000 to ~100 for the guard. No code changes needed.
- **`AccessControl`** — The `onlyRole` modifier is unchanged. Role admin management uses `_setRoleAdmin` with the same semantics. `_setupRole` was removed in favor of `_grantRole`.

### OZ v4 to v5 Breaking Changes Quick Reference

| # | OZ v4 | OZ v5 | Migration Action |
|---|-------|-------|------------------|
| 1 | `Ownable()` (implicit msg.sender) | `Ownable(initialOwner)` | Pass owner to constructor |
| 2 | `_setupRole(role, account)` | `_grantRole(role, account)` | Rename call |
| 3 | `Counters.Counter` | `uint256` | Remove import, use `++` |
| 4 | `_beforeTokenTransfer()` | `_update()` | Rewrite hook logic |
| 5 | `_afterTokenTransfer()` | `_update()` | Merge into `_update` |
| 6 | `Address.isContract(addr)` | `addr.code.length > 0` | Replace call (same caveats) |
| 7 | `ERC721._safeMint(to, id)` | `_safeMint(to, id)` (checks in `_update`) | Override `_update` for custom logic |
| 8 | `ERC20.mint()` as virtual public | `_mint()` internal only | Wrap in your own `mint` function |
| 9 | `ReentrancyGuard` (SSTORE-based) | `ReentrancyGuard` (transient on 0.8.24+) | No change needed; gas savings automatic |
| 10 | `Governor.propose()` sig | Parameters reordered; `clock()` required | Update Governor overrides, add `clock` |

---

## Library Patterns {#libraries}

Libraries in Solidity are reusable code units that can be attached to types. They reduce code duplication and can save deployment gas when used correctly.

### `using LibName for Type` Pattern

Attach a library's functions to a type so they can be called as methods:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

library MathLib {
    error MathOverflow();

    /// @notice Full precision multiplication followed by division: (a * b) / denominator
    /// @dev Reverts on overflow or zero denominator
    function mulDiv(uint256 a, uint256 b, uint256 denominator) internal pure returns (uint256 result) {
        if (denominator == 0) revert MathOverflow();
        // Use 512-bit intermediate to avoid overflow
        uint256 prod0 = a * b;           // least significant 256 bits
        uint256 prod1;                    // most significant 256 bits
        assembly {
            let mm := mulmod(a, b, not(0))
            prod1 := sub(sub(mm, prod0), lt(mm, prod0))
        }
        if (prod1 == 0) return prod0 / denominator;
        if (prod1 >= denominator) revert MathOverflow();

        uint256 remainder;
        assembly {
            remainder := mulmod(a, b, denominator)
            prod1 := sub(prod1, gt(remainder, prod0))
            prod0 := sub(prod0, remainder)
        }
        uint256 twos = denominator & (0 - denominator);
        assembly {
            denominator := div(denominator, twos)
            prod0 := div(prod0, twos)
            twos := add(div(sub(0, twos), twos), 1)
        }
        prod0 |= prod1 * twos;
        uint256 inverse = (3 * denominator) ^ 2;
        inverse *= 2 - denominator * inverse;
        inverse *= 2 - denominator * inverse;
        inverse *= 2 - denominator * inverse;
        inverse *= 2 - denominator * inverse;
        inverse *= 2 - denominator * inverse;
        inverse *= 2 - denominator * inverse;
        result = prod0 * inverse;
    }

    /// @notice Integer square root (floor) using the Babylonian method
    function sqrt(uint256 x) internal pure returns (uint256 result) {
        if (x == 0) return 0;
        result = x;
        uint256 xDiv2 = (x >> 1) + 1;
        while (result > xDiv2) {
            result = xDiv2;
            xDiv2 = (x / result + result) >> 1;
        }
    }
}

contract Vault {
    using MathLib for uint256;

    function calculateShares(
        uint256 assets,
        uint256 totalShares,
        uint256 totalAssets
    ) public pure returns (uint256) {
        // Called as a method on uint256 — first param is the receiver
        return assets.mulDiv(totalShares, totalAssets);
    }

    function sqrtPrice(uint256 price) public pure returns (uint256) {
        return price.sqrt();
    }
}
```

### Selective Function Binding (Solidity 0.8.24+)

You can bind specific functions instead of an entire library, and use `global` to make the binding file-wide:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

type Uint128 is uint128;

library Uint128Lib {
    function add(Uint128 a, Uint128 b) internal pure returns (Uint128) {
        return Uint128.wrap(Uint128.unwrap(a) + Uint128.unwrap(b));
    }

    function sub(Uint128 a, Uint128 b) internal pure returns (Uint128) {
        return Uint128.wrap(Uint128.unwrap(a) - Uint128.unwrap(b));
    }

    function toUint256(Uint128 a) internal pure returns (uint256) {
        return uint256(Uint128.unwrap(a));
    }
}

// Bind specific functions globally — available in all files that import this one
using { Uint128Lib.add, Uint128Lib.sub, Uint128Lib.toUint256 } for Uint128 global;
```

### SafeCast Library Pattern

A common pattern for safe numeric downcasting:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

library SafeCast {
    error SafeCastOverflow();

    function toUint128(uint256 value) internal pure returns (uint128) {
        if (value > type(uint128).max) revert SafeCastOverflow();
        return uint128(value);
    }

    function toUint64(uint256 value) internal pure returns (uint64) {
        if (value > type(uint64).max) revert SafeCastOverflow();
        return uint64(value);
    }

    function toUint32(uint256 value) internal pure returns (uint32) {
        if (value > type(uint32).max) revert SafeCastOverflow();
        return uint32(value);
    }

    function toInt128(int256 value) internal pure returns (int128) {
        if (value < type(int128).min || value > type(int128).max) revert SafeCastOverflow();
        return int128(value);
    }

    function toInt256(uint256 value) internal pure returns (int256) {
        if (value > uint256(type(int256).max)) revert SafeCastOverflow();
        return int256(value);
    }
}

contract StakingPool {
    using SafeCast for uint256;
    using SafeCast for int256;

    struct UserInfo {
        uint128 amount;
        uint64  lastStakeTime;
        int128  rewardDebt;
    }

    mapping(address => UserInfo) public users;

    function stake(uint256 amount) external {
        UserInfo storage user = users[msg.sender];
        user.amount = (uint256(user.amount) + amount).toUint128();
        user.lastStakeTime = block.timestamp.toUint64();
    }
}
```

### Embedded vs Deployed Libraries

| Aspect | Embedded Library | Deployed Library |
|--------|-----------------|------------------|
| Function visibility | `internal` only | `public` or `external` |
| Deployment | Inlined into calling contract | Deployed separately, linked at compile |
| Gas (call overhead) | None (inlined) | `DELEGATECALL` overhead per call |
| Code duplication | Duplicated in every contract that uses it | Single deployed copy shared by all |
| Best for | Small utility functions | Large reusable functions shared by many contracts |

```solidity
// Embedded library — all functions are internal, code is inlined
library EmbeddedLib {
    function max(uint256 a, uint256 b) internal pure returns (uint256) {
        return a > b ? a : b;
    }
}

// Deployed library — has public/external functions, must be linked
library DeployedLib {
    function heavyComputation(uint256[] memory data) public pure returns (uint256 result) {
        for (uint256 i; i < data.length; ++i) {
            result += data[i] * data[i];
        }
    }
}

contract Consumer {
    using EmbeddedLib for uint256;  // inlined — no separate deployment
    using DeployedLib for uint256[];  // requires linking at deploy time

    function process(uint256[] memory data) external pure returns (uint256, uint256) {
        uint256 sum = DeployedLib.heavyComputation(data);
        uint256 biggest = data[0].max(data[1]);
        return (sum, biggest);
    }
}
```

**Foundry linking for deployed libraries:**

```toml
# foundry.toml
[profile.default]
libraries = ["src/DeployedLib.sol:DeployedLib:0x1234...abcd"]
```

### When to Use a Library vs Internal Functions vs Abstract Contract

| Approach | Use When |
|----------|----------|
| **Library + `using for`** | Stateless utility functions on a type (math, casting, array ops) |
| **Internal functions** | Helper logic tightly coupled to one contract's state |
| **Abstract contract** | Shared state + logic across a family of contracts; enforces an interface via `virtual` |

### Remappings for Library Imports

Remappings let you use clean import paths instead of relative `../node_modules/...` paths:

```toml
# foundry.toml — remappings
[profile.default]
remappings = [
    "@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/",
    "@openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts/",
    "@chainlink/=lib/chainlink/",
    "@uniswap/v3-core/=lib/v3-core/",
    "solmate/=lib/solmate/src/",
]
```

```
# remappings.txt — alternative format (also supported by Foundry)
@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/
solmate/=lib/solmate/src/
```

Then import cleanly:

```solidity
import { Math } from "@openzeppelin/contracts/utils/math/Math.sol";
import { ERC20 } from "solmate/tokens/ERC20.sol";
```

---

## Interface & Abstract Contract Patterns {#interfaces}

### `interface` vs `abstract contract`

| Feature | `interface` | `abstract contract` |
|---------|-------------|---------------------|
| State variables | Not allowed | Allowed |
| Function implementations | Not allowed | Partial or full |
| Constructor | Not allowed | Allowed |
| Inheritance | Can inherit interfaces only | Can inherit anything |
| Purpose | Define an external API surface | Provide reusable base logic |

### Interfaces: Define the API Surface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IVault {
    // Events
    event Deposited(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);

    // Errors
    error InsufficientBalance(uint256 available, uint256 requested);
    error ZeroAmount();

    // Functions — no implementation, all implicitly `external`
    function deposit(uint256 amount) external;
    function withdraw(uint256 amount) external;
    function balanceOf(address user) external view returns (uint256);
    function totalAssets() external view returns (uint256);
}
```

### Abstract Contracts: Partial Implementation with Enforced Overrides

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

abstract contract BaseVault is IVault {
    mapping(address => uint256) internal _balances;
    uint256 internal _totalAssets;

    // Implemented — concrete logic shared by all vaults
    function balanceOf(address user) external view override returns (uint256) {
        return _balances[user];
    }

    function totalAssets() external view override returns (uint256) {
        return _totalAssets;
    }

    // Partially implemented — common logic with a hook
    function deposit(uint256 amount) external override {
        if (amount == 0) revert ZeroAmount();
        _balances[msg.sender] += amount;
        _totalAssets += amount;
        _onDeposit(msg.sender, amount);  // subclass must implement
        emit Deposited(msg.sender, amount);
    }

    // Abstract — subclass MUST implement
    function _onDeposit(address user, uint256 amount) internal virtual;
    function _onWithdraw(address user, uint256 amount) internal virtual;
}
```

### Concrete Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { SafeERC20 } from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract ERC20Vault is BaseVault {
    using SafeERC20 for IERC20;

    IERC20 public immutable token;

    constructor(IERC20 _token) {
        token = _token;
    }

    function _onDeposit(address user, uint256 amount) internal override {
        token.safeTransferFrom(user, address(this), amount);
    }

    function _onWithdraw(address user, uint256 amount) internal override {
        token.safeTransfer(user, amount);
    }

    function withdraw(uint256 amount) external override {
        if (amount > _balances[msg.sender]) {
            revert InsufficientBalance(_balances[msg.sender], amount);
        }
        _balances[msg.sender] -= amount;
        _totalAssets -= amount;
        _onWithdraw(msg.sender, amount);
        emit Withdrawn(msg.sender, amount);
    }
}
```

### Multiple Inheritance and C3 Linearization

Solidity uses **C3 linearization** to resolve the inheritance order. Contracts are linearized from **most derived** to **most base**, and the inheritance list is read **left-to-right, most base-like first**:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract A {
    function foo() public pure virtual returns (string memory) {
        return "A";
    }
}

contract B is A {
    function foo() public pure virtual override returns (string memory) {
        return "B";
    }
}

contract C is A {
    function foo() public pure virtual override returns (string memory) {
        return "C";
    }
}

// Linearization: D → C → B → A
// foo() returns "C" because C is the most derived in the linearization
contract D is B, C {
    function foo() public pure override(B, C) returns (string memory) {
        return super.foo();  // calls C.foo() → returns "C"
    }
}
```

**Rule:** In `contract D is B, C`, the list is read left-to-right as "most base-like first." So `C` is more derived than `B`, and `super.foo()` in `D` calls `C.foo()`.

### The Diamond Inheritance Problem and `super`

When multiple parent contracts define the same function, `super` walks the linearization chain and calls each parent **exactly once**:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Base {
    event Called(string name);

    function init() public virtual {
        emit Called("Base");
    }
}

contract Left is Base {
    function init() public virtual override {
        emit Called("Left");
        super.init();  // calls Base.init()
    }
}

contract Right is Base {
    function init() public virtual override {
        emit Called("Right");
        super.init();  // in linearization, calls Left.init() — NOT Base directly
    }
}

// Linearization: Diamond → Right → Left → Base
contract Diamond is Left, Right {
    function init() public override(Left, Right) {
        super.init();
        // Call order: Right.init() → Left.init() → Base.init()
        // Each is called exactly once despite the diamond shape
    }
}
```

### `virtual` and `override` Keywords

```solidity
contract Parent {
    // `virtual` — this function CAN be overridden
    function fee() public pure virtual returns (uint256) {
        return 100;
    }

    // No `virtual` — this function CANNOT be overridden
    function version() public pure returns (uint256) {
        return 1;
    }
}

contract Child is Parent {
    // `override` — this function overrides a parent's virtual function
    // `virtual` — and it can ALSO be overridden by further children
    function fee() public pure override virtual returns (uint256) {
        return 200;
    }
}

contract GrandChild is Child {
    function fee() public pure override returns (uint256) {
        return 300;  // final — no `virtual`, cannot be overridden further
    }
}
```

### Common Pitfall: Forgetting `override` with Multiple Inheritance

When two parent contracts define the same function signature, the child **must** explicitly override both:

```solidity
interface IERC165 {
    function supportsInterface(bytes4 interfaceId) external view returns (bool);
}

contract ERC721 is IERC165 {
    function supportsInterface(bytes4 interfaceId) public view virtual override returns (bool) {
        return interfaceId == type(IERC165).interfaceId;
    }
}

contract ERC721Enumerable is ERC721 {
    function supportsInterface(bytes4 interfaceId) public view virtual override returns (bool) {
        return interfaceId == 0x780e9d63 || super.supportsInterface(interfaceId);
    }
}

contract AccessControl is IERC165 {
    function supportsInterface(bytes4 interfaceId) public view virtual override returns (bool) {
        return interfaceId == type(IERC165).interfaceId;
    }
}

// ❌ Compile error — ambiguous supportsInterface, must override explicitly
// contract MyNFT is ERC721Enumerable, AccessControl {}

// ✅ Correct — explicit override listing all conflicting parents
contract MyNFT is ERC721Enumerable, AccessControl {
    function supportsInterface(bytes4 interfaceId)
        public
        view
        override(ERC721Enumerable, AccessControl)
        returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
}
```

---

## Deterministic Deployment (CREATE2) {#create2}

`CREATE2` allows deploying contracts to **predictable addresses** that are independent of the deployer's nonce, enabling same-address deployments across multiple chains.

### How CREATE2 Works

The address is computed as:

```
address = keccak256(0xff ++ deployerAddress ++ salt ++ keccak256(initCode))[12:]
```

Where:
- `0xff` — a constant prefix to avoid collisions with `CREATE`
- `deployerAddress` — the contract calling `CREATE2`
- `salt` — a `bytes32` value chosen by the deployer
- `initCode` — the contract creation code (bytecode + constructor args)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract CREATE2Factory {
    event Deployed(address indexed addr, bytes32 indexed salt);

    error DeploymentFailed();

    /// @notice Deploy a contract using CREATE2
    /// @param salt Deterministic salt
    /// @param creationCode The contract's creation bytecode (including constructor args)
    function deploy(bytes32 salt, bytes memory creationCode)
        external
        payable
        returns (address deployed)
    {
        assembly {
            deployed := create2(callvalue(), add(creationCode, 0x20), mload(creationCode), salt)
        }
        if (deployed == address(0)) revert DeploymentFailed();
        emit Deployed(deployed, salt);
    }

    /// @notice Predict the address of a CREATE2 deployment
    function computeAddress(bytes32 salt, bytes memory creationCode)
        external
        view
        returns (address)
    {
        return address(uint160(uint256(keccak256(abi.encodePacked(
            bytes1(0xff),
            address(this),
            salt,
            keccak256(creationCode)
        )))));
    }
}
```

### The Canonical CREATE2 Deployer (Nick's Factory)

The keyless deployment factory at `0x4e59b44847b379578588920cA78FbF26c0B4956C` exists on virtually every EVM chain. It was deployed using a pre-signed transaction (no private key needed), guaranteeing the same address everywhere.

**Usage:** Send a transaction to this address with `salt ++ initCode` as calldata. The contract deploys using `CREATE2` with the provided salt.

```solidity
// Using Nick's factory from a Foundry script
interface ICreate2Deployer {
    // No ABI — just send raw calldata: salt (32 bytes) ++ initCode
}

contract DeployViaNick is Script {
    address constant NICK_FACTORY = 0x4e59b44847b379578588920cA78FbF26c0B4956C;

    function run() external {
        bytes32 salt = bytes32(uint256(1));
        bytes memory initCode = abi.encodePacked(
            type(MyContract).creationCode,
            abi.encode(constructorArg1, constructorArg2)
        );

        // Predict the address
        address predicted = address(uint160(uint256(keccak256(abi.encodePacked(
            bytes1(0xff),
            NICK_FACTORY,
            salt,
            keccak256(initCode)
        )))));

        vm.startBroadcast();
        (bool success,) = NICK_FACTORY.call(abi.encodePacked(salt, initCode));
        require(success, "CREATE2 deploy failed");
        vm.stopBroadcast();

        console.log("Deployed to:", predicted);
    }
}
```

### Same-Address Deployment Across Chains

To get the same contract address on multiple chains:

1. Use the **same** CREATE2 deployer (Nick's factory exists on all major chains)
2. Use the **same** salt
3. Use the **same** init code (bytecode + constructor arguments)

**Important:** If constructor arguments differ between chains (e.g., different admin address), the init code changes and the address will differ.

```solidity
// Foundry script for cross-chain deterministic deployment
contract CrossChainDeploy is Script {
    address constant NICK_FACTORY = 0x4e59b44847b379578588920cA78FbF26c0B4956C;

    // Same salt on every chain
    bytes32 constant SALT = keccak256("myprotocol.v1.vault");

    function run() external {
        // Constructor args must be IDENTICAL across chains for same address
        address admin = 0xaAaAaAaaAaAaAaaAaAAAAAAAAaaaAaAaAaaAaaAa;  // same admin everywhere
        bytes memory initCode = abi.encodePacked(
            type(Vault).creationCode,
            abi.encode(admin)
        );

        address predicted = _predictAddress(SALT, initCode);
        console.log("Expected address on all chains:", predicted);

        vm.startBroadcast();
        (bool success,) = NICK_FACTORY.call(abi.encodePacked(SALT, initCode));
        require(success, "deploy failed");
        vm.stopBroadcast();
    }

    function _predictAddress(bytes32 salt, bytes memory initCode)
        internal
        pure
        returns (address)
    {
        return address(uint160(uint256(keccak256(abi.encodePacked(
            bytes1(0xff),
            NICK_FACTORY,
            salt,
            keccak256(initCode)
        )))));
    }
}
```

### Foundry Script for CREATE2 Deployment

A production-ready Foundry deployment pattern:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import { Script, console } from "forge-std/Script.sol";
import { Vault } from "../src/Vault.sol";

contract DeployCreate2 is Script {
    function run() external {
        // Salt derived from deployer + version for reproducibility
        bytes32 salt = keccak256(abi.encodePacked(msg.sender, "vault-v1"));

        bytes memory initCode = abi.encodePacked(
            type(Vault).creationCode,
            abi.encode(msg.sender)  // constructor(address owner)
        );

        // Compute expected address
        address factory = 0x4e59b44847b379578588920cA78FbF26c0B4956C;
        address expected = address(uint160(uint256(keccak256(abi.encodePacked(
            bytes1(0xff),
            factory,
            salt,
            keccak256(initCode)
        )))));

        console.log("Expected address:", expected);

        // Check if already deployed
        if (expected.code.length > 0) {
            console.log("Already deployed, skipping");
            return;
        }

        vm.startBroadcast();
        (bool success,) = factory.call(abi.encodePacked(salt, initCode));
        require(success, "CREATE2 deployment failed");
        vm.stopBroadcast();

        // Verify
        require(expected.code.length > 0, "Deployment verification failed");
        console.log("Deployed successfully to:", expected);
    }
}
```

### Salt Selection Strategies

| Strategy | Example | Use Case |
|----------|---------|----------|
| Fixed string | `keccak256("myprotocol.v1")` | Reproducible, public deploys |
| Sender + nonce | `keccak256(abi.encode(msg.sender, nonce))` | Per-user deterministic proxies |
| Sender + version | `keccak256(abi.encode(msg.sender, "v1"))` | Upgradeable-friendly deployments |
| Chain-specific | `keccak256(abi.encode(block.chainid, "v1"))` | When different addresses per chain are OK |
| Vanity mining | Loop salts until `addr` starts with `0x0000...` | Gas optimization (zero-byte addresses are cheaper in calldata) |

### Security: Front-Running and Metamorphic Contracts

**Front-running CREATE2 deploys:**

An attacker who sees your CREATE2 deployment transaction in the mempool can front-run it with the same salt and init code, deploying the contract first. The contract will have the same code, but the attacker deployed it — which matters if the constructor grants privileges.

**Mitigation:** Include `msg.sender` in the salt so only a specific deployer can use it:

```solidity
contract ProtectedFactory {
    error Unauthorized();

    function deploy(bytes32 userSalt, bytes memory creationCode)
        external
        returns (address deployed)
    {
        // Salt includes msg.sender — only this sender can produce this address
        bytes32 salt = keccak256(abi.encodePacked(msg.sender, userSalt));
        assembly {
            deployed := create2(0, add(creationCode, 0x20), mload(creationCode), salt)
        }
        if (deployed == address(0)) revert Unauthorized();
    }
}
```

**Metamorphic contract attack:**

A contract deployed via CREATE2 can be `SELFDESTRUCT`ed (pre-Dencun) and then redeployed to the **same address** with **different code**. This is the "metamorphic contract" attack — the address looks the same, but the logic changed.

**Mitigation:** After Dencun (EIP-6780), `SELFDESTRUCT` only sends ETH without destroying the contract (except in the same transaction as creation), making this attack much harder. For pre-Dencun chains, verify that contracts at CREATE2 addresses have not been redeployed by checking code hashes.

### CREATE3 Pattern

`CREATE3` provides init-code-independent deterministic addresses. The address depends only on the deployer and salt, **not** on the init code. This is achieved by combining CREATE2 + CREATE:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// @notice CREATE3 factory — address depends only on (deployer, salt), not init code
contract CREATE3Factory {
    error DeploymentFailed();

    /// @dev The proxy bytecode used in the CREATE2 step.
    ///      This is a minimal contract that deploys whatever data is in its constructor:
    ///      PUSH1 0x00 CALLDATASIZE PUSH1 0x00 CALLDATACOPY CALLDATASIZE PUSH1 0x00 RETURN
    bytes internal constant PROXY_BYTECODE = hex"67363d3d37363d34f03d5260086018f3";

    function deploy(bytes32 salt, bytes memory creationCode)
        external
        payable
        returns (address deployed)
    {
        // Step 1: CREATE2 a minimal proxy with a fixed bytecode
        // Since PROXY_BYTECODE is constant, the proxy address depends only on salt
        address proxy;
        bytes memory proxyCode = PROXY_BYTECODE;
        assembly {
            proxy := create2(0, add(proxyCode, 0x20), mload(proxyCode), salt)
        }
        if (proxy == address(0)) revert DeploymentFailed();

        // Step 2: The proxy uses CREATE (nonce-based) to deploy the actual contract
        // Since the proxy always has nonce=1 on its first CREATE, the address is deterministic
        (bool success,) = proxy.call{value: msg.value}(creationCode);
        if (!success) revert DeploymentFailed();

        // The deployed contract address is determined by the proxy address + nonce (1)
        deployed = address(uint160(uint256(keccak256(abi.encodePacked(
            bytes1(0xd6), bytes1(0x94), proxy, bytes1(0x01)
        )))));

        require(deployed.code.length > 0, "CREATE3: deployment failed");
    }

    /// @notice Predict the deployment address (depends only on salt, NOT init code)
    function predictAddress(bytes32 salt) external view returns (address) {
        // First predict the proxy address (CREATE2)
        address proxy = address(uint160(uint256(keccak256(abi.encodePacked(
            bytes1(0xff),
            address(this),
            salt,
            keccak256(PROXY_BYTECODE)
        )))));

        // Then predict the deployed contract address (CREATE from proxy with nonce 1)
        return address(uint160(uint256(keccak256(abi.encodePacked(
            bytes1(0xd6), bytes1(0x94), proxy, bytes1(0x01)
        )))));
    }
}
```

**Why CREATE3?** With standard CREATE2, changing constructor arguments changes the init code, which changes the deployed address. CREATE3 removes this dependency — you can deploy contracts with different constructor arguments on different chains and still get the same address.

| Feature | CREATE2 | CREATE3 |
|---------|---------|---------|
| Address depends on | deployer + salt + initCode | deployer + salt only |
| Same address if constructor args change | No | Yes |
| Gas cost | Lower | Higher (two deployments) |
| Cross-chain same address | Only if init code is identical | Even with different constructor args |
| Complexity | Simple | Moderate (proxy intermediary) |