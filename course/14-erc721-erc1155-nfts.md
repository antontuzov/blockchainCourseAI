# Module 14 — ERC-721 & ERC-1155 NFTs

> **Part 4 · Token Standards & DeFi Primitives**

---

## Learning Objectives

After completing this module you will be able to:

1. Explain the difference between ERC-721 (unique NFTs) and ERC-1155 (semi-fungible)
2. Implement an ERC-721 NFT collection with metadata
3. Build a basic NFT marketplace with listing, buying, and royalty enforcement
4. Implement ERC-1155 for multi-token contracts
5. Understand metadata standards (on-chain vs off-chain, IPFS)

---

## 1. ERC-721 — Non-Fungible Tokens

Each token is **unique** and indivisible. Used for:
- Digital art (BAYC, CryptoPunks)
- Domain names (ENS)
- Game items
- Real-world asset deeds

### Core Interface

```solidity
interface IERC721 {
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);

    function balanceOf(address owner) external view returns (uint256);
    function ownerOf(uint256 tokenId) external view returns (address);
    function safeTransferFrom(address from, address to, uint256 tokenId) external;
    function transferFrom(address from, address to, uint256 tokenId) external;
    function approve(address to, uint256 tokenId) external;
    function setApprovalForAll(address operator, bool approved) external;
    function getApproved(uint256 tokenId) external view returns (address);
    function isApprovedForAll(address owner, address operator) external view returns (bool);
}
```

---

## 2. Implementing an ERC-721 Collection

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC721} from "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import {ERC721Enumerable} from "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {Strings} from "@openzeppelin/contracts/utils/Strings.sol";

/// @title PixelArt — an ERC-721 NFT collection
contract PixelArt is ERC721Enumerable, Ownable {
    using Strings for uint256;

    uint256 public constant MAX_SUPPLY = 10_000;
    uint256 public mintPrice = 0.05 ether;
    string private _baseTokenURI;
    bool public publicSaleActive;

    mapping(uint256 => string) private _tokenURIs; // For per-token metadata

    error MaxSupplyReached();
    error InsufficientPayment();
    error SaleNotActive();

    constructor() ERC721("Pixel Art", "PIXEL") Ownable(msg.sender) {
        _baseTokenURI = "ipfs://QmYourBaseURI/";
    }

    /// @notice Public mint
    function mint(uint256 quantity) external payable {
        if (!publicSaleActive) revert SaleNotActive();
        if (totalSupply() + quantity > MAX_SUPPLY) revert MaxSupplyReached();
        if (msg.value < mintPrice * quantity) revert InsufficientPayment();

        uint256 startId = totalSupply();
        for (uint256 i; i < quantity;) {
            _safeMint(msg.sender, startId + i);
            unchecked { ++i; }
        }
    }

    /// @notice Owner airdrop
    function airdrop(address to, uint256 quantity) external onlyOwner {
        if (totalSupply() + quantity > MAX_SUPPLY) revert MaxSupplyReached();
        uint256 startId = totalSupply();
        for (uint256 i; i < quantity;) {
            _safeMint(to, startId + i);
            unchecked { ++i; }
        }
    }

    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        _requireOwned(tokenId);
        if (bytes(_tokenURIs[tokenId]).length > 0) {
            return _tokenURIs[tokenId];
        }
        return string(abi.encodePacked(_baseTokenURI, tokenId.toString(), ".json"));
    }

    function setBaseURI(string memory newBaseURI) external onlyOwner {
        _baseTokenURI = newBaseURI;
    }

    function withdraw() external onlyOwner {
        (bool success,) = owner().call{value: address(this).balance}("");
        require(success);
    }

    function toggleSale() external onlyOwner {
        publicSaleActive = !publicSaleActive;
    }
}
```

### Metadata JSON (hosted on IPFS)

```json
{
  "name": "Pixel Art #1",
  "description": "A unique pixel art character",
  "image": "ipfs://QmImageHash/1.png",
  "attributes": [
    {"trait_type": "Background", "value": "Blue"},
    {"trait_type": "Rarity", "value": "Legendary"}
  ]
}
```

---

## 3. Simple NFT Marketplace

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title NFTMarketplace — list and buy NFTs
contract NFTMarketplace is ReentrancyGuard {
    struct Listing {
        address seller;
        address nftContract;
        uint256 tokenId;
        uint256 price;
        bool active;
    }

    uint256 public listingCount;
    mapping(uint256 => Listing) public listings;
    uint256 public feeBps = 250; // 2.5%

    event Listed(uint256 indexed listingId, address indexed seller, uint256 price);
    event Bought(uint256 indexed listingId, address indexed buyer, uint256 price);
    event Cancelled(uint256 indexed listingId);

    error NotTokenOwner();
    error ListingNotActive();
    error InsufficientPayment();
    error NotSeller();

    function list(address nftContract, uint256 tokenId, uint256 price) external returns (uint256) {
        if (IERC721(nftContract).ownerOf(tokenId) != msg.sender) revert NotTokenOwner();

        IERC721(nftContract).transferFrom(msg.sender, address(this), tokenId);

        uint256 id = listingCount++;
        listings[id] = Listing({
            seller: msg.sender,
            nftContract: nftContract,
            tokenId: tokenId,
            price: price,
            active: true
        });

        emit Listed(id, msg.sender, price);
        return id;
    }

    function buy(uint256 listingId) external payable nonReentrant {
        Listing storage listing = listings[listingId];
        if (!listing.active) revert ListingNotActive();
        if (msg.value < listing.price) revert InsufficientPayment();

        listing.active = false;

        uint256 fee = (listing.price * feeBps) / 10_000;
        uint256 sellerProceeds = listing.price - fee;

        IERC721(listing.nftContract).safeTransferFrom(address(this), msg.sender, listing.tokenId);

        (bool success,) = listing.seller.call{value: sellerProceeds}("");
        require(success, "Seller payment failed");

        emit Bought(listingId, msg.sender, listing.price);
    }

    function cancel(uint256 listingId) external {
        Listing storage listing = listings[listingId];
        if (listing.seller != msg.sender) revert NotSeller();
        if (!listing.active) revert ListingNotActive();

        listing.active = false;
        IERC721(listing.nftContract).safeTransferFrom(address(this), msg.sender, listing.tokenId);
        emit Cancelled(listingId);
    }
}
```

