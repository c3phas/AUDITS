## Findings

## MEDIUM: Call() should be used instead of transfer() on an address payable

https://github.com/code-423n4/2022-07-ens/blob/ff6e59b9415d0ead7daf31c2ed06e86d9061ae22/contracts/ethregistrar/ETHRegistrarController.sol#L182-L186
https://github.com/code-423n4/2022-07-ens/blob/ff6e59b9415d0ead7daf31c2ed06e86d9061ae22/contracts/ethregistrar/ETHRegistrarController.sol#L204
https://github.com/code-423n4/2022-07-ens/blob/ff6e59b9415d0ead7daf31c2ed06e86d9061ae22/contracts/ethregistrar/ETHRegistrarController.sol#L211

### Vulnerability details

The use of the deprecated `transfer()` function for an address will inevitably make the transaction fail when :

```
    The withdrawer smart contract does not implement a payable fallback function.
    The withdrawer smart contract implements a payable fallback function which uses more than 2300 gas units.
    The withdrawer smart contract implements a payable fallback function which needs less than 2300 gas units but is called through a proxy that raises the call’s gas usage above 2300.
```

### Proof of Concept

```solidity
File: ETHRegistrarController.sol line 182-186

        if (msg.value > (price.base + price.premium)) {
            payable(msg.sender).transfer(
                msg.value - (price.base + price.premium)
            );
        }

File: ETHRegistrarController.sol line 204

            payable(msg.sender).transfer(msg.value - price.base);

File: ETHRegistrarController.sol line 211

        payable(owner()).transfer(address(this).balance);
```

### Recommended Mitigation Steps

`Call()` should be used instead of `transfer()` on an address payable
Additionally, note that the sendValue function available in OpenZeppelin Contract’s Address library can be used to transfer the withdrawn Ether without being limited to 2300 gas units. Risks of reentrancy stemming from the use of this function can be mitigated by tightly following the “Check-effects-interactions” pattern and using OpenZeppelin Contract’s ReentrancyGuard contract. For further reference on why using Solidity’s transfer is no longer recommended, refer to these articles:

- [Stop using Solidity’s transfer now](https://diligence.consensys.net/blog/2019/09/stop-using-soliditys-transfer-now/)
- [Reentrancy after Istanbul](https://blog.openzeppelin.com/reentrancy-after-istanbul/)
