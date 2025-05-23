## Findings

## MEDIUM: call() should be used instead of transfer() on address payable

https://github.com/code-423n4/2022-07-fractional/blob/e2c5a962a94106f9495eb96769d7f60f7d5b14c9/src/modules/Migration.sol#L172
https://github.com/code-423n4/2022-07-fractional/blob/e2c5a962a94106f9495eb96769d7f60f7d5b14c9/src/modules/Migration.sol#L325

### Vulnerability details

`call()` should be used instead of `transfer()` on address payable

The use of the deprecated `transfer()` function for an address wll make the transaction fail when

```
    The withdrawer smart contract does not implement a payable function.
    The withdrawer smart contract implements a payable fallback function which uses more than 2300 gas units.
    The withdrawer smart contract implements a payable fallback function which needs less than 2300 gas units but is called through a proxy that raises the call’s gas usage above 2300.
```

To prevent unexpected behavior and potential loss of funds, consider explicitly warning end-users about the mentioned shortcomings to raise awareness before they deposit Ether into the protocol. Additionally, note that the sendValue function available in OpenZeppelin Contract’s Address library can be used to transfer the withdrawn Ether without being limited to 2300 gas units. Risks of reentrancy stemming from the use of this function can be mitigated by tightly following the “Check-effects-interactions” pattern and using OpenZeppelin Contract’s ReentrancyGuard contract.

### Affected code

```solidity
File: Migration.sol line 172

        payable(msg.sender).transfer(ethAmount);

File: Migration.sol line 325

        payable(msg.sender).transfer(userEth);
```

### Recommendation

Use solidity's low level call() function or the sendValue function from openzeppelin
