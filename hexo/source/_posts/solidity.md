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

**Solidity中的引用类型`memory`与`storage`**

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

**Solidty中的`payable`,`view`,`pure`修饰**

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
