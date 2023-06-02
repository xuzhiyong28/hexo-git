---
title: openzeppelin(1)_token详解
tags:
  - 区块链
  - openzeppelin
categories:  区块链
date: 2020-08-30 16:04:21
---

## ERC20

openzeppelin提供了ERC20的`标准实现`和对应的`扩展实现`。

### 两个接口

- IERC20 : 定义了标准ERC20的接口
  - totalSupply() : ERC20代币总量
  - balanceOf(account) : 查询某个账户的代币数量
  - transfer(to, amount) : 转账，将msg.sender中转amount个代币到to上
  - allowance(owner, spender) : 查询授权额度
  - approve(spender, amount) : 授权转账，此函数作用是msg.sender授权允许spender操作transferFrom函数将amount个代币转给别人
  - transferFrom(from, to, amount) : 转账函数，msg.sender调用将amount个代币从from转给to,前提是msg.sender有获得from的授权额度
- IERC20Metadata : 定义了ERC20的元数据接口
  - name() : ERC20代币名称
  - symbol() : ERC20代币符号
  - decimals() : 标记的小数位

### 一个标准实现

`ERC20.sol`是一个标准的ERC20实现，实现了`IERC20`和`IERC20Metadata`接口。一般我们通过继承这个合约来开发一个ERC20的标准。

```typescript
pragma solidity ^0.8.0;

import {ERC20} from "../token/ERC20/ERC20.sol";
contract ERC20Mock is ERC20 {
	constructor() ERC20("ERC20Mock", "E20M") {}
}
```

### 多个扩展

- ERC20Burnable.sol
  
  - 扩展ERC20实现，并增加了`burn`方法，允许销毁代币。
  
- ERC20Capped.sol
  
  - 扩展ERC20实现，并增加一个字段来控制代币数量上限。
  
- ERC20Pausable.sol
  
  - 扩展ERC20实现，并继承Pausable.sol合约，可暂停的代币转让、铸币和焚烧。
  
- ERC20Snapshot.sol
  
  - 扩展ERC20实现，该合约扩展了具有快照机制的ERC20代币，当快照被创建时，当时的余额和总供应量被记录下来，供以后使用
  
- ERC20Votes.sol

- ERC20VotesComp.sol

- ERC20Wrapper.sol
  - 扩展ERC20实现，并增加了`depositFor`和`withdrawTo`方法，该合约允许用户可以存入和提取 "基础代币"，并获得相应数量的 "包装代币"。怎么理解呢?
    - 例如compound，存入ERC20代币获得cERC20代币
  
- ERC20FlashMint.sol
  
  - 扩展ERC20实现，根据ERC-3156中的定义，实现ERC3156闪存贷款扩展。添加`flashLoan`方法，它在token级别提供闪电贷支持。默认情况下，没有费用，但可以通过覆盖flashFee来改变。
  
- ERC4626.sol

