# Building a Digital Marketplace on Ethereum

![This workshop will walk you through creating a digital marketplace on Ethereum.
](header.png)

The technologies used in this workshop are [React](https://reactjs.org/), [Next.js](https://nextjs.org/), [Tailwind CSS](https://tailwindcss.com/), [HardHat](https://hardhat.org/), [Solidity](https://docs.soliditylang.org/en/v0.8.5/), and [Ethers](https://docs.ethers.io/v5/).

### Getting started

The first thing we need to do is write the smart contracts.

The marketplace will consist of two main smart contracts:

- NFT Contract for minting NFTs
- Markeplace contract for facilitating the sale of NFTs

For writing an NFT, we can use the [ERC721](https://eips.ethereum.org/EIPS/eip-721) standard that is available via [OpenZeppelin](https://docs.openzeppelin.com/contracts/3.x/erc721).

To get started, head over to [https://remix.ethereum.org/](https://remix.ethereum.org/) and create a new file called __Marketplace.sol__.

Here, we can get started by importing the contracts that we'll be needing to get started from Open Zeppelin:

```solidity
// SPDX-License-Identifier: MIT OR Apache-2.0
pragma solidity ^0.8.3;

import "@openzeppelin/contracts/utils/Counters.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

import "hardhat/console.sol";
```

### Creating the NFT contract

Next, let's create the NFT contract:

```solidity
contract NFT is ERC721URIStorage {
    using Counters for Counters.Counter;
    Counters.Counter private _tokenIds;
    address contractAddress;

    constructor(address marketplaceAddress) ERC721("Eat The Blocks NFTs", "ETBNFT") {
        contractAddress = marketplaceAddress;
    }

    function createToken(string memory tokenURI) public returns (uint) {
        _tokenIds.increment();
        uint256 newItemId = _tokenIds.current();

        _mint(msg.sender, newItemId);
        _setTokenURI(newItemId, tokenURI);
        setApprovalForAll(contractAddress, true);
        return newItemId;
    }
}
```

The contstructor takes an argument for the `marketplaceAddress` address, saving the value and making it available in the smart contract. This way when someone calls `createToken`, the contract can allow the Market contract approval to transfer the token away from the owner to the seller.

The `newItemId` value is returned from the function as we will be needing it in our client application to know the dynamic value of the `tokenId` that was generated by the smart contract.

### Creating the Market contract

The Market contract is more complex than the NFT contract since we will be writing all of it from scratch.

```solidity
contract NFTMarket is ReentrancyGuard {
  using Counters for Counters.Counter;
  Counters.Counter private _itemIds;
  Counters.Counter private _itemsSold;
  uint[] marketItems;

  struct MarketItem {
    uint itemId;
    address nftContract;
    uint256 tokenId;
    address payable seller;
    address payable owner;
    uint256 price;
  }

  mapping(uint256 => MarketItem) private idToMarketItem;

  event MarketItemCreated (
    uint indexed itemId,
    address indexed nftContract,
    uint256 indexed tokenId,
    address seller,
    address owner,
    uint256 price
  );

  function getMarketItem(uint256 marketItemId) public view returns (MarketItem memory) {
    return idToMarketItem[marketItemId];
  }

  function createMarketItem(
    address nftContract,
    uint256 tokenId,
    uint256 price
  ) public payable nonReentrant {
    require(price > 0, "Price must be at least 1 wei");

    _itemIds.increment();
    uint256 itemId = _itemIds.current();
    marketItems.push(itemId);
  
    idToMarketItem[itemId] =  MarketItem(
      itemId,
      nftContract,
      tokenId,
      payable(msg.sender),
      payable(address(0)),
      price
    );

    IERC721(nftContract).transferFrom(msg.sender, address(this), tokenId);

    emit MarketItemCreated(
      itemId,
      nftContract,
      tokenId,
      msg.sender,
      address(0),
      price
    );
  }

  function createMarketSale(
    address nftContract,
    uint256 itemId
    ) payable public {
    uint price = idToMarketItem[itemId].price;
    uint tokenId = idToMarketItem[itemId].tokenId;
    require(msg.value == price, "Please submit the asking price in order to complete the purchase");

    idToMarketItem[itemId].seller.transfer(msg.value);
    IERC721(nftContract).transferFrom(address(this), msg.sender, tokenId);
    idToMarketItem[itemId].owner = payable(msg.sender);
    _itemsSold.increment();
  }

  function fetchMarketItem(uint itemId) public view returns (MarketItem memory) {
    MarketItem memory item = idToMarketItem[itemId];
    return item;
  }

  function fetchMarketItems() public view returns (MarketItem[] memory) {
    uint itemCount = _itemIds.current();
    uint unsoldItemCount = _itemIds.current() - _itemsSold.current();
    uint currentIndex = 0;

    MarketItem[] memory items = new MarketItem[](unsoldItemCount);
    for (uint i = 0; i < itemCount; i++) {
      if (idToMarketItem[i + 1].owner == address(0)) {
        uint currentId = idToMarketItem[i + 1].itemId;
        MarketItem storage currentItem = idToMarketItem[currentId];
        items[currentIndex] = currentItem;
        currentIndex += 1;
      }
    }
   
    return items;
  }

  function fetchMyNFTs() public view returns (MarketItem[] memory) {
    uint totalItemCount = _itemIds.current();
    uint itemCount = 0;
    uint currentIndex = 0;

    for (uint i = 0; i < totalItemCount; i++) {
      if (idToMarketItem[i + 1].owner == msg.sender) {
        itemCount += 1;
      }
    }

    MarketItem[] memory items = new MarketItem[](itemCount);
    for (uint i = 0; i < totalItemCount; i++) {
      if (idToMarketItem[i + 1].owner == msg.sender) {
        uint currentId = idToMarketItem[i + 1].itemId;
        MarketItem storage currentItem = idToMarketItem[currentId];
        items[currentIndex] = currentItem;
        currentIndex += 1;
      }
    }
   
    return items;
  }
}
```

There is a lot going on in this contract, so let's walk through some of it.

#### What is ReentrancyGuard

Inheriting from [`ReentrancyGuard`](https://docs.openzeppelin.com/contracts/2.x/api/utils#ReentrancyGuard) will make the `nonReentrant` modifier available, which can be applied to functions to make sure there are no nested (reentrant) calls to them.

#### MarketItem

The `MarketItem` struct allows us to store records of items that we want to make available in the marketplace.

#### idToMarketItem

This mapping allows us to create a key value pairing between IDs and `MarketItem`s.

#### createMarketItem

This function transfers an NFT to the contract address of the market, and puts the item for sale.

#### createMarketSale

This function enables the transfer of the NFT as well as Eth between the buyer and seller.

#### fetchMarketItems

This function returns all market items that are still for sale.

#### fetchMyNFTs

This function returns the NFTs that the user has purchased.

## Building the front end

To build the front end, you can use the starting project in the __marketplace_starter__ folder.

<details>
  <summary>If you'd like to know how to bootstrap the project from scratch yourself, follow these steps</summary>
  
  1. Create a new Next.js app and change into the directory:

  ```sh
  npx create-next-app marketplace-app
  cd marketplace-app
  ```

  2. Install the necessary dependencies:

  ```sh
  npm install web3modal ethers hardhat @nomiclabs/hardhat-waffle ethereum-waffle chai \ 
  @nomiclabs/hardhat-ethers axios ipfs-http-client @openzeppelin/contracts
  ```

  Optional, install tailwind and dependencies:
  ```sh
  npm install -D tailwindcss@latest postcss@latest autoprefixer@latest

  npx tailwindcss init -p

  update styles/globals.css with the following
  @tailwind base;
  @tailwind components;
  @tailwind utilities;
  ```

  3. Initialize Hardhat environment

  ```sh
  npx hardhat
  ? What do you want to do? Create a sample project
  ? Hardhat project root: <Choose default path>
  ```

  4. Open hardhat.config.js and update the module.exports to look like this:

  ```javascript
  module.exports = {
    solidity: "0.8.3",
    paths: {
      artifacts: './src/artifacts',
    },
    networks: {
      hardhat: {
        chainId: 1337
      }
    }
  };
  ```
</details>

### Dowloading and configuring the project

Clone the repo, then change into the __marketplace-starter__ directory, and install the dependencies:

```sh
git clone https://github.com/dabit3/full-stack-ethereum-marketplace-workshop.git

cd full-stack-ethereum-marketplace-workshop/marketplace-starter

yarn

# or

npm install
```

Next, take a look at the tests in __test/test.js__. Here, we can mock the functionality that we'll need for our project.

To run the test and see the output, run the following command:

```sh
npx hardhat test
```

### The smart contracts

The smart contracts are located in the __contracts__ directory.

To use the smart contracts in our front-end (React) applications, we'll need to compile the contract into ABIs.

The Hardhat development environment allows us to do this using the Hardhat CLI.

We can define the location of the artifacts output by configuring its location in __hardhat.config.js__ along with any other properties for our Hardhat development environment.

In our project, the location is set to __./artifacts__ in the root directory.

### Writing the deployment script

Next, let's write the script to deploy the contracts.

To do so, open __scripts/deploy.js__ and add the following to the `main` function body:

```javascript
async function main() {
  const NFTMarket = await hre.ethers.getContractFactory("NFTMarket");
  const nftMarket = await NFTMarket.deploy();
  await nftMarket.deployed();
  console.log("nftMarket deployed to:", nftMarket.address);

  const NFT = await hre.ethers.getContractFactory("NFT");
  const nft = await NFT.deploy(nftMarket.address);
  await nft.deployed();
  console.log("nft deployed to:", nft.address);
}
```

### Running a local node and deploying the contract

Next, let's run a local node and deploy our contract to the node.

To do so, open your terminal and run the following command:

```sh
npx hardhat node
```

This should create and launch a local node.

Next, open another terminal and deploy the contracts:

```sh
npx hardhat run scripts/deploy.js --network localhost
```

When this completes successfully, you should have the addresses printed out to the console for your smart contract deployments.

Take those values and update the configuration in __config.js__:

```javascript
export const nftmarketaddress = "market-contract-address"
export const nftaddress = "nft-contract-address"
```

### Running the app

Now, we should be able to run the app.

To do so, open the terminal and run the following command:

```sh
npm run dev
```

## Building the UI

### Fetching NFTs

The first piece of functionality we'll implement will be fetching NFTs from the marketplace and rendering them to the UI.

To do so, open __pages/index.js__ and update the `loadNFTs` function with the following:

```javascript
async function loadNFTs() {
  const provider = new ethers.providers.JsonRpcProvider()
  const tokenContract = new ethers.Contract(nftaddress, NFT.abi, provider)
  const marketContract = new ethers.Contract(nftmarketaddress, Market.abi, provider)
  const data = await marketContract.fetchMarketItems()
  
  const items = await Promise.all(data.map(async i => {
    const tokenUri = await tokenContract.tokenURI(i.tokenId)
    const meta = await axios.get(tokenUri)
    let price = web3.utils.fromWei(i.price.toString(), 'ether');
    let item = {
      price,
      tokenId: i.tokenId.toNumber(),
      seller: i.seller,
      owner: i.owner,
      image: meta.data.image,
    }
    return item
  }))
  setNfts(items)
  setLoaded('loaded') 
}
```

The `loadNFTs` function calls the `fetchMarketItems` function in our smart contract and returns all unsold NFTs.

We then map over each NFT to transform the response into something more readable for the user interface.

### Creating an NFT and placing it for sale

Next, let's create the function for minting the NFT and putting it for sale in the market.

To do so, open __pages/create-item.js__ and update the `createSale` function with the following code:

```javascript
async function createSale(url) {
  const web3Modal = new Web3Modal({
    network: "mainnet",
    cacheProvider: true,
  });
  const connection = await web3Modal.connect()
  const provider = new ethers.providers.Web3Provider(connection)    
  const signer = provider.getSigner()
  
  let contract = new ethers.Contract(nftaddress, NFT.abi, signer)
  let transaction = await contract.createToken(url)
  let tx = await transaction.wait()
  let event = tx.events[0]
  let value = event.args[2]
  let tokenId = value.toNumber()
  const price = web3.utils.toWei(formInput.price, 'ether')

  contract = new ethers.Contract(nftmarketaddress, Market.abi, signer)
  transaction = await contract.createMarketItem(nftaddress, tokenId, price)
  await transaction.wait()
  router.push('/')
}
```

This function writes two transactions to the network:

__createToken__ mints the new token.

__createMarketItem__ places the item for sale.

Once the item has been crated, we redict the user back to the main page.

### Allowing a user to purchase an NFT

Next, let's create the function for allowing a user to purchase an NFT.

To do so, open __pages/index.js__ and update the `buyNft` function with the following code:

```javascript
async function buyNft(nft) {
  const web3Modal = new Web3Modal({
    network: "mainnet",
    cacheProvider: true,
  });
  const connection = await web3Modal.connect()
  const provider = new ethers.providers.Web3Provider(connection)
  const signer = provider.getSigner()
  const contract = new ethers.Contract(nftmarketaddress, Market.abi, signer)
  
  const price = web3.utils.toWei(nft.price.toString(), 'ether');
  
  const transaction = await contract.createMarketSale(nftaddress, nft.tokenId, {
    value: price
  })
  await transaction.wait()
  loadNFTs()
}
```

This function writes the `createMarketSale` to the network, allowing the user to transfer the amount from their wallet to the seller's wallet.

### Allowing a user to view their own NFTs

The last piece of functionality we want to implement is to give users the ability to view the NFTs they have purchased. To do so, we'll be calling the `fetchMyNFTs` function from the smart contract.

Open __pages/my-nfts.js__ and update the `loadNFTs` function with the following:

```javascript
async function loadNFTs() {
  const web3Modal = new Web3Modal({
    network: "mainnet",
    cacheProvider: true,
  });
  const connection = await web3Modal.connect()
  const provider = new ethers.providers.Web3Provider(connection)
  const signer = provider.getSigner()
    
  const marketContract = new ethers.Contract(nftmarketaddress, Market.abi, signer)
  const tokenContract = new ethers.Contract(nftaddress, NFT.abi, provider)
  const data = await marketContract.fetchMyNFTs()
  
  const items = await Promise.all(data.map(async i => {
    const tokenUri = await tokenContract.tokenURI(i.tokenId)
    const meta = await axios.get(tokenUri)
    let price = web3.utils.fromWei(i.price.toString(), 'ether');
    let item = {
      price,
      tokenId: i.tokenId.toNumber(),
      seller: i.seller,
      owner: i.owner,
      image: meta.data.image,
    }
    return item
  }))

  setNfts(items)
  setLoaded('loaded') 
}
```

## Enabling listing fees

Next, we want the operator of the marketplace to collect listing fees. We can add this functionality with only a few lines of code.

First, open the __NFTMarket.sol__ contract located in the __contracts__ directory.

Here, we will set a listing price that you want to be using. We will also, create a variable that we can use to store the owner of the contract.

Add the following lines of code below the `marketItems` variable initialization:

```solidity
address payable owner;
uint256 listingPrice = 0.01 ether;
```

Next, create a `constructor` to store the contract owner's address:


```solidity
constructor() {
  owner = payable(msg.sender);
}
```

In the `createMarketItem` function, set a check to make sure that the listing fee has been passed into the transaction:

```solidity
require(price > 0, "Price must be at least 1 wei"); // below this line 👇
require(msg.value == listingPrice, "Price must be equal to listing price");
```

Finally, in the `createMarketSale` function, send the `listingPrice` value to the contract owner once the sale is completed. This can go at the end of the function:

```solidity
payable(owner).transfer(listingPrice);
```

Next, in the client-side code, open __pages/create-item.js__ and add the payment value to be sent along with the transaction in the `createSale` function:

```javascript
// above code omitted
const listingPrice = web3.utils.toWei('0.01', 'ether')

contract = new ethers.Contract(nftmarketaddress, Market.abi, signer)
transaction = await contract.createMarketItem(nftaddress, tokenId, price, { value: listingPrice })
// below code omitted
```

### Conclusion

The project should now be complete. You should be able to create, view, and purchase NFTs from the marketplace.

To view a completed version of the project, check out the code located in the __marketplace-complete__ folder.