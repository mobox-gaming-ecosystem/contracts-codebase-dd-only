#CertiK Audit Reply

###KTM-01 | Missing zero address validation
* 相关代码:  
    ```
    function setFarm(address farm_) external {
        if (moboxFarm == address(0)) {
            require(msg.sender == owner(), "not owner");
        } else {
            require(msg.sender == moboxFarm, "not farm");
        }
        moboxFarm = farm_;
    }
    ```
* 处理方式: 不做修改
* 处理原因: 修改moboxFarm的msg.sender必须是owner或者mboxfarm自己, 这里允许将其设置成address(0), 目的是为了停止mint

---

###MFM-01 | No return values
* 相关代码:  
    ```
    function getUserInfo(uint256 pIndex_, address user_) 
        external 
        view 
        returns(
            address wantToken,
            uint256 wantShares,
            uint256 wantAmount,     
            uint256 workingBalance,
            uint256 gracePeriod
        )
    {
        if (pIndex_ > 0) { 
           ... code for return value ...
        }
    }
    ```
* 处理方式: 不做修改  
* 处理原因: 如果进入pIndex_ == 0 判定分支, 编译器自动将返回值全部设置成0, 符合返回期望值, multiple returns please see example: https://docs.soliditylang.org/en/v0.6.6/contracts.html#return-variables

###MFM-02 | Contract sets array length with a user controlled value
* 相关代码:  
    ```
    function addPool(address wantToken_, uint256 allocPoint_, address strategy_) external onlyRewardMgr {
        ......

        uint256 poolIndex = poolInfoArray.length;
        poolInfoArray.push(PoolInfo({
            wantToken: wantToken_,
            allocPoint: SafeMathExt.safe32(allocPoint_),
            lastRewardTime: uint64(block.timestamp),
            rewardPerShare: 0,
            workingSupply: 0,
            strategy: strategy_
        }));

        ......
        
        emit AllocPointChange(poolIndex, allocPoint_, totalAllocPoint);
    }
    ```
* 处理方式: 不做修改
* 处理原因: 此处只是取poolInfoArray.length作为emit AllocPointChange的参数, 并未设置Array.length

###MFM-03 | Missing Return Value Handling
* 相关代码:  
    ```
    function setMoMoMinter(address addr_) external onlyOwner {
        require(keyToken != address(0) && addr_ != address(0), "invalid param");
        if (momoMinter != address(0)) {
            IERC20(keyToken).approve(momoMinter, 0);
        }
        ......
    }
    ```
* 处理方式:  
  修改为  
    ```
    function setMoMoMinter(address addr_) external onlyOwner {
        require(keyToken != address(0) && addr_ != address(0), "invalid param");
        if (momoMinter != address(0)) {
            require(IERC20(keyToken).approve(momoMinter, 0), "approve fail");
        }
        ......
    }
    ```

###MFM-04 | Dangerous usage of block.timestamp
* 处理方式: 不做修改
* 处理原因: 挖矿产出基于timestamp, 并且BSC节点均为官方超级节点

###MFM-05 | Missing Checks for Reentrancy
* 处理方式: 已增加nonReentrant
* 处理原因: The internal function _depositFor 无需添加nonReentrant, 因为所有调用_depositFor的external function已使用nonReentrant

---

###MSP-01 | Integer Overflow
* 处理方式: 使用SafeMath.add来处理
* 关联修改: MoboxStrategyV.sol/MoboxStrategyVBNB.sol将做相同的修改

###MSP-02 | Missing Checks for Reentrancy
* 相关代码:  
    ```
    function harvest() whenNotPaused external {
        if (!recoverPublic) {
            require(_msgSender() == strategist, "not strategist");
        }
        ......
    }
    ```
* 处理方式:  
  修改为  
    ```
    function harvest() whenNotPaused nonReentrant external {
        if (!recoverPublic) {
            require(_msgSender() == strategist, "not strategist");
        }
        ......
    }
    ```
* 修改原因:  
  目前调用者未strategist, 但放开未recoverPublic的时候, 确实有reentrancy attack的风险
* 关联修改: MoboxStrategyV.sol/MoboxStrategyVBNB.sol将做相同的修改

###MSP-03 | Missing zero address validation
* 相关代码:  
    ```
    function setStrategist(address strategist_) external onlyStrategist {
        strategist = strategist_;
    }

    function setDevAddress(address newDev_) external onlyStrategist {
        devAddress = newDev_;
    }
    ```
* 处理方式:  
    修改为  
    ```
    function setStrategist(address strategist_) external onlyStrategist {
        require(strategist_ != address(0), "addr 0");
        strategist = strategist_;
    }

    function setDevAddress(address newDev_) external onlyStrategist {
        require(newDev_ != address(0), "addr 0");
        devAddress = newDev_;
    }
    ```
* 关联修改: MoboxStrategyV.sol/MoboxStrategyVBNB.sol将做相同的修改

###MSP-04 | Missing slippage protection
* 处理方式: 不做修改
* 处理原因: strategy的处理方式是以实时价格卖掉cake

###MSP-05 | Wrong withdraw amount
* 相关代码:
    ```
    function withdraw(address user_, uint256 amount_, uint256 feeRate_) 
        external 
        onlyOwner 
        nonReentrant 
        returns(uint256) 
    {
        require(amount_ > 0 && feeRate_ <= 50, "invalid param");
        uint256 lpBalance = IERC20(wantToken).balanceOf(address(this));
        if (lpBalance < amount_) {
            IPancakeMasterChef(pancakeFarmer).withdraw(pancakePid, amount_.sub(lpBalance));
            lpBalance = IERC20(wantToken).balanceOf(address(this));
        }

        uint256 wantAmount = lpBalance;
        if (wantTotal < wantAmount) {
            wantAmount = wantTotal;
        }
        ......
    }
    ```
