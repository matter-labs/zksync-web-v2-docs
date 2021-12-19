# Accounts

`zksync-web3` exports four classes which can sign transactions on zkSync:

- `Wallet` class is an extension of the `ethers.Wallet` with additional zkSync features.
- `EIP712Signer` class which is used to sign `EIP712`-typed zkSync transactions.
- `Signer` and `L1Signer` classes, which should be used for browser integration.

## `Wallet`

### Creating wallet from private key

Just like `ethers.Wallet`, the `Wallet` object from `zksync-web3` can be created from Ethereum private key.  

```typescript
constructor(
    privateKey: ethers.BytesLike | utils.SigningKey,
    providerL2?: Provider,
    providerL1?: ethers.providers.Provider
)
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| privateKey                   | The Ethereum private key of the account.                                         |
| providerL2 (optional)    | zkSync node provider. Needed to interact with zkSync.                            |
| providerL1 (optional) | Ethereum node provider. Needed to interact with L1. |
| returns                          | `Wallet` object.                                           |

> Example

```typescript
import * as zksync from "zksync";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://z2-dev-api.zksync.dev");
const ethereumProvider = ethers.getDefaultProvider("rinkeby");
const wallet = new Wallet(PRIVATE_KEY, zkSyncProvider, ethereumProvider);
```

### Other ways to create wallet

`Wallet` class supports all the methods from `ethers.Wallet` for creating wallets, e.g. creating from mnemonic, creating from encrypted json, creating random wallet, etc. All these methods take the same parameters as `ethers.Wallet`, so you should refer to its documentation on how to use them.

### Connecting to zkSync provider

In order to interact with zkSync network, a `Wallet` object should be connected to a `Provider` by either passing it to the constructor or with the `connect` method.

```typescript
connect(provider: Provider): Wallet
```


#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| provider                   | zkSync node provider.                                  |
| returns                          | A new `Wallet` object, connected to zkSync                                           |

> Example

```typescript
import { Wallet, Provider } from 'zksync-web3'

const unconnectedWallet = new Wallet(PRIVATE_KEY);

// ... 

const provider = new Provider("https://z2-dev-api.zksync.dev");
const wallet = unconnectedWallet.connect(provider)
```


### Connecting to Ethereum provider

In order to perform L1 operations, the `Wallet` object needs to be connect to an `ethers.providers.Provider` object.

```typescript
connectToL1(provider: ethers.providers.Provider)
```

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| provider                   | Ethereum node provider.                                  |
| returns                          | A new `Wallet` object, connected to zkSync                                           |

> Example

```typescript
import { Wallet } from 'zksync-web3'
import { ethers } from 'ethers'

const unconnectedWallet = new Wallet(PRIVATE_KEY);

// ... 

const ethProvider = ethers.getDefaultProvider("rinkeby");
const wallet = unconnectedWallet.connectToL1(ethProvider);
```

It is possible to chain `connect` and `connectToL1` nethods:

```typescript
const wallet = unconnectedWallet.connect(provider).connectToL1(ethProvider);
```

### Getting zkSync smart contract

```typescript
async getMainContract(): Promise<Contract>
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| returns                          |  `Contract` wrapper of the zkSync smart contract                                           |


> Example

```typescript
import * as zksync from "zksync";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://z2-dev-api.zksync.dev");
const ethereumProvider = ethers.getDefaultProvider("rinkeby");
const wallet = new Wallet(PRIVATE_KEY, zkSyncProvider, ethereumProvider);

const contract = await wallet.getMainContract();
console.log(contract.address);
```


### Approving deposit of tokens

Bridging ERC-20 tokens from Ethereum requires approving the tokens to the zkSync Ethereum smart contract. The `Wallet` object contains a convenient method for that:

