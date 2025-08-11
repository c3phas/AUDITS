<!-- vscode-markdown-toc -->

- 1. [HIGH: First Depositor Exploit allows eToken Inflation](#HIGH:FirstDepositorExploitallowseTokenInflation)
  - 1.1. [Summary](#Summary)
  - 1.2. [Finding Description](#FindingDescription)
  - 1.3. [Impact Explanation](#ImpactExplanation)
  - 1.4. [Likelihood Explanation](#LikelihoodExplanation)
  - 1.5. [Proof of Concept](#ProofofConcept)
  - 1.6. [Recommendation](#Recommendation)
- 2. [MEDIUM: setBorrowingRateConfig Does Not Immediately Update currentBorrowingRate](#MEDIUM:setBorrowingRateConfigDoesNotImmediatelyUpdatecurrentBorrowingRate)
  - 2.1. [Summary](#Summary-1)
  - 2.2. [Finding Description](#FindingDescription-1)
  - 2.3. [Impact Explanation](#ImpactExplanation-1)
  - 2.4. [Likelihood Explanation](#LikelihoodExplanation-1)
  - 2.5. [Proof of Concept](#ProofofConcept-1)
  - 2.6. [Recommendation](#Recommendation-1)
- 3. [LOW: Rounding Error in Debt Accounting Prevents debt Repayment](#LOW:RoundingErrorinDebtAccountingPreventsdebtRepayment)
  - 3.1. [Summary](#Summary-1)
  - 3.2. [Finding Description](#FindingDescription-1)
  - 3.3. [Impact Explanation](#ImpactExplanation-1)
  - 3.4. [Likelihood Explanation](#LikelihoodExplanation-1)
  - 3.5. [Proof of Concept](#ProofofConcept-1)
  - 3.6. [Recommendation](#Recommendation-1)
- 4. [LOW: Disabling borrowing affects repaying making it impossible to repay borrowed tokens](#LOW:Disablingborrowingaffectsrepayingmakingitimpossibletorepayborrowedtokens)
  - 4.1. [Summary](#Summary-1)
  - 4.2. [Finding Description](#FindingDescription-1)
  - 4.3. [Impact Explanation](#ImpactExplanation-1)
  - 4.4. [Likelihood Explanation](#LikelihoodExplanation-1)
  - 4.5. [Proof of Concept](#ProofofConcept-1)
  - 4.6. [Recommendation](#Recommendation-1)
- 5. [INFORMATIONAL: Potential Overcrediting in repay() Due to Misordered Logic](#INFORMATIONAL:PotentialOvercreditinginrepayDuetoMisorderedLogic)
  - 5.1. [Summary](#Summary-1)
  - 5.2. [Finding Description](#FindingDescription-1)
  - 5.3. [Impact Explanation](#ImpactExplanation-1)
  - 5.4. [Likelihood Explanation](#LikelihoodExplanation-1)
  - 5.5. [Proof of Concept](#ProofofConcept-1)
  - 5.6. [Recommendation](#Recommendation-1)

<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->

## 1. <a name='HIGH:FirstDepositorExploitallowseTokenInflation'></a>HIGH: First Depositor Exploit allows eToken Inflation

### 1.1. <a name='Summary'></a>Summary

The first depositor can deposit a minimal amount, eg 1 wei, to receive 1 wei of eToken. If they then transfer a large sum of liquidity directly to the eToken contract address, it inflates the eToken rate, allowing their initial 1 wei of eToken to redeem the full liquidity amount, potentially draining the reserve

### 1.2. <a name='FindingDescription'></a>Finding Description

To deposit we call the function `_deposit` shown below https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/lendingpool/LendingPool.sol?lines=205,220

```solidity
function _deposit(uint256 reserveId, uint256 amount, address onBehalfOf) internal returns (uint256 eTokenAmount) {
    DataTypes.ReserveData storage reserve = getReserve(reserveId);
    require(!reserve.getFrozen(), Errors.VL_RESERVE_FROZEN);
    // update states
    reserve.updateState(getTreasury());
    // validate
    reserve.checkCapacity(amount);
    uint256 exchangeRate = reserve.reserveToETokenExchangeRate();
    // Transfer the user's reserve token to eToken contract
    pay(reserve.underlyingTokenAddress, _msgSender(), reserve.eTokenAddress, amount);
    // Mint eTokens for the user
    eTokenAmount = amount.mul(exchangeRate).div(Precision.FACTOR1E18);
    IExtraInterestBearingToken(reserve.eTokenAddress).mint(onBehalfOf, eTokenAmount);
    // update the interest rate after the deposit
    reserve.updateInterestRates();
}
```

To get the exchangeRate we call a function `reserveToETokenExchangeRate()` shown below:

```solidity
function reserveToETokenExchangeRate(DataTypes.ReserveData storage reserve) internal view returns (uint256) {
    (uint256 totalLiquidity,) = totalLiquidityAndBorrows(reserve);
    uint256 totalETokens = IERC20(reserve.eTokenAddress).totalSupply();
    if (totalETokens == 0 || totalLiquidity == 0) {
        return Precision.FACTOR1E18;
    }
    return totalETokens.mul(Precision.FACTOR1E18).div(totalLiquidity);
}
```

During the first deposit, `if totalETokens == 0`, the depositor receives `eTokens 1:1` based on a default exchange rate (1e18). However, the eToken contract allows direct transfers of underlying tokens via IERC20.transfer, which are not accounted for in `eToken supply`. if an attacker transfers some tokens directly, `balanceOf() ` would include this amount

```solidity
function availableLiquidity(DataTypes.ReserveData storage reserve) internal view returns (uint256 liquidity) {
    liquidity = IERC20(reserve.underlyingTokenAddress).balanceOf(reserve.eTokenAddress);
}
```

Because `totalLiquidity` (used in the exchange rate calculation) includes the full token balance of the eToken contract, but `totalETokens` remains unchanged (since it only tracks minted eTokens), the rate becomes incorrect. This breaks assumptions around proportional accounting.

if the attacker then redeems their 1 wei, the underlying amount is calculated as follows

```solidity
function _redeem(uint256 reserveId, uint256 eTokenAmount, address to, bool receiveNativeETH)
internal
returns (uint256)
{
    DataTypes.ReserveData storage reserve = getReserve(reserveId);
    // update states
    reserve.updateState(getTreasury());
    // calculate underlying tokens using eTokens
    uint256 underlyingTokenAmount = reserve.eTokenToReserveExchangeRate().mul(eTokenAmount).div(Precision.FACTOR1E18);
```

The function `eTokenToReserveExchangeRate()` basically returns `totalLiquidity/totalETokens`. Now since `totalLiquidity` returns the full token balance of the eToken contract we multiply `etokenAmount by the exchangeRate` . if 1,000,000 tokens was transferred to the eTokenContract then the attacker can simply get the entire amount by just redeeming 1 wei

### 1.3. <a name='ImpactExplanation'></a>Impact Explanation

This issue enables attackers to drain the entire pool with a minimal initial amount

### 1.4. <a name='LikelihoodExplanation'></a>Likelihood Explanation

This is very likely to happen as all the attacker has to do is be the first person to call deposit

### 1.5. <a name='ProofofConcept'></a>Proof of Concept

    - Make a minimal deposit (e.g., 1 wei) to receive 1 wei worth of eTokens.
    - Directly transfer a large amount of underlying tokens (e.g., 1,000 ETH) to the eToken contract.
    - Redeem the initial eTokens at an inflated exchange rate, withdrawing the full underlying balance.

### 1.6. <a name='Recommendation'></a>Recommendation

Burn a minimum amount of eTokens upon the first deposit, add the following to the `_deposit()` function

```diff
eTokenAmount = amount.mul(exchangeRate).div(Precision.FACTOR1E18);
+ require(eTokenAmount > MINIMUM_ETOKEN_AMOUNT, Errors.VL_ETOKEN_AMOUNT_TOO_SMALL);
+ if (IExtraInterestBearingToken(reserve.eTokenAddress).totalSupply() == 0) {
+ // Burn the first 1000 etoken, to defend against lp inflation attacks
+ IExtraInterestBearingToken(reserve.eTokenAddress).mint(
+  DEAD_ADDRESS,
+  MINIMUM_ETOKEN_AMOUNT
+  );
+ eTokenAmount -= MINIMUM_ETOKEN_AMOUNT;
+ } IExtraInterestBearingToken(reserve.eTokenAddress).mint(onBehalfOf, eTokenAmount);
```

## 2. <a name='MEDIUM:setBorrowingRateConfigDoesNotImmediatelyUpdatecurrentBorrowingRate'></a>MEDIUM: setBorrowingRateConfig Does Not Immediately Update currentBorrowingRate

### 2.1. <a name='Summary-1'></a>Summary

The `LendingPool::setBorrowingRateConfig()` function updates the interest rate configuration based on reserve utilization, but it does not immediately apply the updated values. As a result, the `currentBorrowingRate` remains outdated until a user-triggered action (deposit or borrow) forces the system to recalculate the interest rates.

### 2.2. <a name='FindingDescription-1'></a>Finding Description

The function `setBorrowingRateConfig()` is used to update the current configuration for the `borrowingRates` depending on the `utilisationRate of the reserve`. The function is implemented as shown below: https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/lendingpool/LendingPool.sol?lines=401,421

```solidity
function setBorrowingRateConfig(
    DataTypes.ReserveData storage reserve,
    uint16 utilizationA,
    uint16 borrowingRateA,
    uint16 utilizationB,
    uint16 borrowingRateB,
    uint16 maxBorrowingRate
    ) internal {
        // (0%, 0%) -> (utilizationA, borrowingRateA) -> (utilizationB, borrowingRateB) -> (100%, maxBorrowingRate)
        reserve.borrowingRateConfig.utilizationA = uint128(Precision.FACTOR1E18.mul(utilizationA).div(Constants.PERCENT_100));
        reserve.borrowingRateConfig.borrowingRateA = uint128(Precision.FACTOR1E18.mul(borrowingRateA).div(Constants.PERCENT_100));
        reserve.borrowingRateConfig.utilizationB = uint128(Precision.FACTOR1E18.mul(utilizationB).div(Constants.PERCENT_100));
        reserve.borrowingRateConfig.borrowingRateB = uint128(Precision.FACTOR1E18.mul(borrowingRateB).div(Constants.PERCENT_100));
        reserve.borrowingRateConfig.maxBorrowingRate = uint128(Precision.FACTOR1E18.mul(maxBorrowingRate).div(Constants.PERCENT_100));
    }
```

This sets the configuration,but it does not trigger a recalculation of the borrowing rate. Therefore, the `currentBorrowingRate` remains outdated until an external interaction (such as deposit or borrow) causes the reserve to update its state.

### 2.3. <a name='ImpactExplanation-1'></a>Impact Explanation

The protocol loses control over when the updated borrowing rate becomes active. Between the time the new configuration is set and the next reserve update (which is unpredictable), the old `currentBorrowingRate` will still be used. This means that the protocol can lose funds since they will want for example for the current utilization rate to have 25% interest rate but instead they will have the previous 15% interest rate until someone update the state by deposit/borrow

### 2.4. <a name='LikelihoodExplanation-1'></a>Likelihood Explanation

The likelihood of this happening is high as every time the protocol intends to change the borrowingRateConfig, the time the new change takes effect will always depend on when the next state update happens.

### 2.5. <a name='ProofofConcept-1'></a>Proof of Concept

    - A reserve is deployed with an initial borrowingRateConfig
    - The protocol updates this configuration by calling setBorrowingRateConfig
    - The updated configuration is not applied immediately. Instead, it takes effect only after the next user-triggered action that updates the reserve state.

### 2.6. <a name='Recommendation-1'></a>Recommendation

Consider calling `updateState` before the change of the configuration and `updateInterestRates` after the change of the configuration during the `LendingPool::setBorrowingRateConfig` call, in order to the update of the configuration to have instant and immediate effect on the `reserve.currentBorrowingRate`.

## 3. <a name='LOW:RoundingErrorinDebtAccountingPreventsdebtRepayment'></a>LOW: Rounding Error in Debt Accounting Prevents debt Repayment

### 3.1. <a name='Summary-1'></a>Summary

A vulnerability in the lending pool can prevent users from repaying their debts due to rounding errors. The reserves total debt `reserve.totalBorrows` may become less than the sum of individual debt positions borrowed amounts `debtPosition.borrowed` because of frequent updates to the reserves debt index and less frequent updates to debt positions. This can cause the `repay` function to revert due to underflow when attempting to subtract the `repayment amount` from `reserve.totalBorrows`, which would prevent users from repaying the debt

### 3.2. <a name='FindingDescription-1'></a>Finding Description

When repaying debt, we call the following function:

https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/lendingpool/LendingPool.sol?lines=324,360

```solidity
// update states
reserve.updateState(getTreasury());
updateDebtPosition(debtPosition, reserve.borrowingIndex);
// only vaultPositionManager contract has credits to borrow tokens
// when this function is called from the vaultPositionManager contracts,
// the _msgSender() is the contract's address
uint256 credit = credits[debtPosition.reserveId][_msgSender()];
credits[debtPosition.reserveId][_msgSender()] = credit.add(amount);
if (amount > debtPosition.borrowed) {
    amount = debtPosition.borrowed;
}
reserve.totalBorrows = reserve.totalBorrows.sub(amount);
debtPosition.borrowed = debtPosition.borrowed.sub(amount);
// Transfer the underlying tokens from the vaultPosition to the eToken contract
IERC20(reserve.underlyingTokenAddress).safeTransferFrom(_msgSender(), reserve.eTokenAddress, amount);
reserve.updateInterestRates();
emit Repay(debtPosition.reserveId, onBehalfOf, _msgSender(), amount);
return amount;
}
```

Of interest to us is the line `reserve.totalBorrows = reserve.totalBorrows.sub(amount);`

Debt calculations use an index variable `reserve.borrowingIndex` which grows with interest accrual. Each time the pool is updated (e.g., during deposits or repayments), the pool index and `reserve.totalBorrows` are adjusted based on the interest rate and time elapsed.

https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/libraries/logic/ReserveLogic.sol?lines=156,170

```solidity
newTotalBorrows = newBorrowingIndex.mul(reserve.totalBorrows).div(reserve.borrowingIndex);
```

The `newBorrowingIndex` is calculated as follows: https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/libraries/logic/ReserveLogic.sol?lines=112,121

```solidity
return reserve.borrowingIndex.mul(
    InterestRateUtils.calculateCompoundedInterest(reserve.currentBorrowingRate, reserve.lastUpdateTimestamp)
    ).div(Precision.FACTOR1E18);
```

This division in `newTotalBorrows` truncates small fractions, negligible unless this is often called.

Each debt position tracks its own index and borrowed amount `debtPosition.borrowed`, updated only when the position is modified (e.g., during repayment).

```solidity
function updateDebtPosition(DataTypes.DebtPositionData storage debtPosition, uint256 latestBorrowingIndex)
internal    {
    debtPosition.borrowed = debtPosition.borrowed.mul(latestBorrowingIndex).div(debtPosition.borrowedIndex);
    debtPosition.borrowedIndex = latestBorrowingIndex;
}
```

This also includes divisions but fewer updates mean fewer rounding errors, so `debtPosition.borrowed` is more accurate.

`reserve.totalBorrows` should equal the sum of all `debtPosition.borrowed` amounts. However, integer arithmetic in pool updates causes small rounding errors, as fractions are truncated.

These errors accumulate in `reserve.totalBorrows` with frequent updates, while debt positions, updated less often, retain more precision. Over time, this can make `reserve.totalBorrows` slightly less than the total of all debt positions, preventing users from repaying their debts due to underflow errors in the repay function.

### 3.3. <a name='ImpactExplanation-1'></a>Impact Explanation

This makes it impossible for a user who wishes to repay their debt,which leads to locking their positions and potential liquidation risks. The risk of liquidation makes this a high risk

### 3.4. <a name='LikelihoodExplanation-1'></a>Likelihood Explanation

As this relies on multiple pool updates, high interest rates, or long time intervals to accumulate significant errors, the likelihood is low

### 3.5. <a name='ProofofConcept-1'></a>Proof of Concept

    - Pool debt: reserve.totalBorrows = 1000 USDC.
    - User debt: debtPosition.borrowed = 1000.0001 USDC (due to less rounding).
    - User tries to repay 1000.0001 USDC, but the contract attempts 1000 - 1000.0001, which fails.

```solidity
reserve.totalBorrows = reserve.totalBorrows.sub(amount);
```

### 3.6. <a name='Recommendation-1'></a>Recommendation

Cap the repayment amount using both the user’s position and the reserve’s borrow total to avoid reverts due to rounding.

```solidity
if (amount > debtPosition.borrowed) {
    amount = debtPosition.borrowed;
}
if (amount > reserve.totalBorrows) {
    amount = reserve.totalBorrows;
}
```

## 4. <a name='LOW:Disablingborrowingaffectsrepayingmakingitimpossibletorepayborrowedtokens'></a>LOW: Disabling borrowing affects repaying making it impossible to repay borrowed tokens

### 4.1. <a name='Summary-1'></a>Summary

Unnecessary check on function repay might make it impossible to repay tokens borrowed.

### 4.2. <a name='FindingDescription-1'></a>Finding Description

When borrowing from lending pool, we have some constraints, ie only the vault contracts can

```solidity
// Whitelist of vault contracts
// only vault contracts in the whitelist can borrow from the lending pool
mapping(address => bool) public borrowingWhiteList;
```

When we call function `borrow()` we have a check that the `msg.sender` is in the whitelist which reverts if not in whitelist

```solidity
function borrow(address onBehalfOf, uint256 debtId, uint256 amount) external notPaused nonReentrant {
    require(borrowingWhiteList[msg.sender], Errors.VL_BORROWING_CALLER_NOT_IN_WHITELIST);
```

This is all good and ensures only those in the whitelist can actually borrow.

The issue arises with function `repay()` as it also has the same check.

```solidity
function repay(address onBehalfOf, uint256 debtId, uint256 amount)
external
notPaused
nonReentrant
returns (uint256)
{
    require(borrowingWhiteList[msg.sender], Errors.VL_BORROWING_CALLER_NOT_IN_WHITELIST);
```

Now note, an admin can add or remove an address from the whitelist . Of interest to us is removing or disabling

```solidity
function disableVaultToBorrow(uint256 vaultId) external onlyOwner notPaused {
    address vaultAddr = getVault(vaultId);
    borrowingWhiteList[vaultAddr] = false;
    emit DisableVaultToBorrow(vaultId, vaultAddr);
}
```

If this function is called, the following check would always be false.

```solidity
require(borrowingWhiteList[msg.sender], Errors.VL_BORROWING_CALLER_NOT_IN_WHITELIST);
```

This works well if the check only exists in the function `borrow()` but in our case , the check is also in the function `repay()`. If an admin calls this `disableVaultToBorrow()` and pass the `vaultId` that returns the address that has already borrowed ,then it would not be possible to ever repay the borrowed tokens as function `repay()` would always revert

Note As the whitelist is about borrowing, having this check on repay function is not needed

### 4.3. <a name='ImpactExplanation-1'></a>Impact Explanation

Borrowed funds cannot be repayed, medium as the admin can just enable/add the address the whitelist again

### 4.4. <a name='LikelihoodExplanation-1'></a>Likelihood Explanation

Whenever we disable borrowing for an address,if the address has already borrowed prior to this disabling, then calling repay would always revert

### 4.5. <a name='ProofofConcept-1'></a>Proof of Concept

A whitelisted User calls function borrow , this passes as the user is in the whitelist

```solidity
require(borrowingWhiteList[msg.sender], Errors.VL_BORROWING_CALLER_NOT_IN_WHITELIST);
```

- An admin calls `disableVaultToBorrow()` and passes the the vault id for the above which in turn disables the address from borrowing
- User now tries to repay what they borrowed by calling `repay()`
- step 3 would revert since we are checking if `user(vault)` is allowed to borrow which was disabled

```solidity
function repay(address onBehalfOf, uint256 debtId, uint256 amount)
external
notPaused
nonReentrant
returns (uint256)   {
    require(borrowingWhiteList[msg.sender], Errors.VL_BORROWING_CALLER_NOT_IN_WHITELIST);
```

This makes it impossible to repay the borrowed tokens

### 4.6. <a name='Recommendation-1'></a>Recommendation

When repaying borrowed tokens, we should not enforce the `borrowingWhiteList`

```diff
function repay(address onBehalfOf, uint256 debtId, uint256 amount)
external
notPaused
nonReentrant
returns (uint256)    {
- require(borrowingWhiteList[msg.sender], Errors.VL_BORROWING_CALLER_NOT_IN_WHITELIST);
DataTypes.DebtPositionData storage debtPosition = debtPositions[debtId];
require(msg.sender == debtPosition.owner, Errors.VL_INVALID_DEBT_OWNER);
```

## 5. <a name='INFORMATIONAL:PotentialOvercreditinginrepayDuetoMisorderedLogic'></a>INFORMATIONAL: Potential Overcrediting in repay() Due to Misordered Logic

### 5.1. <a name='Summary-1'></a>Summary

In `repay()`, user(vault) credit is incorrectly updated using the input repayment amount instead of the actual debt repaid. This can result in users receiving excessive credit if the repayment amount exceeds the debt.

### 5.2. <a name='FindingDescription-1'></a>Finding Description

```solidity
// Credits is the borrowing power of vaults
// Each vault should own a specific credits so as to borrow tokens from the pool
// Only the contract that have enough credits can borrow from lending pool
// Only owners can set this map to grant new credits to a contract
// Credits, mapping(reserveId => mapping(contract_address => credits))
mapping(uint256 => mapping(address => uint256)) public credits;
```

Whenever `repay()` is called, credit is being increased(added) which means more borrowing power for the vault.

see: https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/lendingpool/LendingPool.sol?lines=341,351

```solidity
uint256 credit = credits[debtPosition.reserveId][_msgSender()];credits[debtPosition.reserveId][_msgSender()] = credit.add(amount);
if (amount > debtPosition.borrowed) {
    amount = debtPosition.borrowed;
}
reserve.totalBorrows = reserve.totalBorrows.sub(amount);debtPosition.borrowed = debtPosition.borrowed.sub(amount);
```

The problem with this approach is , the amount used to update the user credit is not the actual borrowed amount. This means that if a user (or vault) repays more than the outstanding debt, they will receive more credit than they should. This is because, we only cap the amount after updating credit, meaning that, if we pass a big amount in relation to the actual borrowed amount, credits will be increased by that big amount.

```solidity
if (amount > debtPosition.borrowed) {    amount = debtPosition.borrowed;}
```

### 5.3. <a name='ImpactExplanation-1'></a>Impact Explanation

This issues allows the vault to gain excess credit(borrowing power) by inflating the amount to be repaid. Note, passing a big amount has no effect to the caller as this is later capped to only capture the actual borrowed amount.

### 5.4. <a name='LikelihoodExplanation-1'></a>Likelihood Explanation

Every time amount is greater than actual borrowed amount, then credit will be flawed, allowing the vault to have higher borrowing power.

### 5.5. <a name='ProofofConcept-1'></a>Proof of Concept

- Call `repay()` with higher amount than actual borrowed amount
- credit value is increased by the amount passed instead of the actual repaid amount

### 5.6. <a name='Recommendation-1'></a>Recommendation

Refactor the logic so that the actual amount repaid is calculated before updating the credit

```diff
function repay(address onBehalfOf, uint256 debtId, uint256 amount)
external
notPaused
nonReentrant
returns (uint256)
{
    require(borrowingWhiteList[msg.sender], Errors.VL_BORROWING_CALLER_NOT_IN_WHITELIST);
    DataTypes.DebtPositionData storage debtPosition = debtPositions[debtId];
    require(_msgSender() == debtPosition.owner, Errors.VL_INVALID_DEBT_OWNER);
    DataTypes.ReserveData storage reserve = getReserve(debtPosition.reserveId);
    // update states
    reserve.updateState(getTreasury());
    updateDebtPosition(debtPosition, reserve.borrowingIndex);
    - uint256 credit = credits[debtPosition.reserveId][_msgSender()];
    - credits[debtPosition.reserveId][_msgSender()] = credit.add(amount);
    if (amount > debtPosition.borrowed) {
        amount = debtPosition.borrowed;
    }
    reserve.totalBorrows = reserve.totalBorrows.sub(amount);
    debtPosition.borrowed = debtPosition.borrowed.sub(amount);
    + uint256 credit = credits[debtPosition.reserveId][_msgSender()];
    + credits[debtPosition.reserveId][_msgSender()] = credit.add(amount);
```
