<!-- vscode-markdown-toc -->

- 1. [HIGH: Invalid withdrawal logic: actual ETH received ignored in favor of requested amount](#HIGH:Invalidwithdrawallogic:actualETHreceivedignoredinfavorofrequestedamount)
  - 1.1. [Summary](#Summary)
  - 1.2. [Description](#Description)
  - 1.3. [Impact Explanation](#ImpactExplanation)
  - 1.4. [Likelihood Explanation](#LikelihoodExplanation)
  - 1.5. [Proof of Concept](#ProofofConcept)
  - 1.6. [Recommendations](#Recommendations)
- 2. [HIGH: Available ETH currentWithHeldEth erased during withdrawal, causing accounting errors and payout](#HIGH:AvailableETHcurrentWithHeldEtherasedduringwithdrawalcausingaccountingerrorsandpayout)
  - 2.1. [Summary](#Summary-1)
  - 2.2. [Description](#Description-1)
  - 2.3. [Impact Explanation](#ImpactExplanation-1)
  - 2.4. [Likelihood Explanation](#LikelihoodExplanation-1)
  - 2.5. [Proof of concept](#Proofofconcept)
  - 2.6. [Recommendation](#Recommendation)
- 3. [HIGH: Withheld funds cannot be staked since the function stakeWithheld always reverts](#HIGH:WithheldfundscannotbestakedsincethefunctionstakeWithheldalwaysreverts)
  - 3.1. [Summary](#Summary-1)
  - 3.2. [Finding Description](#FindingDescription)
  - 3.3. [Impact Explanation](#ImpactExplanation-1)
  - 3.4. [Likelihood Explanation](#LikelihoodExplanation-1)
  - 3.5. [Proof of Concept](#ProofofConcept-1)
- 4. [HIGH: Impossible to withdraw fees earned from withdrawals](#HIGH:Impossibletowithdrawfeesearnedfromwithdrawals)
  - 4.1. [Summary](#Summary-1)
  - 4.2. [Description](#Description-1)
  - 4.3. [Impact Explanation](#ImpactExplanation-1)
  - 4.4. [Likelihood Explantion](#LikelihoodExplantion)
  - 4.5. [Proof Of Concept](#ProofOfConcept)
  - 4.6. [Recommendation](#Recommendation-1)
- 5. [HIGH: Lack of access controls on function restake allows anyone to restake ETH](#HIGH:LackofaccesscontrolsonfunctionrestakeallowsanyonetorestakeETH)
  - 5.1. [Summary](#Summary-1)
  - 5.2. [Description](#Description-1)
  - 5.3. [Impact Explanation](#ImpactExplanation-1)
  - 5.4. [Likelihood](#Likelihood)
  - 5.5. [Proof Of Concept](#ProofOfConcept-1)
  - 5.6. [Recommendation](#Recommendation-1)
- 6. [MEDIUM: Splitting deposits to multiple validators prevents users from submitting(depositing) successfully](#MEDIUM:Splittingdepositstomultiplevalidatorspreventsusersfromsubmittingdepositingsuccessfully)
  - 6.1. [Summary](#Summary-1)
  - 6.2. [Description](#Description-1)
  - 6.3. [Impact Explanation](#ImpactExplanation-1)
  - 6.4. [Likelihood Explanation](#LikelihoodExplanation-1)
  - 6.5. [Proof of Concept](#ProofofConcept-1)
  - 6.6. [poc](#poc)
  - 6.7. [Recommendation:](#Recommendation:)
- 7. [MEDIUM: Improper Use of logical OR (||) in Capacity Check Causes Underflow and Deposit Failures](#MEDIUM:ImproperUseoflogicalORinCapacityCheckCausesUnderflowandDepositFailures)
  - 7.1. [Summary](#Summary-1)
  - 7.2. [Description](#Description-1)
  - 7.3. [Impact explanation](#Impactexplanation)
  - 7.4. [Likelihood explanation](#Likelihoodexplanation)
  - 7.5. [Recommendation](#Recommendation-1)

<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->

## 1. <a name='HIGH:Invalidwithdrawallogic:actualETHreceivedignoredinfavorofrequestedamount'></a>HIGH: Invalid withdrawal logic: actual ETH received ignored in favor of requested amount

### 1.1. <a name='Summary'></a>Summary

withdrawn is wrongly updated to amount when withdrawn is less than amount, which makes the contract believe we have more ETH available to send to user.

### 1.2. <a name='Description'></a>Description

When users are withdrawing after the cooldown period, if amount > currentWithheldETH we end up withdrawing from plumeStaking Original function implementation: https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/Liquid-Staking/src/stPlumeMinter.sol?lines=164,194 Let's see the function with some added logs for debugging

```solidity
function withdraw(address recipient) external nonReentrant returns (uint256 amount) {
    _rebalance();
    WithdrawalRequest storage request = withdrawalRequests[msg.sender];
    amount = request.amount;
    uint256 withdrawn;
    console.log("inside withdraw: amount requested", amount);
    if (amount > currentWithheldETH) {
        withdrawn = plumeStaking.withdraw();
        console.log("Amount withdrawn from plumeStaking", withdrawn);
        currentWithheldETH = 0;
    } else {
        withdrawn = amount;
        currentWithheldETH -= amount;
    }
    withdrawn = withdrawn > amount ? withdrawn : amount; //fees could be taken by staker contract so that less than requested amount is sent
    console.log("Withdrawn value after comparison", withdrawn);
    uint256 withholdFee = amount * WITHHOLD_FEE / 10000;
    currentWithheldETH += withdrawn - amount; //keep the rest of the funds for the rest of users that might have
    uint256 cachedAmount = withdrawn > amount ? amount : withdrawn;
    amount -= withholdFee;
    withHoldEth += cachedAmount - amount;
    address(recipient).call{value: amount}(""); //send amount to user
    emit Withdrawn(msg.sender, amount);
    return amount;
}
```

The case we are interested in is the first path where `amount > currentWithheldETH` When we withdraw on the line `plumeStaking.withdraw()`; the amount is saved in the variable `withdrawn`

Now, let's see how this variable is being handled afterwards.

The first thing we do is compare the `withdrawn` against `amount`(this is the amount user has requested to get)

```solidity
withdrawn = withdrawn > amount ? withdrawn : amount;
```

If withdrawn is greater than amount ie we have withdrawn more than what we need, then store withdran value as withdrawn otherwise store amount as withdrawn

Now, let's think of this for a min,

Suppose `plumeStaking.withdraw()` returned 5 ether, user had unstaked 10 ether so `amount=10 ether`, we end up with

```solidity
withdrawn = 5 > 10: 5 : 10
withdrawn = 10 // this is what we end up with
```

Now, why is this an issue, well this is now making the assumption that we have `withdrawn 10 ether`, which is not true.

As a result , going forward, we are working with `withdrawn=amount=10 ether`

The code then attempts to send eth to the user after taking out fees

```solidity
amount -= withholdFee;
withHoldEth += cachedAmount - amount;
address(recipient).call{value: amount}(""); //send amount to user
```

Now, note we are attempting to send the amount minus fee. Problem, is we are still working under the assumption that we did withdraw enough amount to cover this transfer.

Even though we only have around 5 ether we are attempting to send 10 ether - fee to the user which our contract doesn't have.

Due to lack of checks on the return values of the low level call sending ether, this went without being noticed - the bot should have found the missing check for return values.

### 1.3. <a name='ImpactExplanation'></a>Impact Explanation

This bug causes the contract to assume it has received more ETH than it actually has, leading to inflated user payouts, corrupted internal accounting. The contract will attempt to send more funds to users than it has.

### 1.4. <a name='LikelihoodExplanation'></a>Likelihood Explanation

High likelihood of this happening as everytime a user withdraws and we end up calling `plumeStaking.withdraw()`.

### 1.5. <a name='ProofofConcept'></a>Proof of Concept

```solidity
function test_withdraw_overwrites_withdrawnAmount() public {
    // First submit ETH
    for (uint256 i; i < 20; i++) {
        vm.prank(user1);
        minter.submit{value: 0.09 ether}();
        vm.prank(user1);
        minter.submit{value: 0.01 ether}();
    }
    vm.startPrank(user1);
    minter.submit{value: 20 ether}();
    // Unstake
    frxETHToken.approve(address(minter), 10 ether);
    minter.unstake(10 ether);
    vm.stopPrank();
    // Check withdrawal request
    (uint256 requestAmount, uint256 requestTimestamp) = minter.withdrawalRequests(user1);
    assertEq(requestAmount, 10 ether);
    vm.prank(address(minter));
    assertEq(requestTimestamp, mockPlumeStaking.cooldownEndDate());
    // Fast-forward past cooldown period
    //plumeStaking.cooldownEndDate();
    vm.warp(block.timestamp + 3 days);
    // Withdraw
    vm.startPrank(user1);
    minter.withdraw(user1);
    vm.stopPrank();
}
```

**logs**

```bash
[PASS] test_withdraw_overwrites_withdrawnAmount() (gas: 1640397)Logs:  unstake: Inside else block where currentWithheldETH > 0  inside withdraw: currentWithheldETH 88506034726988  inside withdraw: amount user unstaked 10000000000000000000  Amount > currentWithheldETH  Amount withdrawn from plumeStaking 7995674810443170545  Withdrawn value after comparison 10000000000000000000  CachedAmount  10000000000000000000  amount being sent to user after fee 9900000000000000000
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 29.49s (24.98s CPU time)
```

Let's add the missing check for return values of the low level call to see the actual error

```solidity
+    (bool success,) = address(recipient).call{value: amount}(""); //send amount to user
+   require(success, "Unable to send amount");
```

**logs**

```bash
Ran 1 test for test/fork/stPlumeMinter.t.sol:StPlumeMinterForkTest[FAIL: revert: Unable to send amount] test_withdraw_overwrites_withdrawnAmount() (gas: 1837109)Logs:  unstake: Inside else block where currentWithheldETH > 0  inside withdraw: currentWithheldETH 88506035449512  inside withdraw: amount user unstaked 10000000000000000000  Amount > currentWithheldETH  Amount withdrawn from plumeStaking 7995674746363149390  Withdrawn value after comparison 10000000000000000000  CachedAmount  10000000000000000000  amount being sent to user after fee 9900000000000000000
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 28.67s (24.06s CPU time)
Ran 1 test suite in 29.73s (28.67s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)
Failing tests:Encountered 1 failing test in test/fork/stPlumeMinter.t.sol:StPlumeMinterForkTest[FAIL: revert: Unable to send amount] test_withdraw_overwrites_withdrawnAmount() (gas: 1837109)
```

### 1.6. <a name='Recommendations'></a>Recommendations

Do not overwrite the amount withdrawn from `plumeStaking.withdraw()` when comparing that amount with the amount user has requested to withdraw.

## 2. <a name='HIGH:AvailableETHcurrentWithHeldEtherasedduringwithdrawalcausingaccountingerrorsandpayout'></a>HIGH: Available ETH currentWithHeldEth erased during withdrawal, causing accounting errors and payout

### 2.1. <a name='Summary-1'></a>Summary

When users withdraw , the code does not track `currentWithHeldEth` properly leading to incorrect data, as the ETH tracked by this variable is simply reset to zero.

### 2.2. <a name='Description-1'></a>Description

Users can withdraw their stakes(after unstaking) by calling the function `withdraw()`

There are several paths from when someone calls unstake: https://github.com/cantina-competitions/mystic-monorepo/blob/42b70871d73c7acfd3b3f2b8ca19dee576be2123/Liquid-Staking/src/stPlumeMinter.sol#L86-L132

unstake function: (added some logs for easier debugging and tracking interesting variables :)

```solidity
function unstake(uint256 amount) external nonReentrant returns (uint256 amountUnstaked) {
    _rebalance();
    frxETHToken.minter_burn_from(msg.sender, amount);
    require(withdrawalRequests[msg.sender].amount == 0, "Withdrawal already requested");
    uint256 cooldownTimestamp;
    // Check if we can cover this with withheld ETH
    if (currentWithheldETH >= amount) {
        amountUnstaked = amount;
        cooldownTimestamp = block.timestamp + 1 days;
    } else {
        uint256 remainingToUnstake = amount;
        amountUnstaked = 0;
        if (currentWithheldETH > 0) {
            amountUnstaked = currentWithheldETH;
            console.log("unstake: Inside else block where currentWithheldETH > 0 ");
            remainingToUnstake -= currentWithheldETH;
            currentWithheldETH = 0;
        }
        uint16 validatorId = 1;
        uint256 numVals = numValidators();
        while (remainingToUnstake > 0 && validatorId <= numVals) {
            (bool active,, uint256 stakedAmount,) = plumeStaking.getValidatorStats(uint16(validatorId));
            if (active && stakedAmount > 0) {
                // Calculate how much to unstake from this validator
                uint256 unstakeFromValidator = remainingToUnstake > stakedAmount ? stakedAmount : remainingToUnstake;
                uint256 actualUnstaked = plumeStaking.unstake(validatorId, unstakeFromValidator);
                amountUnstaked += actualUnstaked;
                remainingToUnstake -= actualUnstaked;
                if (remainingToUnstake == 0) break;
            }
            validatorId++;
            require(validatorId <= 10, "Too many validators checked");
        }
        cooldownTimestamp = plumeStaking.cooldownEndDate();
    }
    require(amountUnstaked > 0, "No funds were unstaked");
    require(amountUnstaked >= amount, "Not enough funds unstaked");
    withdrawalRequests[msg.sender] = WithdrawalRequest({amount: amountUnstaked, timestamp: cooldownTimestamp});
    emit Unstaked(msg.sender, amountUnstaked);
    return amountUnstaked;
    }
```

In our case, the `currentWithheldETH` is less than amount so we follow the 2nd path(else block). Inside the else block, our `currentWithheldETH` is greater than 0, so we use some of `currentWithheldETH` to cover the amount requested, therefore we reset the `currentWithheldETH to 0 ` and calculate how much we'd still need.

```solidity
remainingToUnstake -= currentWithheldETH;
currentWithheldETH = 0;
```

Then we call `plumeStaking.unstake()` to get the remaining amount we require to cover the amount user requested. After all this, the withdrawal request is added

```solidity
withdrawalRequests[msg.sender] = WithdrawalRequest({amount: amountUnstaked, timestamp: cooldownTimestamp});
```

Note: At this point, we have reset `currentWithheldETH` but once `withdraw` is called, it makes a call to `rebalance()` which could change the value of `currentWithheldETH` from non zero.

After the cooldown period the user decides to call `withdraw()` function: https://github.com/cantina-competitions/mystic-monorepo/blob/42b70871d73c7acfd3b3f2b8ca19dee576be2123/Liquid-Staking/src/stPlumeMinter.sol#L164-L193

Withdraw function is implemented as follows(added logs for ease of debugging)

```solidity
/// @notice Withdraw available funds that have completed cooling
function withdraw(address recipient) external nonReentrant returns (uint256 amount) {
    _rebalance();
    WithdrawalRequest storage request = withdrawalRequests[msg.sender];
    uint256 totalWithdrawable = plumeStaking.amountWithdrawable() + currentWithheldETH;
    require(block.timestamp >= request.timestamp, "Cooldown not complete");
    require(totalWithdrawable > 0, "Withdrawal not available yet");
    amount = request.amount;
    uint256 withdrawn;
    request.amount = 0;
    request.timestamp = 0;
    console.log("inside withdraw: currentWithheldETH", currentWithheldETH);
    console.log("inside withdraw: amount", amount);
    if (amount > currentWithheldETH) {
        console.log("Amount > currentWithheldETH");
        withdrawn = plumeStaking.withdraw();
        console.log("Amount withdrawn from plumeStaking", withdrawn);
        currentWithheldETH = 0;
    } else {
        withdrawn = amount;
        currentWithheldETH -= amount;
    }
    withdrawn = withdrawn > amount ? withdrawn : amount; //fees could be taken by staker contract so that less than requested amount is sent
    uint256 withholdFee = amount * WITHHOLD_FEE / 10000;
    currentWithheldETH += withdrawn - amount; //keep the rest of the funds for the rest of users that might have
    uint256 cachedAmount = withdrawn > amount ? amount : withdrawn;
    amount -= withholdFee;
    withHoldEth += cachedAmount - amount;
    address(recipient).call{value: amount}(""); //send amount to user
    emit Withdrawn(msg.sender, amount);
    return amount;
}
```

As a result of calling `rebalance()` inside this function, `currentWithheldETH` may have increased.

We are interested in the if block, we check if the amount the user had requested is greater than `currentWithheldETH`. If that's the case,it means, we don't have enough to give to the user so we resort to withdrawing some from plumeStaking

```solidity
withdrawn = plumeStaking.withdraw()
```

This is as opposed to the else block where we know we have enough `currentWithheldETH` to cover the withdrawal without withdrawing any more funds from the plumeStaking. Note, the else block would just deduct the amount user requested from the state variable `currentWithheldETH`

However, the if block, does something weird, after we withdraw from plumeStaking

```solidity
if (amount > currentWithheldETH) {
    withdrawn = plumeStaking.withdraw()
    currentWithheldETH = 0;
}
```

The state variable `currentWithheldETH` is reset to 0. The old value is not stored anywhere, we just overwrite it to 0

The function exits the block and continues with the following:

```solidity
withdrawn = withdrawn > amount ? withdrawn : amount;
uint256 withholdFee = amount * WITHHOLD_FEE / 10000;
currentWithheldETH += withdrawn - amount; //keep the rest of the funds for the rest of users that might have
```

### 2.3. <a name='ImpactExplanation-1'></a>Impact Explanation

The contract fails to track the `currentWithheldETH` correctly which affects all calculations that rely on on this state variable. This Understates funds and miscalculates remaining ETH as we are essentially wiping out real ETH out by resetting what is meant to track it.

### 2.4. <a name='LikelihoodExplanation-1'></a>Likelihood Explanation

The likelihood of this happening is high as this will happen every time a user calls withdraw and we use the first path

### 2.5. <a name='Proofofconcept'></a>Proof of concept

We first do small deposits amounting to 2 eth, we do small deposits to ensure it's not staked but added to `currentWithheldETH`. We then make a full deposit of say 20 ether which is now staked, then we unstake a small amount 10 ether. The `currentWithheldETH being > 0` is necessary to achieve the path we want to follow.

```solidity
function test_withdraw_resetsCurrentWithheldWrongly() public {
    // First submit ETH
    for (uint256 i; i < 20; i++) {
        vm.prank(user1);
        minter.submit{value: 0.09 ether}();
        vm.prank(user1);
        minter.submit{value: 0.01 ether}();
    }
    vm.startPrank(user1);
    minter.submit{value: 20 ether}();
    console.log("currentWithheldETH at the start", minter.currentWithheldETH());
    // Unstake
    frxETHToken.approve(address(minter), 10 ether);
    minter.unstake(10 ether);
    console.log("currentWithheldETH after calling unstake", minter.currentWithheldETH());
    vm.stopPrank();
    // Check withdrawal request
    (uint256 requestAmount, uint256 requestTimestamp) = minter.withdrawalRequests(user1);
    assertEq(requestAmount, 10 ether);
    vm.prank(address(minter));
    assertEq(requestTimestamp, mockPlumeStaking.cooldownEndDate());
    // Fast-forward past cooldown period
    //plumeStaking.cooldownEndDate();
    vm.warp(block.timestamp + 3 days);
    // Withdraw
    vm.startPrank(user1);
    minter.withdraw(user1);
    console.log("currentWithheldETH after calling withdraw", minter.currentWithheldETH());
    vm.stopPrank();
}
```

**logs**

```bash
Ran 1 test for test/fork/stPlumeMinter.t.sol:StPlumeMinterForkTest[PASS] test_withdraw_resetsCurrentWithheldWrongly() (gas: 1636814)Logs:  currentWithheldETH at the start 2000000000000000000  unstake: Inside else block where currentWithheldETH > 0  currentWithheldETH after calling unstake 0  inside withdraw: currentWithheldETH 88506023321534  inside withdraw: amount 10000000000000000000  Amount > currentWithheldETH  Amount withdrawn from plumeStaking 7995675821975431599  currentWithheldETH after calling withdraw 0
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 30.92s (25.70s CPU time)
```

Even though, `currentWithheldETH` was non zero, we have reset it to now zero and this amount was never tracked or added anywhere.

### 2.6. <a name='Recommendation'></a>Recommendation

Instead of resetting the `currentWithheldETH to 0`, we should add it to the amount we withdraw from plumeStaking to get the actual amount we now have at our disposal to send to user

```solidity
if (amount > currentWithheldETH) {
    withdrawn = plumeStaking.withdraw();
    withdrawn += currentWithheldETH;
    currentWithheldETH = 0;
} else {
    withdrawn = amount;
    currentWithheldETH -= amount;
}
```

Alternatively, introduce a new variable that records the sum of withdrawn and `currentWithheldETH` before we reset `currentWithheldETH`. This would then be the amount available that we can send to user

## 3. <a name='HIGH:WithheldfundscannotbestakedsincethefunctionstakeWithheldalwaysreverts'></a>HIGH: Withheld funds cannot be staked since the function stakeWithheld always reverts

### 3.1. <a name='Summary-1'></a>Summary

The function `stakeWitheld()` would always revert whenever called in an attempt to stake withheld tokens.

### 3.2. <a name='FindingDescription'></a>Finding Description

Once the withold ratio is set, some ETH is always withheld and not staked.

```solidity
// Track the amount of ETH that we are keeping
uint256 withheld_amt = 0;
if (withholdRatio != 0) {
    withheld_amt = (msg.value * withholdRatio) / RATIO_PRECISION;
    currentWithheldETH += withheld_amt;
    }
    amount = msg.value - withheld_amt;
```

The amount withheld can later be staked by those with the role of `REBALANCER_ROLE` by calling the function `stakeWitheld`

https://github.com/cantina-competitions/mystic-monorepo/blob/42b70871d73c7acfd3b3f2b8ca19dee576be2123/Liquid-Staking/src/stPlumeMinter.sol#L144-L152

```solidity
function stakeWitheld(uint16 validatorId, uint256 amount)
external
nonReentrant
onlyRole(REBALANCER_ROLE)
returns (uint256 amountRestaked)
{
    _rebalance(); //@audit rebance will deposit the entire balance here.
    currentWithheldETH -= amount;
    depositEther(amount); // @audit at this point we don't have any funds since rebalance deposited them
    emit ETHSubmitted(address(this), address(this), amount, 0);
    return amount;
}
```

The function calls the internal function `_rebalance()` which is implemented as follows.

https://github.com/cantina-competitions/mystic-monorepo/blob/42b70871d73c7acfd3b3f2b8ca19dee576be2123/Liquid-Staking/src/stPlumeMinter.sol#L287-L293

```solidity
/// @notice Rebalance the contract
function _rebalance() internal {
    uint256 amount = _claim();
    frxETHToken.minter_mint(address(this), amount);
    frxETHToken.transfer(address(sfrxETHToken), amount);
    depositEther(address(this).balance);
}
```

Among the things `_rebalance()` does, is call `depositEther()` function passing the entire contract balance.

```solidity
depositEther(address(this).balance);
```

The main function now `stakeWithheld` adjusts the amount of `currentWithheldETH` to reflect that we are now holding less funds than we did initially since we are staking some of it.

```solidity
currentWithheldETH -= amount;
```

We adjust by the amount passed as the argument to `stakeWithheld()`, then we call `depositEther()` passing the amount as the argument of deposit function

Inside `deposit()` we would attempt to stake this amount if it's more than `minStakeAmount`

```solidity
while (remainingAmount > 0) {
    uint256 depositSize = remainingAmount;
    (uint256 validatorId, uint256 capacity) = getNextValidator(remainingAmount);
            if (capacity < depositSize) {
                depositSize = capacity;
            }
            plumeStaking.stake{value: depositSize}(uint16(validatorId));
            remainingAmount -= depositSize;
            depositedAmount += depositSize;
            emit DepositSent(uint16(validatorId));
            }
        return depositedAmount;
}
```

The only problem is, the function `_rebalance()` that we called initially, has already deposited the entire amount of funds inside this contract which means, by the time we get to the line `depositEther(amount);` there are no funds left as such the second call to deposit would never pass

**Example case:**

We are witholding 50% of the deposited amount.

    - user deposits, 10 ether, we withhold 5 ether -> contract balance = 5 ether
    - stakeWithheld is called with amount as 2 ether - > this calls `rebalance()` which now calls deposit

```solidity
depositEther(address(this).balance);
```

After calling rebalance, `address(this).balance = 0` since we have deposited.

    - stakeWithheld now updates the state variable that currentWithheldEth and then tries to call deposit one more time

```solidity
depositEther(2 ether);
```

Only issue is, we don't have the 2 ether now, as our contract has 0 balance. So attempting to call plumeStaking would always revert with `EvmError: OutOfFunds`

### 3.3. <a name='ImpactExplanation-1'></a>Impact Explanation

`StakeWithheld()` function is broken and withheld tokens cannot be staked. As much as as call to `rebalance` succeeds in depositing the balance, the follow up call inside the same function `(stakeWithheld)` causes the entire state change to revert.

### 3.4. <a name='LikelihoodExplanation-1'></a>Likelihood Explanation

High likelihood as this function would always revert when called. This would prevent the witheheld tokens from being staked.

### 3.5. <a name='ProofofConcept-1'></a>Proof of Concept

```solidity
function test_stakeWithheld_alwaysReverts() public {
    // set withhold ratio
    console.log("current withheld", minter.currentWithheldETH());
    console.log("contract balance", address(minter).balance);
        vm.startPrank(owner);
        // Set withhold ratio to 50% (500000 = 50%)
        minter.setWithholdRatio(500000);
        vm.stopPrank();
        // Submit ETH with 50% to be withheld
        vm.prank(user1);
        minter.submit{value: 10 ether}();
        // Check that ETH was withheld
        assertEq(minter.currentWithheldETH(), 5 ether);
        console.log("current withheld after submit with 50% withholding", minter.currentWithheldETH());
        console.log("contract balance after submit with 50% withholding", address(minter).balance);
        // REBALANCER decides to stake some of the withheld funds, this would revert
        vm.startPrank(owner);
        vm.expectRevert();
        uint256 amountStaked = minter.stakeWitheld(1, 2 ether);
        // deposit is already called on balance
        vm.stopPrank();
        console.log("amount staked", amountStaked);
}
```

**logs**

```bash
Ran 1 test for test/fork/stPlumeMinter.t.sol:StPlumeMinterForkTest[PASS] test_stakeWithheld_alwaysReverts() (gas: 676387)Logs:  current withheld 0  contract balance 0  current withheld after submit with 50% withholding 5000000000000000000  contract balance after submit with 50% withholding 5000000000000000000  amount staked 0
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 19.94s (15.43s CPU time)
```

**extended logs**

```bash
│   ├─ [0] 0xA20bfe49969D4a0E9abfdb6a46FeD777304ba07f::stake{value: 2000000000000000000}(1)    │   │   └─ ← [OutOfFunds] EvmError: OutOfFunds    │   └─ ← [Revert] EvmError: Revert    ├─ [0] VM::stopPrank()    │   └─ ← [Return]    ├─ [0] console::log("amount staked", 0) [staticcall]

Note, the initial call to deposit from the _rebalance() indeed passed

│   ├─ [7691] 0xA20bfe49969D4a0E9abfdb6a46FeD777304ba07f::stake{value: 5001087451381946277}(1)    │   │   ├─ [6810] 0x76eF355e6DdB834640a6924957D5B1d87b639375::stake{value: 5001087451381946277}(1) [delegatecall]    │   │   │   ├─  emit topic 0: 0x521d5961e1d8e7e104af28f00e1f7e11655e7cc7e8d7a9b7a07e959a1598e215    │   │   │   │        topic 1: 0x000000000000000000000000fa0f05a76ebcbf400856ccccf29ff66d8acc843f    │   │   │   │        topic 2: 0x0000000000000000000000000000000000000000000000000000000000000001    │   │   │   │           data: 0x00000000000000000000000000000000000000000000000045676e8a4648d7a50000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000045676e8a4648d7a5    │   │   │   └─ ← [Return] 5001087451381946277 [5.001e18]    │   │   └─ ← [Return] 5001087451381946277 [5.001e18]
```

## 4. <a name='HIGH:Impossibletowithdrawfeesearnedfromwithdrawals'></a>HIGH: Impossible to withdraw fees earned from withdrawals

### 4.1. <a name='Summary-1'></a>Summary

Due to how the function `withdrawFees` is implemented, the owner cannot send the fees collected from withdrawals to his address.

### 4.2. <a name='Description-1'></a>Description

When users withdraw by calling `withdraw()` they are charged a fee. see below https://github.com/cantina-competitions/mystic-monorepo/blob/42b70871d73c7acfd3b3f2b8ca19dee576be2123/Liquid-Staking/src/stPlumeMinter.sol#L164-L194

```solidity
uint256 withholdFee = amount * WITHHOLD_FEE / 10000;
currentWithheldETH += withdrawn - amount;
//keep the rest of the funds for the rest of users that might have unstaked to avoid gas loss to unstake, withdraw but fees are taken by staker too so recognize that
uint256 cachedAmount = withdrawn > amount ? amount : withdrawn;
amount -= withholdFee;        withHoldEth += cachedAmount - amount;
address(recipient).call{value: amount}("");
```

The fees are added to the state variable `withHoldEth.`

The contract provides another function which can only be called by `onlyByOwnGov` which is called `withdrawFee()` The function is implemented as follows(added logs for debugging); https://github.com/cantina-competitions/mystic-monorepo/blob/42b70871d73c7acfd3b3f2b8ca19dee576be2123/Liquid-Staking/src/stPlumeMinter.sol#L155-L161

```solidity
function withdrawFee() external nonReentrant onlyByOwnGov returns (uint256 amount) {
    _rebalance();
    address(owner).call{value: withHoldEth}("");
    amount = withHoldEth;
    withHoldEth = 0;
    return amount;
}
```

This should allow owners to send the fees to address owner.

The problem comes from the fact that this function begines by calling `_rebalance()`, see `rebalance()` below

```solidity
function _rebalance() internal {
    uint256 amount = _claim();
    frxETHToken.minter_mint(address(this), amount);
    frxETHToken.transfer(address(sfrxETHToken), amount);
    depositEther(address(this).balance);
}
```

This function calls `depositEther()` passing the entire contract balance as the argument. This in turn calls deposit which stakes the entire contract balance.

This means, by the time we execute the rest of the function `withdrawFee()` there's no ETH left as it has all been staked. Therefore the call `address(owner).call{value: withHoldEth}("");` would revert.

To make things worse, the contract didn't check the return value for this low level call so this was never spotted.

### 4.3. <a name='ImpactExplanation-1'></a>Impact Explanation

As fees are meant to be sent to a given address, this bug prevents this from happening as all amount ends up being staked. Medium over high as funds are not lost just redirected somewhere else.

### 4.4. <a name='LikelihoodExplantion'></a>Likelihood Explantion

High likelihood as everytime this function is called, the accumulated fees would always be staked instead of being sent to the owner.

### 4.5. <a name='ProofOfConcept'></a>Proof Of Concept

To make sure we always have enough balance when about to call withdrawFees, we make initial deposit, call unstake and then we withdraw which gets the fees.

Before we withdraw Fees,just to be certain the contract had enough balance when getting inside the withdrawFee function, the poc will set a withholding ratio allowing the contract to keep 20% of the amount being deposited, then have another user call deposit with 10 ether, allowing us to keep 2 ether. This means contract balance should be enough to cover the fees collected.

```solidity
function test_withdraw_flow() public {
    // First submit ETH
    vm.prank(user1);
    minter.submit{value: 5 ether}();
    // Unstake
    vm.startPrank(user1);
    frxETHToken.approve(address(minter), 3 ether);
    minter.unstake(3 ether);
    // Fast-forward past cooldown period
    vm.warp(block.timestamp + 3 days);
    console.log("Contract balance before withdrawal", address(minter).balance);
    // Withdraw
    minter.withdraw(user1);
    vm.stopPrank();
    console.log("contract balance after withdrawal", address(minter).balance);
    console.log("fees stored in withHoldEth ", minter.withHoldEth());
    // Check withdrawal request was cleared
    (uint256 requestAmount, uint256 requestTimestamp) = minter.withdrawalRequests(user1);
    assertEq(requestAmount, 0);
    assertEq(requestTimestamp, 0);
    vm.startPrank(owner);
    // Set withhold ratio to 50% (200000 = 20%)
    minter.setWithholdRatio(200000);
    assertEq(minter.withholdRatio(), 200000);
    vm.stopPrank();
    //deposit something to increase contract balance
    vm.prank(user2);
    minter.submit{value: 10 ether}();
    console.log("contract balance after user2 deposits", address(minter).balance);
    // let's attempt to collect fees
    vm.startPrank(owner);
    uint256 amount = minter.withdrawFee();
    console.log("contract balance after calling withdrawFee", address(minter).balance);
    console.log("amount collected as fees", amount);
    vm.stopPrank();    }
```

**logs**

```bash
[PASS] test_withdraw_flow() (gas: 1046280)Logs:  Contract balance before withdrawal 1088478523175913  contract balance after withdrawal 28918152422595508  fees stored in withHoldEth  30000000000000000  contract balance after user2 deposits 2028918152422595508  balance before calling send just after calling rebalance 0  contract balance after calling withdrawFee 0  amount collected as fees 30000000000000000
```

**Notice, the contract balance after calling rebalance=0**

since return value of low level call is not being checked, the contract assumes everything went right

Let's fix the return value check and run the test again

```solidity
function withdrawFee() external nonReentrant onlyByOwnGov returns (uint256 amount) {
    _rebalance();
    console.log("balance before calling send just after calling rebalance", address(this).balance);
    (bool success,) = address(owner).call{value: withHoldEth}("");
    require(success, "Failed to send fees");
    amount = withHoldEth;
    withHoldEth = 0;
    return amount;
}
```

**logs from running the test again**

```bash
Ran 1 test for test/fork/stPlumeMinter.t.sol:StPlumeMinterForkTest[FAIL: revert: Failed to send fees] test_withdraw_flow() (gas: 1268217)Logs:  Contract balance before withdrawal 1088548824898604  contract balance after withdrawal 28918082121330964  fees stored in withHoldEth  30000000000000000  contract balance after user2 deposits 2028918082121330964  balance before calling send just after calling rebalance 0

│   ├─ [0] console::log("balance before calling send just after calling rebalance", 0) [staticcall]    │   │   └─ ← [Stop]    │   ├─ [0] 0x0000000000000000000000000000000000001234::fallback{value: 30000000000000000}()    │   │   └─ ← [OutOfFunds] EvmError: OutOfFunds    │   ├─  storage changes:    │   │   @ 0x9b779b17422d0df92223018b32b4d1fa46e071723d6817e2486d003becc55f00: 2 → 1    │   │   @ 4: 1 → 2    │   └─ ← [Revert] revert: Failed to send fees
```

**Note:** we can now clearly see where the error happened

### 4.6. <a name='Recommendation-1'></a>Recommendation

Several things need to be fixed: Ensure we have enough balance in the contract to send fees Check the return value of the low level call to ensure everything went through As rebalance always ends up staking the entire contract balance, might need to reevaluate if we need it inside the withdrawFees function

## 5. <a name='HIGH:LackofaccesscontrolsonfunctionrestakeallowsanyonetorestakeETH'></a>HIGH: Lack of access controls on function restake allows anyone to restake ETH

### 5.1. <a name='Summary-1'></a>Summary

The function `restake()` lacks access control as such anyone can restake the funds currently in cooldown.

### 5.2. <a name='Description-1'></a>Description

After calling `unstake()` ,the unstaked amount is subjected to a cooldown period before which a user cannot withdraw. As seen in `withdraw()` we check if cooldown period is over before proceeding.

```solidity
// @notice Withdraw available funds that have completed cooling
function withdraw(address recipient) external nonReentrant returns (uint256 amount) {
    _rebalance();
    WithdrawalRequest storage request = withdrawalRequests[msg.sender];
    uint256 totalWithdrawable = plumeStaking.amountWithdrawable() + currentWithheldETH;
    require(block.timestamp >= request.timestamp, "Cooldown not complete");
}
```

Another function exists `restakes()` that allows restaking the funds that are currently in cooling stage, see below

Affected line of code https://github.com/cantina-competitions/mystic-monorepo/blob/42b70871d73c7acfd3b3f2b8ca19dee576be2123/Liquid-Staking/src/stPlumeMinter.sol#L135-L142

```solidity
/// @notice Restake from cooling/parked funds to a specific validator
function restake(uint16 validatorId) external nonReentrant returns (uint256 amountRestaked) {
    _rebalance();
    (PlumeStakingStorage.StakeInfo memory info) = plumeStaking.stakeInfo(address(this));
    amountRestaked = plumeStaking.restake(validatorId, info.cooled + info.parked);
    emit Restaked(address(this), validatorId, amountRestaked);
    return amountRestaked;
}
```

The issue here is this function is not protected and as such anyone can be able to restake the funds in cooldown.

### 5.3. <a name='ImpactExplanation-1'></a>Impact Explanation

This is a critical function as such there should be limit to who can call it. When users call `unstake()`, the tokens are held in a cooldown state for a given period, then users can just withdraw them. However, this function restake is meant to restake this tokens which means removing them from cooldown. Only priviliged roles should be allowed to do this.

### 5.4. <a name='Likelihood'></a>Likelihood

High likelihood of this happening as the function is not protected in any way, anyone can just call it

### 5.5. <a name='ProofOfConcept-1'></a>Proof Of Concept

First we deposit and unstake as user1, then while the funds are in cooldown, user2 calls `restakes` which now restakes all the funds in cooldown.

```solidity
function test_restake_flow_no_accessControl() public {
    // User1 first submit ETH
    vm.prank(user1);
    minter.submit{value: 5 ether}();
    // User1 Unstake
    vm.startPrank(user1);
    frxETHToken.approve(address(minter), 3 ether);
    minter.unstake(3 ether);
    vm.stopPrank();
    // Confirm amount in cooled state
    PlumeStakingStorage.StakeInfo memory stakeInfo = minter.stakeInfo();
    console.log("In cooldown after user1 unstakes", stakeInfo.cooled);
    // Now user2 decides to call restake
    vm.prank(user2);
    uint256 amountRestaked = minter.restake(1);
    console.log("amount restaked", amountRestaked);
    // Check cooled amount is now 0
    PlumeStakingStorage.StakeInfo memory stakeInfo_ = minter.stakeInfo();
    assertEq(stakeInfo_.cooled, 0);
    console.log("In cooldown after user2 restakes", stakeInfo_.cooled);
}
```

**Run**

```bash
forge test --fork-url https://phoenix-rpc.plumenetwork.xyz/ --match-test test_restake_flow_no_accessControl -vv
```

**logs**

```bash
Ran 1 test for test/fork/stPlumeMinter.t.sol:StPlumeMinterForkTest[PASS] test_restake_flow_no_accessControl() (gas: 866738)Logs:  In cooldown after user1 unstakes 2997826306151267800  amount restaked 2997826306151267800  In cooldown after user2 restakes 0
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 28.26s (23.65s CPU time)
```

### 5.6. <a name='Recommendation-1'></a>Recommendation

Add access control to the function `restakes()`.

## 6. <a name='MEDIUM:Splittingdepositstomultiplevalidatorspreventsusersfromsubmittingdepositingsuccessfully'></a>MEDIUM: Splitting deposits to multiple validators prevents users from submitting(depositing) successfully

### 6.1. <a name='Summary-1'></a>Summary

Users cannot successfully `submit(deposit)` in some cases: Due to how splitting stakes across multiple validators is implemented, a user might not be able to deposit ETH as the amount is split and no follow up on `minStakeAmount` is made after the split. The check for `minStakeAmount` is only performed on the initial call to deposit, if this deposit ends up being split, we don't check this, which means , if the split allowed first deposit to go through, if the second deposit(from split amount) could be less than `minStakeAmount`

### 6.2. <a name='Description-1'></a>Description

When users call submit, we are rerouted to the function `deposit()` which checks if amount being deposited is greater than `minStakeAmount`. If so we get inside a loop where the deposit will be split among multiple validators if one validator has no enough capacity. This is where the issue is.

https://github.com/cantina-competitions/mystic-monorepo/blob/42b70871d73c7acfd3b3f2b8ca19dee576be2123/Liquid-Staking/src/stPlumeMinter.sol#L256-L285

```solidity
/// @notice Deposit ETH to validators, splitting across multiple if needed
function depositEther(uint256 _amount) internal returns (uint256 depositedAmount) {
    // Initial pause check
    require(!depositEtherPaused, "Depositing ETH is paused");
    require(_amount > 0, "Amount must be greater than 0");
    uint256 remainingAmount = _amount;
    depositedAmount = 0;
    if (remainingAmount < plumeStaking.getMinStakeAmount()) {
        currentWithheldETH += remainingAmount;
        return 0;
    }
    while (remainingAmount > 0) {
        uint256 depositSize = remainingAmount;
        (uint256 validatorId, uint256 capacity) = getNextValidator(remainingAmount);
        if (capacity < depositSize) {
            depositSize = capacity;
        }
        plumeStaking.stake{value: depositSize}(uint16(validatorId));
        remainingAmount -= depositSize;
        depositedAmount += depositSize;
        emit DepositSent(uint16(validatorId));
    }
    return depositedAmount;
}
```

If first validator has less capacity, we only deposit to their capacity `depositSize = capacity`;, As remaining amount is still greater than 0, we loop one more time `while (remainingAmount > 0) { `. The problem is now we don't check `if remainingAmount < plumeStaking.getMinStakeAmount()`. This means , we will call `stake()` for second validator without validating the `depositSize`. This would cause the call to revert as deposit must be greater or equal to `minStakeAmount`

### 6.3. <a name='ImpactExplanation-1'></a>Impact Explanation

User would be unable to deposit or submit any ETH to the contract if their deposit would end up being split among several validators.

### 6.4. <a name='LikelihoodExplanation-1'></a>Likelihood Explanation

There is a high likelihood of this happening, however some requirements are , the deposit is split among several validators, a small amount which is less than minStakeAmount remains after the split.

### 6.5. <a name='ProofofConcept-1'></a>Proof of Concept

There is a minStakeAmount of 0.1 ETH

As we don't have access to the function used to set the capacity, we will make one minor adjustment to hardcode the capacity to 2 ether for testing purposes.

Capacity is returned by the function `getNextValidator()` see the relevant part

```solidity
if (info.maxCapacity != 0 || totalStaked < info.maxCapacity) {
    return (validatorId, info.maxCapacity - totalStaked);
}
if (info.maxCapacity == 0) {
    return (validatorId, type(uint256).max - 1);
}
```

In our case,`info.maxCapacity == 0` so we always return capacity as `type(uint256).max - 1`. To simulate setting the capacity, we can force this function to return 2 ether instead

```solidity
if (info.maxCapacity == 0) {
    return (validatorId, 2 ether);
}
```

Now, our poc, we just try to `deposit 2.05 ether` which is greater than `minStakeAmount`. Since the first call `getNextValidator()` returns a validator with a capacity of 2 ether, only 2 ether is staked in the first case, we then check if we still have some amount remaining, in our case, 0.05 eth is still not staked. Since the loop only checks `if remainingAmount > 0`, we loop one more time, this time we intend to `deposit 0.05 ether` which is `less than minStakeAmount` but not checked this time. We end up calling `stake()` with `amount 0.05` which would obviously cause a revert.

Add the following test to stPlumeMinter.t.sol after setting capacity to 2 ether and run

```bash
forge test --fork-url https://phoenix-rpc.plumenetwork.xyz/ --match-test test_splitDepositsCouldFail
```

### 6.6. <a name='poc'></a>poc

```solidity
function test_splitDepositsCouldFail() public {
    // a deposit of 2.05 would be split into 2 if validator cap is 2 ether
    // As we attempt to deposit less than minStake in second iteration, we should revert
    vm.startPrank(user1);
    //error StakeAmountTooSmall(uint256 providedAmount, uint256 minAmount);
    vm.expectRevert(abi.encodeWithSelector(StakeAmountTooSmall.selector, 0.05 ether, 0.1 ether));
    minter.submit{value: 2.05 ether}(); //2.05 ether        // first validator takes the 2 ether
    // we should have 0.05 left which is less than minStakeAmount , thus should be added to currentWithHeldEth
    console.log("current withheld After", minter.currentWithheldETH());
    vm.stopPrank();
}
```

`error StakeAmountTooSmall(uint256 providedAmount, uint256 minAmount);` is defined in the staking contract so to use it in our test please also add to the file stPlumeMinter.t.sol

**results**

```bash
Ran 1 test for test/fork/stPlumeMinter.t.sol:StPlumeMinterForkTest[PASS] test_splitDepositsCouldFail() (gas: 403770)Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 15.16s (10.65s CPU time)
```

**extended**

```bash
│   ├─ [1335] 0xA20bfe49969D4a0E9abfdb6a46FeD777304ba07f::stake{value: 50000000000000000}(1)    │   │   ├─ [447] 0x76eF355e6DdB834640a6924957D5B1d87b639375::stake{value: 50000000000000000}(1) [delegatecall]    │   │   │   └─ ← [Revert] StakeAmountTooSmall(50000000000000000 [5e16], 100000000000000000 [1e17])    │   │   └─ ← [Revert] StakeAmountTooSmall(50000000000000000 [5e16], 100000000000000000 [1e17])    │   ├─  storage changes:    │   │   @ 0x693fae37672bc30bbe20719647917bd9103873c231d017918b8d43203391bb6d: 0 → 1    │   └─ ← [Revert] StakeAmountTooSmall(50000000000000000 [5e16], 100000000000000000 [1e17])
```

### 6.7. <a name='Recommendation:'></a>Recommendation:

When the stake is split to multiple validators, we should track remaining amount after staking with each validator and compare against the `minStakeAmount`. If the `remaining amount > 0` but less than `minStakeAmount` inside the while loop, we should add it to `currentWithHeldEth` and reset `remainingAmount.`

```solidity
function depositEther(uint256 _amount) internal returns (uint256 depositedAmount) {
    // Initial pause check
    require(!depositEtherPaused, "Depositing ETH is paused");
    require(_amount > 0, "Amount must be greater than 0");
    uint256 remainingAmount = _amount;
    depositedAmount = 0;
    if (remainingAmount < plumeStaking.getMinStakeAmount()) {
        currentWithheldETH += remainingAmount;
        return 0;
    }
    while (remainingAmount > 0) {
        uint256 depositSize = remainingAmount;
        (uint256 validatorId, uint256 capacity) = getNextValidator(remainingAmount);
        if (capacity < depositSize) {
            depositSize = capacity;
        }
        plumeStaking.stake{value: depositSize}(uint16(validatorId));
        remainingAmount -= depositSize;
        depositedAmount += depositSize;
        +if (remainingAmount < plumeStaking.getMinStakeAmount()) {
            +  currentWithheldETH += remainingAmount;
            +  remainingAmount = 0;
        + }
        emit DepositSent(uint16(validatorId));
    }
    return depositedAmount;
}
```

## 7. <a name='MEDIUM:ImproperUseoflogicalORinCapacityCheckCausesUnderflowandDepositFailures'></a>MEDIUM: Improper Use of logical OR (||) in Capacity Check Causes Underflow and Deposit Failures

### 7.1. <a name='Summary-1'></a>Summary

Function `getNextValidator()` uses logical `OR` instead of logical `AND` which might lead to deposits being impossible to make.

The operators `|| and &&` apply the common short-circuiting rules. This means that in the expression `f(x) || g(y)`, if `f(x)` evaluates to true, `g(y)` will not be evaluated even if it may have side-effects.

So if `info.maxCapacity != 0` is true, it won’t even check whether `totalStaked < info.maxCapacity`. This leads to unsafe arithmetic if `totalStaked > info.maxCapacity`, causing a revert due to underflow when subtracting.

### 7.2. <a name='Description-1'></a>Description

The function `getNextValidator()` is called everytime we deposit to try and get a validator where we can stake. This function is implemented as follows https://github.com/cantina-competitions/mystic-monorepo/blob/42b70871d73c7acfd3b3f2b8ca19dee576be2123/Liquid-Staking/src/stPlumeMinter.sol#L55-L78

```solidity
function getNextValidator(uint256 depositAmount) public returns (uint256 validatorId, uint256 capacity) {
        // Make sure there are free validators available
        uint256 numVals = numValidators();
        require(numVals != 0, "Validator stack is empty");
        console.log("number of validators", numVals);
        for (uint256 i = 0; i < numVals; i++) {
            validatorId = validators[i].validatorId;
            if (validatorId == 0) break;
            (bool active,,,) = plumeStaking.getValidatorStats(uint16(validatorId));
            if (!active) continue;
            (PlumeStakingStorage.ValidatorInfo memory info, uint256 totalStaked,) = plumeStaking.getValidatorInfo(uint16(validatorId));
            if (info.maxCapacity != 0 || totalStaked < info.maxCapacity) {
                return (validatorId, info.maxCapacity - totalStaked);
                }
            if (info.maxCapacity == 0) {
                return (validatorId, type(uint256).max - 1);
                }
            }
        revert("No validator with sufficient capacity");
        }
```

Of interest to us, is the following block:

```solidity
if (info.maxCapacity != 0 || totalStaked < info.maxCapacity) {
    return (validatorId, info.maxCapacity - totalStaked);
}
```

The function checks if maxCapacity has been set and therefore not zero and also checks if totalStaked < info.maxCapacity

The problem is, we use logical or || which means, if the first check is true, we do not check the second one ie `if info.maxCapacity != 0 `then we just get inside the if block to execute whatever code is there.

The problem arises when we find ourselves in a situation where `totalStaked > info.maxCapacity`. Due to rules of short circuiting, we never ensured that `totalStaked < info.maxCapacity`

As a result of this we will attempt to do the arithmetic `info.maxCapacity - totalStaked` which will revert preventing any deposits.

Capacity might be initially set to `type(uint256).max` then later adjusted to a smaller value using `setValidatorCapacity()`

### 7.3. <a name='Impactexplanation'></a>Impact explanation

All calls to deposit would end up reverting since the compiler will prevent any underflows(checked arithmetic) As deposits are a core part of this contract, this is a high impact bug as it prevents users from staking.

### 7.4. <a name='Likelihoodexplanation'></a>Likelihood explanation

If capacity is set without considering the current staked amount, then there is a chance this would cause a revert
Proof of concept

```solidity
function test_DepositsFail() public {        (, uint256 totalStaked,) = mockPlumeStaking.getValidatorInfo(uint16(1));
        console.log("Total staked before resetting capacity", totalStaked);
        vm.startPrank(address(0xC0A7a3AD0e5A53cEF42AB622381D0b27969c4ab5));        // Guarantees, new cap is less than totalStaked        mockPlumeStaking.setValidatorCapacity(1, totalStaked - 1 ether);        vm.stopPrank();
        vm.startPrank(user1);        minter.submit{value: 1 ether}();        vm.stopPrank();    }
```

**logs**

```bash
Ran 1 test for test/fork/stPlumeMinter.t.sol:StPlumeMinterForkTest[FAIL: panic: arithmetic underflow or overflow (0x11)] test_DepositsFail() (gas: 183621)Logs:  Total staked before resetting capacity 122102148517906405494
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 15.67s (9.52s CPU time)
Ran 1 test suite in 17.31s (15.67s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)
Failing tests:Encountered 1 failing test in test/fork/stPlumeMinter.t.sol:StPlumeMinterForkTest[FAIL: panic: arithmetic underflow or overflow (0x11)] test_DepositsFail() (gas: 183621)
```

Expected We should never even attempt to do the arithmetic if totalStaked > info.maxCapacity thus both conditions should be checked.

### 7.5. <a name='Recommendation-1'></a>Recommendation

Use logical AND(&&) instead of OR(||)

```solidity
if (info.maxCapacity != 0 && totalStaked < info.maxCapacity) {
    return (validatorId, info.maxCapacity - totalStaked);
}
```

With this, `if maxCapacity != 0`, we would still need to check `totalStaked < info.maxCapacity` before performing the operations inside the if block.