```typescript
async approveERC20(
    token: Address,
    amount: BigNumberish,
    overrides?: ethers.CallOverrides
): Promise<ethers.providers.TransactionResponse>
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| token                   | The Ethereum address of the token.                                         |
| amount     | The amount of token to be approved.                            |
| overrides (optional) | Ethereum transaction overrides. May be used to pass `gasLimit`, `gasPrice`, etc. |
| returns                          | `ethers.providers.TransactionResponse` object.                                           |

> Example

```typescript
import * as zksync from "zksync";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://z2-dev-api.zksync.dev");
const ethereumProvider = ethers.getDefaultProvider("rinkeby");
const wallet = new Wallet(PRIVATE_KEY, zkSyncProvider, ethereumProvider);

const USDC_ADDRESS = '0xeb8f08a975ab53e34d8a0330e0d34de942c95926';
const txReceipt = await wallet.approveERC20({
    USDC_ADDRESS,
    amount: "10000000", // 10.0 USDC
});

await txReceipt.wait();
```


### Depositing tokens to zkSync

The wallet also have convenience method for briding tokens to zkSync:

```typescript
async deposit(transaction: {
    token: Address;
    amount: BigNumberish;
    to?: Address;
    approveERC20?: boolean;
    overrides?: ethers.CallOverrides;
}): Promise<PriorityOpResponse> 
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| tramsaction.token                   | The address of the token to deposit.                                         |
| tramsaction.amount    | Amount of the token to deposit.                            |
| tramsaction.to (optional) | The address which will receive the deposited tokens on L2. |
| tramsaction.approveERC20 (optional) | Whether or not should the token approval be performed under the hood. Set this flag to `true` if you bridge ERC-20 token and didn't call the `approveERC20` function beforehand. |
| tramsaction.overrides (optional) | Ethereum transaction overrides. May be used to pass `gasLimit`, `gasPrice`, etc. |
| returns                          | `PriorityOpResponse` object                                           |

> Example

```typescript
import * as zksync from "zksync";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://z2-dev-api.zksync.dev");
const ethereumProvider = ethers.getDefaultProvider("rinkeby");
const wallet = new Wallet(PRIVATE_KEY, zkSyncProvider, ethereumProvider);

const USDC_ADDRESS = '0xeb8f08a975ab53e34d8a0330e0d34de942c95926';
const usdcDepositReceipt = await wallet.deposit({
    token: USDC_ADDRESS,
    amount: "10000000",
    approveERC20: true
});
// Note that we wait not only for the L1 transaction to complete, but also for it to be
// processed by zkSync. If we want to wait only for transaction to be processed on L1,
// we can use `await usdcDepositReceipt.waitL1Commit()`
await usdcDepositReceipt.wait();

const ethDepositReceipt = await wallet.deposit({
    token: zksync.utils.ETH_ADDRESS,
    amount: "10000000",
});
// Note that we wait not only for the L1 transaction to complete, but also for it to be
// processed by zkSync. If we want to wait only for transaction to be processed on L1,
// we can use `await ethDepositReceipt.waitL1Commit()`
await ethDepositReceipt.wait();
```

### Adding native token to zkSync

To add a new native token to zkSync, `addToken` function should be called on the zkSync smart contract:

```typescript
async addToken(token: Address, overrides?: ethers.CallOverrides): Promise<PriorityOpResponse>
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| token                   | The address of the token.                                         |
| overrides (optional)    | Ethereum transaction overrides. May be used to pass `gasLimit`, `gasPrice` etc.                            |
| returns                          | `PriorityOpResponse` object.                                           |

> Example

```typescript
import * as zksync from "zksync";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://z2-dev-api.zksync.dev");
const ethereumProvider = ethers.getDefaultProvider("rinkeby");
const wallet = new Wallet(PRIVATE_KEY, zkSyncProvider, ethereumProvider);

const MLTT_ADDRESS = "0x690f4886c6911d81beb8130db30c825c27281f22";
const addTokenReceipt = await wallet.addToken(MLTT_ADDRESS);

