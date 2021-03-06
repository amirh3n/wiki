In this tutorial, we will show you how you can use the [@0x/connect](https://github.com/0xProject/0x.js/tree/development/packages/connect) package in conjunction with [0x.js](https://github.com/0xProject/0x.js/tree/development/packages/0x.js) in order to:

-   ask a relayer for fee information
-   submit signed orders to a relayer with appropriate fees
-   ask a relayer for a ZRX/WETH orderbook
-   find the best orders in the orderbook
-   fill orders from the orderbook using `0x.js`

You can find all the `@0x/connect` documentation [here](https://0xproject.com/docs/connect).

You can find a list of relayers that comply with the Standard Relayer API [here](https://github.com/0xProject/0x-relayer-registry)

### Setup

Since 0x.js helps you build apps that interface with Ethereum, you will need to use it in conjunction with an Ethereum node. For development, we recommend using [Ganache CLI](https://github.com/trufflesuite/ganache-cli). Since there is some setup required to getting started with Ganache and the 0x smart contracts, we are going to use a [0x starter project](https://github.com/0xProject/0x-starter-project) which handles a lot of this for us.

Clone the repo:

```
git clone https://@github.com/0xProject/0x-starter-project.git
```

Install all the dependencies (we use Yarn: `brew install yarn --without-node`):

```
yarn install
```

Pull the latest Ganache 0x snapshot with all the 0x contracts pre-deployed and an account with ZRX balance:

```
yarn download_snapshot
```

In a separate terminal, navigate to the project directory and start TestRPC:

```
yarn ganache-cli
```

In another terminal, navigate to the project directory and start a local http server that conforms to a portion of the standard relayer API, listening to port 3000:

```
yarn fake_sra_server
```

You can now run the code we are going to walk you through in the rest of this tutorial:

```
yarn scenario:fill_order_sra
```

### Importing packages

The first step to interacting with `@0x/connect` is to import the following relevant packages:

```javascript
import {
    assetDataUtils,
    BigNumber,
    ContractWrappers,
    generatePseudoRandomSalt,
    Order,
    orderHashUtils,
    signatureUtils,
} from '0x.js';
import { HttpClient, OrderbookRequest, OrderConfigRequest } from '@0x/connect';
import { Web3Wrapper } from '@0x/web3-wrapper';
import { getContractAddressesForNetworkOrThrow } from '@0x/contract-addresses';
```

**0x.js** is a package that pulls in a number of underlying 0x packages and exposes their respective functionality. You can choose to pull these packages directly without using 0x.js. These packages allow you to interact with the 0x smart contracts (contract wrappers) and create, sign and validate orders (order utils).
**BigNumber** is a JavaScript library for arbitrary-precision decimal and non-decimal arithmetic.
**HttpClient** is a Standard Relayer API client implementation that allows you to easily query any Standard Relayer API endpoint.
**Web3Wrapper** is a package for interacting with an Ethereum node, retrieving account information. This is an optional package and you can choose to use alternatives like Web3.js or Ethers.js.
**contract-addresses** is a package where we have our deployed contract addresses for the various networks. This also includes an address for a deployed Wrapped Ether contract.

### Instantiating a Provider and ContractWrappers

First, we need to instantiate an instance of ContractWrappers and a Provider. In our case, since we are using our local node (ganache), we will use **http://localhost:8545**. You can read about what providers are [here](#Web3-Provider-Explained).

```typescript
import { RPCSubprovider, Web3ProviderEngine } from '0x.js';

export const providerEngine = new Web3ProviderEngine();
providerEngine.addProvider(new RPCSubprovider('http://localhost:8545'));
providerEngine.start();
// Instantiate ContractWrappers with the provider
const contractWrappers = new ContractWrappers(providerEngine, { networkId: NETWORK_CONFIGS.networkId });
```

### Retreiving Accounts from the Provider

We will use Web3Wrapper to retrieve accounts from the provider. The first account will the the maker, and the second account will be the taker for the purposes of this tutorial.

```typescript
const web3Wrapper = new Web3Wrapper(providerEngine);
const [maker, taker] = await web3Wrapper.getAvailableAddressesAsync();
```

### Declaring decimals and addresses

Since we are dealing with a few contracts, we will specify them now to reduce the syntax load. Fortunately for us, `0x.js` library comes with a couple of contract addresses that can be useful to have at hand. One thing that is important to remember is that there are no decimals in the Ethereum virtual machine (EVM), which means you always need to keep track of how many "decimals" each token possesses. Since we will sell some ZRX for some ETH and since they both have 18 decimals, we can use a shared constant.

```javascript
// Token Addresses
const contractAddresses = getContractAddressesForNetworkOrThrow(NETWORK_CONFIGS.networkId);
const zrxTokenAddress = contractAddresses.zrxToken;
const etherTokenAddress = contractAddresses.etherToken;
```

0x Protocol uses the ABI encoding scheme for asset data. For example, the ERC20 Token address which is being traded on 0x needs to be encoded. Encoding the address informs the 0x smart contracts on which type of asset is being traded (e.g ERC20 or ERC721) and has optional parameters for different token types (e.g ERC721 token id). In this tutorial we are trading 5 ZRX (ERC20) for 0.1 WETH (ERC20).

```typescript
const makerAssetData = assetDataUtils.encodeERC20AssetData(zrxTokenAddress);
const takerAssetData = assetDataUtils.encodeERC20AssetData(etherTokenAddress);
// the amount the maker is selling of maker asset
const makerAssetAmount = Web3Wrapper.toBaseUnitAmount(new BigNumber(5), DECIMALS);
// the amount the maker wants of taker asset
const takerAssetAmount = Web3Wrapper.toBaseUnitAmount(new BigNumber(0.1), DECIMALS);
```

### Approvals and WETH Balance

To trade on 0x, the participants (maker and taker) require a small amount of initial set up. They need to approve the 0x smart contracts to move funds on their behalf. In order to give 0x protocol smart contract access to funds, we need to set _allowances_ (you can read about allowances [here](https://tokenallowance.io/)). In this tutorial the taker is using WETH (or Wrapper ETH), as ETH is not an ERC20 token it must first be converted into WETH to be used on 0x. Concretely, "converting" ETH to WETH means that we will deposit some ETH in a smart contract acting as a ERC20 wrapper. In exchange of depositing ETH, we will get some ERC20 compliant tokens called WETH at a 1:1 conversion rate. For example, depositing 10 ETH will give us back 10 WETH and we can revert the process at any time. The ContractWrappers package has helpers for general ERC20 tokens as well as the WETH token.

```typescript
// Allow the 0x ERC20 Proxy to move ZRX on behalf of makerAccount
const makerZRXApprovalTxHash = await contractWrappers.erc20Token.setUnlimitedProxyAllowanceAsync(
    zrxTokenAddress,
    maker,
);
await web3Wrapper.awaitTransactionSuccessAsync(makerZRXApprovalTxHash);

// Allow the 0x ERC20 Proxy to move WETH on behalf of takerAccount
const takerWETHApprovalTxHash = await contractWrappers.erc20Token.setUnlimitedProxyAllowanceAsync(
    etherTokenAddress,
    taker,
);
await web3Wrapper.awaitTransactionSuccessAsync(takerWETHApprovalTxHash);

// Convert ETH into WETH for taker by depositing ETH into the WETH contract
const takerWETHDepositTxHash = await contractWrappers.etherToken.depositAsync(
    etherTokenAddress,
    takerAssetAmount,
    taker,
);
await web3Wrapper.awaitTransactionSuccessAsync(takerWETHDepositTxHash);
```

At this point, it is worth mentioning why we need to await all those transactions. Calling an 0x.js function returns immediately after submitting a transaction with a transaction hash, so the user interface (UI) might show some useful information to the user before the transaction is mined (it sometimes takes long time). In our use-case we just want it to be confirmed, which happens immediately on ganache. It is nevertheless a good habit to interact with the blockchain with these async/await calls.

## Creating an HttpClient

Next, we create an `HttpClient` instance with a url pointing to a local standard relayer api http server running at **http://localhost:3000**. The `HttpClient` is our programmatic gateway to any relayer that conforms to the standard relayer api http standard.

```javascript
// Instantiate relayer client pointing to a local server on port 3000
const relayerApiUrl = 'http://localhost:3000/v2';
const relayerClient = new HttpClient(relayerApiUrl);
```

### Getting configuration information

Before creating an order to submit to a relayer, it is necessary to understand the requirements a relayer may have to host the order on their orderbook. This is what the `/order_config` endpoint is for.

Create a partial order (or `OrderConfigRequest`) and query the API using `getOrderConfigAsync`:

```javascript
// Generate and expiration time and find the exchange smart contract address
const randomExpiration = getRandomFutureDateInSeconds();
const exchangeAddress = contractAddresses.exchange;

// Ask the relayer about the parameters they require for the order
const orderConfigRequest: OrderConfigRequest = {
    exchangeAddress,
    makerAddress: maker,
    takerAddress: NULL_ADDRESS,
    expirationTimeSeconds: randomExpiration,
    makerAssetAmount,
    takerAssetAmount,
    makerAssetData,
    takerAssetData,
};
const orderConfig = await httpClient.getOrderConfigAsync(orderConfigRequest);
```

Here are the fields in an order config request:

-   **makerAddress** : Ethereum address of our **Maker**.
-   **takerAddress** : Ethereum address of our **Taker**.
-   **makerAssetData**: The token address the **Maker** is offering.
-   **takerAssetData**: The token address the **Maker** is requesting from the **Taker**.
-   **exchangeAddress** : The Exchange address.
-   **makerAssetAmount**: The amount of token the **Maker** is offering.
-   **takerAssetAmount**: The amount of token the **Maker** is requesting from the **Taker**.
-   **expirationTimeSeconds**: When will the order expire (in unix time).

Here are the fields that are provided in the order config response:

-   **senderAddress** : Ethereum address of a required **Sender** (none for now).
-   **feeRecipientAddress** : Ethereum address of our **Relayer** (none for now).
-   **makerFee**: How many ZRX the **Maker** will pay as a fee to the **Relayer**.
-   **takerFee** : How many ZRX the **Taker** will pay as a fee to the **Relayer**.

You can see that together they form a full Order object, except for the salt.

### Creating an order

Now that we have the configuration information, we can create a full order.

```javascript
// Create the order
const order: Order = {
    salt: generatePseudoRandomSalt(),
    ...orderConfigRequest,
    ...orderConfig,
};
```

### Signing the order

Now that we created an order as a **Maker**, we need to prove that we actually own the address specified as `makerAddress`. After all, we could always try pretending to be someone else just to annoy an exchange and other traders! To do so, we will sign the orders with the corresponding private key and append the signature to our order.

```javascript
// Generate the order hash and sign it
const orderHashHex = orderHashUtils.getOrderHashHex(order);
const signature = await signatureUtils.ecSignHashAsync(providerEngine, orderHashHex, maker);
const signedOrder = { ...order, signature };
```

With this, anyone can verify that the signature is authentic and this will prevent any change to the order by a third party. If the order is changed by even a single bit, then the hash of the order will be different and therefore invalid when compared to the signed hash.

Now let's actually verify whether the order we created is valid

```javascript
await contractWrappers.exchange.validateOrderFillableOrThrowAsync(signedOrder);
```

If something was wrong with our order, this function would throw an informative error. If it passes, then the order is currently fillable. A relayer should constantly be [pruning their orderbook](#0x-OrderWatcher) of invalid orders using this method.

### Submitting the order to the relayer

We can finally submit our signed order to the relayer for them to host on their orderbook.

```javascript
// Submit the order to the SRA Endpoint
await httpClient.submitOrderAsync(signedOrder, { networkId: NETWORK_CONFIGS.networkId };
```

### Requesting an orderbook

If in an application we need exchange functionality between two assets, we can find a suitable order for our needs using the `getOrderbookAsync()` method of `HttpClient`. In this example, we use the addresses of the ZRX and WETH tokens we retrieved earlier and use them to generate an `OrderbookRequest` to send to the relayerClient. In response, we get an `OrderbookResponse` containing orders that correspond with the provided `quoteAssetData` and `baseAssetData` (learn more about the quote/base token terminology [here](https://en.wikipedia.org/wiki/Currency_pair)).

```javascript
// Taker queries the Orderbook from the Relayer
const orderbookRequest: OrderbookRequest = { baseAssetData: makerAssetData, quoteAssetData: takerAssetData };
const response = await httpClient.getOrderbookAsync(orderbookRequest, { networkId: NETWORK_CONFIGS.networkId });
if (response.asks.total === 0) {
    throw new Error('No orders found on the SRA Endpoint');
}
```

### Filling an order

`OrderbookResponse` contains two fields, `bids` and `asks`. `Bids` is a [`PaginatedCollection`](https://0xproject.com/docs/connect#types-PaginatedCollection) of [`APIOrder`](https://0xproject.com/docs/connect#types-APIOrder)s where for each order, the `makerAssetData` field is equal to the `quoteAssetData` provided by the `OrderbookRequest` and the `takerAssetData` field is equal to `baseAssetData`. `Asks` is the opposite of `bids`. For each order, the `makerAssetData` field is equal to the `baseAssetData` and the `takerAssetData` field is equal to `quoteAssetData`.

The Standard Relayer API guarantees that the orders are sorted by price, and then by _taker fee price_ which is defined as the `takerFee` divided by `takerAmount`. After _taker fee price_, orders are to be sorted by expiration in ascending order.

Given the above, we can just pick an order from the top of the asks paginated collection and fill it:

```javascript
const sraOrder = response.asks.records[0].order;
```

Now we validate the order is fillable given the maker and taker. This checks the balances and allowances of both the Maker and Taker, this way if there are any issues we do not waste gas on a failed transaction.

```javascript
await contractWrappers.exchange.validateFillOrderThrowIfInvalidAsync(sraOrder, takerAssetAmount, taker);
```

Now that the order is validated we submit it to the blockchain by calling fillOrder.

```javascript
txHash = await contractWrappers.exchange.fillOrderAsync(sraOrder, takerAssetAmount, taker, {
    gasLimit: TX_DEFAULTS.gas,
});
await web3Wrapper.awaitTransactionSuccessAsync(txHash);
```

### Wrapping up

Through this tutorial we learned how to:

-   ask a relayer for fee information
-   submit signed orders to a relayer with appropriate fees
-   ask a relayer for a ZRX/WETH orderbook
-   find the best orders in the orderbook
-   fill orders from the orderbook using `0x.js`

While all of these tasks were performed using Ganache and a local standard relayer api compliant HTTP server, you can start using [@0x/connect](https://www.npmjs.com/package/@0x/connect) in conjunction with Radar Relay's standard relayer api HTTP url: https://api.radarrelay.com/0x/v2/ for executing trades on the main Ethereum network. For more information on how to use `0x.js`, go [here](https://0xproject.com/docs/0x.js).
