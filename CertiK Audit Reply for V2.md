#CertiK Audit Reply

###MOBOX-01 | Privileged Ownerships
* 处理方式: 无需修改
* 处理原因: Strategist目前是合约本身就是TimeLockController.sol, 我们会在后续版本持续完善timelock

---

###CMO-01 | Redundant Expression
* 相关代码:  
* 处理方式: 无需修改
* 处理原因: 如果删除this, 会产生编译器警告: Function state mutability can be restricted to pure function _msgData() internal view virtual returns (bytes memory) { ^ (Relevant source part starts here and spans across multiple lines). 

---

###MSM-01 | Divide Before Multiply
* 相关代码:  
    ```
    if (amount > 0) {
        if (amount <= 10) {
            percent = amount / 5 * 200;                     // 0% ~ 4%
        } else if (amount <= 50) {
            percent = (amount - 10) / 5 * 100 + 400;        // 4% ~ 12%
        } else if (amount <= 100) {
            percent = (amount - 50) / 10 * 100 + 1200;      // 12% ~ 17%
        } else if (amount <= 200) {
            percent = (amount - 100) / 25 * 100 + 1700;     // 17% ~ 21%
        } else if (amount <= 500) {
            percent = (amount - 200) / 50 * 100 + 2100;     // 21% ~ 27%
        } else {
            percent = 2700;                                 // 27%
        }
    }
    ```
* 处理方式: 无需修改
* 处理原因: 这里必须先除法再乘法, 这里的逻辑是, 当用户ERC721-NFT(V4)数量在10个以内时, 每5个增加2%的hashrate, 如果是6个, 仍然是增加2%, 下面的逻辑类似

###MSM-02 | userHashratePercent is Undefined for One amountV Holders

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
* 处理方式: 无需修改
* 处理原因: 所有声明的变量, 编译器都会将其设置成默认值0, 请查看文档https://docs.soliditylang.org/en/v0.6.12/control-structures.html#scoping-and-declarations, 所以, 当ammount == 1时, 函数返回值为0, 显式的赋值0会浪费gas

###MSM-03 | Incorrect ERC20 Interface
* 处理方式: 无需修改
* 处理原因: MoMoToken是标准ERC721-Token, 请查看文档https://eips.ethereum.org/EIPS/eip-721, safeTranferFrom无返回值

###MFM-04 | Dangerous Strict Equalities
* 处理方式: 无需修改
* 处理原因: rewardPerTokenStored默认初始化是0, 当totalHashrate == 0(没有质押任何NFT), 或者 block.timestamp  <= rewardStartTime(未开始挖矿)时, 将返回当前的rewardPerTokenStored, 用户的token是通过两次rewardPerTokenStored的差值进行对比的, 如果返回同一个说明token为0

###MSM-05 | Inaccurate Parameter of HashrateChange Event
* 处理方式: 无需修改
* 相关代码:
```
    function levelUp(uint256 tokenId_, uint256[] memory protosV1V2V3_, uint256[] memory amountsV1V2V3_, uint256[] memory tokensV4V5_) external nonReentrant 
    {        
        ......
        // doTransfer = false, realRemove = true
        uint256 hashrateFixedSub = _removeNft(msg.sender, protosV1V2V3_, amountsV1V2V3_, tokensV4V5_, 0x02, 0);
        ...... 
        emit LevelUpBurnStake(msg.sender, protosV1V2V3_, amountsV1V2V3_, tokensV4V5_);
        emit HashrateChange(msg.sender, 4, oldHashRate, newHashRate);
    }
```
* 处理原因: 这里传的参数changeType_为0, 是要求internal函数_removeNft无需emit HashrateChange, 因为levelUp在函数最后会emit HashrateChange, 如果_removeNft也emit了这个事件, 会导致事件重复, 而且数值不准确

###MSM-06 | Inaccurate Parameter of HashrateChange Event
* 处理方式: 无需修改
* 相关代码
```
    function mintAndStake(uint256 amount_) external nonReentrant {
        ......
        (ids, vals, tokenIds) = momoMinter.mintByStaker(msg.sender, amount_);
        _stakeNft(msg.sender, ids, vals, tokenIds, false, 2);
        ......
    }
```
* 处理原因: 在momoMinter.mintByStaker的逻辑中, NFT的minter后的owner直接设置为MomoStaker, 这样可以减少一次NFT从用户tansfer到MomoStaker的过程, 减少gas消耗, 所以, 这里的_stakeNft()不需要doing real tansfer


###MSM-07 | Reentrancy vulnerabilities
* 处理方式: 为MoMoStaker.removeNft/stakeNft/mintAndStake/setMomoName/addMomoStory等外部调用的函数增加nonReentrant

###MSM-08 | Uninitialized Local Variable
* 处理方式: 无需修改
* 处理原因: 所有声明的变量, 编译器都会将其设置成默认值0, 请查看文档https://docs.soliditylang.org/en/v0.6.12/control-structures.html#scoping-and-declarations, 所以, 显示的将hashrateFixed设置为0会浪费gas


###MSM-09 | Unused Return
* 处理方式: 已修改, 处理了返回值

###MSM-10 | 3rd Party Dependencies
* 处理方式: 无需修改
* 处理原因: MomoToken是我们开发者发布的ERC-721 token, 并非 3rd contract

###MSM-11 | Missing Zero Address Validation
* 处理方式: 已添加zero address check

###MSM-12 | Calls Inside A Loop
* 处理方式: 无需修改
* 处理原因: mintAndStake: _stakeNft, doTransfer_ = false: 此处在momoMinter.mintByStaker逻辑中是需要burn足够数量的KeyToken来mint出相等数量的ERC1155或ERC721, 并且在 mintByStaker中限制了一次最多burn 10 KeyToken, 所以此处不会出现denial-of-service attack
其他地方调用_stakeNft会分别触发_momomToken.safeBatchTransferFrom和_momoToken.transferFrom, 是需要用户持有参数ids_ & amounts_中传入的ERC1155, 和tokenIds_中传入的ERC721, 会transfer失败, 另外就算用户有足够多的NFT, 也会因为gas超过上限而交易失败, 所以不能发起denial-of-service attack

###MSM-13 | Public Function Could Be Declared External
* 处理方式: 无需修改
* 处理原因: Ownable在所有合约做是作为一个基类合约, 部分继承自Ownable的合约会在内部调用transferOwnership函数, 单独为每个合约使用不同逻辑的Ownable合约反而会增加整套合约的维护难度