// Note that we wait not only for the L1 transaction to complete, but also for it to be
// processed by zkSync. If we want to wait only for transaction to be processed on L1,
// we can use `await addTokenReceipt.waitL1Commit()`
addTokenReceipt.wait();
```

### Getting token balance

```typescript
async getBalance(token?: Address, blockTag: BlockTag = 'committed'): Promise<BigNumber>
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| token (optional)                | The address of the token. ETH by default.                                         |
| blockTag (optional)    | Which block should we check the balance on. `committed`, i.e. the latest processed one is the default option.                             |
| returns                          | The amount of token the `Wallet` has                                        |

> Example

```typescript
import * as zksync from "zksync";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://z2-dev-api.zksync.dev");
const ethereumProvider = ethers.getDefaultProvider("rinkeby");
const USDC_ADDRESS = '0xeb8f08a975ab53e34d8a0330e0d34de942c95926';
const wallet = new Wallet(PRIVATE_KEY, zkSyncProvider);

// Getting balance in USDC
console.log(await wallet.getBalance(USDC_ADDRESS));

// Getting balance in ETH
console.log(await wallet.getBalance());
```

### Getting token balance on L1

```typescript
async getBalanceL1(token?: Address, blockTag?: ethers.providers.BlockTag): Promise<BigNumber>
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| token (optional)                | The address of the token. ETH by default.                                         |
| blockTag (optional)    | Which block should we check the balance on. The latest processed one is the default option.                             |
| returns                          | The amount of token the `Wallet` has on Ethereum                                       |

> Example

```typescript
import * as zksync from "zksync";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://z2-dev-api.zksync.dev");
const ethereumProvider = ethers.getDefaultProvider("rinkeby");
const USDC_ADDRESS = '0xeb8f08a975ab53e34d8a0330e0d34de942c95926';
const wallet = new Wallet(PRIVATE_KEY, zkSyncProvider);

// Getting balance in USDC
console.log(await wallet.getBalanceL1(USDC_ADDRESS));

// Getting balance in ETH
console.log(await wallet.getBalanceL1());
```

### Getting nonce

`Wallet` also provided the `getNonce` method which is an alias for [getTransactionCount](https://docs.ethers.io/v5/api/signer/#Signer-getTransactionCount).

```typescript
async getNonce(blockTag?: BlockTag): Promise<number>
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| blockTag (optional)    | Which block should we get the nonce on. `committed`, i.e. the latest processed one is the default option.                             |
| returns                          | The amount of token the `Wallet` has                                        |

> Example

```typescript
import * as zksync from "zksync";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://z2-dev-api.zksync.dev");
const USDC_ADDRESS = '0xeb8f08a975ab53e34d8a0330e0d34de942c95926';
// Note that we don't need ethereum provider to get the nonce
const wallet = new Wallet(PRIVATE_KEY, zkSyncProvider);

console.log(await wallet.getNonce());
```

### Transfering tokens inside zkSync

Please note that for now unlike Ethereum, zkSync does not support native transfers, i.e. the `value` field of all transactions is equal to `0`. All the token transfers are done through ERC-20 `transfer` function call. 

But for convenience, `Wallet` object has `transfer` method, which can transfer any `ERC-20` tokens.

```typescript
async transfer(tx: {
    to: Address;
    amount: BigNumberish;
    token?: Address;
    feeToken?: Address;
    nonce?: Address;
    validFrom?: number;
    validUntil?: number;
}): Promise<ethers.ContractTransaction>
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| tx.to                   | The address of the recipient                                        |
| tx.amount    | The amount of the token to transfer                            |
| token (optional) | The address of the token. `ETH` by default. |
| feeToken (optional) | The address of the token which is used to pay fees. `ETH` by default. |
| nonce (optional) | The nonce to be supplied. If not provided, the `wallet` will fetch the nonce in the latest committed block. |
| validFrom (optional) | The UNIX timestamp from which the transaction is valid. |
| validUntil (optional) | The UNIX timestamp until which the transaction is valid. |
| returns                          | `Wallet` object.                                           |

> Example

```typescript
import * as zksync from "zksync";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://z2-dev-api.zksync.dev");
const ethereumProvider = ethers.getDefaultProvider("rinkeby");
const wallet = new zksync.Wallet(PRIVATE_KEY, zkSyncProvider, ethereumProvider);

