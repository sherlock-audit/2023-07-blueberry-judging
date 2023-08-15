Sunny Iris Elephant

medium

# getIsolatedCollateralValue() is not using the up-to-date exchange rate
## Summary

BlueBerryBank.sol:: getIsolatedCollateralValue() is using a wrong exchange rate which doesnt accrue interest before the value is returned so it will return not the most recent value. Because of that a user can get liquidated even if his position is still healthy.


## Vulnerability Detail

getIsolatedCollateralValue() is used in isLiquidatable() when calculating if a given position can be liquidated based on its risk ratio. This function computes the USD value of the isolated collateral for a given position.  

However the problem is that interest is not accrued before the Compounds exchange rate is returned. The correct function to call is exchangeRateCurrent() which first accrues interest and then returns the stored exchange rate. 

getIsolatedCollateralValue() is not a view function so exchangeRateCurrent() can and should be used here. 

## Impact

The user will be unable to call execute() where isLiquidatable() is used but more importantly he could get liquidated even if his position is still healthy


## Code Snippet

https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/BlueBerryBank.sol#L482-L493

```solidity
482: /// @notice Ensure to call `accrue` beforehand to get the most recent value.
483: /// @param positionId ID of the position to compute the isolated collateral value for.
484: /// @return icollValue USD value of the isolated collateral.
485: function getIsolatedCollateralValue(
486:     uint256 positionId
487: ) public override returns (uint256 icollValue) {
488:     Position memory pos = positions[positionId];
489:     /// NOTE: exchangeRateStored has 18 decimals.
490:     uint256 underlyingAmount;
491:     if (_isSoftVault(pos.underlyingToken)) {
492:         underlyingAmount =
493:             (ICErc20(banks[pos.debtToken].bToken).exchangeRateStored() *


```

As you can see exchangeRateStored() is used and the comment above says that accrue should be called but exchangeRateStored() does not accrue interest. This is not a view function so exchangeRateCurrent() should be used.


## Tool used

Manual Review

## Recommendation

Use exchangeRateCurrent() so that interest is accrued before the exchange rate is returned.
