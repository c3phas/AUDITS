## Findings

## HIGH: Calling refund does not update the array of contributors

When refund is called, the array of contributors should be updated to remove the user who called `refund()`

### Finding Description

When users make a contribution, they are added to the array `address[] public contributors;` as show below https://github.com/daaoai/daaoai_contracts/blob/99e1743582cc01439b8f1140ca929f57afeacd5d/src/Daao.sol#L174-L176

```solidity
        if (contributions[msg.sender] == 0) {
            contributors.push(msg.sender);
        }
```

This list is mainly used when we finalize a fundraising as we loop on it

https://github.com/daaoai/daaoai_contracts/blob/99e1743582cc01439b8f1140ca929f57afeacd5d/src/Daao.sol#L295-L304

```solidity
        for (uint256 i = 0; i < contributors.length; i++) {
            address contributor = contributors[i];
            uint256 contribution = contributions[contributor];
            uint256 tokensToMint = (contribution * SUPPLY_TO_FUNDRAISERS) /
                totalRaised;

            emit MintDetails(contributor, tokensToMint);

            token.mint(contributor, tokensToMint);
        }
```

As there is no specific way to remove users from this array, if the array grows too much, this might lead to DOS trying to loop and minting token, as such we should be careful with how big the array gets Note: Lightchaser found the DOS linked to this array but this is not what this finding is all about

The problem arises when a goal is not reached and the fundraising deadline has reached, this allows users to call function `refund()` to get their contribution back. This should help reduce the contributors array because we don't need to have the user in the array if they have no contribution, so once a user calls refund, we should remove them from the array. However, the current refund function does not do this.

If owner or protocol admin decides to update the `fundraisingDeadline` allowing contributions to continue, when we eventually hit the target, calling finalize would have interesting data This is because , `contributors.length` would still contain info for users who called `refund()`. To get the amount contributed, `finalizeFundraising()` does the following in the loop

```solidity
address contributor = contributors[i];
uint256 contribution = contributions[contributor];
```

However for users who called `refund()` earlier, their contribution is currently set to zero contributions `[msg.sender] = 0;`

This means we would repeatedly call `token.mint()` with `tokensToMint=0` This wastes too much gas and could even lead to a denial of service.

### Impact Explanation

As users are mostly whitelisted, the chances of this growing too big is small, therefore I think low impact is accurate.

### Likelihood Explanation

Whenever `refund()` is called, the value of contributors would always return wrong length

### Recommendation

When users call `refund()`, they should be removed from the list of contributors. This would prevent looping on too many items and also avoids us from making unnecessary external calls ie `token.mint()`

## HIGH: Lack of state updates when calling refund()

### Summary

Incorrect state updates when calling function `refund()` makes it impossible to hit the fundraising goal. When calling `refund` only the users mapping is being updated while `totalRaised` is not being updated.

### Finding Description

If the fundraising goal is not reached and deadline has reached, contributors are allowed to call `refund` function to get their contributions back. This resets their contributions to 0 in `contributions[msg.sender] = 0`; and transfers the funds to the user see https://github.com/daaoai/daaoai_contracts/blob/99e1743582cc01439b8f1140ca929f57afeacd5d/src/Daao.sol#L403-L417

```solidity
    function refund() external nonReentrant {
        require(!goalReached, "Fundraising goal was reached");
        require(
            block.timestamp > fundraisingDeadline,
            "Deadline not reached yet"
        );
        require(contributions[msg.sender] > 0, "No contributions to refund");

        uint256 contributedAmount = contributions[msg.sender];
        contributions[msg.sender] = 0;

        payable(msg.sender).transfer(contributedAmount);

        emit Refund(msg.sender, contributedAmount);
    }
```

When contributing, users call function `contribute` which updates some few state variables if one succeeds in making a contribution, of interest to us is `contributions[msg.sender] += effectiveContribution;` and `totalRaised`

https://github.com/daaoai/daaoai_contracts/blob/99e1743582cc01439b8f1140ca929f57afeacd5d/src/Daao.sol#L178-L179

```solidity
        contributions[msg.sender] += effectiveContribution;
        totalRaised += effectiveContribution;
```

When refunding , ideally, we want to do the opposite of this which means reduce the users contributions from the mapping contributions and also reduce the `totalRaised` amount with the user's contributions(value they are being refunded) as this no longer counts towards the totalRaised

The problem arises in that the function `refund()` only refunds the user but does not update the state variable `totalRaised` which would still reflect the old variable

### Impact Explanation

Wrong state updates will lead to incorrect calculations , especially in this case where we get a false sense of the fundraising goal being achieved

### Likelihood Explanation

The likelihood is high as this would happen every time the goal was not reached and a user calls `refund()`

### Proof of Concept

Suppose we have a goal of 50K, user A contributes 20K, user B contributes 10K, totalRaised=30K, which is 20 shy of the goal. The contribution deadline passes and the goal was not reached yet as we only have 30K. Now user B calls refund after we failed to hit the goal , user B gets his 10K back and `contributions[USER B] = 0`. now, only user A is left but `totalRaised` still = 30K. as this was not updated in `refund()`

Suppose the protocol admin decides to extend the deadline which allows contributions to continue by calling function `extendFundraisingDeadline()`. If contributions resume, our state variable `totalRaised` is still at 30K, meaning once another user contributes say 20K, we would have 50K which equals the goal. we can only raise 20K before the goal is reached , but in real sense we need 30K

### Recommendation

To mitigate this, just update the state variable `totalRaised` when `refund()` is called

```diff
    function refund() external nonReentrant {
        require(!goalReached, "Fundraising goal was reached");
        require(
            block.timestamp > fundraisingDeadline,
            "Deadline not reached yet"
        );
        require(contributions[msg.sender] > 0, "No contributions to refund");
        uint256 contributedAmount = contributions[msg.sender];
        contributions[msg.sender] = 0;
     + totalRaised -= contributedAmount
        payable(msg.sender).transfer(contributedAmount);
        emit Refund(msg.sender, contributedAmount);
    }
```