const recipient = zksync.Wallet.createRandom();

const USDC_ADDRESS = '0xeb8f08a975ab53e34d8a0330e0d34de942c95926';
// We transfer 0.01 ETH to the recipient and pay the fee in USDC 
const transferHandle = wallet.transfer({ 
    to: recipient.address, 
    amount: ethers.utils.parseEther('0.01'), 
    feeToken: USDC_ADDRESS 
});
```

### Retrieving the underlying L1 wallet

You can get an `ethers.Wallet` object with the same private key with `ethWallet()` method.


#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| returns                          | `ethers.Wallet` object with the same private key.                                           |

> Example

```typescript
import * as zksync from "zksync";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://z2-dev-api.zksync.dev");
const ethereumProvider = ethers.getDefaultProvider("rinkeby");
const wallet = new zksync.Wallet(PRIVATE_KEY, zkSyncProvider, ethereumProvider);

const ethWallet = wallet.ethWallet();
```

## `EIP712Signer`

Signer which is used to sign EIP712 transactions only. Its methods are mostly used internally. More examples are coming soon!

## `Signer`

This class is to be used in browser environemt. The easiest way to construct it is to use the `getSigner` method of the `Web3Provider`. This structure extends `ethers.providers.JsonRpcSigner` and so supports all the methods available for it.

```typescript
import { Web3Provider } from 'zksync-web3'

const provider = new Web3Provider(window.ethereum);
const signer = provider.getSigner();
```


### Getting token balance

```typescript
async getBalance(token?: Address, blockTag: BlockTag = 'committed'): Promise<BigNumber>
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| token (optional)                | The address of the token. ETH by default.                                         |
| blockTag (optional)    | Which block should we check the balance on. `committed`, i.e. the latest processed one is the default option.                             |
| returns                          | The amount of token the `Wallet` has                                        |

> Example

```typescript
import { Web3Provider } from 'zksync-web3'
import { ethers } from "ethers";

const provider = new Web3Provider(window.ethereum);
const signer = provider.getSigner();

// Getting balance in USDC
console.log(await signer.getBalance(USDC_ADDRESS));

// Getting balance in ETH
console.log(await signer.getBalance());
```

### Getting nonce

