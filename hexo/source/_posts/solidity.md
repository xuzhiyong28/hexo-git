---
title: Solidity开发
tags:
  - 区块链
categories:  区块链
description : Solidity开发
date: 2021-05-30 13:53:30
---

### 常见知识点解释

**Solidity中的`状态变量`与`局部变量`**

`_age`和`_name`就属于状态变量。

`age` `name` `name1`就属于局部变量。

```java
pragma solidity >=0.4.22 <0.6.0;
contract Person {
    int public _age;
    string public _name;
    function Person(int age,string name) {
          _age = age;
          _name = name;
    }
    function f(string name) {
          var name1 = name;
    }
}
```

### Solidity中的引用类型`memory`与`storage`

`storage` : 是在合约部署创建时，根据你的合约中状态变量的声明，就固定下来了，并且不能在将来的合约方法调用中改变这个结构。

`memory` : solidity应当在该函数运行时为变量创建一块空间，使其大小和结构满足函数运行的需要。

注 : 

- 状态变量强制是storage类型
- 任何函数参数当它的类型为引用类型时，这个函数参数都默认为memory类型。
- memory类型的变量会临时拷贝一份值存储到内存中，当我们将这个参数值赋给一个新的变量，并尝试去修改这个新的变量的值时，最原始的变量的值并不会发生变化。
- 当函数参数为`memory`类型时相当于**值传递**。
- 当函数参数为`storage`类型时是**指针传递**。

```java
pragma solidity >=0.4.22 <0.6.0;
contract Person {
    string public  _name;
    function Person() {
        _name = "liyuechun";
    }
    function f() {
        modifyName(_name);
    }
    function modifyName(string memory name)  {
        var name1 = name;
        bytes(name1)[0] = 'L';
    }
}
```

在本案例中，当创建合约时，`_name`的值为liyuechun，当我们调用f()函数时，f()函数中会将`_name`的值赋给临时的memory变量name，换句话说，因为name的类型为memory，所以name和`_name`会分别指向不同的对象，当我们尝试去修改name指针指向的值时，_name所指向的内容不会发生变化。

```java
pragma solidity >=0.4.22 <0.6.0;
contract Person {
    string public  _name;
    function Person() {
        _name = "liyuechun";
    }
    function f() {
        modifyName(_name);
    }
    function modifyName(string storage name) internal {
        var name1 = name;
        bytes(name1)[0] = 'L';
    }
}
```

如果想要在modifyName函数中通过传递过来的指针修改`_name`的值，那么必须将函数参数的类型显示设置为storage类型，storage类型拷贝的不是值，而是`_name`指针，当调用modifyName(`_name`)函数时，相当于同时有`_name`，name,name1三个指针同时指向同一个对象，我们可以通过三个指针中的任何一个指针修改他们共同指向的内容的值。**函数默认是public类型的，当我们函数参数有storage类型时，必须加internal或者private修饰表示只能内部调用**。

### Solidty中的`payable`,`view`,`pure`修饰

- `payable`方法是一种可以接收以太的特殊函数。如果一个函数需要进行货币操作，必须带上payable关键字这样才能正常接收`msg.value`。
- `view`修饰的函数，是constant的别名，只能读取storage变量的值。
- `pure`修饰的函数 ，不能对storage变量进行读写。

**Solidity中internal、private、external、public区别**

对于`public`和`private`，相信学过其他主流语言的人都能明白：

- `public`修饰的变量和函数，任何用户或者合约都能调用和访问。
- `private`修饰的变量和函数，只能在其所在的合约中调用和访问，即使是其子合约也没有权限访问。

除 public 和 private 属性之外，Solidity 还使用了另外两个描述函数可见性的修饰词：internal（内部） 和 external（外部）。

- internal 和 private 类似，不过， 如果某个合约继承自其父合约，这个合约即可以访问父合约中定义的“内部”函数。
- external 与public 类似，只不过这些函数只能在合约之外调用 - 它们不能被合约内的其他函数调用。

