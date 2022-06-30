---
title: ethereum
date: 2022-05-05 10:24:26
tags:
    - 区块链
---

### 以太坊交易签名流程

https://learnblockchain.cn/article/1225

```go
func TestTxSign(t *testing.T) {
    // 1.获取合约调用方法的前4个字节，合约方法是包括方法名+参数类型的方式
    // Keccak256 用来计算哈希值
    methodId := crypto.Keccak256([]byte("setA(uint256)"))[:4]
    fmt.Println("methodId: ", common.Bytes2Hex(methodId))
    paramValue := math.U256Bytes(new(big.Int).Set(big.NewInt(123)))
    fmt.Println("paramValue: ", common.Bytes2Hex(paramValue))
    input := append(methodId, paramValue...)
    fmt.Println("input: ", common.Bytes2Hex(input))

    // 2.构建交易对象
    nonce := uint64(1)
    value := big.NewInt(0)
    gasLimit := uint64(3000000)
    gasPrice := big.NewInt(20000000000)
    to := common.HexToAddress("0x05e56888360ae54acf2a389bab39bd41e3934d2b")
    data := &types.LegacyTx{
        To:       &to,
        Nonce:    nonce,
        Gas:      gasLimit,
        GasPrice: gasPrice,
        Value:    value,
        Data:     input,
    }
    rawTx := types.NewTx(data)
    jsonRawTx, _ := rawTx.MarshalJSON()
    fmt.Println("rawTx: ", string(jsonRawTx))

    //3. 交易签名
    // 采用EIP155Signer签名方式， 1表示ChainId
    signer := types.NewEIP155Signer(big.NewInt(1))
    // 私钥 *ecdsa.PrivateKey - secp256k1标准的私钥
    key, err := crypto.HexToECDSA("e8e14120bb5c085622253540e886527d24746cd42d764a5974be47090d3cbc42")
    if err != nil {
        fmt.Println("crypto.HexToECDSA failed: ", err.Error())
        return
    }
    // SignTx的签名过程分成如下几步 ：
    // 1. 对交易信息计算rlpHash
    // 2. 对rlpHash使用私钥进行签名
    // 3. 填充交易对象中的V,R,S字段
    sigTransaction, err := types.SignTx(rawTx, signer, key)
    if err != nil {
        fmt.Println("types.SignTx failed: ", err.Error())
        return
    }
    jsonSigTx, _ := sigTransaction.MarshalJSON()
    fmt.Println("sigTransaction: ", string(jsonSigTx))

    // 4.发送交易
    // 1. 对签名后的交易进行rlp编码
    // 2. jsonrpc的eth_sendRawTransaction 方法发送交易
    ethClient, err := ethclient.Dial("http://127.0.0.1:7545")
    if err != nil {
        fmt.Println("ethclient.Dial failed: ", err.Error())
        return
    }
    // sigTransaction当前已经是填充了VRS，表示是签名后的交易
    err = ethClient.SendRawTransaction(context.Background(), sigTransaction)
    if err != nil {
        fmt.Println("ethClient.SendTransaction failed: ", err.Error())
        return
    }
    fmt.Println("send transaction success,tx: ", sigTransaction.Hash().Hex()) 

    // 获取sigTransaction使用rlp编码后转16进制的数据
    //binary, _ := sigTransaction.MarshalBinary()
    //sigTransactionStr := hexutil.Encode(binary)
    //fmt.Println("sigTransactionStr : ", sigTransactionStr)

}
```

### 以太坊交易解签

签名后的交易发送到以太坊节点后，节点需要从签名交易中还原出公钥（从公钥中单向计算出账号地址），进而将交易放入交易池中。

https://learnblockchain.cn/article/1250

```go
func TestTxUnSign(t *testing.T) {
    // encodedTxStr是具有签名的交易对象的rlp编码
    encodedTxStr := "0xf889018504a817c800832dc6c09405e56888360ae54acf2a389bab39bd41e3934d2b80a4ee919d50000000000000000000000000000000000000000000000000000000000000007b26a0b9fa34e065ea5589135f39f4f6b2ec6954a6635a68e5a9787beb2928c80c356ea01cbb0fb3bb7700f24be544f065c9967af774d01c0c0dfd7b5bede7343b7fe213"
    // 16进制转byte
    encodedTx, err := hexutil.Decode(encodedTxStr)
    if err != nil {
        log.Fatal("hexutil.Decode failed: ", err.Error())
    }
    // rlp解码
    tx := new(types.Transaction)
    if err := rlp.DecodeBytes(encodedTx,tx); err != nil{
        log.Fatal("rlp.DecodeBytes failed: ", err.Error())
    }
    signer := types.NewEIP155Signer(big.NewInt(1))
    // 使用签名器从已签名的交易中还原出账户公钥
    from, err := types.Sender(signer, tx)
    if err != nil {
        log.Fatal("types.Sender: ", err.Error())
    }
    fmt.Println("from: ", from.Hex())
}
```