`Wallet` also provided the `getNonce` method which is an alias for [getTransactionCount](https://docs.ethers.io/v5/api/signer/#Signer-getTransactionCount).

```typescript
async getNonce(blockTag?: BlockTag): Promise<number>
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| blockTag (optional)    | Which block should we get the nonce on. `committed`, i.e. the latest processed one is the default option.                             |
| returns                          | The amount of token the `Wallet` has                                        |

> Example

```typescript
import { Web3Provider } from 'zksync-web3'

const provider = new Web3Provider(window.ethereum);
const signer = provider.getSigner();

console.log(await signer.getNonce());
```

### Transfering tokens inside zkSync

Please note that for now unlike Ethereum, zkSync does not support native transfers, i.e. the `value` field of all transactions is equal to `0`. All the token transfers are done through ERC-20 `transfer` function call. 

But for convenience, `Wallet` object has `transfer` method, which can transfer any `ERC-20` tokens.

```typescript
async transfer(tx: {
    to: Address;
    amount: BigNumberish;
    token?: Address;
    feeToken?: Address;
    nonce?: Address;
    validFrom?: number;
    validUntil?: number;
}): Promise<ethers.ContractTransaction>
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| tx.to                   | The address of the recipient                                        |
| tx.amount    | The amount of the token to transfer                            |
| token (optional) | The address of the token. `ETH` by default. |
| feeToken (optional) | The address of the token which is used to pay fees. `ETH` by default. |
| nonce (optional) | The nonce to be supplied. If not provided, the `wallet` will fetch the nonce in the latest committed block. |
| validFrom (optional) | The UNIX timestamp from which the transaction is valid. |
| validUntil (optional) | The UNIX timestamp until which the transaction is valid. |
| returns                          | `Wallet` object.                                           |

> Example

```typescript
import { Wallet, Web3Provider } from 'zksync-web3'
import { ethers } from "ethers";

const provider = new Web3Provider(window.ethereum);
const signer = provider.getSigner();

const recipient = Wallet.createRandom();

const USDC_ADDRESS = '0xeb8f08a975ab53e34d8a0330e0d34de942c95926';
// We transfer 0.01 ETH to the recipient and pay the fee in USDC 
const transferHandle = signer.transfer({ 
    to: recipient.address, 
    amount: ethers.utils.parseEther('0.01'), 
    feeToken: USDC_ADDRESS 
});
```

## `L1Signer`

This class is to be used in browser environemt to do zkSync-related operations on Layer 1. This class extends `ethers.providers.JsonRpcSigner` and so supports all the methods available for it.


The easiest way to construct it from the `Web3Provider`.

```typescript
import { Web3Provider, Provider, L1Signer } from 'zksync-web3'

const provider = new ethers.Web3Provider(window.ethereum);
const zksyncProvider = new Provider('https://z2-dev-api.zksync.dev');
const signer = L1Signer.from(provider.getSigner(), zksyncProvider);
```


### Approving deposit of tokens

Bridging ERC-20 tokens from Ethereum requires approving the tokens to the zkSync Ethereum smart contract. The `Wallet` object contains a convenient method for that:

```typescript
async approveERC20(
    token: Address,
    amount: BigNumberish,
    overrides?: ethers.CallOverrides
): Promise<ethers.providers.TransactionResponse>
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| token                   | The Ethereum address of the token.                                         |
| amount     | The amount of token to be approved.                            |
| overrides (optional) | Ethereum transaction overrides. May be used to pass `gasLimit`, `gasPrice`, etc. |
| returns                          | `ethers.providers.TransactionResponse` object.                                           |

> Example

```typescript
import { Web3Provider, Provider, L1Signer } from 'zksync-web3'
import { ethers } from 'ethers';

const provider = new ethers.Web3Provider(window.ethereum);
const zksyncProvider = new Provider('https://z2-dev-api.zksync.dev');
const signer = L1Signer.from(provider.getSigner(), zksyncProvider);

const USDC_ADDRESS = '0xeb8f08a975ab53e34d8a0330e0d34de942c95926';
const txReceipt = await signer.approveERC20({
    USDC_ADDRESS,
    amount: "10000000", // 10.0 USDC
});

await txReceipt.wait();
```


### Getting zkSync smart contract

```typescript
async getMainContract(): Promise<Contract>
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| returns                          | `Contract` wrapper of the zkSync smart contract                                           |

> Example

```typescript
import { Web3Provider, Provider, L1Signer } from 'zksync-web3'
import { ethers } from 'ethers';

const provider = new ethers.Web3Provider(window.ethereum);
const zksyncProvider = new Provider('https://z2-dev-api.zksync.dev');
const signer = L1Signer.from(provider.getSigner(), zksyncProvider);

const mainContract = await signer.getMainContract();
console.log(mainContract.address);
```

### Getting token balance on L1

```typescript
async getBalanceL1(token?: Address, blockTag?: ethers.providers.BlockTag): Promise<BigNumber>
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| token (optional)                | The address of the token. ETH by default.                                         |
| blockTag (optional)    | Which block should we check the balance on. The latest processed one is the default option.                             |
| returns                          | The amount of token the `Wallet` has on Ethereum                                       |

> Example

```typescript
import { Web3Provider, Provider, L1Signer } from 'zksync-web3'
import { ethers } from 'ethers';

const provider = new ethers.Web3Provider(window.ethereum);
const zksyncProvider = new Provider('https://z2-dev-api.zksync.dev');
const signer = L1Signer.from(provider.getSigner(), zksyncProvider);

// Getting balance in USDC
console.log(await signer.getBalanceL1(USDC_ADDRESS));

// Getting balance in ETH
console.log(await signer.getBalanceL1());
```

### Depositing tokens to zkSync

The wallet also have convenience method for briding tokens to zkSync:

```typescript
async deposit(transaction: {
    token: Address;
    amount: BigNumberish;
    to?: Address;
    approveERC20?: boolean;
    overrides?: ethers.CallOverrides;
}): Promise<PriorityOpResponse> 
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| tramsaction.token                   | The address of the token to deposit.                                         |
| tramsaction.amount    | Amount of the token to deposit.                            |
| tramsaction.to (optional) | The address which will receive the deposited tokens on L2. |
| tramsaction.approveERC20 (optional) | Whether or not should the token approval be performed under the hood. Set this flag to `true` if you bridge ERC-20 token and didn't call the `approveERC20` function beforehand. |
| tramsaction.overrides (optional) | Ethereum transaction overrides. May be used to pass `gasLimit`, `gasPrice`, etc. |
| returns                          | `PriorityOpResponse` object                                           |

> Example

```typescript
import { Web3Provider, Provider, L1Signer } from 'zksync-web3'
import { ethers } from 'ethers';

const provider = new ethers.Web3Provider(window.ethereum);
const zksyncProvider = new Provider('https://z2-dev-api.zksync.dev');
const signer = L1Signer.from(provider.getSigner(), zksyncProvider);

const USDC_ADDRESS = '0xeb8f08a975ab53e34d8a0330e0d34de942c95926';
const usdcDepositReceipt = await signer.deposit({
    token: USDC_ADDRESS,
    amount: "10000000",
    approveERC20: true
});
// Note that we wait not only for the L1 transaction to complete, but also for it to be
// processed by zkSync. If we want to wait only for transaction to be processed on L1,
// we can use `await usdcDepositReceipt.waitL1Commit()`
await usdcDepositReceipt.wait();

const ethDepositReceipt = await signer.deposit({
    token: zksync.utils.ETH_ADDRESS,
    amount: "10000000",
});
// Note that we wait not only for the L1 transaction to complete, but also for it to be
// processed by zkSync. If we want to wait only for transaction to be processed on L1,
// we can use `await ethDepositReceipt.waitL1Commit()`
await ethDepositReceipt.wait();
```

### Adding native token to zkSync

To add a new native token to zkSync, `addToken` function should be called on the zkSync smart contract:

```typescript
async addToken(token: Address, overrides?: ethers.CallOverrides): Promise<PriorityOpResponse>
``` 

#### Inputs and outputs

| Name                             | Description                                                            |
| -------------------------------- | ---------------------------------------------------------------------- |
| token                   | The address of the token.                                         |
| overrides (optional)    | Ethereum transaction overrides. May be used to pass `gasLimit`, `gasPrice` etc.                            |
| returns                          | `PriorityOpResponse` object.                                           |

> Example

```typescript
import { Web3Provider, Provider, L1Signer } from 'zksync-web3'
import { ethers } from 'ethers';

const provider = new ethers.Web3Provider(window.ethereum);
const zksyncProvider = new Provider('https://z2-dev-api.zksync.dev');
const signer = L1Signer.from(provider.getSigner(), zksyncProvider);

const MLTT_ADDRESS = "0x690f4886c6911d81beb8130db30c825c27281f22";
const addTokenReceipt = await signer.addToken(MLTT_ADDRESS);

// Note that we wait not only for the L1 transaction to complete, but also for it to be
// processed by zkSync. If we want to wait only for transaction to be processed on L1,
// we can use `await addTokenReceipt.waitL1Commit()`
addTokenReceipt.wait();
```