---

## 4. ERC-1155 — Multi-Token Standard

One contract manages **multiple token types** — fungible, semi-fungible, and NFTs.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC1155} from "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

/// @title GameItems — sword, shield, gold, potions
contract GameItems is ERC1155, Ownable {
    uint256 public constant GOLD = 0;     // Fungible (1M supply)
    uint256 public constant SWORD = 1;    // Semi-fungible (1000 supply)
    uint256 public constant SHIELD = 2;   // Semi-fungible (500 supply)
    uint256 public constant LEGENDARY = 3; // NFT (1 of each)

    constructor() ERC1155("ipfs://QmGameItems/{id}.json") Ownable(msg.sender) {
        _mint(msg.sender, GOLD, 1_000_000 * 10 ** 18, "");
        _mint(msg.sender, SWORD, 1000, "");
        _mint(msg.sender, SHIELD, 500, "");
    }

    function mintLegendary(address to, uint256 id) external onlyOwner {
        _mint(to, id, 1, ""); // Only 1 of each legendary
    }

    function mintBatch(address to, uint256[] memory ids, uint256[] memory amounts) external onlyOwner {
        _mintBatch(to, ids, amounts, "");
    }

    function setURI(string memory newuri) external onlyOwner {
        _setURI(newuri);
    }
}
```

### ERC-721 vs ERC-1155

| Feature | ERC-721 | ERC-1155 |
|---------|---------|----------|
| Token types | One type per contract | Multiple types per contract |
| Batch transfers | No | Yes (`safeBatchTransferFrom`) |
| Fungible tokens | No | Yes |
| Gas for bulk mint | High (separate calls) | Low (one call) |
| Use case | Unique NFTs | Games, mixed assets |

---

## 5. Royalties — EIP-2981

```solidity
import {ERC2981} from "@openzeppelin/contracts/token/common/ERC2981.sol";

contract RoyaltyNFT is ERC721, ERC2981, Ownable {
    constructor() ERC721("Royalty Art", "RA") Ownable(msg.sender) {
        _setDefaultRoyalty(msg.sender, 500); // 5% royalty
    }

    function royaltyInfo(uint256 tokenId, uint256 salePrice)
        public view override returns (address, uint256)
    {
        return super.royaltyInfo(tokenId, salePrice);
    }

    // Required override for ERC-165
    function supportsInterface(bytes4 interfaceId)
        public view override(ERC721, ERC2981) returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
}
```

---

## Checkpoint ✅

1. Deploy an ERC-721 contract on anvil, mint 5 NFTs, and transfer one
2. Create an NFT marketplace listing and execute a purchase
3. Implement an ERC-1155 game items contract with 4 token types
4. Write a test that verifies royalty information is returned correctly

---

## Common Pitfalls

| Pitfall | Clarification |
|---------|---------------|
| `safeMint` vs `mint` | `safeMint` checks if recipient can handle NFTs (via `onERC721Received`) |
| Storing images on-chain | Extremely expensive; use IPFS or Arweave for media |
| Not implementing `tokenURI` correctly | Must return valid JSON conforming to the metadata standard |
| Marketplace escrow issues | Ensure the marketplace has approval before listing |

---

## Further Reading

- [EIP-721: Non-Fungible Token Standard](https://eips.ethereum.org/EIPS/eip-721)
- [EIP-1155: Multi Token Standard](https://eips.ethereum.org/EIPS/eip-1155)
- [EIP-2981: Royalty Standard](https://eips.ethereum.org/EIPS/eip-2981)
- [OpenSea Metadata Standards](https://docs.opensea.io/docs/metadata-standards)

---

**Previous →** [Module 13: ERC-20 Tokens](./13-erc20-tokens.md)  
**Next →** [Module 15: DeFi Foundations](./15-defi-foundations.md)
