

# AAVE-V2-Summary

学习aave的时候参考大佬们的总结

https://learnblockchain.cn/article/3137#%E6%B5%AE%E5%8A%A8%E5%80%9F%E6%AC%BE%E4%BB%A3%E5%B8%81

https://learnblockchain.cn/article/3099

这些文章更偏向于经济模型的总结，直接摆出公式，适合有过aave源码基础去阅读。而对于初学者，直接看到这些公式难免头皮发麻，不如从自己擅长的地方入手，从ERC20切入分析AAVE的源码。

相比于直接摆出公式，或者直接看白皮书，个人感觉还是结合源码，带着问题来研究会比较深刻。

仓库地址：https://github.com/aave/protocol-v2.git


- [AAVE-V2-Summary](#aave-v2-summary)
  - [尝试用一句高度总结aave-v2 =\> aave-v2 本质上是一个以银行为参考的ERC20智能合约](#尝试用一句高度总结aave-v2--aave-v2-本质上是一个以银行为参考的erc20智能合约)
- [1. Aave中的两类货币 AToken DebtToken 这些代币与标准ERC20有何不同？](#1-aave中的两类货币-atoken-debttoken-这些代币与标准erc20有何不同)
  - [AToken: Aave中代表收益、利息的货币](#atoken-aave中代表收益利息的货币)
    - [重写ERC20 balanceOf, totalSupply方法。由于利息随时间的增长是一直在变化的，所以每当时间T时调用这些方法，需要计算ΔT时间内的线性利率](#重写erc20-balanceof-totalsupply方法由于利息随时间的增长是一直在变化的所以每当时间t时调用这些方法需要计算δt时间内的线性利率)
    - [重写mint burn transfer方法，使其与流动性指数有相关性，流动性指数会受到deposit withdraw等操作影响](#重写mint-burn-transfer方法使其与流动性指数有相关性流动性指数会受到deposit-withdraw等操作影响)
  - [DebtToken：代表债务的货币](#debttoken代表债务的货币)
    - [StableDebtToken 稳定债务](#stabledebttoken-稳定债务)
      - [重写的ERC20接口balanceOf toatlSupply 由于利息随时间的增长是一直在变化的，所以每当时间T时调用这些方法，计算复利](#重写的erc20接口balanceof-toatlsupply-由于利息随时间的增长是一直在变化的所以每当时间t时调用这些方法计算复利)
      - [重写ERC20的 mint(对应借贷产生的负债) burn(对应还贷是销毁负债token或者再平衡)](#重写erc20的-mint对应借贷产生的负债-burn对应还贷是销毁负债token或者再平衡)
    - [VaribleDebtToken 可变债务](#varibledebttoken-可变债务)
      - [重写的ERC20 balanceOf toatlSupply 同StableDebtToken 计算复利](#重写的erc20-balanceof-toatlsupply-同stabledebttoken-计算复利)
      - [重写mint burn 等方法, 与稳定借贷不同，浮动借贷没有平均浮动借贷指数这一概念(因为是随时可变的)，mint 和 brun的参数index都是当前浮动借款指数](#重写mint-burn-等方法-与稳定借贷不同浮动借贷没有平均浮动借贷指数这一概念因为是随时可变的mint-和-brun的参数index都是当前浮动借款指数)
- [2. LendingPool 如何用这些代币来实现despoit withdraw borrow repay ...？](#2-lendingpool-如何用这些代币来实现despoit-withdraw-borrow-repay-)
    - [despoit 存款 -\> AToken](#despoit-存款---atoken)
    - [withdraw 提现 -\> AToken](#withdraw-提现---atoken)
    - [borrow 借贷 -\> DebtToken](#borrow-借贷---debttoken)
    - [repay 还款 -\> DebtToken](#repay-还款---debttoken)
    - [计算指数、利率。关键计算公式以注释标注](#计算指数利率关键计算公式以注释标注)
      - [更新流动性、浮动贷款指数](#更新流动性浮动贷款指数)
      - [更新稳定/浮动贷款利率、当前流动率](#更新稳定浮动贷款利率当前流动率)
- [3. LendingPool中的闪电贷flashLoan](#3-lendingpool中的闪电贷flashloan)
  - [闪电贷、闪电兑换是什么？乐观转账？](#闪电贷闪电兑换是什么乐观转账)
  - [闪电贷攻击](#闪电贷攻击)
- [4. 抵押、清算、健康度相关 aave如何保证风险可控？](#4-抵押清算健康度相关-aave如何保证风险可控)
  - [LTV(抵押率) Liquidation Threashold(清算阈值) Health Factor(健康度)](#ltv抵押率-liquidation-threashold清算阈值-health-factor健康度)
  - [Cover(清算) 的触发和计算 何时触发清算? 如何清算](#cover清算-的触发和计算-何时触发清算-如何清算)
- [5. 从Aave合约学到的一些其他东西](#5-从aave合约学到的一些其他东西)
  - [预言机Oracle](#预言机oracle)
    - [什么是预言机？](#什么是预言机)
    - [预言机操控攻击](#预言机操控攻击)
  - [代理合约](#代理合约)
    - [利用代理合约实现合约的热更新，合约升级](#利用代理合约实现合约的热更新合约升级)
    - [破坏了合约的不可修改性？](#破坏了合约的不可修改性)



## 尝试用一句高度总结aave-v2 => aave-v2 本质上是一个以银行为参考的ERC20智能合约

银行：提供存、取、借、还等基本能力，甚至还提供闪电贷这种可以加杠杆的能力。通过稳定币的多抵押，少借贷的模式使其借贷的风险比较保守，相对流动性也会较差。存款收益，借贷利息等也参考银行的线性、复利计算方式。

ERC20：AToken，DebtToken。为了防止债务转移，限制债务的流动性，aave-v2对DebtToken transfer, allowance, approve等相关功能其增加了限制



# 1. Aave中的两类货币 AToken DebtToken 这些代币与标准ERC20有何不同？
## AToken: Aave中代表收益、利息的货币
  
### 重写ERC20 balanceOf, totalSupply方法。由于利息随时间的增长是一直在变化的，所以每当时间T时调用这些方法，需要计算ΔT时间内的线性利率
<details>

```solidity
  /*
    计算线性收益率，1 + currentLiquidityRate * ΔT时间
  */
  // 线性复利计算公式
  function calculateLinearInterest(uint256 rate, uint40 lastUpdateTimestamp)
    internal
    view
    returns (uint256)
  {
    //solium-disable-next-line
    uint256 timeDifference = block.timestamp.sub(uint256(lastUpdateTimestamp));

    return (rate.mul(timeDifference) / SECONDS_PER_YEAR).add(WadRayMath.ray());
  }
  // 线性复利计算公式
  function getNormalizedIncome(DataTypes.ReserveData storage reserve)
    internal
    view
    returns (uint256)
  {
    uint40 timestamp = reserve.lastUpdateTimestamp;

    //solium-disable-next-line
    if (timestamp == uint40(block.timestamp)) {
      //if the index was updated in the same block, no need to perform any calculation
      return reserve.liquidityIndex;
    }

    uint256 cumulated =
      MathUtils.calculateLinearInterest(reserve.currentLiquidityRate, timestamp).rayMul(
        reserve.liquidityIndex
      );

    return cumulated;
  }
  // 线性复利计算
  function getReserveNormalizedIncome(address asset)
    external
    view
    virtual
    override
    returns (uint256)
  {
    return _reserves[asset].getNormalizedIncome();
  }
  /*
    则T时间余额 = T-1余额 * (1 + currentLiquidityRate * ΔT时间)
  */
  function balanceOf(address user)
    public
    view
    override(IncentivizedERC20, IERC20)
    returns (uint256)
  {
    return super.balanceOf(user).rayMul(_pool.getReserveNormalizedIncome(_underlyingAsset));
  }
// 与balanceOf 计算方式相同
  function totalSupply() public view override(IncentivizedERC20, IERC20) returns (uint256) {
    uint256 currentSupplyScaled = super.totalSupply();

    if (currentSupplyScaled == 0) {
      return 0;
    }

    return currentSupplyScaled.rayMul(_pool.getReserveNormalizedIncome(_underlyingAsset));
  }
```
</details>

###  重写mint burn transfer方法，使其与流动性指数有相关性，流动性指数会受到deposit withdraw等操作影响
<details>

```solidity
    /*      
       newLiquidityIndex = cumulatedLiquidityInterest = （年化流动性利率LR(t) + 1）* 原liquidityIndex
            LI(t) = (LR(t) * (Delta(Tyear) + 1)) * LI(t-1)        LI(0) = 1*10^27 = 1ray
    */
    uint256 cumulatedLiquidityInterest =
    MathUtils.calculateLinearInterest(currentLiquidityRate, timestamp);
    newLiquidityIndex = cumulatedLiquidityInterest.rayMul(liquidityIndex);

  function mint(
    address user,
    uint256 amount,
    uint256 index
  ) external override onlyLendingPool returns (bool) {
    uint256 previousBalance = super.balanceOf(user);
    // index 即是流动性指数 下面的brun transfer同理，均使用线性利率计算方式
    uint256 amountScaled = amount.rayDiv(index);
    require(amountScaled != 0, Errors.CT_INVALID_MINT_AMOUNT);
    _mint(user, amountScaled);

    emit Transfer(address(0), user, amount);
    emit Mint(user, amount, index);

    return previousBalance == 0;
  }
  
  function burn(
    address user,
    address receiverOfUnderlying,
    uint256 amount,
    uint256 index
  ) external override onlyLendingPool {
    uint256 amountScaled = amount.rayDiv(index);
    require(amountScaled != 0, Errors.CT_INVALID_BURN_AMOUNT);
    _burn(user, amountScaled);

    IERC20(_underlyingAsset).safeTransfer(receiverOfUnderlying, amount);

    emit Transfer(user, address(0), amount);
    emit Burn(user, receiverOfUnderlying, amount, index);
  }
  
  function _transfer(
    address from,
    address to,
    uint256 amount,
    bool validate
  ) internal {
    address underlyingAsset = _underlyingAsset;
    ILendingPool pool = _pool;

    uint256 index = pool.getReserveNormalizedIncome(underlyingAsset);

    uint256 fromBalanceBefore = super.balanceOf(from).rayMul(index);
    uint256 toBalanceBefore = super.balanceOf(to).rayMul(index);

    super._transfer(from, to, amount.rayDiv(index));

    if (validate) {
      pool.finalizeTransfer(underlyingAsset, from, to, amount, fromBalanceBefore, toBalanceBefore);
    }

    emit BalanceTransfer(from, to, amount, index);
  }
```
</details>


## DebtToken：代表债务的货币
其派生出 稳定借贷利率债务货币(StableDebtToken)，浮动借贷利率债务货币(VaribleDebtToken), 债务货币被限制流动性，但其他ERC20的接口没有限制，balanceOf,totalSupply, mint, burn ...不同的借贷模式有不同的实现
<details>

```solidity
  /**
   * @dev Being non transferrable, the debt token does not implement any of the
   * standard ERC20 functions for transfer and allowance.
   **/
  function transfer(address recipient, uint256 amount) public virtual override returns (bool) {
    recipient;
    amount;
    revert('TRANSFER_NOT_SUPPORTED');
  }

  function allowance(address owner, address spender)
    public
    view
    virtual
    override
    returns (uint256)
  {
    owner;
    spender;
    revert('ALLOWANCE_NOT_SUPPORTED');
  }

  function approve(address spender, uint256 amount) public virtual override returns (bool) {
    spender;
    amount;
    revert('APPROVAL_NOT_SUPPORTED');
  }
  ...
```
</details>


### StableDebtToken 稳定债务
#### 重写的ERC20接口balanceOf toatlSupply 由于利息随时间的增长是一直在变化的，所以每当时间T时调用这些方法，计算复利

<details>

```solidity
  稳定借贷的贷款利率是以复利的方式计算的，也就是利滚利。计算公式也比较简单(1 + 年化稳定借贷利率) ^ ΔT
  (1+x)^n = 1+n*x+[n/2*(n-1)]*x^2+[n/6*(n-1)*(n-2)*x^3
  Aave这里用了泰勒展开 级数为3，可以节省gas， 详情可以参考MathUtils.sol 不再贴代码
  /**
   * @dev Calculates the current user debt balance 
   * @return The accumulated debt of the user
   **/
  function balanceOf(address account) public view virtual override returns (uint256) {
    uint256 accountBalance = super.balanceOf(account);
    uint256 stableRate = _usersStableRate[account];
    if (accountBalance == 0) {
      return 0;
    }
    uint256 cumulatedInterest =
      MathUtils.calculateCompoundedInterest(stableRate, _timestamps[account]);
    return accountBalance.rayMul(cumulatedInterest);
  }
   /**
   * @dev Returns the total supply
   **/
  function totalSupply() public view override returns (uint256) {
    return _calcTotalSupply(_avgStableRate);
  }

  /**
   * @dev Calculates the total supply
   * @param avgRate The average rate at which the total supply increases
   * @return The debt balance of the user since the last burn/mint action
   **/
  function _calcTotalSupply(uint256 avgRate) internal view virtual returns (uint256) {
    uint256 principalSupply = super.totalSupply();

    if (principalSupply == 0) {
      return 0;
    }

    uint256 cumulatedInterest =
      MathUtils.calculateCompoundedInterest(avgRate, _totalSupplyTimestamp);

    return principalSupply.rayMul(cumulatedInterest);
  }

```
</details>


#### 重写ERC20的 mint(对应借贷产生的负债) burn(对应还贷是销毁负债token或者再平衡)

<details>

``` solidity
  function mint(
    address user,
    address onBehalfOf,
    uint256 amount,
    uint256 rate
  ) external override onlyLendingPool returns (bool) {
    MintLocalVars memory vars;

    if (user != onBehalfOf) {
      _decreaseBorrowAllowance(onBehalfOf, user, amount);
    }
    // 计算复利 和 利息
    (, uint256 currentBalance, uint256 balanceIncrease) = _calculateBalanceIncrease(onBehalfOf);

    // 当前总供应量，时刻计算复利 SB不受时间影响
    // previousSupply = SD(t) = 总借款量SB * (1+复利利率/Tyear)^Delta(t) 
    vars.previousSupply = totalSupply();
    vars.currentAvgStableRate = _avgStableRate;
    vars.nextSupply = _totalSupply = vars.previousSupply.add(amount);

    vars.amountInRay = amount.wadToRay();

    vars.newStableRate = _usersStableRate[onBehalfOf]
      .rayMul(currentBalance.wadToRay())
      .add(vars.amountInRay.rayMul(rate))
      .rayDiv(currentBalance.add(amount).wadToRay());

    require(vars.newStableRate <= type(uint128).max, Errors.SDT_STABLE_DEBT_OVERFLOW);
    // 保存用户借款时利率，在还款时会用到
    _usersStableRate[onBehalfOf] = vars.newStableRate;

    //solium-disable-next-line
    _totalSupplyTimestamp = _timestamps[onBehalfOf] = uint40(block.timestamp);

    // Calculates the updated average stable rate
    /*                            ASR(t-1) * SD(t) + SR(t) * SB(borrow)
        平均稳定利率 = ASR(t) =  ————————————————————————————————————
                                    (SD(t) + SB(borrow))
    */
    vars.currentAvgStableRate = _avgStableRate = vars
      .currentAvgStableRate
      .rayMul(vars.previousSupply.wadToRay())
      .add(rate.rayMul(vars.amountInRay))
      .rayDiv(vars.nextSupply.wadToRay());

    _mint(onBehalfOf, amount.add(balanceIncrease), vars.previousSupply);

    emit Transfer(address(0), onBehalfOf, amount);

    emit Mint(
      user,
      onBehalfOf,
      amount,
      currentBalance,
      balanceIncrease,
      vars.newStableRate,
      vars.currentAvgStableRate,
      vars.nextSupply
    );

    return currentBalance == 0;
  }


   function burn(address user, uint256 amount) external override onlyLendingPool {
    // 计算复利 和 利息
    (, uint256 currentBalance, uint256 balanceIncrease) = _calculateBalanceIncrease(user);
    // 当前总供应量，时刻计算复利 SB不受时间影响
    // previousSupply = SD(t) = 总借款量SB * (1+复利利率/Tyear)^Delta(t) 
    uint256 previousSupply = totalSupply();
    uint256 newAvgStableRate = 0;
    uint256 nextSupply = 0;
    uint256 userStableRate = _usersStableRate[user];

    // Since the total supply and each single user debt accrue separately,
    // there might be accumulation errors so that the last borrower repaying
    // mght actually try to repay more than the available debt supply.
    // In this case we simply set the total supply and the avg stable rate to 0
    // 两次判断 还的比欠的多的情况
    if (previousSupply <= amount) {
      _avgStableRate = 0;
      _totalSupply = 0;
    } else {
      nextSupply = _totalSupply = previousSupply.sub(amount);
      uint256 firstTerm = _avgStableRate.rayMul(previousSupply.wadToRay());
      uint256 secondTerm = userStableRate.rayMul(amount.wadToRay());

      // For the same reason described above, when the last user is repaying it might
      // happen that user rate * user balance > avg rate * total supply. In that case,
      // we simply set the avg rate to 0
      if (secondTerm >= firstTerm) {
        newAvgStableRate = _avgStableRate = _totalSupply = 0;
      } else {

        /*
          ASR(t) =  SD(t) * ASR(t-1) - SB(repay) * SR(x)->用户以多少利率还款，在借款mint时该值被更新，代表借款时的利率，还款时直接取用该值
                    ————————————————————————————————————
                            SD(t) - SB(repay)
        */

        newAvgStableRate = _avgStableRate = firstTerm.sub(secondTerm).rayDiv(nextSupply.wadToRay());
      }
    }

    if (amount == currentBalance) {
      _usersStableRate[user] = 0;
      _timestamps[user] = 0;
    } else {
      //solium-disable-next-line
      _timestamps[user] = uint40(block.timestamp);
    }
    //solium-disable-next-line
    _totalSupplyTimestamp = uint40(block.timestamp);
    // 如果还款金额未超过利息，则将剩余利息作为新增借款，进行 mint() 这将修改 _userStableRate[user]（以还款时的利率重新借贷），也是 rebalancing 
    if (balanceIncrease > amount) {
      uint256 amountToMint = balanceIncrease.sub(amount);
      _mint(user, amountToMint, previousSupply);
      emit Mint(
        user,
        user,
        amountToMint,
        currentBalance,
        balanceIncrease,
        userStableRate,
        newAvgStableRate,
        nextSupply
      );
    } else {
      uint256 amountToBurn = amount.sub(balanceIncrease);
      _burn(user, amountToBurn, previousSupply);
      emit Burn(user, amountToBurn, currentBalance, balanceIncrease, newAvgStableRate, nextSupply);
    }

    emit Transfer(user, address(0), amount);
  }
```
</details>

### VaribleDebtToken 可变债务
#### 重写的ERC20 balanceOf toatlSupply 同StableDebtToken 计算复利
<details>

```solidity
  function balanceOf(address user) public view virtual override returns (uint256) {
    uint256 scaledBalance = super.balanceOf(user);

    if (scaledBalance == 0) {
      return 0;
    }

    return scaledBalance.rayMul(_pool.getReserveNormalizedVariableDebt(_underlyingAsset));
  }

 function totalSupply() public view virtual override returns (uint256) {
    return super.totalSupply().rayMul(_pool.getReserveNormalizedVariableDebt(_underlyingAsset));
  }

  /**
   * @dev Returns the normalized variable debt per unit of asset
   * @param asset The address of the underlying asset of the reserve
   * @return The reserve normalized variable debt
   */
  function getReserveNormalizedVariableDebt(address asset)
    external
    view
    override
    returns (uint256)
  {
    return _reserves[asset].getNormalizedDebt();
  }

  function getNormalizedDebt(DataTypes.ReserveData storage reserve)
    internal
    view
    returns (uint256)
  {
    uint40 timestamp = reserve.lastUpdateTimestamp;

    //solium-disable-next-line
    if (timestamp == uint40(block.timestamp)) {
      //if the index was updated in the same block, no need to perform any calculation
      return reserve.variableBorrowIndex;
    }
    // 复利计算债务, 同稳定借贷计算公式相同，只不过参数变成了浮动借贷利率，浮动借贷指数
    uint256 cumulated =
      MathUtils.calculateCompoundedInterest(reserve.currentVariableBorrowRate, timestamp).rayMul(
        reserve.variableBorrowIndex
      );

    return cumulated;
  }
```
</details>

#### 重写mint burn 等方法, 与稳定借贷不同，浮动借贷没有平均浮动借贷指数这一概念(因为是随时可变的)，mint 和 brun的参数index都是当前浮动借款指数
<details>

``` solidity
 function mint(
    address user,
    address onBehalfOf,
    uint256 amount,
    uint256 index
  ) external override onlyLendingPool returns (bool) {
    if (user != onBehalfOf) {
      _decreaseBorrowAllowance(onBehalfOf, user, amount);
    }

    uint256 previousBalance = super.balanceOf(onBehalfOf);
    uint256 amountScaled = amount.rayDiv(index);
    require(amountScaled != 0, Errors.CT_INVALID_MINT_AMOUNT);

    _mint(onBehalfOf, amountScaled);

    emit Transfer(address(0), onBehalfOf, amount);
    emit Mint(user, onBehalfOf, amount, index);

    return previousBalance == 0;
  }

  function burn(
    address user,
    uint256 amount,
    uint256 index
  ) external override onlyLendingPool {
    uint256 amountScaled = amount.rayDiv(index);
    require(amountScaled != 0, Errors.CT_INVALID_BURN_AMOUNT);

    _burn(user, amountScaled);

    emit Transfer(user, address(0), amount);
    emit Burn(user, amount, index);
  }
```
</details>

# 2. LendingPool 如何用这些代币来实现despoit withdraw borrow repay ...？
如果研究清楚上面的几个代币，下面的会比较容易理解


### despoit 存款 -> AToken
<details>

```solidity
  function deposit(
    address asset,
    uint256 amount,
    address onBehalfOf,
    uint16 referralCode
  ) external override whenNotPaused {
    DataTypes.ReserveData storage reserve = _reserves[asset];

    ValidationLogic.validateDeposit(reserve, amount);

    address aToken = reserve.aTokenAddress;

    // 更新指数 更新累计浮动借款指数，更新流动性指数
    reserve.updateState();
    // 更新稳定 浮动借款利率，更新当前流动率
    reserve.updateInterestRates(asset, aToken, amount, 0);

    IERC20(asset).safeTransferFrom(msg.sender, aToken, amount);

    // 第一次mint后emit个事件，可能其他地方会用这个事件做一些奖励发放
    // _mint(onBehalfOf, amount / reserve.liquidityIndex)
    bool isFirstDeposit = IAToken(aToken).mint(onBehalfOf, amount, reserve.liquidityIndex);

    if (isFirstDeposit) {
      _usersConfig[onBehalfOf].setUsingAsCollateral(reserve.id, true);
      emit ReserveUsedAsCollateralEnabled(asset, onBehalfOf);
    }

    emit Deposit(asset, msg.sender, onBehalfOf, amount, referralCode);
  }
```
</details>

### withdraw 提现 -> AToken
存款时会mint AToken， 提现时会Burn AToken。之前的过程相同，都是更新T时刻 稳定、浮动借款利率、指数，流动率等
<details>

```solidity
  function withdraw(
    address asset,
    uint256 amount,
    address to
  ) external override whenNotPaused returns (uint256) {
    DataTypes.ReserveData storage reserve = _reserves[asset];

    address aToken = reserve.aTokenAddress;

    uint256 userBalance = IAToken(aToken).balanceOf(msg.sender);

    uint256 amountToWithdraw = amount;

    if (amount == type(uint256).max) {
      amountToWithdraw = userBalance;
    }

    ValidationLogic.validateWithdraw(
      asset,
      amountToWithdraw,
      userBalance,
      _reserves,
      _usersConfig[msg.sender],
      _reservesList,
      _reservesCount,
      _addressesProvider.getPriceOracle()
    );

    reserve.updateState();

    reserve.updateInterestRates(asset, aToken, 0, amountToWithdraw);

    if (amountToWithdraw == userBalance) {
      _usersConfig[msg.sender].setUsingAsCollateral(reserve.id, false);
      emit ReserveUsedAsCollateralDisabled(asset, msg.sender);
    }

    IAToken(aToken).burn(msg.sender, to, amountToWithdraw, reserve.liquidityIndex);

    emit Withdraw(asset, msg.sender, to, amountToWithdraw);

    return amountToWithdraw;
  }
```
</details>

### borrow 借贷 -> DebtToken
根据用户选择要借贷的模式，稳定/浮动，计算实时借贷利率，调用对应派生合约的mint方法
<details>

```solidity
 struct ExecuteBorrowParams {
    address asset;
    address user;
    address onBehalfOf;
    uint256 amount;
    uint256 interestRateMode;
    address aTokenAddress;
    uint16 referralCode;
    bool releaseUnderlying;
  }

  function _executeBorrow(ExecuteBorrowParams memory vars) internal {
    DataTypes.ReserveData storage reserve = _reserves[vars.asset];
    DataTypes.UserConfigurationMap storage userConfig = _usersConfig[vars.onBehalfOf];

    address oracle = _addressesProvider.getPriceOracle();

    uint256 amountInETH =
      IPriceOracleGetter(oracle).getAssetPrice(vars.asset).mul(vars.amount).div(
        10**reserve.configuration.getDecimals()
      );

    ValidationLogic.validateBorrow(
      vars.asset,
      reserve,
      vars.onBehalfOf,
      vars.amount,
      amountInETH,
      vars.interestRateMode,
      _maxStableRateBorrowSizePercent,
      _reserves,
      userConfig,
      _reservesList,
      _reservesCount,
      oracle
    );

    reserve.updateState();

    uint256 currentStableRate = 0;

    bool isFirstBorrowing = false;
    if (DataTypes.InterestRateMode(vars.interestRateMode) == DataTypes.InterestRateMode.STABLE) {
      currentStableRate = reserve.currentStableBorrowRate;

      isFirstBorrowing = IStableDebtToken(reserve.stableDebtTokenAddress).mint(
        vars.user,
        vars.onBehalfOf,
        vars.amount,
        currentStableRate
      );
    } else {
      isFirstBorrowing = IVariableDebtToken(reserve.variableDebtTokenAddress).mint(
        vars.user,
        vars.onBehalfOf,
        vars.amount,
        reserve.variableBorrowIndex
      );
    }

    if (isFirstBorrowing) {
      userConfig.setBorrowing(reserve.id, true);
    }

    reserve.updateInterestRates(
      vars.asset,
      vars.aTokenAddress,
      0,
      vars.releaseUnderlying ? vars.amount : 0
    );

    if (vars.releaseUnderlying) {
      IAToken(vars.aTokenAddress).transferUnderlyingTo(vars.user, vars.amount);
    }

    emit Borrow(
      vars.asset,
      vars.user,
      vars.onBehalfOf,
      vars.amount,
      vars.interestRateMode,
      DataTypes.InterestRateMode(vars.interestRateMode) == DataTypes.InterestRateMode.STABLE
        ? currentStableRate
        : reserve.currentVariableBorrowRate,
      vars.referralCode
    );
  }

```
</details>

### repay 还款 -> DebtToken 
借款对应派生类的mint方法，还款对应派生合约的burn方法
<details>

```solidity
  function repay(
    address asset,
    uint256 amount,
    uint256 rateMode,
    address onBehalfOf
  ) external override whenNotPaused returns (uint256) {
    DataTypes.ReserveData storage reserve = _reserves[asset];

    (uint256 stableDebt, uint256 variableDebt) = Helpers.getUserCurrentDebt(onBehalfOf, reserve);

    DataTypes.InterestRateMode interestRateMode = DataTypes.InterestRateMode(rateMode);

    ValidationLogic.validateRepay(
      reserve,
      amount,
      interestRateMode,
      onBehalfOf,
      stableDebt,
      variableDebt
    );

    uint256 paybackAmount =
      interestRateMode == DataTypes.InterestRateMode.STABLE ? stableDebt : variableDebt;

    if (amount < paybackAmount) {
      paybackAmount = amount;
    }

    reserve.updateState();

    if (interestRateMode == DataTypes.InterestRateMode.STABLE) {
      IStableDebtToken(reserve.stableDebtTokenAddress).burn(onBehalfOf, paybackAmount);
    } else {
      IVariableDebtToken(reserve.variableDebtTokenAddress).burn(
        onBehalfOf,
        paybackAmount,
        reserve.variableBorrowIndex
      );
    }

    address aToken = reserve.aTokenAddress;
    reserve.updateInterestRates(asset, aToken, paybackAmount, 0);

    if (stableDebt.add(variableDebt).sub(paybackAmount) == 0) {
      _usersConfig[onBehalfOf].setBorrowing(reserve.id, false);
    }

    IERC20(asset).safeTransferFrom(msg.sender, aToken, paybackAmount);

    IAToken(aToken).handleRepayment(msg.sender, paybackAmount);

    emit Repay(asset, onBehalfOf, msg.sender, paybackAmount);

    return paybackAmount;
  }

```
</details>

### 计算指数、利率。关键计算公式以注释标注
可以看到存取款，借款还款都离不开更新指数，更新利率这两个操作，所以单独看下这两个函数
#### 更新流动性、浮动贷款指数
<details>

```solidity
function _updateIndexes(
    DataTypes.ReserveData storage reserve,
    uint256 scaledVariableDebt,
    uint256 liquidityIndex,
    uint256 variableBorrowIndex,
    uint40 timestamp
  ) internal returns (uint256, uint256) {
    uint256 currentLiquidityRate = reserve.currentLiquidityRate;

    uint256 newLiquidityIndex = liquidityIndex;
    uint256 newVariableBorrowIndex = variableBorrowIndex;

    //only cumulating if there is any income being produced
    /*
       newLiquidityIndex = cumulatedLiquidityInterest = 线性利息 * liquidityIndex
          LI(t) = (LR(t) * (Delta(Tyear) + 1)) * LI(t-1)        LI(0) = 1*10^27 = 1ray 
          每单位存款本金，未来将变成多少本金（含利息收入）
          NI(t) = (LR(t) * (Delta(Tyear) + 1)) * LI(t-1)        LI(0) = 1*10^27 = 1ray 


       newVariableBorrowIndex = 阶乘复利 * variableBorrowIndex
          VI(t) = cumulatedVariableBorrowInterest * VI(t-1)
                = ((1 + VI(t) / Tyear) ^ Delta(t)) * VI(t-1)
    */
    if (currentLiquidityRate > 0) {
      // LR(t) * (Delta(Tyear) + 1)
      uint256 cumulatedLiquidityInterest =
        MathUtils.calculateLinearInterest(currentLiquidityRate, timestamp);
      newLiquidityIndex = cumulatedLiquidityInterest.rayMul(liquidityIndex);
      require(newLiquidityIndex <= type(uint128).max, Errors.RL_LIQUIDITY_INDEX_OVERFLOW);

      reserve.liquidityIndex = uint128(newLiquidityIndex);

      //as the liquidity rate might come only from stable rate loans, we need to ensure
      //that there is actual variable debt before accumulating
      if (scaledVariableDebt != 0) {
        // (1+x)^n = 1+n*x+[n/2*(n-1)]*x^2+[n/6*(n-1)*(n-2)*x^3
        uint256 cumulatedVariableBorrowInterest =
          MathUtils.calculateCompoundedInterest(reserve.currentVariableBorrowRate, timestamp);
        newVariableBorrowIndex = cumulatedVariableBorrowInterest.rayMul(variableBorrowIndex);
        require(
          newVariableBorrowIndex <= type(uint128).max,
          Errors.RL_VARIABLE_BORROW_INDEX_OVERFLOW
        );
        reserve.variableBorrowIndex = uint128(newVariableBorrowIndex);
      }
    }

    //solium-disable-next-line
    reserve.lastUpdateTimestamp = uint40(block.timestamp);
    return (newLiquidityIndex, newVariableBorrowIndex);
  }
```
</details>

#### 更新稳定/浮动贷款利率、当前流动率
<details>

```solidity
  function calculateInterestRates(
    address reserve,
    uint256 availableLiquidity,
    uint256 totalStableDebt,
    uint256 totalVariableDebt,
    uint256 averageStableBorrowRate,
    uint256 reserveFactor
  )
    public
    view
    override
    returns (
      uint256,
      uint256,
      uint256
    )
  {
    CalcInterestRatesLocalVars memory vars;

    vars.totalDebt = totalStableDebt.add(totalVariableDebt);
    vars.currentVariableBorrowRate = 0;
    vars.currentStableBorrowRate = 0;
    vars.currentLiquidityRate = 0;
    // utilizationRate = 1 / (1 + availableLiquidity / totalDebt) = 1 / (1 + (Balanceof(AToken) + TokenAdd - TokenTaken) / (totalStableDebt + totalVariableDebt))
    vars.utilizationRate = vars.totalDebt == 0
      ? 0
      : vars.totalDebt.rayDiv(availableLiquidity.add(vars.totalDebt));

    vars.currentStableBorrowRate = ILendingRateOracle(addressesProvider.getLendingRateOracle())
      .getMarketBorrowRate(reserve);
/*
当前时刻的浮动/稳定借款利率
https://docs.aave.com/risk/liquidity-risk/borrow-interest-rate
USDT      U{optimal}   U{base}    Slope1     Slope2
Variable  90%          0%         4%         60%
Stable	  90%          3.5%       2%         60%

if(utilizationRate > OPTIMAL_UTILIZATION_RATE)
  #R(t) = #R(t-1) + (#slope1 * utilizationRate / OPTIMAL_UTILIZATION_RATE) 
else
  EXCESS_UTILIZATION_RATE = WadRayMath.ray().sub(optimalUtilizationRate);
  optimalUtilizationRate = utilizationRate
  excessUtilizationRateRatio = (utilizationRate - OPTIMAL_UTILIZATION_RATE) / EXCESS_UTILIZATION_RATE = (utilizationRate - OPTIMAL_UTILIZATION_RATE) / (1 - optimalUtilizationRate)
                             = (utilizationRate - OPTIMAL_UTILIZATION_RATE) / (1 - utilizationRate)
  #R(t) = #R(t-1) + #slope1 + (#slope2 * excessUtilizationRateRatio) 
*/
    if (vars.utilizationRate > OPTIMAL_UTILIZATION_RATE) {
      uint256 excessUtilizationRateRatio =
        vars.utilizationRate.sub(OPTIMAL_UTILIZATION_RATE).rayDiv(EXCESS_UTILIZATION_RATE);

      vars.currentStableBorrowRate = vars.currentStableBorrowRate.add(_stableRateSlope1).add(
        _stableRateSlope2.rayMul(excessUtilizationRateRatio)
      );

      vars.currentVariableBorrowRate = _baseVariableBorrowRate.add(_variableRateSlope1).add(
        _variableRateSlope2.rayMul(excessUtilizationRateRatio)
      );
    } else {
      vars.currentStableBorrowRate = vars.currentStableBorrowRate.add(
        _stableRateSlope1.rayMul(vars.utilizationRate.rayDiv(OPTIMAL_UTILIZATION_RATE))
      );
      vars.currentVariableBorrowRate = _baseVariableBorrowRate.add(
        vars.utilizationRate.rayMul(_variableRateSlope1).rayDiv(OPTIMAL_UTILIZATION_RATE)
      );
    }
    // LRt 当前流动性率：等于总的借贷利率乘以此时的资金利用率, 注意：这里的利率是一个年化利率
    vars.currentLiquidityRate = _getOverallBorrowRate(
      totalStableDebt,
      totalVariableDebt,
      vars
        .currentVariableBorrowRate,
      averageStableBorrowRate
    )
      .rayMul(vars.utilizationRate)
      .percentMul(PercentageMath.PERCENTAGE_FACTOR.sub(reserveFactor));

    return (
      vars.currentLiquidityRate,
      vars.currentStableBorrowRate,
      vars.currentVariableBorrowRate
    );
  }
```
</details>

# 3. LendingPool中的闪电贷flashLoan 
## 闪电贷、闪电兑换是什么？乐观转账？
https://www.aqniu.com/vendor/87395.html 这篇文章对aave的闪电贷说的挺详细的，和源码也能对的上，就不再赘述了，这里只标记下关键操作
<details>

```solidity
 struct FlashLoanLocalVars {
    IFlashLoanReceiver receiver;
    address oracle;
    uint256 i;
    address currentAsset;
    address currentATokenAddress;
    uint256 currentAmount;
    uint256 currentPremium;
    uint256 currentAmountPlusPremium;
    address debtToken;
  }

  /**
   * @dev Allows smartcontracts to access the liquidity of the pool within one transaction,
   * as long as the amount taken plus a fee is returned.
   * IMPORTANT There are security concerns for developers of flashloan receiver contracts that must be kept into consideration.
   * For further details please visit https://developers.aave.com
   * @param receiverAddress The address of the contract receiving the funds, implementing the IFlashLoanReceiver interface
   * @param assets The addresses of the assets being flash-borrowed
   * @param amounts The amounts amounts being flash-borrowed
   * @param modes Types of the debt to open if the flash loan is not returned:
   *   0 -> Don't open any debt, just revert if funds can't be transferred from the receiver
   *   1 -> Open debt at stable rate for the value of the amount flash-borrowed to the `onBehalfOf` address
   *   2 -> Open debt at variable rate for the value of the amount flash-borrowed to the `onBehalfOf` address
   * @param onBehalfOf The address  that will receive the debt in the case of using on `modes` 1 or 2
   * @param params Variadic packed params to pass to the receiver as extra information
   * @param referralCode Code used to register the integrator originating the operation, for potential rewards.
   *   0 if the action is executed directly by the user, without any middle-man
   **/
  function flashLoan(
    address receiverAddress,
    address[] calldata assets,
    uint256[] calldata amounts,
    uint256[] calldata modes,
    address onBehalfOf,
    bytes calldata params,
    uint16 referralCode
  ) external override whenNotPaused {
    FlashLoanLocalVars memory vars;

    ValidationLogic.validateFlashloan(assets, amounts);

    address[] memory aTokenAddresses = new address[](assets.length);
    uint256[] memory premiums = new uint256[](assets.length);

    vars.receiver = IFlashLoanReceiver(receiverAddress);

    for (vars.i = 0; vars.i < assets.length; vars.i++) {
      aTokenAddresses[vars.i] = _reserves[assets[vars.i]].aTokenAddress;
      // 手续费
      premiums[vars.i] = amounts[vars.i].mul(_flashLoanPremiumTotal).div(10000);

      IAToken(aTokenAddresses[vars.i]).transferUnderlyingTo(receiverAddress, amounts[vars.i]);
    }

    require(
      // 要求获取到approve权限，失败回滚
      vars.receiver.executeOperation(assets, amounts, premiums, msg.sender, params),
      Errors.LP_INVALID_FLASH_LOAN_EXECUTOR_RETURN
    );

    for (vars.i = 0; vars.i < assets.length; vars.i++) {
      vars.currentAsset = assets[vars.i];
      vars.currentAmount = amounts[vars.i];
      vars.currentPremium = premiums[vars.i];
      vars.currentATokenAddress = aTokenAddresses[vars.i];
      vars.currentAmountPlusPremium = vars.currentAmount.add(vars.currentPremium);

      if (DataTypes.InterestRateMode(modes[vars.i]) == DataTypes.InterestRateMode.NONE) {
        _reserves[vars.currentAsset].updateState();
        _reserves[vars.currentAsset].cumulateToLiquidityIndex(
          IERC20(vars.currentATokenAddress).totalSupply(),
          vars.currentPremium
        );
        _reserves[vars.currentAsset].updateInterestRates(
          vars.currentAsset,
          vars.currentATokenAddress,
          vars.currentAmountPlusPremium,
          0
        );
        // 执行还款，失败回滚
        IERC20(vars.currentAsset).safeTransferFrom(
          receiverAddress,
          vars.currentATokenAddress,
          vars.currentAmountPlusPremium
        );
      } else {
        // If the user chose to not return the funds, the system checks if there is enough collateral and
        // eventually opens a debt position
        // 闪电贷转普通借贷
        _executeBorrow(
          ExecuteBorrowParams(
            vars.currentAsset,
            msg.sender,
            onBehalfOf,
            vars.currentAmount,
            modes[vars.i],
            vars.currentATokenAddress,
            referralCode,
            false
          )
        );
      }
      emit FlashLoan(
        receiverAddress,
        msg.sender,
        vars.currentAsset,
        vars.currentAmount,
        vars.currentPremium,
        referralCode
      );
    }
  }
```
</details>

## 闪电贷攻击
https://www.geekmeta.com/flash/4171407.html
写合约要更注重逻辑合法性，边界判断。

# 4. 抵押、清算、健康度相关 aave如何保证风险可控？
## LTV(抵押率) Liquidation Threashold(清算阈值) Health Factor(健康度)

<details>

```solidity
  struct CalculateUserAccountDataVars {
    uint256 reserveUnitPrice;
    uint256 tokenUnit;
    uint256 compoundedLiquidityBalance;
    uint256 compoundedBorrowBalance;
    uint256 decimals;
    uint256 ltv;
    uint256 liquidationThreshold;
    uint256 i;
    uint256 healthFactor;
    uint256 totalCollateralInETH;
    uint256 totalDebtInETH;
    uint256 avgLtv;
    uint256 avgLiquidationThreshold;
    uint256 reservesLength;
    bool healthFactorBelowThreshold;
    address currentReserveAddress;
    bool usageAsCollateralEnabled;
    bool userUsesReserveAsCollateral;
  }

  /**
   * @dev Calculates the user data across the reserves.
   * this includes the total liquidity/collateral/borrow balances in ETH,
   * the average Loan To Value, the average Liquidation Ratio, and the Health factor.
   * @param user The address of the user
   * @param reservesData Data of all the reserves
   * @param userConfig The configuration of the user
   * @param reserves The list of the available reserves
   * @param oracle The price oracle address
   * @return The total collateral and total debt of the user in ETH, the avg ltv, liquidation threshold and the HF
   **/
  function calculateUserAccountData(
    address user,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    DataTypes.UserConfigurationMap memory userConfig,
    mapping(uint256 => address) storage reserves,
    uint256 reservesCount,
    address oracle
  )
    internal
    view
    returns (
      uint256,
      uint256,
      uint256,
      uint256,
      uint256
    )
  {
    CalculateUserAccountDataVars memory vars;

    if (userConfig.isEmpty()) {
      return (0, 0, 0, 0, uint256(-1));
    }
    for (vars.i = 0; vars.i < reservesCount; vars.i++) {
      if (!userConfig.isUsingAsCollateralOrBorrowing(vars.i)) {
        continue;
      }

      vars.currentReserveAddress = reserves[vars.i];
      DataTypes.ReserveData storage currentReserve = reservesData[vars.currentReserveAddress];

      (vars.ltv, vars.liquidationThreshold, , vars.decimals, ) = currentReserve
        .configuration
        .getParams();

      vars.tokenUnit = 10**vars.decimals;
      vars.reserveUnitPrice = IPriceOracleGetter(oracle).getAssetPrice(vars.currentReserveAddress);

      if (vars.liquidationThreshold != 0 && userConfig.isUsingAsCollateral(vars.i)) {
        vars.compoundedLiquidityBalance = IERC20(currentReserve.aTokenAddress).balanceOf(user);
        // 清算奖励，抵押物的拍卖折扣
        uint256 liquidityBalanceETH =
          vars.reserveUnitPrice.mul(vars.compoundedLiquidityBalance).div(vars.tokenUnit);
        // 加到总抵押物中
        vars.totalCollateralInETH = vars.totalCollateralInETH.add(liquidityBalanceETH);
        // 抵押货物增加，后面再计算平均抵押率
        vars.avgLtv = vars.avgLtv.add(liquidityBalanceETH.mul(vars.ltv));
        // 清算阈值增加，后面再计算平均清算阈值
        vars.avgLiquidationThreshold = vars.avgLiquidationThreshold.add(
          liquidityBalanceETH.mul(vars.liquidationThreshold)
        );
      }

      if (userConfig.isBorrowing(vars.i)) {
        // 浮动债务 + 稳定债务 = 当前总债务
        vars.compoundedBorrowBalance = IERC20(currentReserve.stableDebtTokenAddress).balanceOf(
          user
        );
        vars.compoundedBorrowBalance = vars.compoundedBorrowBalance.add(
          IERC20(currentReserve.variableDebtTokenAddress).balanceOf(user)
        );
      
        // 总债务占用的抵押物的资产数 += 单价 * 总债务 /  10**vars.decimals（万分比）
        vars.totalDebtInETH = vars.totalDebtInETH.add(
          vars.reserveUnitPrice.mul(vars.compoundedBorrowBalance).div(vars.tokenUnit)
        );
      }
    }
    // 平均抵押率 平均清算阈值
    vars.avgLtv = vars.totalCollateralInETH > 0 ? vars.avgLtv.div(vars.totalCollateralInETH) : 0;
    vars.avgLiquidationThreshold = vars.totalCollateralInETH > 0
      ? vars.avgLiquidationThreshold.div(vars.totalCollateralInETH)
      : 0;
    // 健康度计算
    // 总抵押ETH * 总清算阈值 / 总债务ETH * 100%
    vars.healthFactor = calculateHealthFactorFromBalances(
      vars.totalCollateralInETH,
      vars.totalDebtInETH,
      vars.avgLiquidationThreshold
    );
    return (
      vars.totalCollateralInETH,
      vars.totalDebtInETH,
      vars.avgLtv,
      vars.avgLiquidationThreshold,
      vars.healthFactor
    );
  }
```

</details>

## Cover(清算) 的触发和计算 何时触发清算? 如何清算
健康度 < 1 时触发清算逻辑，根据用户欠债数量计算需要清算的额度进行清算
<details>

```solidity
 /**
   * @dev Calculates how much of a specific collateral can be liquidated, given
   * a certain amount of debt asset.
   * - This function needs to be called after all the checks to validate the liquidation have been performed,
   *   otherwise it might fail.
   * @param collateralReserve The data of the collateral reserve
   * @param debtReserve The data of the debt reserve
   * @param collateralAsset The address of the underlying asset used as collateral, to receive as result of the liquidation
   * @param debtAsset The address of the underlying borrowed asset to be repaid with the liquidation
   * @param debtToCover The debt amount of borrowed `asset` the liquidator wants to cover
   * @param userCollateralBalance The collateral balance for the specific `collateralAsset` of the user being liquidated
   * @return collateralAmount: The maximum amount that is possible to liquidate given all the liquidation constraints
   *                           (user balance, close factor)
   *         debtAmountNeeded: The amount to repay with the liquidation
   **/
  function _calculateAvailableCollateralToLiquidate(
    DataTypes.ReserveData storage collateralReserve,
    DataTypes.ReserveData storage debtReserve,
    address collateralAsset,
    address debtAsset,
    uint256 debtToCover,
    uint256 userCollateralBalance
  ) internal view returns (uint256, uint256) {
    uint256 collateralAmount = 0;
    uint256 debtAmountNeeded = 0;
    IPriceOracleGetter oracle = IPriceOracleGetter(_addressesProvider.getPriceOracle());

    AvailableCollateralToLiquidateLocalVars memory vars;

    vars.collateralPrice = oracle.getAssetPrice(collateralAsset);
    vars.debtAssetPrice = oracle.getAssetPrice(debtAsset);

    (, , vars.liquidationBonus, vars.collateralDecimals, ) = collateralReserve
      .configuration
      .getParams();
    vars.debtAssetDecimals = debtReserve.configuration.getDecimals();

    // This is the maximum possible amount of the selected collateral that can be liquidated, given the
    // max amount of liquidatable debt
    vars.maxAmountCollateralToLiquidate = vars
      .debtAssetPrice
      .mul(debtToCover)
      .mul(10**vars.collateralDecimals)
      .percentMul(vars.liquidationBonus)
      .div(vars.collateralPrice.mul(10**vars.debtAssetDecimals));

    if (vars.maxAmountCollateralToLiquidate > userCollateralBalance) {
      collateralAmount = userCollateralBalance;
      debtAmountNeeded = vars
        .collateralPrice
        .mul(collateralAmount)
        .mul(10**vars.debtAssetDecimals)
        .div(vars.debtAssetPrice.mul(10**vars.collateralDecimals))
        .percentDiv(vars.liquidationBonus);
    } else {
      collateralAmount = vars.maxAmountCollateralToLiquidate;
      debtAmountNeeded = debtToCover;
    }
    return (collateralAmount, debtAmountNeeded);
  }


 /**
   * @dev Function to liquidate a position if its Health Factor drops below 1
   * - The caller (liquidator) covers `debtToCover` amount of debt of the user getting liquidated, and receives
   *   a proportionally amount of the `collateralAsset` plus a bonus to cover market risk
   * @param collateralAsset The address of the underlying asset used as collateral, to receive as result of the liquidation
   * @param debtAsset The address of the underlying borrowed asset to be repaid with the liquidation
   * @param user The address of the borrower getting liquidated
   * @param debtToCover The debt amount of borrowed `asset` the liquidator wants to cover
   * @param receiveAToken `true` if the liquidators wants to receive the collateral aTokens, `false` if he wants
   * to receive the underlying collateral asset directly
   **/
  function liquidationCall(
    address collateralAsset,
    address debtAsset,
    address user,
    uint256 debtToCover,
    bool receiveAToken
  ) external override returns (uint256, string memory) {
    DataTypes.ReserveData storage collateralReserve = _reserves[collateralAsset];
    DataTypes.ReserveData storage debtReserve = _reserves[debtAsset];
    DataTypes.UserConfigurationMap storage userConfig = _usersConfig[user];

    LiquidationCallLocalVars memory vars;

    (, , , , vars.healthFactor) = GenericLogic.calculateUserAccountData(
      user,
      _reserves,
      userConfig,
      _reservesList,
      _reservesCount,
      _addressesProvider.getPriceOracle()
    );

    (vars.userStableDebt, vars.userVariableDebt) = Helpers.getUserCurrentDebt(user, debtReserve);

    (vars.errorCode, vars.errorMsg) = ValidationLogic.validateLiquidationCall(
      collateralReserve,
      debtReserve,
      userConfig,
      vars.healthFactor,
      vars.userStableDebt,
      vars.userVariableDebt
    );

    if (Errors.CollateralManagerErrors(vars.errorCode) != Errors.CollateralManagerErrors.NO_ERROR) {
      return (vars.errorCode, vars.errorMsg);
    }

    vars.collateralAtoken = IAToken(collateralReserve.aTokenAddress);

    vars.userCollateralBalance = vars.collateralAtoken.balanceOf(user);

    vars.maxLiquidatableDebt = vars.userStableDebt.add(vars.userVariableDebt).percentMul(
      LIQUIDATION_CLOSE_FACTOR_PERCENT
    );
    // 计算需要清算的债务数量 
    vars.actualDebtToLiquidate = debtToCover > vars.maxLiquidatableDebt
      ? vars.maxLiquidatableDebt
      : debtToCover;
    (
      vars.maxCollateralToLiquidate,
      vars.debtAmountNeeded
    ) = _calculateAvailableCollateralToLiquidate(
      collateralReserve,
      debtReserve,
      collateralAsset,
      debtAsset,
      vars.actualDebtToLiquidate,
      vars.userCollateralBalance
    );

    // If debtAmountNeeded < actualDebtToLiquidate, there isn't enough
    // collateral to cover the actual amount that is being liquidated, hence we liquidate
    // a smaller amount

    if (vars.debtAmountNeeded < vars.actualDebtToLiquidate) {
      vars.actualDebtToLiquidate = vars.debtAmountNeeded;
    }
    // 执行清算的逻辑，主要是一些债务利率、流动性更新，货币transfer 
    ...
  }
```

</details>

# 5. 从Aave合约学到的一些其他东西
## 预言机Oracle
### 什么是预言机？ 
https://mirror.xyz/iamdk.eth/w5-I6EmlQuECxkVnyHpqIrysALjf4LucpDTU23HwdS8

### 预言机操控攻击

## 代理合约
### 利用代理合约实现合约的热更新，合约升级

https://learnblockchain.cn/article/1102

### 破坏了合约的不可修改性？
持否定看法吧
