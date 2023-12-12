Create a Cross-EVM bridge using Solidity, Typescript, Hardhat & ethers.js
Thank for source code Angsin
# Prerequisite 
* Node.js 
* npm 

If you know what you’re doing and would like to get straight to the source code, you can find it here: https://github.com/AngSin/cross-evm-bridge. But if you would like to follow along step by step, let’s get to it!

# Project Set Up
Let’s create the directory we will be working in:

```bash
mkdir cross-evm-bridge
cd cross-evm-bridge
npm init -y
npm install --save-dev hardhat
```

# initialize the hardhat project
```bash
npx hardhat init
```
Go down and pick the Create a `TypeScript` Projectoption

# Writing the Contracts
We will need a sample `ERC20 token contract` to test out our bridge with. To do this, let’s install OpenZeppelin, which will give us the base for our `ERC20 token contract`:
```bash
npm i @openzeppelin/contracts
```

`OpenZeppelin uses 0.8.20` version of `Solidity`, so let’s also go to our `hardhat.config.ts` file and change the version 
```bash
// hardhat.config.ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";

const config: HardhatUserConfig = {
  solidity: "0.8.20",
};

export default config;
```
Our set up for `OpenZeppelin` is now complete. We can now use it to quickly create a basic `ERC20 token` like so:
```bash
// contracts/MyToken.sol
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MyToken is ERC20 {
    constructor() ERC20("MyToken", "MTK") {
        _mint(msg.sender, 2_000 ether);
    }
}
```
We’re starting with the fun part now. Let’s make the bridge contract that will accept the tokens on one chain and be able to release them on another. To do this we must realise that we have to communicate with the other chain, so there must be some events that our bridge contract emits to communicate that tokens have been deposited on one chain and are ready to be released on another.
this is how a bare-bones bridge contract should look like:
```bash
// contracts/Bridge.sol
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract Bridge is Ownable {
    address public token;
    event Deposit(address by, uint256 amount);

    constructor(address _token) Ownable(msg.sender) {
        token = _token;
    }

    function deposit(uint256 _amount) public {
        IERC20(token).transferFrom(msg.sender, address(this), _amount);
        emit Deposit(msg.sender, _amount);
    }

    function release(address _to, uint256 _amount) public onlyOwner {
        IERC20(token).transfer(_to, _amount);
    }
}
```
# Deploying the `Contracts`
Our contracts are now ready! Let’s get to deploying them. To do that, we will need a few things such as some `RPC’s` for the network’s we want to deploy to, a `private key` of a wallet that is funded with some ETH or native token of that chain (to pay for the deployment gas fees). For this tutorial we will be using the `sepolia testnet` and the `mumbai testnet`. To receive funds for these testnets, you can use https://sepoliafaucet.com/ and https://mumbaifaucet.com/ respectively. 
Once your wallet is funded, let’s get you some RPC’s,
you can get them at: https://dashboard.alchemy.com/. Here, go to `create new` app and create one for `Ethereum’s testnet Sepolia` and one for `Polygon’s testnet Sepolia`. Once you have created the two App on `Alchemy`, you will be able to view their api keys, like so:


The part that is interesting to us right now is the `HTTPS api` so let’s copy those for both the `sepolia` and `mumbai` apps and paste them into a new file called `.env` at the root of the project. Provided you have also managed to fund your wallet with some `testnet tokens`, you can get the `private key` of that wallet on `metamask`, prefix it with a 0x and you can then add all these `secrets/keys` in your `.env` file like so:
```bash
# .env
SEPOLIA_RPC_URL=<YOUR_SEPOLIA_RPC_URL>
MUMBAI_RPC_URL=<YOUR_MUMBAI_RPC_URL>
PRIVATE_KEY=0x<YOUR_PRIVATE_KEY>
```
Ok, now how do we use these variables in our code to deploy the `contracts`? all of this will be part of the configuration for hardhat (in the `hardhat.config.ts` file), but to do that, we will need a node library called `dotenv`, which will let our code source these variables into the environment. 
# install the library
```bash
npm i dotenv
```
Now we can use our `testnet RPC` urls by changing the config code to:
```bash
// hardhat.config.ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import * as dotenv from "dotenv";

dotenv.config();

const config: HardhatUserConfig = {
  solidity: {
    version: "0.8.20",
    settings: {
      optimizer: {
        enabled: true,
      },
    },
  },
  networks: {
    hardhat: {},
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL!!,
      accounts: [process.env.PRIVATE_KEY!!],
    },
    mumbai: {
      url: process.env.MUMBAI_RPC_URL!!,
      accounts: [process.env.PRIVATE_KEY!!],
    },
  },
};

export default config;
```
Note that in addition to adding the testnet network configurations, we have also added some optimization to our solidity compiler setting, this will make the deployment more gas efficient.