### FetCher与Downloader区别

Fetcher用于同步网络节点的新区块和新的交易数据，如果新区块和本地最新的区块相隔距离较远，说明本地区块数据太旧，Fetcher就不会同步这些区块。这时候就要借助Downloader来同步完整的区块数据。

### MPT存储涉及到的三种编码方式

- KeyBytes编码
- Hex编码
- Compact编码

在完成Compact编码后，会通过折叠操作把子结点替换成子结点的hash值，然后以键值对的形式将所有结点存储到数据库中。下面详细介绍上面3中编码方式。

参考 ：

- https://blog.csdn.net/TurkeyCock/article/details/80616341
- https://www.cnblogs.com/1314xf/p/14240439.html

#### KeyBytes编码

图 ：[https://img-blog.csdn.net/20180607223625109?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1R1cmtleUNvY2s=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70]

即原始关键字，比如图中0x811344、0x879337等。每个字节中包含2个nibble（半字节，4 bits），每个nibble的数值范围时0x0~0xF。

```go
// keybytes编码转换为Hex编码
func keybytesToHex(str []byte) []byte {
    l := len(str) * 2 + 1
    var nibbles = make([]byte , l)
    for i, b := range str {
		nibbles[i*2] = b / 16
		nibbles[i*2+1] = b % 16
	}
	nibbles[l-1] = 16
	return nibbles
}

func TestHexKeybytes(t *testing.T) {
	tests := []struct{ key, hexIn, hexOut []byte }{
		{key: []byte{}, hexIn: []byte{16}, hexOut: []byte{16}},
		{key: []byte{}, hexIn: []byte{}, hexOut: []byte{16}},
		{
			key:    []byte{0x12, 0x34, 0x56},
			hexIn:  []byte{1, 2, 3, 4, 5, 6, 16},
			hexOut: []byte{1, 2, 3, 4, 5, 6, 16},
		},
		{
			key:    []byte{0x12, 0x34, 0x5},
			hexIn:  []byte{1, 2, 3, 4, 0, 5, 16},
			hexOut: []byte{1, 2, 3, 4, 0, 5, 16},
		},
		{
			key:    []byte{0x12, 0x34, 0x56},
			hexIn:  []byte{1, 2, 3, 4, 5, 6},
			hexOut: []byte{1, 2, 3, 4, 5, 6, 16},
		},
	}
	for _, test := range tests {
		if h := keybytesToHex(test.key); !bytes.Equal(h, test.hexOut) {
			t.Errorf("keybytesToHex(%x) -> %x, want %x", test.key, h, test.hexOut)
		}
		if k := hexToKeybytes(test.hexIn); !bytes.Equal(k, test.key) {
			t.Errorf("hexToKeybytes(%x) -> %x, want %x", test.hexIn, k, test.key)
		}
	}
}
```

#### Hex编码

由于我们需要以nibble为单位进行编码并插入MPT，因此需要把一个字节拆分成两个，转换为Hex编码。

#### Compact编码

当我们需要把内存中MPT存储到数据库中时，还需要再把两个字节合并为一个字节进行存储。这时候会碰到这些问题:

- 关键字长度为奇数，有一个字节无法合并
- 需要区分结点是扩展结点还是叶子结点，为了解决这个问题，以太坊设计了一种Compact编码方式，具体规则如下：
  - 扩展结点，关键字长度为偶数，前面加00前缀
  - 扩展结点，关键字长度为奇数，前面加1前缀（前缀和第1个字节合并为一个字节）
  - 叶子结点，关键字长度为偶数，前面加20前缀（因为是Big Endian）
  - 叶子结点，关键字长度为奇数，前面加3前缀（前缀和第1个字节合并为一个字节）

```go
// hex编码转换为Compact编码
func hexToCompact(hex []byte) []byte {
    terminator := byte(0)
    if hasTerm(hex) {
        terminator = 1
        hex = hex[:len(hex)-1]
    }
    buf := make([]byte, len(hex)/2+1)
    buf[0] = terminator << 5 // the flag byte
    if len(hex)&1 == 1 {
        buf[0] |= 1 << 4 // odd flag
        buf[0] |= hex[0] // first nibble is contained in the first byte
        hex = hex[1:]
    }
    decodeNibbles(hex, buf[1:])
    return buf
}
func TestHexCompact(t *testing.T) {
	tests := []struct{ hex, compact []byte }{
		// empty keys, with and without terminator.
		{hex: []byte{}, compact: []byte{0x00}},
		{hex: []byte{16}, compact: []byte{0x20}},
		// odd length, no terminator
		{hex: []byte{1, 2, 3, 4, 5}, compact: []byte{0x11, 0x23, 0x45}},
		// even length, no terminator
		{hex: []byte{0, 1, 2, 3, 4, 5}, compact: []byte{0x00, 0x01, 0x23, 0x45}},
		// odd length, terminator
		{hex: []byte{15, 1, 12, 11, 8, 16 /*term*/}, compact: []byte{0x3f, 0x1c, 0xb8}},
		// even length, terminator
		{hex: []byte{0, 15, 1, 12, 11, 8, 16 /*term*/}, compact: []byte{0x20, 0x0f, 0x1c, 0xb8}},
	}
	for _, test := range tests {
		if c := hexToCompact(test.hex); !bytes.Equal(c, test.compact) {
			t.Errorf("hexToCompact(%x) -> %x, want %x", test.hex, c, test.compact)
		}
		if h := compactToHex(test.compact); !bytes.Equal(h, test.hex) {
			t.Errorf("compactToHex(%x) -> %x, want %x", test.compact, h, test.hex)
		}
	}
}
```

### ethereum中P2P-nodeID如何生成？

- 通过crypto.GenerateKey随机生成一对公私钥。（取出后会把私钥放到nodeKey文件中去）
- 通过crypto.FromECDSAPub(&key.PublicKey)[1:]取最终的nodeID。

```go
func TestNodeGet(t *testing.T){
	// 生成随机私钥
	key, _ := crypto.GenerateKey()
	// 取私钥对应的公钥
	nodeId := fmt.Sprintf("%x", crypto.FromECDSAPub(&key.PublicKey)[1:])
	// 得到最终的nodeID
	fmt.Println(fmt.Sprintf("enode://%s", nodeId))

	keyFromFile , _ := crypto.LoadECDSA("D:\\blockchainspack\\matic\\end\\bor_config\\miner1\\data\\bor\\nodekey")
	nodeidFromFile := fmt.Sprintf("%x", crypto.FromECDSAPub(&keyFromFile.PublicKey)[1:])
	fmt.Println(fmt.Sprintf("enode://%s", nodeidFromFile))
}
```

### 开启挖矿的逻辑

```go
// backend.go
eth.miner = miner.New(eth, &config.Miner, chainConfig, eth.EventMux(), eth.engine, eth.isLocalBlock)
 - go miner.update()

func (miner *Miner) update() {
    // 区块同步的事件
	events := miner.mux.Subscribe(downloader.StartEvent{}, downloader.DoneEvent{}, downloader.FailedEvent{})
	// .. 省略代码
	shouldStart := false
	canStart := true	// 是否可以挖矿
	dlEventCh := events.Chan()
	for {
		select {
		case ev := <-dlEventCh:
			if ev == nil {
				dlEventCh = nil
				continue
			}
			switch ev.Data.(type) {
			case downloader.StartEvent:	// 情况1 ：区块正在下载事件
				wasMining := miner.Mining()
				miner.worker.stop() // 停止挖矿
				canStart = false	// 设置不挖矿
				if wasMining {
					shouldStart = true
					log.Info("Mining aborted due to sync")
				}
			case downloader.FailedEvent: //情况2 ： 下载失败事件
				canStart = true	 // 设置挖矿
				if shouldStart {
					miner.SetEtherbase(miner.coinbase)
					miner.worker.start() // 开启挖矿
				}
			case downloader.DoneEvent:	// 情况3 ：下载区块完成
				canStart = true
				if shouldStart {
					miner.SetEtherbase(miner.coinbase)
					miner.worker.start() // 开启挖矿
				}
				// Stop reacting to downloader events
				events.Unsubscribe()
			}
        case addr := <-miner.startCh: // 情况3：执行了 miner.start(1)命令
			miner.SetEtherbase(addr)
			if canStart {
				miner.worker.start()// 开启挖矿
			}
			shouldStart = true
		case <-miner.stopCh:
			shouldStart = false
			miner.worker.stop()
		case <-miner.exitCh:
			miner.worker.close()
			return
		}
	}
}
```

### downloader下载代码导图

```go
func New(stack *node.Node, config *ethconfig.Config) // backend.go
  - eth.handler, err = newHandler(&handlerConfig{.....})
     - h.downloader = downloader.New(h.checkpointNumber, config.Database, h.stateBloom, h.eventMux, h.chain, nil, h.removePeer)
```

