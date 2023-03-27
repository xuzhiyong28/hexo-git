## Compound的合约结构

Compound合约模块主要包含

- **InterestRateModel**: 利率模型抽象合约，分为 V1 和 V2 版本。V2 比 V1 增加了拐点型利率模型
- **Comptroller**: 审计合约，对存取借贷等核心业务进行审查和校验
- **PriceOracle**: 价格预言机，用来获取资产价格
- **CToken**: 也称为生息代币，是用户在 Compound 上存入资产的凭证
- **Governance**: 治理相关，不属于核心业务逻辑
- **Lens**: 聚合查询，方便前端调用接口。不属于核心业务逻辑

```
/contracts
├── CToken                              // CToken相关
|  ├── CDaiDelegate.sol
|  ├── CErc20.sol
|  ├── CErc20Delegate.sol
|  ├── CErc20Delegator.sol
|  ├── CErc20Immutable.sol
|  ├── CEther.sol
|  ├── CToken.sol
|  └── CTokenInterfaces.sol
├── Comptroller                         // 审计相关
|  ├── Comptroller.sol
|  ├── ComptrollerG7.sol
|  ├── ComptrollerInterface.sol
|  ├── ComptrollerStorage.sol
|  └── Unitroller.sol
├── Governance                          // 治理相关
|  ├── Comp.sol
|  ├── GovernorAlpha.sol
|  ├── GovernorBravoDelegate.sol
|  ├── GovernorBravoDelegateG1.sol
|  ├── GovernorBravoDelegateG2.sol
|  ├── GovernorBravoDelegator.sol
|  └── GovernorBravoInterfaces.sol
├── InterestRateModel                   // 利率模型
|  ├── BaseJumpRateModelV2.sol
|  ├── DAIInterestRateModelV3.sol
|  ├── InterestRateModel.sol
|  ├── JumpRateModel.sol
|  ├── JumpRateModelV2.sol
|  └── WhitePaperInterestRateModel.sol
├── Lens                                // 聚合查询
|  └── CompoundLens.sol
├── PriceOracle                         // 价格预言机
|  ├── PriceOracle.sol
|  └── SimplePriceOracle.sol
├── interfaces                          // 接口
|  ├── EIP20Interface.sol
|  └── EIP20NonStandardInterface.sol
└── utils                               // 其他全局基础工具类
   ├── ErrorReporter.sol
   ├── ExponentialNoError.sol
   ├── Maximillion.sol
   ├── Reservoir.sol
   ├── SafeMath.sol
   └── Timelock.sol
```

## 利率模型合约

InterestRateModel合约是用来计算 Compound 上特定代币借贷利率的合约，分析它可以知道借贷利率的计算方法。他只要有两种实现**WhitePaperInterestRateModel.sol(直线型利率)**和**JumpRateModelV2.sol(拐点型利率)**。

### 直线型

```typescript
contract WhitePaperInterestRateModel is InterestRateModel {
	using SafeMath for uint;
    event NewInterestParams(uint baseRatePerBlock, uint multiplierPerBlock);
    // 利率模型假设的每年大约块数
    uint public constant blocksPerYear = 2102400;
	// 利率斜率的利用率乘数
    uint public multiplierPerBlock;
    // 基准利率
    uint public baseRatePerBlock;
	
    /*
    	baseRatePerYear 表示年化利率
    	multiplierPerYear 利用率乘数
    **/
    constructor(uint baseRatePerYear, uint multiplierPerYear) public {
        // 每个块的利率 = 年化利率 / 2102400
        baseRatePerBlock = baseRatePerYear.div(blocksPerYear);
        // 每个块的利率乘数 = 利用率乘数 / 2102400
        multiplierPerBlock = multiplierPerYear.div(blocksPerYear);
        emit NewInterestParams(baseRatePerBlock, multiplierPerBlock);
    }
    
    /**
    	cash #资金池余额  borrows #总借款 reserves #储备金
    */
    function utilizationRate(uint cash, uint borrows, uint reserves) public pure returns(uint) {
        if (borrows == 0) {
            return 0;
        }
        // 资金使用率 = 总借款 / (资金池余额 + 总借款 - 储备金)
        return borrows.mul(1e18).div(cash.add(borrows).sub(reserves));
    }
    
    //  计算当前每个区块的借款利率
    function getBorrowRate() public view returns(uint) {
        // 获取资金使用率
        uint ur = utilizationRate(cash, borrows, reserves);
        // 借款利率 = 资金使用率 * 区块斜率 + 基准利率
        return ur.mul(multiplierPerBlock).div(1e18).add(baseRatePerBlock);
    }
    
    /**
    	计算存款利率
    	cash #资金池余额  borrows #总借款 reserves #储备金
    	reserveFactorMantissa 储备金率
    **/
    function getSupplyRate(uint cash, uint borrows, uint reserves, uint reserveFactorMantissa) public view returns(uint) {
        // 1 - 储备金率(扩大的精度)
        uint oneMinusReserveFactor = uint(1e18).sub(reserveFactorMantissa);
        // 获取每个区块的借款利率
        uint borrowRate = getBorrowRate(cash, borrows, reserves);
        // 借款利率 * 储备金 - 1e18(减去扩大的精度)
        uint rateToPool = borrowRate.mul(oneMinusReserveFactor).div(1e18);
		// 存款利率 = 资金使用率 * 借款利率 *（1 - 储备金率）
		// utilizationRate * borrowRate * (1 - reserveFactor)
        return utilizationRate(cash, borrows, reserves).mul(rateToPool).div(1e18);
    }
    
}
```

