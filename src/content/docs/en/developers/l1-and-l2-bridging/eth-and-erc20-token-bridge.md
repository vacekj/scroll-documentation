---
section: developers
date: Last Modified
title: "ETH and ERC20 token bridge"
lang: "en"
permalink: "TODO"
excerpt: "TODO"
---

# ETH and ERC20 Token Bridge

## Deposit ETH and ERC20 tokens from Goerli

The Gateway Router allows ETH and ERC20 token bridging from L1 to L2 using the `depositETH` and `depositERC20` functions respectively. It is a permissionless bridge deployed on Goerli Testnet at `0xeF37207c1A1efF6D6a9d7BfF3cF4270e406d319b` serving Scroll Alpha testnet. Notice that ERC20 tokens will have a different address on L2, you can use the `getL2ERC20Address` function to query the new address.

{% hint style="info" %}
**`depositETH`** and **`depositERC20`** are payable functions, the amount of ETH sent to these functions will be used to pay for L2 fees. If the amount is not enough, the transaction will not be sent. All excess ETH will be sent back to the sender. `0.00001 ETH` should be more than enough to process a token deposit.
{% endhint %}

When bridging ERC20 tokens, you don’t have to worry about selecting the right Gateway. This is because the `L1GatewayRouter` will choose the correct underlying entry point to send the message:

- `L1StandardERC20Gateway`: This Gateway permits any ERC20 deposit and will be selected as the default by the L1GatewayRouter for an ERC20 token that doesn’t need custom logic on Scroll. On the very first token bridging, a new token will be created on Scroll L2 that implements the ScrollStandardERC20. To bridge a token, call the `depositERC20` function on the `L1GatewayRouter`.
- `L1CustomERC20Gateway`: This Gateway will be selected by the `L1GatewayRouter` for tokens with custom logic. For an L1/L2 token pair to work on the Scroll Custom ERC20 Bridge, the L2 token contract has to implement `IScrollStandardERC20`. Additionally, the token should grant `mint` or `burn` capability to the `L2CustomERC20Gateway`. The address of `L2CustomERC20Gateway` is `0xa07Cb742657294C339fB4d5d6CdF3fdBeE8C1c68`. Visit the [bridge-erc20-through-the-custom-gateway.md](../../developer-guides/bridge-erc20-through-the-custom-gateway.md "mention") guide for a step-by-step example of how to bridge a custom token.

All Gateway contracts will form the message and send it to the `L1ScrollMessenger` which can send arbitrary messages to L2. The `L1ScrollMessenger` passes the message to the `L1MessageQueue`. Any user can send messages directly to the Messenger to execute arbitrary data on L2. This means they can execute any function on L2 from a transaction made on L1 via the bridge. Although an application could directly pass messages to existing token contracts, the Gateway abstracts the specifics and simplifies making transfers and calls.

{% hint style="info" %}
Users can also bypass the **`L1ScrollMessenger`** and send messages directly to the **`L1MessageQueue`**. If a message is sent via the `L1MessageQueue`, the transaction's sender will be the address of the user sending the transaction, not the address of the `L1ScrollMessenger`. Learn more about sending arbitrary messages in the [the-scroll-messenger.md](the-scroll-messenger.md "mention") documentation.&#x20;
{% endhint %}

When a new block gets created on L1, the Watcher will detect the message on the `L1MessageQueue` and will pass it to the Relayer service, which will submit the transaction to the L2 via the l2geth node. Finally, the l2geth node will pass the transaction to the `L2ScrollMessagner` contract for execution on L2.

## Withdraw ETH and ERC20 tokens from Scroll Alpha

The L2 Gateway is very similar to the L1 Gateway. We can withdraw ETH and ERC20 tokens back to L1 using the `withdrawETH` and `withdrawERC20` functions. The contract address is deployed on Scroll Alpha at `0x6d79Aa2e4Fbf80CF8543Ad97e294861853fb0649`. We use the `getL1ERC20Address` to retrieve the token address on L1.

{% hint style="info" %}
**`withdrawETH`** and **`withdrawERC20`** are payable functions, and the amount of ETH sent to these functions will be used to pay for L1 fees. If the amount is not enough, the transaction will not be sent. All excess ETH will be sent back to the sender. Fees will depend on Goerli activity but `0.005 ETH` should be enough to process a token withdrawal.
{% endhint %}

{% hint style="warning" %}
**Make sure the transactions won't revert on Goerli** while sending from Scroll Alpha. There is no way to recover bridged ETH, tokens, or NFTs if your transaction reverts on Goerli. All assets are burnt on Scroll when the transaction is sent, and it's impossible to mint them again.
{% endhint %}

## Creating an ERC20 token with custom logic on L2

If a token needs custom logic on Scroll Alpha, it will need to be bridged through a `L1CustomERC20Gateway` and `L2CustomERC20Gateway` respectively. The custom token on Scroll Alpha will need to give permission to the Gateway to mint new tokens when a deposit occurs and to burn when tokens are withdrawn.&#x20;

The following interface is the `IScrollStandardERC20` needed for deploying tokens compatible with the `L2CustomERC20Gateway` on Scroll Alpha.

