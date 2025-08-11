## Findings Summary

## No way to refund users if they pay excess when buying curves tokens

### Lines of code

https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L263-L279

### Impact

Excess ether sent by the user when buying curves token is lost

### Proof of concept

https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L263-L279

File: /contracts/Curves.sol

```solidity
    function _buyCurvesToken(address curvesTokenSubject, uint256 amount) internal {
        uint256 supply = curvesTokenSupply[curvesTokenSubject];
        if (!(supply > 0 || curvesTokenSubject == msg.sender)) revert UnauthorizedCurvesTokenSubject();


        uint256 price = getPrice(supply, amount);
        (, , , , uint256 totalFee) = getFees(price);


        if (msg.value < price + totalFee) revert InsufficientPayment();


        curvesTokenBalance[curvesTokenSubject][msg.sender] += amount;
        curvesTokenSupply[curvesTokenSubject] = supply + amount;
        _transferFees(curvesTokenSubject, true, price, amount, supply);


        // If is the first token bought, add to the list of owned tokens
        if (curvesTokenBalance[curvesTokenSubject][msg.sender] - amount == 0) {
            _addOwnedCurvesTokenSubject(msg.sender, curvesTokenSubject);
        }
```

When buying curves token, we have a check for the amount being sent `if (msg.value < price + totalFee) revert InsufficientPayment();`
This ensures that the buyer always sends enough amount to handle the payment.
If a user sends a greater amount than required, there is no way for them to get refunded the extra amount

### Recommendation

Track the amount the user pays and have a way to refund the excess amount

## Medium

### When tokens are sold, we should remove them from the list of owned tokens

### Impact

The mapping ownedCurvesTokenSubjects will contain false data once we sell our curve tokens

### Lines of code

https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L282-L293

```solidity
File: /contracts/Curves.sol
    function sellCurvesToken(address curvesTokenSubject, uint256 amount) public {
        uint256 supply = curvesTokenSupply[curvesTokenSubject];
        if (supply <= amount) revert LastTokenCannotBeSold();
        if (curvesTokenBalance[curvesTokenSubject][msg.sender] < amount) revert InsufficientBalance();


        uint256 price = getPrice(supply - amount, amount);


        curvesTokenBalance[curvesTokenSubject][msg.sender] -= amount;
        curvesTokenSupply[curvesTokenSubject] = supply - amount;


        _transferFees(curvesTokenSubject, false, price, amount, supply);
    }
```

When we call `_buyCurvesToken()` or `_transfer()` we always make sure we add the tokens to this list ie for `_buyCurvesToken` we add it if it's the first token bought and for `_transfer()` we add it to the list unless we are transferring from oneself
This ensures we keep track of tokens owned by the user using the mapping `ownedCurvesTokenSubjects`

However, when a user sells their curve tokens, the list of `ownedCurvesTokenSubjects` is not updated.

Similary the function `_transfer()` only seems to handle one side ie we update the list of the user receiving the tokens but we do not update the list of the person transfering the tokens

If we sell our tokens then attempt to call `transferAllCurvesTokens()` would give us some false results.

```solidity
    function transferAllCurvesTokens(address to) external {
        if (to == address(this)) revert ContractCannotReceiveTransfer();
        address[] storage subjects = ownedCurvesTokenSubjects[msg.sender];
        for (uint256 i = 0; i < subjects.length; i++) {
            uint256 amount = curvesTokenBalance[subjects[i]][msg.sender];
            if (amount > 0) {
                _transfer(subjects[i], msg.sender, to, amount);
            }
        }
    }
```

Note, we are reading `ownedCurvesTokenSubjects[msg.sender]` and storing the results in an array `subjects`.

### Recommendation

We should update the list by removing the tokens whenever we sell the tokens