Compound计算利率时是按照一个block的利率来进行计算的，所以每个计息周期为**1 block = 15 秒**。基本上直线型的借款利率符合公式

```
# x 表示资金使用率  k 区块斜率  b 基准利率
y = k * x + b
```

### 拐点型

拐点型的合约是**JumpRateModelV2.sol**，而具体详细实现大多在**BaseJumpRateModelV2.sol**上。

```typescript
//代码片段
// 计算借款利率
function getBorrowRateInternal(
        uint cash, // 代币余额
        uint borrows, // 用户借出代币总数
        uint reserves // 储备代币总数
    ) internal view returns (
        uint    // 块借出利率
    ) {
        // 获取市场使用率
        uint util = utilizationRate(cash, borrows, reserves);
        // 判断市场使用率 是否到达指定拐点
        if (util <= kink) {
            console.log("util.mul(multiplierPerBlock).div(1e18).add(baseRatePerBlock)", util.mul(multiplierPerBlock).div(1e18).add(baseRatePerBlock));
             // util * multiplierPerBlock + baseRatePerBlock
            //  块借出利率 = 资金借出率 * 块利率乘数 + 块基准利率
            return util.mul(multiplierPerBlock).div(1e18).add(baseRatePerBlock);
        } else {
            // 块借出利率 = (资金借出率 - 拐点资金借出率) * 拐点块利率乘数 + 拐点资金借出率 * 块利率乘数 + 块基准利率

            // 拐点前块借出利率 = kink * multiplierPerBlock + baseRatePerBlock
            uint normalRate = kink.mul(multiplierPerBlock).div(1e18).add(baseRatePerBlock);
            // 超出拐点资金借出率 = util - kink
            uint excessUtil = util.sub(kink);
            // 根据超过拐点计算
            // 块借出利率 = (util - kink) * jumpMultiplierPerBlock + kink * multiplierPerBlock + baseRatePerBlock
            return excessUtil.mul(jumpMultiplierPerBlock).div(1e18).add(normalRate);
        }
    }
```

## cToken资金池

​	Compound支持的代币都有一个对应的资金池。放贷人将ERC20Token放到资金池中获得对应的cToken。**cToken.sol**合约是基类合约，没有构造函数且声明了几个抽象函数，由上层的合约实现，那么他的上层合约包括两类，一类是用来处理ETH的**CEther.sol**，一类是用来处理ERC20的**cERC20.sol**。

​	简而言之，**ETH** 的 cToken 交互入口是 **CEther** 合约，仅此一份；而 **ERC20** 的 cToken 交互入口则是 **CErc20Delegator** 合约，每种 ERC20 资产都各有一份入口合约。

### cERC20的代理模式

- CToken.sol为基类函数
- CErc20.sol继承cToken.sol，实现CErc20Interface.sol合约接口
- CErc20Delegate.sol继承cErc20.sol，里面比CErc20.sol多了两个方法，具体不知道为何要这样做
- CErc20Delegator.sol为代理合约，代理CErc20Delegate.sol。

### cToken合约详解

cToken.sol是所有资金池的基础合约，他是整个借贷存款的核心代码。

**初始化方法**

cErc20.sol的初始化方法里面调用了cToken.sol的初始化方法。主要是设置了一些值

