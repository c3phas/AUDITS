## Findings

## MEDIUM: Using Stale price in pyth network

Using the method `getPriceUnsafe()` from `pyth` might return a price from arbitrary far in the past which might lead to wrong price calculation

### Root Cause

In `pythOracle.sol` the function `_getLatestPrice()` uses the function `getPriceUnsafe()` from pyth
This method returns the price object containing last updated price for the requested price feed ID.

```solidity
    function _getLatestPrice(address token) internal view returns (uint256, uint256) {
        // Return 0 if price feed id is not set, reverts are handled by caller
        if (priceFeedIds[token] == bytes32(0)) return (0, 0);


        bytes32 priceFeedId = priceFeedIds[token];
        PythStructs.Price memory pythPrice = pyth.getPriceUnsafe(priceFeedId);


        uint256 price = uint256(uint64(pythPrice.price));
        uint256 expo = uint256(uint32(-pythPrice.expo));


        return (price, expo);
    }
```

According to the [pyth docs](https://api-reference.pyth.network/price-feeds/evm/getPriceUnsafe) It is the caller's responsibility to check the returned `publishTime` to ensure that the update is recent enough for their use case

### Impact

This function may return a price from arbitrarily in the past leading to wrong calculations

### Mitigation

If you need the latest price, update the price using [updatePriceFeeds()](https://api-reference.pyth.network/price-feeds/evm/updatePriceFeeds) and then call [getPrice()](https://api-reference.pyth.network/price-feeds/evm/getPrice).
