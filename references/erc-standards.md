# ERC Token Standards & Interface Reference

## Table of Contents
1. [ERC-20 — Fungible Token](#erc20)
2. [ERC-721 — NFT (Non-Fungible Token)](#erc721)
3. [ERC-1155 — Multi-Token Standard](#erc1155)
4. [ERC-4626 — Tokenized Vault](#erc4626)
5. [ERC-2981 — NFT Royalties](#erc2981)
6. [EIP-712 — Typed Structured Data Signing](#eip712)
7. [ERC-4337 — Account Abstraction](#erc4337)
8. [ERC-165 — Interface Detection](#erc165)
9. [Other Key EIPs](#other-eips)
10. [Token Integration Checklist](#token-checklist)

---

## ERC-20 — Fungible Token {#erc20}

### Basic ERC-20

```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";  // EIP-2612
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyToken is ERC20, ERC20Burnable, ERC20Permit, Ownable {
    constructor(address initialOwner)
        ERC20("MyToken", "MTK")
        ERC20Permit("MyToken")
        Ownable(initialOwner)
    {
        _mint(initialOwner, 1_000_000 * 10 ** decimals());
    }

    function mint(address to, uint256 amount) external onlyOwner {
        _mint(to, amount);
    }
}
```

### ERC-20 with Capped Supply

```solidity
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Capped.sol";

contract CappedToken is ERC20Capped, Ownable {
    constructor() ERC20("CappedToken", "CAP") ERC20Capped(100_000_000 * 1e18) Ownable(msg.sender) {
        _mint(msg.sender, 10_000_000 * 1e18);
    }
}
```

### ERC-20 Gotchas & Non-Standard Tokens

```solidity
// NEVER use `token.transfer()` directly — some tokens (USDT, BNB) don't return bool
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract Vault {
    using SafeERC20 for IERC20;

    function deposit(IERC20 token, uint256 amount) external {
        token.safeTransferFrom(msg.sender, address(this), amount);  // handles USDT etc.
    }

    function withdraw(IERC20 token, address to, uint256 amount) external {
        token.safeTransfer(to, amount);  // handles non-standard return values
    }
}
```

**Common non-standard behaviors to handle:**
- USDT: doesn't return bool on transfer
- USDC: upgradeable, can blacklist addresses
- WBTC: 8 decimals (not 18)
- FOT tokens (fee-on-transfer): received amount < sent amount
- Rebase tokens: balance changes without transfer

### ERC-20 Permit (EIP-2612 — Gasless Approvals)

```solidity
// User signs off-chain, relayer submits on-chain (no separate approve tx needed)
function depositWithPermit(
    IERC20Permit token,
    uint256 amount,
    uint256 deadline,
    uint8 v, bytes32 r, bytes32 s
) external {
    token.permit(msg.sender, address(this), amount, deadline, v, r, s);
    IERC20(address(token)).safeTransferFrom(msg.sender, address(this), amount);
}
```

### ERC-20 Approval Race Condition & Permit2

**The classic approve race condition:**
A user has approved a spender for 100 tokens and wants to change the approval to 50. A malicious miner/validator can front-run the `approve(50)` call: first spend the existing 100 allowance, then the new 50 approval lands, giving the spender 150 total instead of the intended 50.

**Solutions:**

1. **`increaseAllowance` / `decreaseAllowance`** (OpenZeppelin pattern) — adjust allowance relative to current value, avoiding the race
2. **Approve to 0 first, then to the new amount** — requires two transactions, but prevents the exploit since a front-runner can only spend 0 after the first tx
3. **Permit2** (Uniswap's universal approval manager) — the modern solution that eliminates per-protocol approvals entirely

**Permit2 explained:**

Permit2 is a canonical contract deployed by Uniswap that acts as a universal token approval manager:

- User approves the Permit2 contract once with max approval for each token
- Individual protocols receive scoped, time-limited, revocable permits via off-chain signatures
- Two modes:
  - `SignatureTransfer` — one-time permit consumed immediately (nonce-based)
  - `AllowanceTransfer` — recurring permit with amount and expiration (traditional allowance model, but managed by Permit2)
- Canonical address (same on all chains): `0x000000000022D473030F116dDEE9F6B43aC78BA3`

**Protocol integration with Permit2:**

```solidity
import {ISignatureTransfer} from "permit2/src/interfaces/ISignatureTransfer.sol";

contract MyProtocol {
    ISignatureTransfer public immutable permit2;

    constructor(ISignatureTransfer _permit2) {
        permit2 = _permit2;
    }

    function depositWithPermit2(
        ISignatureTransfer.PermitTransferFrom calldata permit,
        ISignatureTransfer.SignatureTransferDetails calldata transferDetails,
        bytes calldata signature
    ) external {
        permit2.permitTransferFrom(permit, transferDetails, msg.sender, signature);
        // ... credit user
    }
}
```

**When to use Permit2 vs EIP-2612 Permit:**

| Feature | EIP-2612 | Permit2 |
|---------|----------|---------|
| Token must support it | Yes (built-in) | No (works with any ERC-20) |
| One approval per protocol | No | Yes (approve Permit2 once) |
| Expiration | Yes | Yes |
| Batch support | No | Yes |

---

## ERC-721 — NFT (Non-Fungible Token) {#erc721}

### Basic NFT

```solidity
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Royalty.sol";  // ERC-2981
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyNFT is ERC721, ERC721URIStorage, ERC721Royalty, Ownable {
    uint256 private _nextTokenId;
    uint256 public constant MINT_PRICE = 0.01 ether;
    uint256 public constant MAX_SUPPLY = 10_000;

    constructor(address initialOwner) ERC721("MyNFT", "MNFT") Ownable(initialOwner) {
        // Set default 5% royalty to owner (ERC-2981)
        _setDefaultRoyalty(initialOwner, 500);  // 500 = 5% in basis points
    }

    function mint(address to, string calldata uri) external payable returns (uint256) {
        if (msg.value < MINT_PRICE) revert InsufficientPayment();
        if (_nextTokenId >= MAX_SUPPLY) revert MaxSupplyReached();

        uint256 tokenId = _nextTokenId++;
        _safeMint(to, tokenId);
        _setTokenURI(tokenId, uri);
        return tokenId;
    }

    // Required overrides for multiple inheritance
    function tokenURI(uint256 tokenId) public view override(ERC721, ERC721URIStorage) returns (string memory) {
        return super.tokenURI(tokenId);
    }

    function supportsInterface(bytes4 interfaceId) public view override(ERC721, ERC721URIStorage, ERC721Royalty) returns (bool) {
        return super.supportsInterface(interfaceId);
    }
}
```

### NFT with Merkle Tree Whitelist

```solidity
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";

contract WhitelistNFT is ERC721, Ownable {
    bytes32 public merkleRoot;
    mapping(address => bool) public claimed;

    function setMerkleRoot(bytes32 root) external onlyOwner {
        merkleRoot = root;
    }

    function whitelistMint(bytes32[] calldata proof) external {
        if (claimed[msg.sender]) revert AlreadyClaimed();

        bytes32 leaf = keccak256(abi.encodePacked(msg.sender));
        if (!MerkleProof.verify(proof, merkleRoot, leaf)) revert NotWhitelisted();

        claimed[msg.sender] = true;
        _safeMint(msg.sender, _nextTokenId++);
    }
}
```

### NFT Metadata Standards

```json
// Standard metadata JSON (ERC-721 Metadata URI Schema)
{
  "name": "Token Name",
  "description": "Token description",
  "image": "ipfs://QmHash.../0.png",
  "attributes": [
    { "trait_type": "Background", "value": "Blue" },
    { "trait_type": "Rarity",     "value": "Rare" },
    { "trait_type": "Level",      "display_type": "number", "value": 5 }
  ]
}
```

**On-chain metadata (fully decentralized):**
```solidity
function tokenURI(uint256 tokenId) public view override returns (string memory) {
    string memory json = Base64.encode(bytes(string(abi.encodePacked(
        '{"name":"Token #', tokenId.toString(),
        '","description":"My NFT Collection","attributes":[]}'
    ))));
    return string(abi.encodePacked("data:application/json;base64,", json));
}
```

---

## ERC-1155 — Multi-Token Standard {#erc1155}

Best for gaming, mixed fungible/non-fungible collections:

```solidity
import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/token/ERC1155/extensions/ERC1155Supply.sol";

contract GameItems is ERC1155, ERC1155Supply, Ownable {
    uint256 public constant SWORD  = 1;
    uint256 public constant SHIELD = 2;
    uint256 public constant POTION = 3;

    constructor() ERC1155("https://api.mygame.com/items/{id}.json") Ownable(msg.sender) {}

    function mintBatch(address to, uint256[] calldata ids, uint256[] calldata amounts) external onlyOwner {
        _mintBatch(to, ids, amounts, "");
    }

    // Efficient batch transfer
    function safeTransferFrom(address from, address to, uint256 id, uint256 amount, bytes calldata data)
        public override { super.safeTransferFrom(from, to, id, amount, data); }

    function _update(address from, address to, uint256[] memory ids, uint256[] memory values)
        internal override(ERC1155, ERC1155Supply) {
        super._update(from, to, ids, values);
    }
}
```

**When to use ERC-1155 over ERC-721:**
- Large collections where items share types (100k swords, 50k shields)
- Games with fungible resources AND unique items in same contract
- Batch minting/transfers to save gas

---

## ERC-4626 — Tokenized Vault {#erc4626}

Standard interface for yield-bearing vaults (DeFi building block):

```solidity
import "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";

contract SimpleVault is ERC4626, Ownable {
    constructor(IERC20 asset, address owner)
        ERC4626(asset)
        ERC20("Vault Share", "vSHARE")
        Ownable(owner)
    {}

    // Override to account for yield accrual
    function totalAssets() public view override returns (uint256) {
        return IERC20(asset()).balanceOf(address(this));
        // In real vault: + accrued yield from strategy
    }
}

// Key ERC-4626 functions:
// deposit(assets, receiver) → shares       // user deposits asset
// mint(shares, receiver) → assets          // user buys exact shares
// withdraw(assets, receiver, owner) → shares  // user withdraws asset
// redeem(shares, receiver, owner) → assets    // user burns shares
// convertToShares(assets) → shares         // preview conversion
// convertToAssets(shares) → assets         // preview conversion
// previewDeposit(assets) → shares
// previewWithdraw(assets) → shares
// maxDeposit(receiver) → maxAssets
// maxWithdraw(owner) → maxAssets
```

---

## ERC-2981 — NFT Royalties {#erc2981}

Standard interface for querying royalty information:

```solidity
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Royalty.sol";

contract RoyaltyNFT is ERC721, ERC721Royalty {
    constructor() ERC721("RoyaltyNFT", "RNFT") {
        // Set default 7.5% royalty to this contract
        _setDefaultRoyalty(address(this), 750);  // 750/10000 = 7.5%
    }

    function setTokenRoyalty(uint256 tokenId, address receiver, uint96 feeNumerator) external onlyOwner {
        _setTokenRoyalty(tokenId, receiver, feeNumerator);
    }
}

// Marketplaces call:
// royaltyInfo(tokenId, salePrice) → (receiver, royaltyAmount)
```

---

## EIP-712 — Typed Structured Data Signing {#eip712}

Standard for signing structured data off-chain (permits, orders, meta-transactions):

```solidity
import "@openzeppelin/contracts/utils/cryptography/EIP712.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

contract OrderBook is EIP712 {
    struct Order {
        address maker;
        address tokenIn;
        address tokenOut;
        uint256 amountIn;
        uint256 amountOut;
        uint256 nonce;
        uint256 deadline;
    }

    bytes32 public constant ORDER_TYPEHASH = keccak256(
        "Order(address maker,address tokenIn,address tokenOut,uint256 amountIn,uint256 amountOut,uint256 nonce,uint256 deadline)"
    );

    mapping(address => uint256) public nonces;

    constructor() EIP712("OrderBook", "1") {}

    function fillOrder(Order calldata order, bytes calldata signature) external {
        // Verify deadline
        if (block.timestamp > order.deadline) revert OrderExpired();

        // Reconstruct and verify signature
        bytes32 structHash = keccak256(abi.encode(
            ORDER_TYPEHASH,
            order.maker, order.tokenIn, order.tokenOut,
            order.amountIn, order.amountOut,
            order.nonce, order.deadline
        ));
        bytes32 digest = _hashTypedDataV4(structHash);  // includes domain separator
        address signer = ECDSA.recover(digest, signature);

        if (signer != order.maker) revert InvalidSignature();
        if (nonces[order.maker] != order.nonce) revert InvalidNonce();

        nonces[order.maker]++;
        // ... execute the order
    }
}
```

---

## ERC-4337 — Account Abstraction {#erc4337}

Enables smart contract wallets without protocol changes. Users can batch transactions, pay gas in ERC-20, etc.

**Key concepts:**
- `UserOperation` — structured transaction request
- `Bundler` — off-chain node that submits batched UserOps
- `EntryPoint` — singleton contract that validates and executes UserOps
- `Paymaster` — contract that pays gas on behalf of users

```solidity
import "@account-abstraction/contracts/interfaces/IAccount.sol";
import "@account-abstraction/contracts/core/BaseAccount.sol";

contract SimpleAccount is BaseAccount, Initializable {
    address public owner;
    IEntryPoint private immutable _entryPoint;

    constructor(IEntryPoint entryPoint) {
        _entryPoint = entryPoint;
        _disableInitializers();
    }

    function initialize(address anOwner) external initializer {
        owner = anOwner;
    }

    function entryPoint() public view override returns (IEntryPoint) {
        return _entryPoint;
    }

    function _validateSignature(PackedUserOperation calldata userOp, bytes32 userOpHash)
        internal override returns (uint256 validationData) {
        bytes32 hash = userOpHash.toEthSignedMessageHash();
        if (owner != hash.recover(userOp.signature)) return SIG_VALIDATION_FAILED;
        return SIG_VALIDATION_SUCCESS;
    }

    function execute(address dest, uint256 value, bytes calldata func) external {
        _requireFromEntryPointOrOwner();
        (bool ok, bytes memory result) = dest.call{value: value}(func);
        if (!ok) revert CallFailed(result);
    }
}
```

---

## ERC-165 — Interface Detection {#erc165}

Allows contracts to publish which interfaces they support:

```solidity
// Your contract
function supportsInterface(bytes4 interfaceId) public view override returns (bool) {
    return interfaceId == type(IERC721).interfaceId ||
           interfaceId == type(IERC2981).interfaceId ||
           super.supportsInterface(interfaceId);
}

// Checking if a contract supports an interface
bool isERC721 = IERC165(tokenAddress).supportsInterface(type(IERC721).interfaceId);
```

---

## Other Key EIPs {#other-eips}

| EIP | Name | Purpose |
|-----|------|---------|
| EIP-1167 | Minimal Proxy | Cheap contract cloning (clone factories) |
| EIP-1559 | Fee Market | Base fee + priority fee (maxFeePerGas/maxPriorityFeePerGas) |
| EIP-1967 | Proxy Storage Slots | Standard slot locations for proxy implementation address |
| EIP-2612 | Permit | Off-chain token approvals via signature |
| EIP-3156 | Flash Loan | Standard interface for flash loans |
| EIP-3525 | Semi-Fungible Token | Token with value within a slot (financial instruments) |
| EIP-4361 | Sign-In with Ethereum | Authentication standard |
| EIP-6780 | SELFDESTRUCT | Post-Cancun: only works in constructor |
| EIP-7201 | Namespaced Storage | Safe storage layout for complex upgradeable contracts |
| EIP-1153 | Transient Storage | TSTORE/TLOAD: cheap temporary storage (cleared after tx) |
| EIP-2771 | Meta Transactions | Trusted forwarder pattern for gasless transactions |
| EIP-5267 | EIP-712 Domain Discovery | Retrieve EIP-712 domain info from contracts |

### Transient Storage (EIP-1153 — Solidity 0.8.24+)

```solidity
// Reentrancy guard using transient storage (much cheaper than regular storage)
uint256 transient private _locked;

modifier nonReentrant() {
    if (_locked != 0) revert ReentrantCall();
    _locked = 1;
    _;
    _locked = 0;
}
```

Transient storage costs: 100 gas (write) vs 22,100 gas (regular storage write). Cleared automatically after transaction.

---

## Token Integration Checklist {#token-checklist}

A checklist for protocols that accept arbitrary ERC-20 tokens:

- [ ] Use `SafeERC20` for all transfers (handles non-standard return values)
- [ ] Handle fee-on-transfer tokens (check balance before/after transfer)
- [ ] Handle rebase tokens (don't cache balanceOf, or explicitly exclude them)
- [ ] Handle tokens with decimals != 18 (USDC=6, WBTC=8)
- [ ] Handle tokens that can pause (USDC, USDT)
- [ ] Handle tokens that can blacklist (USDC, USDT)
- [ ] Handle tokens with max uint256 approval meaning "infinite" (some tokens)
- [ ] Don't assume `transferFrom` returns true — some return nothing
- [ ] Check for zero-amount transfer behavior (some tokens revert on 0)
- [ ] Consider tokens with callbacks (ERC-777, ERC-1363) — reentrancy risk

```solidity
// Pattern: safe deposit handling for fee-on-transfer tokens
function deposit(IERC20 token, uint256 amount) external {
    uint256 balanceBefore = token.balanceOf(address(this));
    token.safeTransferFrom(msg.sender, address(this), amount);
    uint256 actualReceived = token.balanceOf(address(this)) - balanceBefore;
    // Credit user with actualReceived, NOT amount
    balances[msg.sender] += actualReceived;
}
```