- underlying_ : 标的合约地址，例如要为DAI合约做资金池，这里underiying表示DAI的合约地址
- comptroller_ : 审计合约地址，所有的cToken都使用同一个审计合约地址
- interestRateModel_ : 利率模型地址
- initialExchangeRateMantissa_ :  初始利率

```typescript
// cErc20.sol的初始化方法
function initialize(
        address underlying_,
        ComptrollerInterface comptroller_,
        InterestRateModel interestRateModel_,
        uint initialExchangeRateMantissa_,
        string memory name_,
        string memory symbol_,
        uint8 decimals_
    ) public {
        // 初始化cToken
        super.initialize(
            comptroller_, 
            interestRateModel_, 
            initialExchangeRateMantissa_, 
            name_, 
            symbol_, 
            decimals_
        );
        underlying = underlying_;
        EIP20Interface(underlying).totalSupply();
    }

// cToken.sol合约的初始化方法
function initialize(
        ComptrollerInterface comptroller_,  //  审计合约
        InterestRateModel interestRateModel_,   //  利率模型合约
        uint initialExchangeRateMantissa_,  //  初始利率
        string memory name_,    //  名字
        string memory symbol_,  //  符号
        uint8 decimals_     //  小数
    ) public {
        // 判断是否为管理员
        require(msg.sender == admin, "only admin may initialize the market");
        // 市场只能初始化一次
        // 校验 上一次计算利息区块和市场开放以来的累计值 都为0
        require(accrualBlockNumber == 0 && borrowIndex == 0, "market may only be initialized once");
        // 通过传入的利率来 设置初始汇率
        initialExchangeRateMantissa = initialExchangeRateMantissa_;
        // 利率必须大于0
        require(initialExchangeRateMantissa > 0, "initial exchange rate must be greater than zero.");
        // 设置审计合约
        uint err = _setComptroller(comptroller_);
        // 确保控制器修改成功
        require(err == uint(Error.NO_ERROR), "setting comptroller failed");
        //  获取当前区块
        accrualBlockNumber = getBlockNumber();
        // 设置 利率累计值 默认为 1e18
        borrowIndex = mantissaOne;
        // 设置利率模型（取决于区块数/借款指数）
        err = _setInterestRateModelFresh(interestRateModel_);
        // 确保返回成功
        require(err == uint(Error.NO_ERROR), "setting interest rate model failed");
        name = name_;
        symbol = symbol_;
        decimals = decimals_;
        // 计数器开始为真以防止将其从零更改为非零（即较小的成本/退款）
        // 防止重入攻击？
        _notEntered = true;
    }
```

cToken的具体方法

- mint : 存款，该操作会将用户的标的资产转给cToken合约，并根据最新的兑换率将对应的 cToken 代币转到用户钱包地址。
- redeem : 赎回存款，即用 cToken 换回标的资产，会根据最新的兑换率计算能换回多少标的资产。
- redeemUnderlying : 与redeem一样，只是这个方法支持指定赎回数量。
- borrow : 借款，会根据用户的抵押物来计算可借额度，借款成功则将所借资产从资金池中直接转到用户钱包地址。
- repayBorrow : 还款，当指定还款金额为 -1 时，则表示全额还款，包括所有利息，否则，则会存在利息没还尽的可能，因为每过一个区块就会产生新的利息。
- repayBorrowBehalf : 代还款，即支付人帮借款人还款。
- liquidateBorrow : 清算，任何人都可以调用此函数来担任清算人，直接借款人、还款金额和清算的 cToken 资产，清算时，清算人帮借款人代还款，并得到借款人所抵押的等值+清算奖励的 cToken 资产。
- accrueInterest : 函数计算新的利息，上面方法执行时都会先执行这个函数。

**利息计算方法accrueInterest**

