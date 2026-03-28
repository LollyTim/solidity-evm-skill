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
```