`internal、private、external、public`这4种关键字都是可见性修饰符，**互不共存**。

### 如何使用call的方式调用其他合约的方法

```java
//Test2.sol
pragma solidity >=0.7.0 <0.9.0;
contract Test2 {
    constructor() public {}

    uint256 public storageData;
    function set(uint256 x) public {
        storageData = x;
    }
}

// Test.sol
pragma solidity >=0.7.0 <0.9.0;
import {Test2} from "./Test2.sol";

contract Test {
    uint  storageData;

    constructor() public {}

    function set(uint x) public {
        storageData = x;
    }  
    // 使用encodeWithSignature编码方法和变量
    function abiEncode(uint256 x) public view returns (bytes memory) {
        return abi.encodeWithSignature("set(uint256)", x);
    }
    // 使用encodeWithSelector编码方法和变量
    function abiEncode2(address addr,uint256 x) public view returns (bytes memory) {
        return abi.encodeWithSelector(Test2(addr).set.selector, x);
    }

    function testCall(address addr , uint256 x) public returns (bool) {
        (bool success, bytes memory result) = addr.call(abi.encodeWithSignature("set(uint256)",x));
        return success;
    }

    function testCall2(address addr, uint256 x) public returns (bool) {
        (bool success, bytes memory result) = addr.call(abi.encodeWithSelector(Test2(addr).set.selector,x));
        return success;
    }

}
```

### ABI编码合约方法详解

我们通过小狐狸调用合约方法发送给以太坊节点时，需要对合约方法和参数进行ABI编码。例如

```javascript
myContract.methods.baz(69,true).encodeABI();
// 下面数值中间的空格是我自己隔开的，方便确认
> 0xcdcd77c0 0000000000000000000000000000000000000000000000000000000000000045 0000000000000000000000000000000000000000000000000000000000000001
```

例如上面，我们对合约方法 `myMethod`方法进行编码得到了一串字符。合约方法的ABI编码包括两部分 ：`函数名编码` + `参数编码`。通过这两部分结合起来得到最终的ABI编码。

**函数名签名**

在EVM中，每个函数都由4个byte长度的16进制值来唯一标识，这4个bytes叫做函数签名。函数签名是对函数名，函数参数做Keccak(SHA-3) 运算后，获得的hash值的前4个bytes。例如上面我们的合约方法`baz(uint32 x, bool y)`。

```javascript
console.log(web3.utils.sha3('baz(uint32,bool)'))
> 0xcdcd77c0992ec5bbfc459984220f8c45084cc24d9b6efed1fae540db8de801d2
// 取前面4个字节最终得到
> cdcd77c0
```

**参数编码**

如上面例子，传入了`69`,`true`

- 69被编码成 `0x0000000000000000000000000000000000000000000000000000000000000045`

- true被编码成 `0x0000000000000000000000000000000000000000000000000000000000000001`

所以最终得到结果

```
0xcdcd77c0 0000000000000000000000000000000000000000000000000000000000000045 0000000000000000000000000000000000000000000000000000000000000001
```

### Fallback回退函数

Fallback回退函数执行的场景 ：

- 如果在一个对合约调用中，没有其他函数与给定的函数标识符匹配，则fallback会被调用。

- 在没有`receive`函数时，且没有提供附加数据对合约调用时，则fallback会被调用。

场景1 ： `Storage.sol`通过调用`callOwner`方法，方法里面使用call调用和`Owner.sol`的`add`方法，但是Owner.sol里面没有add方法，所以触发了Owner.sol合约的`fallback`函数。

场景2 ：`Storage.sol`通过调用`call2Owner`方法，msg.value表示要转的以太币，往`Owner.sol`合约转账msg.value数量的以太币，由于Owner.sol合约没有`receive`方法，所以Owner.sol合约的fallback方法被触发。如果Owner.sol此时有`receive`函数的话，就不会触发fallback函数。

