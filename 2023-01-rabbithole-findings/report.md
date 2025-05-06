## Findings

## HIGH: Anyone can mint a receipt despite efforts of restricting it to onlyMinters

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L58-L61
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleTickets.sol#L47-L50

### Vulnerability details

Anyone can mint a receipt despite efforts of restricting it to `onlyMinters`.
The `onlyMinter` modifer does not do anything to restrict calls to `onlyMinters` which would allow anyone to call the function mint.
The require statement was ommited on the modifier and just does a comparison with no side effects regardless of the comparison results

### Proof of Concept

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L58-L61

```solidity
58:    modifier onlyMinter() {
59:        msg.sender == minterAddress;
60:        _;
61:    }
```

The above modifier is being used on the function mint Line 98 to restrict who can mint receipts.

```solidity
98:    function mint(address to_, string memory questId_) public onlyMinter {
99:        _tokenIds.increment();
100:        uint newTokenID = _tokenIds.current();
101:        questIdForTokenId[newTokenID] = questId_;
102:        timestampForTokenId[newTokenID] = block.timestamp;
103:        _safeMint(to_, newTokenID);
104:    }
```

A closer look at the modifer condition `msg.sender == minterAddress` we see it's missing the require statement , which makes the whole thing useless. The check would simply return true or false and the function would still be executed regardless of who called it.

A similar case on https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleTickets.sol#L47-L50

This would allow anyone to call the function `mint` and `mintBatch`

### Recommended Mitigation Steps

The modifier onlyMinter should be modified as below

diff --git a/contracts/RabbitHoleReceipt.sol b/contracts/RabbitHoleReceipt.sol
index 085b617..2d37f6c 100644

```diff
--- a/contracts/RabbitHoleReceipt.sol
+++ b/contracts/RabbitHoleReceipt.sol
@@ -56,7 +56,7 @@ contract RabbitHoleReceipt is
     }

     modifier onlyMinter() {
-        msg.sender == minterAddress;
+        require(msg.sender == minterAddress);
         _;
     }
```
