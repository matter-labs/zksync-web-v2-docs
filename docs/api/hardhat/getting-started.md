# Getting started

[Hardhat](https://hardhat.org) is an Ethereum development environment, designed for easy smart contract development in Solidity. One of its most prominent features is extendability: you can easily add new plugins to your hardhat project.

zkSync has three plugins for hardhat:

- `@matterlabs/hardhat-zksync-solc` for smart contract compilation using Solidity.
- `@matterlabs/hardhat-zksync-vyper` for smart contract compilation using Vyper.
- `@matterlabs/hardhat-zksync-deploy` for smart contract deployment.

To learn more about hardhat itself, check out their [documentation](https://hardhat.org/getting-started/).

This tutorial shows how to set up a zkSync Solidity project using hardhat from scratch.
If you are using Vyper, check out the [reference](./reference.md#matterlabs-hardhat-zksync-vyper) for our Vyper plugin or the [example](https://github.com/matter-labs/hardhat-zksync/tree/main/examples/vyper-example) in our github repo!

## Prerequisites

For this tutorial, the following programs must be installed:

- `yarn` package manager. `npm` examples will be added soon.
- `Docker` for compilation.
- A wallet with some Görli `ETH` on L1 (Görli `USDC` is also required for the ERC-20 tutorial) to pay for bridging funds to zkSync as well as deploying smart contracts.

## Initializing the project

1. Initialize the project and install the dependencies. Run the following commands in the terminal:

```
mkdir greeter-example
cd greeter-example
yarn init -y
yarn add -D typescript ts-node ethers zksync-web3 hardhat @matterlabs/hardhat-zksync-solc @matterlabs/hardhat-zksync-deploy
```

`typescript` and `ts-node` are optional - plugins will work fine in vanilla JavaScript environment. Although, please note that this tutorial _does_ use TypeScript.

2. Create the `hardhat.config.ts` file and paste the following code within it:

```typescript
require("@matterlabs/hardhat-zksync-deploy");
require("@matterlabs/hardhat-zksync-solc");

module.exports = {
  zksolc: {
    version: "1.1.0",
    compilerSource: "docker",
    settings: {
      optimizer: {
        enabled: true,
      },
      experimental: {
        dockerImage: "matterlabs/zksolc",
      },
    },
  },
  zkSyncDeploy: {
    zkSyncNetwork: "https://zksync2-testnet.zksync.dev",
    ethNetwork: "goerli", // Can also be the RPC URL of the network (e.g. `https://goerli.infura.io/v3/<API_KEY>`)
  },
  networks: {
    // To compile with zksolc, this must be the default network.
    hardhat: {
      zksync: true,
    },
  },
  solidity: {
    version: "0.8.12",
  },
};
```

3. Create the `contracts` and `deploy` folders. The former is the place where all the contracts' `*.sol` files should be stored, and the latter is the place where all the scripts related to deploying the contract will be put.

4. Create the `contracts/Greeter.sol` contract and insert the following code within it:

```solidity
//SPDX-License-Identifier: Unlicense
pragma solidity ^0.8.0;

contract Greeter {
    string private greeting;

    constructor(string memory _greeting) {
        greeting = _greeting;
    }

    function greet() public view returns (string memory) {
        return greeting;
    }

    function setGreeting(string memory _greeting) public {
        greeting = _greeting;
    }
}
```

5. Compile the contracts with the following command:

```
yarn hardhat compile
```

A `artifacts-zk` and `cache-zk` folders were created in the root directory (instead of the regular hardhat's `artifacts` and `cache`).
These are compilation artifacts, and should not be added to version control.

6. Create the deployment script in the `deploy/deploy.ts`:

```typescript
import { utils, Wallet } from "zksync-web3";
import * as ethers from "ethers";
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { Deployer } from "@matterlabs/hardhat-zksync-deploy";

// An example of a deploy script that will deploy and call a simple contract.
export default async function (hre: HardhatRuntimeEnvironment) {
  console.log(`Running deploy script for the Greeter contract`);

  // Initialize the wallet.
  const wallet = new Wallet("<WALLET-PRIVATE-KEY>");

  // Create deployer object and load the artifact of the contract we want to deploy.
  const deployer = new Deployer(hre, wallet);
  const artifact = await deployer.loadArtifact("Greeter");

  // Deposit some funds to L2 in order to be able to perform L2 transactions.
  const depositAmount = ethers.utils.parseEther("0.001");
  const depositHandle = await deployer.zkWallet.deposit({
    to: deployer.zkWallet.address,
    token: utils.ETH_ADDRESS,
    amount: depositAmount,
  });
  // Wait until the deposit is processed on zkSync
  await depositHandle.wait();

  // Deploy this contract. The returned object will be of a `Contract` type, similarly to ones in `ethers`.
  // `greeting` is an argument for contract constructor.
  const greeting = "Hi there!";
  const greeterContract = await deployer.deploy(artifact, [greeting]);

  // Show the contract info.
  const contractAddress = greeterContract.address;
  console.log(`${artifact.contractName} was deployed to ${contractAddress}`);

  // Call the deployed contract.
  const greetingFromContract = await greeterContract.greet();
  if (greetingFromContract == greeting) {
    console.log(`Contract greets us with ${greeting}!`);
  } else {
    console.error(`Contract said something unexpected: ${greetingFromContract}`);
  }

  // Edit the greeting of the contract
  const newGreeting = "Hey guys";
  const setNewGreetingHandle = await greeterContract.setGreeting(newGreeting);
  await setNewGreetingHandle.wait();

  const newGreetingFromContract = await greeterContract.greet();
  if (newGreetingFromContract == newGreeting) {
    console.log(`Contract greets us with ${newGreeting}!`);
  } else {
    console.error(`Contract said something unexpected: ${newGreetingFromContract}`);
  }
}
```

7. After replacing the `WALLET-PRIVATE-KEY` text with the `0x`-prefixed private key of your Ethereum wallet, run the script using the following command:

```
yarn hardhat deploy-zksync
```

## Paying fees in ERC20 tokens

This section explains how to pay fees in `USDC` token as an example.

1. After making sure that the wallet has some Görli `USDC` on L1, change the depositing code to the following:

```typescript
const L1_USDC_ADDRESS = "0xd35cceead182dcee0f148ebac9447da2c4d449c4";
const USDC_DECIMALS = 6;

const deploymentFee = await deployer.estimateDeployFee(artifact, [greeting], L2_USDC_ADDRESS);
// Deposit funds to L2.
const depositHandle = await deployer.zkWallet.deposit({
  to: deployer.zkWallet.address,
  token: L1_USDC_ADDRESS,
  // We deposit more than the minimal required amount to have funds
  // for further iteraction with our smart contract.
  amount: deploymentFee.mul(2),
  // Unlike ETH, ERC20 tokens require approval in order to deposit to zkSync.
  // You can either set the approval in a separate transaction or provide `approveERC20` flag equal to `true`
  // and the SDK will initiate approval transaction under the hood.
  approveERC20: true,
});

// Wait until the deposit is processed by zkSync.
await depositHandle.wait();
```

::: tip Note on L2 token address

Note that on the zkSync network, addresses of ERC-20 tokens are different from their addresses on Ethereum. So if, for example, you deposit an ERC-20 token to zkSync, the token address will not be the same as on Ethereum.
To get the address of an ERC-20 token on L2, call the `getL2TokenAddress()` method of the `Provider` / `Wallet` class:

:::

```typescript
const L2_USDC_ADDRESS = await provider.l2TokenAddress(L1_USDC_ADDRESS);
```

2. To output the fee in a human-readable format:

```typescript
const parsedFee = ethers.utils.formatUnits(deploymentFee.toString(), USDC_DECIMALS);
console.log(`The deployment will cost ${parsedFee} USDC`);
```

Please note that the fees on the testnet do not correctly represent the fees on the future mainnet release.

3. Now pass the L2 `USDC` address as the `feeToken` to the deployment transaction:

```typescript
const greeterContract = await deployer.deploy(artifact, [greeting], { feeToken: L2_USDC_ADDRESS });
```

4. To pay fees in USDC for smart contract interaction, supply the fee token in the `customData` override:

```typescript
const setNewGreetingHandle = await greeterContract.setGreeting(newGreeting, {
  customData: {
    feeToken: L2_USDC_ADDRESS,
  },
});
```

#### Full example:

```typescript
import { utils, Wallet, Provider } from "zksync-web3";
import * as ethers from "ethers";
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { Deployer } from "@matterlabs/hardhat-zksync-deploy";

const L1_USDC_ADDRESS = "0xd35cceead182dcee0f148ebac9447da2c4d449c4";
const USDC_DECIMALS = 6;

// An example of a deploy script that will deploy and call a simple contract.
export default async function (hre: HardhatRuntimeEnvironment) {
  console.log(`Running deploy script for the Greeter contract`);

  // Initialize the wallet.
  const provider = new Provider(hre.userConfig.zkSyncDeploy?.zkSyncNetwork);
  const wallet = new Wallet("<WALLET-PRIVATE-KEY>");
  const L2_USDC_ADDRESS = await provider.l2TokenAddress(L1_USDC_ADDRESS);

  // Create deployer object and load the artifact of the contract we want to deploy.
  const deployer = new Deployer(hre, wallet);
  const artifact = await deployer.loadArtifact("Greeter");

  const greeting = "Hi there!";
  const deploymentFee = await deployer.estimateDeployFee(artifact, [greeting], L2_USDC_ADDRESS);

  // Deposit funds to L2
  const depositHandle = await deployer.zkWallet.deposit({
    to: deployer.zkWallet.address,
    token: L1_USDC_ADDRESS,
    amount: deploymentFee.mul(2),
    approveERC20: true,
  });
  // Wait until the deposit is processed on zkSync
  await depositHandle.wait();

  // Deploy this contract. The returned object will be of a `Contract` type, similarly to ones in `ethers`.
  // `greeting` is an argument for contract constructor.
  const parsedFee = ethers.utils.formatUnits(deploymentFee.toString(), USDC_DECIMALS);
  console.log(`The deployment will cost ${parsedFee} USDC`);

  const greeterContract = await deployer.deploy(artifact, [greeting], { feeToken: L2_USDC_ADDRESS });

  // Show the contract info.
  const contractAddress = greeterContract.address;
  console.log(`${artifact.contractName} was deployed to ${contractAddress}`);

  // Call the deployed contract.
  const greetingFromContract = await greeterContract.greet();
  if (greetingFromContract == greeting) {
    console.log(`Contract greets us with ${greeting}!`);
  } else {
    console.error(`Contract said something unexpected: ${greetingFromContract}`);
  }

  // Edit the greeting of the contract
  const newGreeting = "Hey guys";
  const setNewGreetingHandle = await greeterContract.setGreeting(newGreeting, {
    customData: {
      feeToken: L2_USDC_ADDRESS,
    },
  });
  await setNewGreetingHandle.wait();

  const newGreetingFromContract = await greeterContract.greet();
  if (newGreetingFromContract == newGreeting) {
    console.log(`Contract greets us with ${newGreeting}!`);
  } else {
    console.error(`Contract said something unexpected: ${newGreetingFromContract}`);
  }
}
```

5. After replacing the `WALLET-PRIVATE-KEY` text with the `0x`-prefixed private key of the Ethereum wallet, run the script using the following command:

```
yarn hardhat deploy-zksync
```

## Learn more

- New to `zksync-web3` SDK? [Here](../js) is the documentation.
- Want to dive deeper into zkSync `hardhat` plugins? Head straight to the [reference](./reference).

## Future releases

There are two major points of improvements for the plugins which will be released in the future:

- **Composability with the existing hardhat plugins.** Compatibility with other hardhat plugins is planned for future, but has not been a focus yet.
- **Improved cross-platform support.**