场景3：例如调用`call3Owner`时，msg.value, msg.data都有值的情况下。但是没有`receive`函数，fallback也会触发。

```javascript
// 2_Storage.sol
pragma solidity >=0.7.0 <0.9.0;
import {Owner} from "./2_Owner.sol";

contract Storage {

    Owner h;
    constructor(address payable addr) public{
        h = Owner(addr);
    }
    function callOwner() public {
        payable(address(h)).call(abi.encodeWithSignature("add(uint256,uint256)"));
    }
    function call2Owner() public payable{
        payable(h).call{value : msg.value}("");
    }
    function call3Owner() public payable {
        payable(h).call{value : msg.value}(abi.encodeWithSignature("setX()"));
    }
}

// 2_Owner.sol
pragma solidity >=0.7.0 <0.9.0;
contract Owner {
    uint256 public temp = 0;
    uint256 public temp2 = 0;
    uint256 public balanceOf;
    function SetX() public {

    }

    function initBalance() public {
        balanceOf = address(this).balance;
    }

    fallback() external payable {
        temp = 12345;
    }

    receive() external payable {
        temp2 = 54321;
    }
}
```

### Solidity重入攻击详解

向以太坊合约账户进行转账，发送Ether的时候，会执行合约账户对应合约代码的回调函数（fallback）。在以太坊[智能合约](https://so.csdn.net/so/search?q=%E6%99%BA%E8%83%BD%E5%90%88%E7%BA%A6&spm=1001.2101.3001.7020)中，进行转账操作，一旦向被攻击者劫持的合约地址发起转账操作，迫使执行攻击合约的回调函数，回调函数中包含回调自身代码，将会导致代码执行“重新进入”合约。这种合约漏洞，被称为“重入漏洞”。

Fallback回退函数执行的场景 ：

- 如果在一个对合约调用中，没有其他函数与给定的函数标识符匹配，则fallback会被调用。

- 在没有`receive`函数时，且没有提供附加数据对合约调用时，则fallback会被调用。

例如下面两个合约:

在银行这里例子里，`Hacker.sol`是攻击者合约。Hacker.sol合约通过`hack`方法调用了`Bank.sol`合约的`withdraw`方法。在withdraw方法中，如果msg.sender是一个智能合约的话，`msg.sender.call.value(amount)`调用call，会执行这个智能合约中的Fallback函数。所以又重新执行了Hacker.sol合约的fallback方法。fallback里面又重新执行了Bank.sol合约的`withdraw`方法。所以就造成了**重入攻击**。

```javascript
//Bank.sol
contract Bank {
    mapping(address => uint256) public balanceOf;
    function withdraw(uint256 amount) public {
        require(balanceOf[msg.sender] >= amount);
        msg.sender.call{value:amount}("");
        balanceOf[msg.sender] -= amount;
    }
 }

//Hacker.sol
import {Bank} from "./Bank.sol";
contract Hacker {
    bool status = false;
    Bank b;
    constructor(address addr) public {
        b = Bank(addr);
    }

    function hack() public {
        b.withdraw(1 ether);
    }

    if (!status) {
            status = true;
            b.withdraw(1 ether);
    }
}
```

### solidity合约状态值在底层存储

- 认为一个合约对应了一条无限长的磁带，磁带上以32字节为单位，拥有无数个存储槽；每个存储槽的位置就是它的key，也是用32字节表示。

- 对于简单的，大小在32字节内的变量，以定义变量的顺序作为它的key来存储变量值。即第一个变量的key为0，第二个变量的key为1，……。

- 结构体和定长数组也是顺序存储（只要每个值都是32字节以内的），比如结构体变量定义在位置1，结构体内部要两个成员，则这两个成员的key依序为 1和2。数组类似，只是在处理数组时编译器会多加一些边界检查的代码。

- 连续的若干个小的值，可能被优化为存储的同一个位置，比如：合约中前四个状态变量都是uint64类型的，则四个状态变量的值会被打包成一个32字节的值存储在0位置。

- map中内容的存储，如果map中的value在32字节以内，则会按以下公式得到数据库中的key：keccak256(bytes32(map中的key）+bytes32(map变量的位置))； 例如，一个map变量在合约中最先定义，map中一个key为"abc"，则其在数据库中的存储位置为：keccak256(bytes32("abc"）+bytes32(0))。

- 如果map中的value是一个复杂类型，存储需求超过32字节，则会按上述公式得到第一个存储位置，然后依序加1得到后续数据的存储位置。

- 可变长度数组，与map类似，但更复杂点：以数组变量所在位置为key，存储数组的长度。然后从keccak256(bytes32(position))开始存储数组中的元素。

- 可变长度字节数组和字符串一样：如果长度小于等于31字节，则直接在变量位置处存储字符串值，并用值的最后一个字节存储字符串的编码长度。编码长度 = 字符数 * 2 。比如，"abc"的存储值为"0x6162630000000000000000000000000000000000000000000000000000000006"。

- 当可变长度字节数组或字符串长度大于31字节时，变量位置存储的是 编码长度，而此时编码长度公式变为 编码长度 = 字符数 * 2 + 1 。 然后，从 keccak256(bytes32(position))位置，使用连续的若干个存储槽存储字符串值。 从而，对于字符串，如果编码长度是奇数，则代表的是长字符串，如果是偶数则代表不超过31字节的字符串。

### Solidity通过合约如何转ETH

```javascript
pragma solidity >=0.7.0 <0.9.0;

contract TrantCall{
    constructor() public {   
    }
    // 转账
    function depositEther() public payable{
      // payable(address(this)).call{value:msg.value}("");
    }
    // 取款
    function exitTokens(uint amount) public payable {
        payable(msg.sender).call{value:amount}("");
    }
}
```

- depositEther转账方法，调用者必须执行传msg.value。这样以太币就会将币从msg.sender传给这个合约。合约里面不用做任何东西，做多做个验证。

- exitTokens提取方法，通过指定amount(单位是wei),之后会合约上扣除对应的以太币，然后msg.sender会加入相应的以太币，假如以太币不够将会报错。

### keystore详解

keystore : 它是账户私钥的**加密**版本。

使用keystore的原因 : 直接把账户私钥存储在加密工具中比较危险，你的私钥容易被攻击。但是如果存储用密码加密后的以太坊私钥，则可以保证安全性和可用性。

- 安全性：攻击者需要密钥库文件和密码来才能窃取。

- 可用性：仅需要密码库文件和密码来使用资金。

```
{
  "version": 3,
  // keystore的随机编号
  "id": "2e5fba44-38e9-4733-a52f-6981ca128437",
  // 地址
  "address": "b4551bab04854a09b93492bb61b1b011a82cc27a",
  // 加密描述 - 核心
  "crypto": {
    "ciphertext": "395bf3540ec48a1f31403c635102ab98d61d52989e1aa13a2f031efc7b6ec503",
    "cipherparams": {
      "iv": "30b95477207a06b4613d232a5afc4466"
    },
    "cipher": "aes-128-ctr",
    "kdf": "scrypt",
    "kdfparams": {
      "dklen": 32,
      "salt": "eb660c57dc95991b49b46c78e0f9a7e3d146fd18f495083acb3cad1343136152",
      "n": 8192,
      "r": 8,
      "p": 1
    },
    // 用来校验最终得到的账户私钥明文正不正确
    "mac": "f7182a8e054eb7e7aebe3b96eeb2aa8cd4f6d1db7429183415b8c71b88964e73"
  }
}
```

<img title="" src="solidity/1.png" alt="" data-align="inline">

上图的解密密钥从哪里来？ -- 看下面图，主要也是通过密码

![](solidity/2.png)

最终过程

![](solidity/5.png)
