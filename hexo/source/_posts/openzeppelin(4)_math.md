## Solidity中的溢出

### 0.8版本之前的溢出漏洞

我们先来看一个合约

```javascript
pragma solidity ^0.4.18;
contract TimeLock {
    
     // uint 等价于 uint256
     function sub(uint a, uint b) public pure returns (uint) {
         return a - b;
     }
    
     function add(uint8 a, uint8 b) public pure returns (uint8) {
         return a + b;
     }
    
}
```

- `sub`方法假设我输入的参数为(a = 0, b = -1) , 则会发生下溢，输出的值为115792089237316195423570985008687907853269984665640564039457584007913129639935(超级大的数)。
- `add`方法假设输入的参数为(a = 255, b = 1) 则会发生下溢， 输出的值为 0

所以在0.8版本之前若加减乘除超出了类型的取值返回就会发生溢出，导致值与预期的不同，造成合约有漏洞问题，例如我需要判断`add`方法若返回256则执行某些方法，但是其实返回的是0。由于这个漏洞也造成了一些以太币被盗取的事件。所以在0.8之前的版本要解决溢出问题会使用openzeppelin的`SafeMath`库。

### 0.8版本后

0.8版本之后在语言层面已经解决了这个问题，**一旦发生整数溢出，transaction会直接被revert，同时也提供了unchecked关键字用来忽略报错，若溢出则返回type(类型).max或type(类型).min**。

## 安全的算术运算库：SafeMath

所以`SafeMath.sol`是用来解决0.8版本之前的溢出问题，同时保证溢出时不会抛出异常，而是返回false.我们具体来看一个例子

```javascript 
library SafeMath {
	function tryAdd(uint256 a, uint256 b) internal pure returns (bool, uint256) {
        unchecked {
            uint256 c = a + b;
            if (c < a) return (false, 0);
            return (true, c);
        }
    }
}
```

例如上面的加法运算，他是如何保证不抛出异常且检查溢出的

- 使用unchecked关键字来忽略检查，所以溢出时不会报错
- 若a+b的值c是溢出的，那么当判断c<a时就会返回false.

SafeMath中的方法基本都采用这种方式，具体代码可以自行参考源代码。

## 安全类型转换库：SafeCast

这个库合约库的作用就在于可以帮你完成这些转换而无需担心溢出问题。