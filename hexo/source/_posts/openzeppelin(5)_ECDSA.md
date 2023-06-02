## ethers.js+ECDSA实现Message签名和验签

### 前端

```javascript
const {expect} = require("chai");
const {loadFixture} = require("@nomicfoundation/hardhat-network-helpers");
const { ethers } = require('hardhat');
// npx hardhat test ./test/edcsa_mock_test.js

describe('ECDSAMock Test', () => {
    let ECDSAMock;
    let instanceECDSA;
    beforeEach(async () => {
        this.accounts = await ethers.provider.listAccounts();
        ECDSAMock = await ethers.getContractFactory("ECDSAMock");
        instanceECDSA = await ECDSAMock.deploy();
        await instanceECDSA.deployed();
    })

    it('test_001', async () => {
        let message = "Hello World";
        let [signer] = await ethers.getSigners();
        // 1. 消息使用keccak256进行哈希
        let messageHash = ethers.utils.solidityKeccak256(["string"], [message]);
        const signature = await signer.signMessage(ethers.utils.arrayify(messageHash));
        // 将签名进行转换
        console.log('合约地址:' + instanceECDSA.address);
        console.log('原始消息:' + message);
        console.log('hash(message):' + messageHash);
        console.log('signature:' + signature);
        console.log('signer.address:' + await signer.getAddress());
        let reviceAddress = await instanceECDSA.verifySignature(messageHash, signature);
        console.log("reviceAddress:" + reviceAddress);
    })
});
```

### 合约

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
contract ECDSAMock {
    using ECDSA for bytes32;

    function verifySignature(
        bytes32 messageHash,
        bytes memory signature
    ) public pure returns (address) {
        bytes32 ethSignedMessageHash = ECDSA.toEthSignedMessageHash(messageHash);
        address signer = ECDSA.recover(ethSignedMessageHash,signature);
        return signer;
    }
}

//    function toEthSignedMessageHash(bytes32 hash) internal pure returns (bytes32 message) {
//        assembly {
//            mstore(0x00, "\x19Ethereum Signed Message:\n32")
//            mstore(0x1c, hash)
//            message := keccak256(0x00, 0x3c)
//        }
//    }

```

疑问点：为啥要多加`ECDSA.toEthSignedMessageHash(messageHash)`

以太坊提案对于通用消息签名方法需要添加`\x19Ethereum Signed Message:\n32`。personal_sign(**ethers.getSignersa方法里面调用以太坊的personal_sign接口**)方法在签名时没有添加 "Ethereum Signed Message" 前缀，但在验证签名时，你需要使用 `ECDSA.toEthSignedMessageHash` 方法对消息进行预处理，以确保与签名数据一致的哈希比较。这是为了保持验证签名的一致性。

### 总结

通用签名方法实现逻辑 :

1. 参数Message -> keccak256得到messageHash -> 转成以太坊签名消息哈希messageEthHash -> 通过私钥运算获得签名signature。
   - 消息HASH -> keccak256(abi.encodePacked(parameter...)
   - 以太坊签名消息 -> keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", hash))
2. 合约验证逻辑：ecrecover(_msgHash, v, r, s) 可以"消息"和"签名"中的vrs推出地址，然后我们在我们需要的程序中对其进行对比即可。

## EIP-712提案下签名验签