```typescript
function accrueInterest() public returns (uint) {
	// 获取当前区块
    uint currentBlockNumber = getBlockNumber();
    // 最近一次计算的区块
    uint accrualBlockNumberPrior = accrualBlockNumber;
	// 如果两个区块相等表示已经计算过利息了无需再计算,直接返回
    if (accrualBlockNumberPrior == currentBlockNumber) {
        return uint(Error.NO_ERROR);
    }
    // 获取资金池余额, 例如DAI代币,这里的资金池余额获取的是在DAI合约中当前地址的balanceOf
    uint cashPrior = getCashPrior();
    // 获取总借款
    uint borrowsPrior = totalBorrows;
    // 获取总储备金
    uint reservesPrior = totalReserves;
    // 获取借款指数
    uint borrowIndexPrior = borrowIndex;
    // 获取借款利率，interestRateModel:直线型|拐点型
    uint borrowRateMantissa = interestRateModel.getBorrowRate(cashPrior, borrowsPrior, reservesPrior);
    // 借款利率不能大于最大借款利率
    require(borrowRateMantissa <= borrowRateMaxMantissa, "borrow rate is absurdly high");
    // 计算"当前区块"和"最近计算过利息的区块"之间的区块数
    (MathError mathErr, uint blockDelta) = subUInt(currentBlockNumber, accrualBlockNumberPrior);
    // 当前区块 要大于或者等于 最近计算过利息的区块
    require(mathErr == MathError.NO_ERROR, "could not calculate block delta");
    
    Exp memory simpleInterestFactor;
    uint interestAccumulated;   // 总借款在未计算利率的区块产生的总利息
    uint totalBorrowsNew;       // 新的总借款数
    uint totalReservesNew;      
    uint borrowIndexNew;
    // simpleInterestFactor = borrowRate * blockDelta，区块区间内的单位利息
    // 未计算利率的区块产生的利率 = 借款利率 * 未计算利率的区块
    (mathErr, simpleInterestFactor) = mulScalar(Exp({mantissa: borrowRateMantissa}), blockDelta);
    if (mathErr != MathError.NO_ERROR) {
        return failOpaque(Error.MATH_ERROR, FailureInfo.ACCRUE_INTEREST_SIMPLE_INTEREST_FACTOR_CALCULATION_FAILED, uint(mathErr));
    }
    
    // interestAccumulated = simpleInterestFactor * totalBorrows(总借款额度)
    // interestAccumulated =  未计算利率的区块产生的利率 * 总借款额度
    (mathErr, interestAccumulated) = mulScalarTruncate(simpleInterestFactor, borrowsPrior);
    if (mathErr != MathError.NO_ERROR) {
        return failOpaque(Error.MATH_ERROR, FailureInfo.ACCRUE_INTEREST_ACCUMULATED_INTEREST_CALCULATION_FAILED, uint(mathErr));
    }
    // totalBorrowsNew = interestAccumulated + totalBorrows
    // 借款总数 = 利息总数 + 总借款额度
    (mathErr, totalBorrowsNew) = addUInt(interestAccumulated, borrowsPrior);
    if (mathErr != MathError.NO_ERROR) {
            return failOpaque(Error.MATH_ERROR, FailureInfo.ACCRUE_INTEREST_NEW_TOTAL_BORROWS_CALCULATION_FAILED, uint(mathErr));
    }
    // totalReservesNew = interestAccumulated * reserveFactor + totalReserves
    // 根据储备金率将部分利息累加到储备金中
    (mathErr, totalReservesNew) = mulScalarTruncateAddUInt(Exp({mantissa: reserveFactorMantissa}), interestAccumulated, reservesPrior);
    if (mathErr != MathError.NO_ERROR) {
            return failOpaque(Error.MATH_ERROR, FailureInfo.ACCRUE_INTEREST_NEW_TOTAL_RESERVES_CALCULATION_FAILED, uint(mathErr));
    }
    // borrowIndexNew = simpleInterestFactor * borrowIndex + borrowIndex
    // 累加借款指数
    (mathErr, borrowIndexNew) = mulScalarTruncateAddUInt(simpleInterestFactor, borrowIndexPrior, borrowIndexPrior);
        if (mathErr != MathError.NO_ERROR) {
            return failOpaque(Error.MATH_ERROR, FailureInfo.ACCRUE_INTEREST_NEW_BORROW_INDEX_CALCULATION_FAILED, uint(mathErr));
    }
    
    accrualBlockNumber = currentBlockNumber;    //  当前区块
    borrowIndex = borrowIndexNew;       //  借款指数
    totalBorrows = totalBorrowsNew;     //  借款总量
    totalReserves = totalReservesNew;   //  储备金总量
    // 发事件通知
    emit AccrueInterest(cashPrior,interestAccumulated,borrowIndexNew,totalBorrowsNew);
    return uint(Error.NO_ERROR);
}
```

**存款mint**

​	将用户的标的资产转入 cToken 合约中（数据会存储在代理合约中），并根据最新的兑换率将对应的 cToken 代币转到用户钱包地址。