If you’ve peaked in the repository you will notice that bootstrapping our project with hardhat came with a deployment script written for us in `scripts/deploy.ts`. However, it was for the example contract that Hardhat bootstrapped the project with. So let’s change the script so it can deploy our 2 contracts and also fund our bridge contract with some tokens:
```bash
// scripts/deploy.ts
import { ethers } from "hardhat";

async function main() {
  // deploy the token contract
  const token = await ethers.deployContract("MyToken");
  await token.waitForDeployment();
  const tokenAddress = await token.getAddress();
  console.log(`Token contract deployed on: ${tokenAddress}`);
  // deploy the bridge contract
  const bridge = await ethers.deployContract("Bridge", [tokenAddress]);
  await bridge.waitForDeployment();
  const bridgeAddress = await bridge.getAddress();
  console.log(`Bridge contract deployed on: ${bridgeAddress}`);
  // fund the bridge contract with some tokens
  await token.transfer(bridgeAddress, ethers.parseEther('1000'));
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```
# Deploy contract
We will now deploy our contracts on both `sepolia` and `mumbai`. Let’s first start with `sepolia`. 
Run the following command to deploy:
```bash
npx hardhat run scripts/deploy.ts --network sepolia
```

This is great! We have now have the addresses of our contracts which we can check out on `https://sepolia.etherscan.io`. 
This will help us test out our contracts directly on etherscan. Make sure you save these contract addresses somewhere for ease, but also remember that these will also be available on `https://sepolia.etherscan.io` if you look at your wallet’s history.

# Deployment on mumbai:
```bash
npx hardhat run scripts/deploy.ts --network mumbai 
```
If you would also like to check out the contracts on Polygon’s Mumbai testnet, you can do that here: `https://mumbai.polygonscan.com/`.
You may notice, that the contracts are not verified, i.e. you can’t interact with them directly on `etherscan/polygonscan` yet because these services don’t have the code for the contracts yet:

Let’s fix this.

# Verifying the Contracts
Firstly, 
create an account on etherscan (`https://etherscan.io/login`) and polygonscan (`https://polygonscan.com/login`), and get yourself `API Keys` for both the explorers. Once you have them, add them to your `.env` file like so:
```bash
# /.env
SEPOLIA_RPC_URL=<YOUR_SEPOLIA_RPC_URL>
MUMBAI_RPC_URL=<YOUR_MUMBAI_RPC_URL>
PRIVATE_KEY=0x<YOUR_PRIVATE_KEY>
ETHERSCAN_API_KEY=<YOUR_ETHERSCAN_API_KEY>
POLYGONSCAN_API_KEY=<YOUR_POLYGONSCAN_API_KEY>
```
# Edit `hardhat` configuration
Now let’s edit our `hardhat` configuration to:
```bash
// harhdat.config.ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import * as dotenv from "dotenv";

dotenv.config();

const isMumbaiNetwork = process.argv.includes('mumbai');

const config: HardhatUserConfig = {
  solidity: {
    version: "0.8.20",
    settings: {
      optimizer: {
        enabled: true,
      },
    },
  },
  networks: {
    hardhat: {},
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL!!,
      accounts: [process.env.PRIVATE_KEY!!],
    },
    mumbai: {
      url: process.env.MUMBAI_RPC_URL!!,
      accounts: [process.env.PRIVATE_KEY!!],
    },
  },
  etherscan: {
    apiKey: isMumbaiNetwork ? process.env.POLYGONSCAN_API_KEY!! : process.env.ETHERSCAN_API_KEY!!,
  },
};

export default config;
```
You may note that we now have a condition to check if the network we are deploying to is mumbai or not, if it is we will use the `polygonscan api key`, otherwise we will resort to the etherscan api key.

Now we can resume to verify our contracts on the 2 blockchain explorers by running these commands:
```bash
npx hardhat verify --network sepolia <YOUR_TOKEN_ADDRESS>
npx hardhat verify --network sepolia <YOUR_BRIDGE_ADDRESS> <YOUR_TOKEN_ADDRESS>
npx hardhat verify --network mumbai <YOUR_TOKEN_ADDRESS>
npx hardhat verify --network mumbai <YOUR_BRIDGE_ADDRESS> <YOUR_TOKEN_ADDRESS>
```

Now if we go back to Polygonscan or etherscan, we will be able to interact with our contracts 

However, if we start testing the bridge right now, they will not work as intended, this is because currently there is no connection between the two chains. We will write a small bot/backend that will do so using `Node.js`, `Typescript` and `ethers.js`.