* 处理方式
    修改为  
    ```
    function withdraw(address user_, uint256 amount_, uint256 feeRate_) 
        external 
        onlyOwner 
        nonReentrant 
        returns(uint256) 
    {
        require(amount_ > 0 && feeRate_ <= 50, "invalid param");
        uint256 wantAmount = amount_ > wantTotal ? wantTotal : amount_;
        uint256 lpBalance = IERC20(wantToken).balanceOf(address(this));
        if (lpBalance < wantAmount) {
            IPancakeMasterChef(pancakeFarmer).withdraw(pancakePid, wantAmount.sub(lpBalance));
            lpBalance = IERC20(wantToken).balanceOf(address(this));
        }

        if (wantAmount > lpBalance) {
            wantAmount = lpBalance;
        }
        ......
    }
    ```

###MSP-06 | No return values
* 相关代码:  
    ```
    function _makePath(address _tokenA, address _tokenB) internal pure returns(address[] memory path) {
        if (_tokenA == wbnb) {
            path = new address[](2);
            path[0] = wbnb;
            path[1] = _tokenB;
        } else if(_tokenB == wbnb) {
            path = new address[](2);
            path[0] = _tokenA;
            path[1] = wbnb;
        } else {
            path = new address[](3);
            path[0] = _tokenA;
            path[1] = wbnb;
            path[2] = _tokenB;
        }
    }
    ```
* 处理方式: 不做修改
* 处理原因: 此处已对返回值path做赋值处理, multiple returns please see example: https://docs.soliditylang.org/en/v0.6.6/contracts.html#return-variables

###MSP-07 | Unimpletation constructor function
* 处理方式: 已删除constructor function

---

###MSV-01 | Integer Overflow
* 处理方式: 使用SafeMath.add来处理
* 关联修改: MoboxStrategyP.sol/MoboxStrategyVBNB.sol将做相同的修改

###MSV-02 | Missing slippage protection
* 处理方式: 不做修改
* 处理原因: strategy的处理方式是以实时价格卖掉cake

###MSV-03 | Compile error
* 处理方式: 已处理

###MSV-04 | Missing Checks for Reentrancy
* 相关代码:  
    ```
    function deleverageOnce() external {
        ......
    }

    function deleverageUntilNotOverLevered() external {
        ......
    }

    function farm() external {
        ......
    }

    function harvest() whenNotPaused external {
        ......
    }
    ```
* 处理方式:  
  修改为  
    ```
    function deleverageOnce() external nonReentrant {
        ......
    }

    function deleverageUntilNotOverLevered() external nonReentrant {
        ......
    }

    function farm() external nonReentrant {
        ......
    }

    function harvest() whenNotPaused external nonReentrant {
        ......
    }
    ```

###MSV-05 | Missing access control
* 相关代码:  
    ```
    function init(
        address moboxFarm_,
        address strategist_,
        address wantToken_,
        address vToken_,
        address buyBackPool_,
        address devAddress_,
        uint256 buyBackRate_,
        uint256 devFeeRate_,
        uint256 margin_
    ) external {
        require(wantToken == address(0) && moboxFarm == address(0), "may only be init once");
        ......
    }
    ```
* 处理方式: 增加权限onlyOwner  
* 处理原因: init只能被调用一次, 没有onlyOwner实际上没有问题, 但是加上会更好


###MSV-06 | No return values
* 相关代码
    ```
    function getTotal() public view returns(uint256 wantTotal_, uint256 shareTotal_) {
        wantTotal_ = wantTotal();
        shareTotal_ = shareTotal;
    }

    function _makePath(address _tokenA, address _tokenB) internal pure returns(address[] memory path) {
        if (_tokenA == wbnb) {
            path = new address[](2);
            path[0] = wbnb;
            path[1] = _tokenB;
        } else if(_tokenB == wbnb) {
            path = new address[](2);
            path[0] = _tokenA;
            path[1] = wbnb;
        } else {
            path = new address[](3);
            path[0] = _tokenA;
            path[1] = wbnb;
            path[2] = _tokenB;
        }
    }
    ```
* 处理方式: 不做修改  
* 处理原因: 已在所有判断分支返回返回值, multiple returns please see example: https://docs.soliditylang.org/en/v0.6.6/contracts.html#return-variables  

###MSV-07 | Ignored return values
* 相关代码:  
    ```
    function _supply(uint256 amount_) internal {
        IVToken(vToken).mint(amount_);
    }

    // Withdraw funds
    function _removeSupply(uint256 amount_) internal {
        IVToken(vToken).redeemUnderlying(amount_);
    }


    function _borrow(uint256 amount_) internal {
        IVToken(vToken).borrow(amount_);
    }


    function _repayBorrow(uint256 amount_) internal {
        IVToken(vToken).repayBorrow(amount_); 
    }
    ```
* 处理方式: 不做修改  
* 处理原因: VToken返回值为该操作的执行结果, uint 0=success, otherwise a failure, 我们在每次调用这些函数后, 都会重新获取相关token的balanceOf(address(this)), 并根据实际的token的余额进行后续的逻辑处理, 所以我们并不用关系VToken的执行结果  

###MSV-08 | Leverage risk
* 处理方式: 不做修改  
* 处理原因: 不修改代码, 将borrowDepth参数设置为0, 不使用leverage

---

###MTM-01 | Ignored return value
* 处理方式: 见MFM-03

---
###PMO-01 | Compiler Error
相关代码: 
    ```
    constructor () internal {
        _paused = false;
    }
    ```
处理方式: 不做修改
处理原因: We can compile it success by 0.6.12+commit.27d51765

