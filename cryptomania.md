# cryptomania-nft
It is a Foundry ERC-721 collection with minting, max supply, owner controls, and royalties.
mkdir cryptomania && cd cryptomania
git init
gh repo create cryptomania --public --source=. --remote=origin
cryptomania/
├── README.md
├── LICENSE
├── foundry.toml
├── remappings.txt
├── .gitignore
├── .env.example
├── src/
│   └── Cryptomania.sol
├── script/
│   └── Deploy.s.sol
├── test/
│   └── Cryptomania.t.sol
└── metadata/
    └── 1.json
   # Foundry
cache/
out/
broadcast/

# Env / secrets
.env
.env.local

# OS / editor
.DS_Store
.idea/
.vscode/

# Node (if you add a frontend later)
node_modules/
PRIVATE_KEY=
RPC_URL=
ETHERSCAN_API_KEY=

# Optional collection settings used by the deploy script
NAME=Cryptomania
SYMBOL=CMANIA
BASE_URI=ipfs://YOUR_CID/
MAX_SUPPLY=10000
MINT_PRICE_WEI=10000000000000000
ROYALTY_RECEIVER=
ROYALTY_BPS=500
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
solc = "0.8.24"
optimizer = true
optimizer_runs = 200
via_ir = true
ffi = false

[fmt]
line_length = 100
tab_width = 4
bracket_spacing = true

[rpc_endpoints]
sepolia = "${RPC_URL}"
mainnet = "${RPC_URL}"

[etherscan]
sepolia = { key = "${ETHERSCAN_API_KEY}" }
forge-std/=lib/forge-std/src/
openzeppelin-contracts/=lib/openzeppelin-contracts/contracts/
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC721} from "openzeppelin-contracts/token/ERC721/ERC721.sol";
import {ERC721Royalty} from "openzeppelin-contracts/token/ERC721/extensions/ERC721Royalty.sol";
import {Ownable} from "openzeppelin-contracts/access/Ownable.sol";
import {ReentrancyGuard} from "openzeppelin-contracts/utils/ReentrancyGuard.sol";
import {Strings} from "openzeppelin-contracts/utils/Strings.sol";

/// @title Cryptomania
/// @notice Limited ERC-721 collection with paid minting and EIP-2981 royalties.
contract Cryptomania is ERC721, ERC721Royalty, Ownable, ReentrancyGuard {
    using Strings for uint256;

    uint256 public immutable maxSupply;
    uint256 public mintPrice;
    uint256 public totalMinted;
    bool public mintOpen;
    string private _baseTokenURI;

    error MintClosed();
    error SoldOut();
    error IncorrectPayment();
    error ZeroAddress();
    error InvalidConfig();

    event MintStateChanged(bool open);
    event MintPriceChanged(uint256 newPrice);
    event BaseURIChanged(string newBaseURI);

    constructor(
        string memory name_,
        string memory symbol_,
        string memory baseURI_,
        uint256 maxSupply_,
        uint256 mintPrice_,
        address royaltyReceiver_,
        uint96 royaltyFeeBps_,
        address initialOwner_
    ) ERC721(name_, symbol_) Ownable(initialOwner_) {
        if (maxSupply_ == 0) revert InvalidConfig();
        if (royaltyReceiver_ == address(0) || initialOwner_ == address(0)) revert ZeroAddress();
        if (royaltyFeeBps_ > 1000) revert InvalidConfig(); // cap at 10%

        maxSupply = maxSupply_;
        mintPrice = mintPrice_;
        _baseTokenURI = baseURI_;
        _setDefaultRoyalty(royaltyReceiver_, royaltyFeeBps_);
    }

    function mint(uint256 quantity) external payable nonReentrant {
        if (!mintOpen) revert MintClosed();
        if (quantity == 0) revert InvalidConfig();
        if (totalMinted + quantity > maxSupply) revert SoldOut();
        if (msg.value != mintPrice * quantity) revert IncorrectPayment();

        uint256 startId = totalMinted + 1;
        totalMinted += quantity;

        for (uint256 i = 0; i < quantity; ++i) {
            _safeMint(msg.sender, startId + i);
        }
    }

    function ownerMint(address to, uint256 quantity) external onlyOwner {
        if (to == address(0)) revert ZeroAddress();
        if (quantity == 0) revert InvalidConfig();
        if (totalMinted + quantity > maxSupply) revert SoldOut();

        uint256 startId = totalMinted + 1;
        totalMinted += quantity;

        for (uint256 i = 0; i < quantity; ++i) {
            _safeMint(to, startId + i);
        }
    }

    function setMintOpen(bool open) external onlyOwner {
        mintOpen = open;
        emit MintStateChanged(open);
    }

    function setMintPrice(uint256 newPrice) external onlyOwner {
        mintPrice = newPrice;
        emit MintPriceChanged(newPrice);
    }

    function setBaseURI(string calldata newBaseURI) external onlyOwner {
        _baseTokenURI = newBaseURI;
        emit BaseURIChanged(newBaseURI);
    }

    function setDefaultRoyalty(address receiver, uint96 feeBps) external onlyOwner {
        if (receiver == address(0)) revert ZeroAddress();
        if (feeBps > 1000) revert InvalidConfig();
        _setDefaultRoyalty(receiver, feeBps);
    }

    function withdraw(address payable to) external onlyOwner {
        if (to == address(0)) revert ZeroAddress();
        to.transfer(address(this).balance);
    }

    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        _requireOwned(tokenId);
        return string.concat(_baseTokenURI, tokenId.toString(), ".json");
    }

    function supportsInterface(bytes4 interfaceId)
        public
        view
        override(ERC721, ERC721Royalty)
        returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
}
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Script, console} from "forge-std/Script.sol";
import {Cryptomania} from "../src/Cryptomania.sol";

