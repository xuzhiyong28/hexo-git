---
title: openzeppelin(1)_finance金融详解
tags:
  - 区块链
  - openzeppelin
categories:  区块链
date: 2020-08-30 16:04:21
---

以下介绍openzeppeliin中的finance(金融)模块，主要包含两个合约。

### PaymentSplitter

OpenZeppelin的`PaymentSplitter`合约是一个智能合约，用于在以太坊网络上分割支付。它可以用于分割以太币或任何其他 ERC20 兼容代币的支付。

`PaymentSplitter`合约的主要作用是将一笔支付分割成多个份额，按照预定义的比例将这些份额分配给多个账户。这些账户可以是个人、组织或智能合约，它们在收到支付时可以自动执行其他操作。

具体来说，当一个用户向`PaymentSplitter`合约发送支付时，合约会自动将支付金额按照预先设定的比例分配给多个指定的账户。这些账户可以在合约创建时设定，并且可以随时添加或删除。每个账户都有一个指定的份额比例，表示在总支付金额中该账户应该收到的份额比例。

当支付到达`PaymentSplitter`合约时，合约将检查每个账户的份额比例，并按照这些比例将支付金额分配给相应的账户。这些账户可以随时查询其余额，也可以通过与其他智能合约互动来执行其他操作。

`PaymentSplitter`合约的优点在于它可以提高支付的透明度和可靠性。通过使用智能合约分割支付，用户可以确信支付的准确分配和安全性，而不需要依赖第三方中介。

`PaymentSplitter`合约提供了以下公共方法：

**1. `constructor(address[] memory payees, uint256[] memory shares_)`** 构造函数，用于创建`PaymentSplitter`合约并初始化分配比例。

- `payees`：地址数组，表示接收支付的账户列表。
- `shares_`：整数数组，表示每个接收账户收到的份额比例。

**2. `payee(uint256 index) public view returns (address)`** 返回指定索引位置的接收账户地址。

- `index`：整数，表示要查询的接收账户在列表中的索引位置。

**3. `shares(uint256 index) public view returns (uint256)`** 返回指定索引位置的接收账户的份额比例。

- `index`：整数，表示要查询的接收账户在列表中的索引位置。

**4. `release(address payable account) public virtual`** 释放指定账户的可用余额，将余额转移到指定账户的地址。

- `account`：可支付地址，表示要释放可用余额的账户。

**5. `releaseAll() public`** 释放所有接收账户的可用余额，将余额转移到每个接收账户的地址。

**6. `totalShares() public view returns (uint256)`** 返回所有接收账户的总份额。

**7. `totalReleased() public view returns (uint256)`** 返回已释放的总余额。

**8. `released(address account) public view returns (uint256)`** 返回指定账户已释放的余额。

- `account`：可支付地址，表示要查询的账户。

**9. `withdrawPayments() public virtual`** 释放所有接收账户的可用余额，将余额转移到每个接收账户的地址，然后触发`PaymentSplitter`合约的`PaymentReleased`事件。

**10. `renounceOwnership() public virtual`** 将合约的所有者设置为零地址，防止后续所有者意外更改合约的分配方式。

这些方法提供了分割和释放支付的功能，并允许查询账户余额和份额比例。它们使`PaymentSplitter`合约成为一个灵活且易于使用的工具，可以在多个场景中使用，包括分割奖励、股息、销售收入等。具体合约使用可以看github上的官方代码。

**该如何使用PaymentSplitter？**

当需要将一笔支付分割成多个份额并按照预定义比例分配给多个接收账户时，可以使用`PaymentSplitter`合约。

以下是一个使用`PaymentSplitter`合约的示例场景：假设有一个公司想要将每个月的销售收入分配给三个员工，其中销售经理应该获得50％的销售收入，而其他两名员工应该获得25％的销售收入。在这种情况下，可以使用以下步骤来创建和使用`PaymentSplitter`合约：

1. 创建`PaymentSplitter`合约并指定接收账户和份额比例：

```typescript
contract SalesRevenueSplitter is PaymentSplitter {
  constructor(
    address[] memory payees,
    uint256[] memory shares_
  ) PaymentSplitter(payees, shares_) {}
}
```

在此示例中，我们继承自`PaymentSplitter`合约并使用`constructor`函数指定三个接收账户的地址和份额比例：`[address1, address2, address3]`和`[50, 25, 25]`。

1. 当每个月销售收入到达公司的以太坊地址时，将其发送到`SalesRevenueSplitter`合约地址。

```typescript
function paySalesRevenue() public payable {
  address(this).transfer(msg.value);
}
```

1. 当需要向员工分配销售收入时，调用`releaseAll`函数并指定员工的以太坊地址。

```typescript
function distributeSalesRevenue() public {
  releaseAll();
}
```

在此示例中，我们使用`releaseAll`函数将销售收入释放给所有接收账户，并将余额转移到各自的以太坊地址。

通过这种方式，我们可以使用`PaymentSplitter`合约来分割和分配每个月的销售收入，而不需要手动计算份额或使用第三方中介来处理支付。这使得整个过程更加透明、安全和高效。

### VestingWallet

`VestingWallet`合约是OpenZeppelin提供的一种用于分批释放代币的合约，可以用于限制代币的流通，以防止代币一次性流入市场引起价格波动。

具体来说，`VestingWallet`合约可以将代币分配给一个地址，并根据预设的释放计划在一定时间内分批释放代币。这种方式可以让代币逐步流入市场，防止大量代币一次性投放，从而保持代币的价值稳定。

下面是`VestingWallet`合约的具体方法：

- `constructor(address _token, address _beneficiary, uint256 _start, uint256 _duration, uint256 _interval, uint256 _cliff)`：合约构造函数，初始化受益人（即代币的接收者）、代币合约地址、开始时间、释放总时间、释放时间间隔、锁定时间（即开始时间后多久可以开始释放代币）等参数。
- `vestedAmount(uint256 time)`：计算在给定时间下受益人已经获得的代币数量。
- `release()`：释放受益人当前时间可领取的代币数量。

在使用`VestingWallet`合约时，通常需要按照以下步骤进行：

1. 部署`VestingWallet`合约，指定代币合约地址、受益人地址、开始时间、释放总时间、释放时间间隔和锁定时间等参数。

```
VestingWallet vestingWallet = new VestingWallet(tokenAddress, beneficiaryAddress, startTime, duration, interval, cliff);
```

在此示例中，我们使用`new`运算符创建了一个`VestingWallet`合约，并指定代币合约地址、受益人地址、开始时间、释放总时间、释放时间间隔和锁定时间等参数。

1. 将代币转入到`VestingWallet`合约中，以便在未来的时间段内分批释放。

```
token.transfer(address(vestingWallet), totalAmount);
```

在此示例中，我们使用`transfer`函数将代币转移给`VestingWallet`合约，以便在未来根据设定的释放计划分批释放给受益人。

1. 在受益人需要领取代币时，调用`release`函数。

```
vestingWallet.release();
```

在此示例中，我们使用`release`函数将当前时间可以领取的代币释放给受益人。如果当前时间尚未到达可以领取代币的时间，则无法领取。

通过这种方式，我们可以使用`VestingWallet`合约实现对代币的分批释放，从而保持代币的价值稳定，避免代币一次性投放引起价格波动。