```typescript
function mintInternal(uint mintAmount) internal nonReentrant returns (uint, uint) {
   // 计算利息
   uint error = accrueInterest();
   if (error != uint(Error.NO_ERROR)) {
      // 在同一区块的话 不执行
      return (fail(Error(error), FailureInfo.MINT_ACCRUE_INTEREST_FAILED), 0);
   }
   return mintFresh(msg.sender, mintAmount);
}

function mintFresh(address minter, uint mintAmount) internal returns (uint, uint) {
    // 调用审计合约方法检查是否允许当前地址存入指定数量的代币或ETH
    uint allowed = comptroller.mintAllowed(address(this), minter, mintAmount);
    if (allowed != 0) {
            return (failOpaque(Error.COMPTROLLER_REJECTION, FailureInfo.MINT_COMPTROLLER_REJECTION, allowed), 0);
    }
    // 验证当前区块高度是否等于市场保存的区块高度。如果不一致则抛出异常 accrualBlockNumber计算利息的时候有更新
    if (accrualBlockNumber != getBlockNumber()) {
    	return (fail(Error.MARKET_NOT_FRESH, FailureInfo.MINT_FRESHNESS_CHECK), 0);
    }
    MintLocalVars memory vars;
    // 计算当前兑换率
    // exchangeRateStoredInternal : 首先检查当前总的供给量是否为0，如果是返回初始的兑换率，否则使用 (totalCash + totalBorrows - totalReserves) / totalSupply 计算新的的兑换率
    (vars.mathErr, vars.exchangeRateMantissa) = exchangeRateStoredInternal();
    if (vars.mathErr != MathError.NO_ERROR) {
        return (failOpaque(Error.MATH_ERROR, FailureInfo.MINT_EXCHANGE_RATE_READ_FAILED, uint(vars.mathErr)), 0);
    }
    // 计算发起者实际转入到合约中的 Token 数量
    vars.actualMintAmount = doTransferIn(minter, mintAmount);
    // 计算需要铸造的 cToken 数量
    // 铸造的 cToken 数量等于真实输入到合约中的 Token 数量除以兑换率，公式如下：mintTokens = actualMintAmount / exchangeRate
    (vars.mathErr, vars.mintTokens) = divScalarByExpTruncate(vars.actualMintAmount, Exp({mantissa: vars.exchangeRateMantissa}));
    require(vars.mathErr == MathError.NO_ERROR, "MINT_EXCHANGE_CALCULATION_FAILED");
    // 把铸造出来的 cToken 数量加入到总供给中
    (vars.mathErr, vars.totalSupplyNew) = addUInt(totalSupply, vars.mintTokens);
    require(vars.mathErr == MathError.NO_ERROR, "MINT_NEW_TOTAL_SUPPLY_CALCULATION_FAILED");
    // 把铸造出来的 cToken 数量加入到用户对应的账户中
    (vars.mathErr, vars.accountTokensNew) = addUInt(accountTokens[minter], vars.mintTokens);
    require(vars.mathErr == MathError.NO_ERROR, "MINT_NEW_ACCOUNT_BALANCE_CALCULATION_FAILED");
    // 更新合约的总供给量和用户对应的数量
    totalSupply = vars.totalSupplyNew;
    // 这里就是存用户的token量
    accountTokens[minter] = vars.accountTokensNew;
    // 发射铸造事件和转账事件
    emit Mint(minter, vars.actualMintAmount, vars.mintTokens);
    emit Transfer(address(this), minter, vars.mintTokens);
	return (uint(Error.NO_ERROR), vars.actualMintAmount);
}
```

**赎回redeem**

赎回存款，即用 cToken 换回标的资产，会根据最新的兑换率计算能换回多少标的资产。

