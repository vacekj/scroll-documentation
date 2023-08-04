---
section: developers
date: Last Modified
title: "ERC1155 token bridge"
lang: "en"
permalink: "TODO"
excerpt: "TODO"
---

# ERC1155 Token Bridge

## Deposit ERC1155 tokens from Goerli

ERC1155 bridging from Goerli to Scroll Alpha is done via the L1ERC1155Gateway deployed on Goerli at `0xd1bE599aaCBC21448fD6373bbc7c1b4c7806f135`. Similarly to ERC721 bridging, we don't use a router but the `depositERC1155` function on the Gateway directly.

{% hint style="info" %}
**`depositERC1155`** is a payable function, and the amount of ETH sent to this function will be used to pay for L2 fees. If the amount is not enough, the transaction will not be sent. All excess eth will be sent back to sender. `0.00001 ETH` should be more than enough to process a token deposit.
{% endhint %}

### Creating an ERC1155 token on L2

Similar to ERC721 bridging, in order to bridge ERC1155 tokens, a contract compatible with the `IScrollERC1155` standard has to be launched and mapped on a `L1ERC1155Gateway` and `L2ERC1155Gateway` on Goerli and Scroll Alpha, respectively. This contract has to grant permission to the Gateway on Scroll Alpha to mint when a token is deposited and burn when the token is withdrawn.

The following interface is the `IScrollERC1155` needed for deploying ERC1155 tokens compatible with the `L2ERC1155Gateway` on Scroll Alpha.

```jsx
interface IScrollERC1155 {
    /// @notice Return the address of Gateway the token belongs to.
    function gateway() external view returns (address);

    /// @notice Return the address of counterpart token.
    function counterpart() external view returns (address);

    /// @notice Mint some token to recipient's account.
    /// @dev Gateway Utilities, only gateway contract can call
    /// @param _to The address of recipient.
    /// @param _tokenId The token id to mint.
    /// @param _amount The amount of token to mint.
    /// @param _data The data passed to recipient
    function mint(
        address _to,
        uint256 _tokenId,
        uint256 _amount,
        bytes memory _data
    ) external;

    /// @notice Burn some token from account.
    /// @dev Gateway Utilities, only gateway contract can call
    /// @param _from The address of account to burn token.
    /// @param _tokenId The token id to burn.
    /// @param _amount The amount of token to burn.
    function burn(
        address _from,
        uint256 _tokenId,
        uint256 _amount
    ) external;

    /// @notice Batch mint some token to recipient's account.
    /// @dev Gateway Utilities, only gateway contract can call
    /// @param _to The address of recipient.
    /// @param _tokenIds The token id to mint.
    /// @param _amounts The list of corresponding amount of token to mint.
    /// @param _data The data passed to recipient
    function batchMint(
        address _to,
        uint256[] calldata _tokenIds,
        uint256[] calldata _amounts,
        bytes calldata _data
    ) external;

    /// @notice Batch burn some token from account.
    /// @dev Gateway Utilities, only gateway contract can call
    /// @param _from The address of account to burn token.
    /// @param _tokenIds The list of token ids to burn.
    /// @param _amounts The list of corresponding amount of token to burn.
    function batchBurn(
        address _from,
        uint256[] calldata _tokenIds,
        uint256[] calldata _amounts
    ) external;
}
```

### Adding an ERC1155 token to the Scroll Bridge

All assets can be bridged securely and permissionlessly through Gateway contracts deployed by any developer. However, Scroll also manages an ERC1155 Gateway contract where all NFTs created by the community are welcome. Being part of the Scroll-managed Gateway means you won't need to deploy the Gateway contracts, and your token will appear in the Scroll frontend. To be part of the Scroll Gateway, you must contact the Scroll team to add the token to both Goerli and Scroll Alpha Gateway contracts. To do so, follow the instructions on the [token lists](https://github.com/scroll-tech/token-list) repository to add your token to the Scroll official frontend.

## Withdraw ERC1155 tokens from Scroll

The `L2ERC1155Gateway` contract deployed at `0xfe5Fc32777646bD123564C41f711FF708Dd48360` is used to send tokens from Scroll Alpha to Goerli. Before bridging, the `L2ERC1155Gateway` contract has to be approved by the token contract. Once that is done, `withdrawERC1155` can be called to perform the asset bridge.

{% hint style="info" %}
**`withdrawERC1155`** is a payable function, and the amount of ETH sent to this function will be used to pay for L1 fees. If the amount is not enough, the transaction will not be sent. All excess ETH will be sent back to sender. Fees depend on Goerli activity but `0.005 ETH` should be enough to process a token withdrawal.
{% endhint %}

{% hint style="warning" %}
**Make sure the transaction won't revert on Goerli** while sending from Scroll Alpha. There is no way to recover tokens bridged if you're transaction reverts on Goerli. All assets are burnt on Scroll Alpha when the transaction is sent, and it's impossible to mint them again.
{% endhint %}

## L1ERC1155Gateway API

Please visit the [npm library](https://www.npmjs.com/package/@scroll-tech/contracts?activeTab=code) for the complete Scroll contract API documentation.

### depositERC1155

```solidity
function depositERC1155(
  address _token,
  address _to,
  uint256 _tokenId,
  uint256 _amount,
  uint256 _gasLimit
) external payable;
```

Deposit an ERC1155 token from Goerli to a recipient's account on Scroll Alpha.

| Parameter | Description                                                 |
| --------- | ----------------------------------------------------------- |
| token     | The address of ERC1155 token contract on Goerli.            |
| to        | The address of recipient's account on Scroll Alpha.         |
| tokenId   | The NFT id to deposit.                                      |
| amount    | The amount of tokens to deposit.                            |
| gasLimit  | Gas limit required to complete the deposit on Scroll Alpha. |

### updateTokenMapping

```solidity
function updateTokenMapping(address _l1Token, address _l2Token) external;
```

Update the mapping that connects an ERC1155 token contract from Goerli to Scroll Alpha.

| Parameter | Description                                       |
| --------- | ------------------------------------------------- |
| \_l1Token | The address of the ERC1155 token in L1.           |
| \_l2Token | The address of corresponding ERC1155 token in L2. |

## L2ERC1155Gateway API

### withdrawERC1155

```solidity
function withdrawERC1155(address token, address to, uint256 tokenId, uint256 amount, uint256 gasLimit) external payable;
```

Send ERC1155 tokens from Scroll Alpha to a recipient's account on Goerli.

| Parameter | Description                                                              |
| --------- | ------------------------------------------------------------------------ |
| token     | The address of ERC1155 token contract on Scroll Alpha.                   |
| to        | The address of recipient's account on Goerli.                            |
| tokenId   | The NFT id to withdraw.                                                  |
| amount    | The amount of tokens to withdraw.                                        |
| gasLimit  | Unused, but included for potential forward compatibility considerations. |

### updateTokenMapping

```solidity
function updateTokenMapping(address _l1Token, address _l2Token) external;
```

Update the mapping that connects an ERC1155 token contract from Scroll Alpha to Goerli.

| Parameter | Description                                       |
| --------- | ------------------------------------------------- |
| \_l1Token | The address of th ERC1155 token in L1.            |
| \_l2Token | The address of corresponding ERC1155 token in L2. |