# Creating a Bridging Bot
Before we get into writing the code, we will need to expand our `.env` file to include the addresses of all our contracts as well as the `WebSocket` URLs of our apps from `Alchemy`. I.e. from the WebSocket option here:
Once you’ve got all this info, your .env file should look something like this:
```bash
# .env
SEPOLIA_RPC_URL=<YOUR_SEPOLIA_RPC_URL>
SEPOLIA_WS_RPC_URL=<YOUR_SEPOLIA_WS_RPC_URL>

SEPOLIA_TOKEN_ADDRESS=<YOUR_SEPOLIA_TOKEN_ADDRESS>
SEPOLIA_BRIDGE_ADDRESS=<YOUR_SEPOLIA_BRIDGE_ADDRESS>

MUMBAI_RPC_URL=<YOUR_MUMBAI_RPC_URL>
MUMBAI_WS_RPC_URL=<YOUR_MUMBAI_WS_RPC_URL>

MUMBAI_TOKEN_ADDRESS=<YOUR_MUMBAI_TOKEN_ADDRESS>
MUMBAI_BRIDGE_ADDRESS=<YOUR_MUMBAI_BRIDGE_ADDRESS>

PRIVATE_KEY=0x<YOUR_PRIVATE_KEY>

ETHERSCAN_API_KEY=<YOUR_ETHERSCAN_API_KEY>
POLYGONSCAN_API_KEY=<YOUR_POLYGONSCAN_API_KEY>
```
# Config Bot code
let’s write the code of our bot that will release the tokens by interacting with our bridge contract:
```bash
import {
  AddressLike,
  BigNumberish,
  Contract,
  WebSocketProvider,
  formatEther,
  Wallet,
  JsonRpcProvider,
} from "ethers";
import { Bridge__factory } from "./typechain-types";
import * as dotenv from "dotenv";

dotenv.config();

const sepoliaProvider = new WebSocketProvider(
  process.env.SEPOLIA_WS_RPC_URL!!,
  {
    name: "sepolia",
    chainId: 11155111,
  },
);

const sepoliaBridgeContract = new Contract(
  process.env.SEPOLIA_BRIDGE_ADDRESS!!,
  Bridge__factory.abi,
  sepoliaProvider,
);

console.log("bot listening!");

sepoliaBridgeContract.on(
  "Deposit",
  async (depositor: AddressLike, amount: BigNumberish) => {
    const provider = new JsonRpcProvider(process.env.MUMBAI_RPC_URL!!, {
      name: "mumbai",
      chainId: 80001,
    });
    const sender = new Wallet(process.env.PRIVATE_KEY!!, provider);
    const bridge = Bridge__factory.connect(
      process.env.MUMBAI_BRIDGE_ADDRESS!!,
      sender,
    );
    console.log(`sending ${formatEther(amount)} tokens to ${depositor}`);
    await bridge.release(depositor, amount);
    console.log(
      `sent ${formatEther(amount)} tokens to ${depositor} on Mumbai contract: ${
        process.env.MUMBAI_BRIDGE_ADDRESS
      }`,
    );
  },
);
```
You may notice some `errors` when you paste the bot code into your repository. This is because the `contract’s` types & artifacts have not yet been generated. Do this by running:
```bash
npx hardhat compile
```
Once that is done, 
the `bot.ts` file should have no errors. We now have a bridge that is able to listen for a deposit on sepolia and fund the user’s wallets on mumbai. Let’s test this out, by first running the bot by running the command:
```bash
npx ts-node bot.ts
```
Now to test, let’s go to our token’s contract page on `https://sepolia.etherscan.io`. 
We first have to approve the bridge contract to be able to deposit tokens. 
Let’s do this by calling the function approve in the Write Contract section of our Token contract. 
We will need the address of our bridge contract and the wei value of however many tokens we want to transfer (which you can get here: `https://eth-converter.com/`), so for `100 tokens`, the Wei value would be `100000000000000000000`: 
If we put it together, our function call should look something like:

Once you write this transaction and it succeeds. We can try depositing some tokens into this bridge by going to the `https://sepolia.etherscan.io` page of our bridge contract and calling the deposit function. 
I decided to deposit 1 Token of our `ERC-20` token (MyToken) into the bridge.
After a successful transaction I saw the following message on my bot’s console output:

This is great! the bridge bot seems to have worked, let’s check it out by going to the `https://mumbai.polygonscan.com` and looking at the last transaction made by our mumbai testnet’s bridge contract. 
Voila, we see a transaction made by the bot which sent 1 MTK token to the user.

You may notice that right now our bridge is simply one way, it accepts tokens on sepolia and transfers them out from mumbai. 
Do you think you can make the bridge two way? 
hint: look at the code in `bot.ts`, you will be able to `copy paste` a lot of the code and make the bridge work both ways!

One thing we have not looked into, in this article is what would happen if a bridge contract has no tokens left. 
With our current code, the transaction on the other chain will still pass, however the user will not receive any tokens on their destination chain. This will be a major issue!

To fix this, we would need some communication not just on Deposit but also on Release,
so each time a token is released, we should know how many tokens exist on the destination chain and update them in the source `bridge contract`. 
As mentioned in the start of the article, liquidity issues are out of the scope of this article so we will not get into how to implement this. 
However, it is good to know that these issues exist and a bridge in a production environment will require more thinking, testing and supporting infrastructure.

We have also not paid much attention to the security aspect of this application. 
We have currently stored the “admin” `private key` in a file (`.env`), however, in a production environment,
we should use a SCW (Smart contract wallet), multi-sig wallet or another such layer of abstraction which would make exploits of the bridge harder to maintain. 
Bigger projects are also experimenting with decentralizing the “admin” role which is ideal but more complicated.