- draft-ERC20Permit.sol

  - 对于标准的ERC20的`approve`来说，假设A approve B 100ETH，那么A需要消耗自己的GAS执行approve方法。ERC20Permit提供了一个新的思路，A只需提供离线签名，然后由C代替执行的方式。离线签名的方式参照EIP-2612

    - ```typescript
      bytes32 private constant _PERMIT_TYPEHASH =
              keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)");
      function permit(
              address owner,
              address spender,
              uint256 value,
              uint256 deadline,
              uint8 v,
              bytes32 r,
              bytes32 s
          ) public virtual override {
              require(block.timestamp <= deadline, "ERC20Permit: expired deadline");
              bytes32 structHash = keccak256(abi.encode(_PERMIT_TYPEHASH, owner, spender, value, _useNonce(owner), deadline));
              bytes32 hash = _hashTypedDataV4(structHash);
              address signer = ECDSA.recover(hash, v, r, s);
              require(signer == owner, "ERC20Permit: invalid signature");
              _approve(owner, spender, value);
          }
      ```

  - ERC20PresetMinterPauser

    - 这个合约主要继承了`AccessControlEnumerable`,`ERC20Burnable`,`ERC20Pausable`。可以看出他扩展的功能包括权限，可销毁，可暂停功能。

  - TokenTimeLick

    - 一个代币持有者合同，将允许受益人在特定的释放时间后提取代币。这个合约主要定义了一个时间，受益人只有在这个时间后才能通过`release`方法获得收益。

  - SafeERC20

    - 一个ERC20的`library`. 提供的安全的ERC20操作。为什么说他是安全的，可以看下这个文档(https://learnblockchain.cn/article/3074)

  - ERC20Votes和ERC20VotesComp

    - ERC20VotesComp继承了ERC20Votes

## ERC721

### ERC721标准接口(IERC721.SOL)

```javascript
/**
 * @dev ERC721标准接口.
 */
interface IERC721 is IERC165 {
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);

    function balanceOf(address owner) external view returns (uint256 balance);

    function ownerOf(uint256 tokenId) external view returns (address owner);

    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId,
        bytes calldata data
    ) external;

    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId
    ) external;

    function transferFrom(
        address from,
        address to,
        uint256 tokenId
    ) external;

    function approve(address to, uint256 tokenId) external;

    function setApprovalForAll(address operator, bool _approved) external;

    function getApproved(uint256 tokenId) external view returns (address operator);

    function isApprovedForAll(address owner, address operator) external view returns (bool);
}
```

#### IERC721事件

`IERC721`包含3个事件，其中`Transfer`和`Approval`事件在`ERC20`中也有。

- `Transfer`事件：在转账时被释放，记录代币的发出地址`from`，接收地址`to`和`tokenid`。
- `Approval`事件：在授权时释放，记录授权地址`owner`，被授权地址`approved`和`tokenid`。
- `ApprovalForAll`事件：在批量授权时释放，记录批量授权的发出地址`owner`，被授权地址`operator`和授权与否的`approved`。

#### IERC721函数

- `balanceOf`：返回某地址的NFT持有量`balance`。
- `ownerOf`：返回某`tokenId`的主人`owner`。
- `transferFrom`：普通转账，参数为转出地址`from`，接收地址`to`和`tokenId`。
- `safeTransferFrom`：安全转账（如果接收方是合约地址，会要求实现`ERC721Receiver`接口）。参数为转出地址`from`，接收地址`to`和`tokenId`。
- `approve`：授权另一个地址使用你的NFT。参数为被授权地址`approve`和`tokenId`。
- `getApproved`：查询`tokenId`被批准给了哪个地址。
- `setApprovalForAll`：将自己持有的该系列NFT批量授权给某个地址`operator`。
- `isApprovedForAll`：查询某地址的NFT是否批量授权给了另一个`operator`地址。
- `safeTransferFrom`：安全转账的重载函数，参数里面包含了`data`

### IERC721Receiver接口(防止勿转)

如果一个合约没有实现`ERC721`的相关函数，转入的`NFT`就进了黑洞，永远转不出来了。为了防止误转账，`ERC721`实现了`safeTransferFrom()`安全转账函数，目标合约必须实现了`IERC721Receiver`接口才能接收`ERC721`代币，不然会`revert`。`IERC721Receiver`接口只包含一个`onERC721Received()`函数。

```javascript
// ERC721接收者接口：合约必须实现这个接口来通过安全转账接收ERC721
interface IERC721Receiver {
    function onERC721Received(
        address operator,
        address from,
        uint tokenId,
        bytes calldata data
    ) external returns (bytes4);
}
```

我们看下`ERC721`利用`_checkOnERC721Received`来确保目标合约实现了`onERC721Received()`函数（返回`onERC721Received`的`selector`）：

```
    function _checkOnERC721Received(
        address from,
        address to,
        uint tokenId,
        bytes memory _data
    ) private returns (bool) {
        if (to.isContract()) {
            return
                IERC721Receiver(to).onERC721Received(
                    msg.sender,
                    from,
                    tokenId,
                    _data
                ) == IERC721Receiver.onERC721Received.selector;
        } else {
            return true;
        }
    }
```