contract Deploy is Script {
    function run() external {
        uint256 pk = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(pk);

        string memory name_ = vm.envOr("NAME", string("Cryptomania"));
        string memory symbol_ = vm.envOr("SYMBOL", string("CMANIA"));
        string memory baseURI = vm.envOr("BASE_URI", string("ipfs://YOUR_CID/"));
        uint256 maxSupply = vm.envOr("MAX_SUPPLY", uint256(10_000));
        uint256 mintPrice = vm.envOr("MINT_PRICE_WEI", uint256(0.01 ether));
        address royaltyReceiver = vm.envOr("ROYALTY_RECEIVER", deployer);
        uint96 royaltyBps = uint96(vm.envOr("ROYALTY_BPS", uint256(500)));

        vm.startBroadcast(pk);
        Cryptomania nft = new Cryptomania(
            name_,
            symbol_,
            baseURI,
            maxSupply,
            mintPrice,
            royaltyReceiver,
            royaltyBps,
            deployer
        );
        vm.stopBroadcast();

        console.log("Cryptomania deployed at:", address(nft));
        console.log("Owner:", deployer);
    }
}
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {Cryptomania} from "../src/Cryptomania.sol";

contract CryptomaniaTest is Test {
    Cryptomania internal nft;
    address internal owner = makeAddr("owner");
    address internal user = makeAddr("user");

    function setUp() public {
        nft = new Cryptomania(
            "Cryptomania",
            "CMANIA",
            "ipfs://cid/",
            10,
            0.01 ether,
            owner,
            500,
            owner
        );
        vm.deal(user, 1 ether);
    }

    function testMintWhenOpen() public {
        vm.prank(owner);
        nft.setMintOpen(true);

        vm.prank(user);
        nft.mint{value: 0.01 ether}(1);

        assertEq(nft.ownerOf(1), user);
        assertEq(nft.totalMinted(), 1);
    }

    function testCannotMintWhenClosed() public {
        vm.prank(user);
        vm.expectRevert(Cryptomania.MintClosed.selector);
        nft.mint{value: 0.01 ether}(1);
    }

    function testSoldOut() public {
        vm.prank(owner);
        nft.setMintOpen(true);

        vm.prank(user);
        nft.mint{value: 0.10 ether}(10);

        vm.prank(user);
        vm.expectRevert(Cryptomania.SoldOut.selector);
        nft.mint{value: 0.01 ether}(1);
    }
}
{
  "name": "Cryptomania #1",
  "description": "A collectible from the Cryptomania NFT project.",
  "image": "ipfs://YOUR_IMAGE_CID/1.png",
  "attributes": [
    { "trait_type": "Edition", "value": "Genesis" },
    { "trait_type": "Rarity", "value": "Common" }
  ]
}
# Cryptomania

ERC-721 NFT collection built with Foundry and OpenZeppelin.

## Stack

- Solidity 0.8.24
- Foundry
- OpenZeppelin Contracts
- ERC-721 + EIP-2981 royalties

## Setup

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup

forge init --force
forge install OpenZeppelin/openzeppelin-contracts --no-commit
forge install foundry-rs/forge-std --no-commit
cp .env.example .env
forge build
forge test -vv
forge fmt
cast send $CONTRACT_ADDRESS "setMintOpen(bool)" true \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY

### `LICENSE`

```text
MIT License

Copyright (c) 2026 Cryptomania

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
git add .
git commit -m "Initial Cryptomania NFT repository"
git branch -M main
git push -u origin main