```typescript
function redeemInternal(uint redeemTokens) internal nonReentrant returns (uint) {
    // 计算汇率
    uint error = accrueInterest();
    if (error != uint(Error.NO_ERROR)) {
        return fail(Error(error), FailureInfo.REDEEM_ACCRUE_INTEREST_FAILED);
    }
    // 计算最新赎回数量
    return redeemFresh(msg.sender, redeemTokens, 0);
}
function redeemFresh(
    address payable redeemer, //    用户地址
    uint redeemTokensIn,    //  赎回cToken数量
    uint redeemAmountIn     //  赎回标的资产数量
) internal returns (uint) {
    require(redeemTokensIn == 0 || redeemAmountIn == 0, "one of redeemTokensIn or redeemAmountIn must be zero");
    RedeemLocalVars memory vars;
    // 计算兑换率
    (vars.mathErr, vars.exchangeRateMantissa) = exchangeRateStoredInternal();
    if (vars.mathErr != MathError.NO_ERROR) {
        return failOpaque(Error.MATH_ERROR, FailureInfo.REDEEM_EXCHANGE_RATE_READ_FAILED, uint(vars.mathErr));
    }

    if (redeemTokensIn > 0) {
        // 计算兑换率和待赎回标的金额
        vars.redeemTokens = redeemTokensIn;
        // 计算可以得到的cToken数量， redeemAmount = 存款利率 * 全部额度
        (vars.mathErr, vars.redeemAmount) = mulScalarTruncate(Exp({mantissa: vars.exchangeRateMantissa}), redeemTokensIn);
        if (vars.mathErr != MathError.NO_ERROR) {
            return failOpaque(Error.MATH_ERROR, FailureInfo.REDEEM_EXCHANGE_TOKENS_CALCULATION_FAILED, uint(vars.mathErr));
        }
    }
    // 根据传入标的资产数量能换多少cToken
    else {
        /*
         * 获得当前汇率并计算要兑换的金额：
         *  redeemTokens = redeemAmountIn / exchangeRate
         *  redeemAmount = redeemAmountIn
         */
        // cToken的数量 = 标的资产数量 / 汇率
        (vars.mathErr, vars.redeemTokens) = divScalarByExpTruncate(redeemAmountIn, Exp({mantissa: vars.exchangeRateMantissa}));
        if (vars.mathErr != MathError.NO_ERROR) {
            return failOpaque(Error.MATH_ERROR, FailureInfo.REDEEM_EXCHANGE_AMOUNT_CALCULATION_FAILED, uint(vars.mathErr));
        }
        // 赎回的额度
        vars.redeemAmount = redeemAmountIn;
    }
    // 检查账户是否允许兑换
    uint allowed = comptroller.redeemAllowed(address(this), redeemer, vars.redeemTokens);
    if (allowed != 0) {
        return failOpaque(Error.COMPTROLLER_REJECTION, FailureInfo.REDEEM_COMPTROLLER_REJECTION, allowed);
    }

    /* 验证市场的区块数 == 当前区块数*/
    if (accrualBlockNumber != getBlockNumber()) {
        return fail(Error.MARKET_NOT_FRESH, FailureInfo.REDEEM_FRESHNESS_CHECK);
    }

    /*
     * 计算新的总供应和赎回余额，并检查下溢：
     */
    // totalSupplyNew = totalSupply - redeemTokens
    (vars.mathErr, vars.totalSupplyNew) = subUInt(totalSupply, vars.redeemTokens);
    if (vars.mathErr != MathError.NO_ERROR) {
        return failOpaque(Error.MATH_ERROR, FailureInfo.REDEEM_NEW_TOTAL_SUPPLY_CALCULATION_FAILED, uint(vars.mathErr));
    }
    // accountTokensNew = accountTokens[redeemer] - redeemTokens
    (vars.mathErr, vars.accountTokensNew) = subUInt(accountTokens[redeemer], vars.redeemTokens);
    if (vars.mathErr != MathError.NO_ERROR) {
        return failOpaque(Error.MATH_ERROR, FailureInfo.REDEEM_NEW_ACCOUNT_BALANCE_CALCULATION_FAILED, uint(vars.mathErr));
    }

    /* Fail gracefully if protocol has insufficient cash */
    // 计算价格
    if (getCashPrior() < vars.redeemAmount) {
        return fail(Error.TOKEN_INSUFFICIENT_CASH, FailureInfo.REDEEM_TRANSFER_OUT_NOT_POSSIBLE);
    }

    /////////////////////////
    // EFFECTS & INTERACTIONS
    // (No safe failures beyond this point)

    /*
    *我们为赎回者和赎回金额调用doTransferOut。
    *注意：cToken必须处理ERC-20和ETH基础之间的变化。
    *成功后，cToken的兑换金额减去现金。
    *如果出现任何问题，doTransferOut将恢复，因为我们无法确定是否发生了副作用。
    */
    doTransferOut(redeemer, vars.redeemAmount);
    /* 将先前计算的值写入存储 */
    totalSupply = vars.totalSupplyNew;
    accountTokens[redeemer] = vars.accountTokensNew;
    // 发送事件
    emit Transfer(redeemer, address(this), vars.redeemTokens);
    emit Redeem(redeemer, vars.redeemAmount, vars.redeemTokens);
    // 检查两个是否为0
    comptroller.redeemVerify(address(this), redeemer, vars.redeemAmount, vars.redeemTokens);
    return uint(Error.NO_ERROR);
}
```