```jsx
interface IScrollStandardERC20 {
    /// @notice Return the address of Gateway the token belongs to.
    function gateway() external view returns (address);

    /// @notice Return the address of counterpart token.
    function counterpart() external view returns (address);

    /// @dev ERC677 Standard, see https://github.com/ethereum/EIPs/issues/677
    /// Defi can use this method to transfer L1/L2 token to L2/L1,
    /// and deposit to L2/L1 contract in one transaction
    function transferAndCall(
        address receiver,
        uint256 amount,
        bytes calldata data
    ) external returns (bool success);

    /// @notice Mint some token to recipient's account.
    /// @dev Gateway Utilities, only gateway contract can call
    /// @param _to The address of recipient.
    /// @param _amount The amount of token to mint.
    function mint(address _to, uint256 _amount) external;

    /// @notice Mint some token from account.
    /// @dev Gateway Utilities, only gateway contract can call
    /// @param _from The address of account to burn token.
    /// @param _amount The amount of token to mint.
    function burn(address _from, uint256 _amount) external;
}
```

### Adding a Custom L2 ERC20 token to the Scroll Bridge

Tokens can be bridged securely and permissionlessly through Gateway contracts deployed by any developer. However, Scroll also manages an ERC20 Router and a Gateway where all tokens created by the community are welcome. Being part of the Scroll-managed Gateway means you won't need to deploy the Gateway contracts, and your token will appear in the Scroll frontend. To be part of the Scroll Gateway, you must contact the Scroll team to add the token to both Goerli and Scroll Alpha bridge contracts. To do so, follow the instructions on the [token lists](https://github.com/scroll-tech/token-list) repository to add your new token to the official Scroll frontend.

## L1 Gateway API

Please visit the [npm library](https://www.npmjs.com/package/@scroll-tech/contracts?activeTab=code) for the complete Scroll contract API documentation.

### depositETH

```solidity
function depositETH(address _to, uint256 _amount, uint256 _gasLimit) public payable;
```

Sends ETH from L1 to L2.

| Parameter | Description                                                                                                                 |
| --------- | --------------------------------------------------------------------------------------------------------------------------- |
| to        | The address of recipient's account on L2.                                                                                   |
| amount    | The amount of token to transfer, in wei.                                                                                    |
| gasLimit  | Gas limit required to complete the deposit on L2. From 4000 to 10000 gas limit should be enough to process the transaction. |

### depositERC20

```solidity
function depositERC20(address _token, address _to, uint256 _amount, uint256 _gasLimit) payable;
```

Sends ERC20 tokens from L1 to L2.

| Parameter | Description                                                                                                                 |
| --------- | --------------------------------------------------------------------------------------------------------------------------- |
| token     | The token address on L1.                                                                                                    |
| to        | The address of recipient's account on L2.                                                                                   |
| amount    | The amount of token to transfer, in wei.                                                                                    |
| gasLimit  | Gas limit required to complete the deposit on L2. From 4000 to 10000 gas limit should be enough to process the transaction. |

### getL2ERC20Address

```solidity
function getL2ERC20Address(address _l1Token) external view returns (address);
```

Returns the corresponding L2 token address given L1 token address.

| Parameter | Description              |
| --------- | ------------------------ |
| \_l1Token | The address of l1 token. |

### updateTokenMapping

```solidity
function updateTokenMapping(address _l1Token, address _l2Token) external;
```

Update the mapping that connects an ERC20 token from Goerli to Scroll Alpha.

| Parameter | Description                                     |
| --------- | ----------------------------------------------- |
| \_l1Token | The address of the ERC20 token in L1.           |
| \_l2Token | The address of corresponding ERC20 token in L2. |

## L2 Gateway API

### withdrawETH

```solidity
function withdrawETH(address to, uint256 amount, uint256 gasLimit) external payable;
```

Sends ETH from L2 to L1.

| Parameter | Description                                                                                             |
| --------- | ------------------------------------------------------------------------------------------------------- |
| to        | The address of recipient's account on L1.                                                               |
| amount    | The amount of token to transfer, in wei.                                                                |
| gasLimit  | Gas limit required to complete the deposit on L1. This is optional, send 0 if you don’t want to set it. |

### withdrawERC20

```solidity
function withdrawERC20(address token, address to, uint256 amount, uint256 gasLimit) external payable;
```

Sends ERC20 tokens from L2 to L1.

| Parameter | Description                                                                                             |
| --------- | ------------------------------------------------------------------------------------------------------- |
| token     | The token address on L2.                                                                                |
| to        | The address of recipient's account on L1.                                                               |
| amount    | The amount of token to transfer, in wei.                                                                |
| gasLimit  | Gas limit required to complete the deposit on L1. This is optional, send 0 if you don’t want to set it. |

### getL1ERC20Address

```solidity
function getL1ERC20Address(address l2Token) external view returns (address);
```

Returns the corresponding L1 token address given an L2 token address.

| Parameter | Description                  |
| --------- | ---------------------------- |
| l2Token   | The address of the L2 token. |

### updateTokenMapping

```solidity
function updateTokenMapping(address _l1Token, address _l2Token) external;
```

Update the mapping that connects an ERC20 contract from Scroll Alpha to Goerli.

| Parameter | Description                                     |
| --------- | ----------------------------------------------- |
| \_l2Token | The address of the ERC20 token in L2.           |
| \_l1Token | The address of corresponding ERC20 token in L1. |