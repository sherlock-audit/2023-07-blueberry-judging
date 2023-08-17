Sunny Iris Elephant

medium

# Users will fail to close their Convex position if the Curve pool is killed
## Summary

If the Curve pool that the ConvexSpell.sol uses becomes killed, users will be unable to close their position because remove_liquidity_one_coin() will revert. Users will be unable to repay their debt so their assets will be liquidated.

## Vulnerability Detail

ConvexSpell.sol::closePositionFarm() is used to close an existing liquidity position. After the collateral is taken and the rewards are swapped, _removeLiquidity() is called which removes liquidity from a Curve pool by calling remove_liquidity_one_coin(). 


The problem arises when the Curve pool is killed(paused) so if self.is_killed in the curve pool contract is true, calling remove_liquidity_one_coin() function will always revert and closePositionFarm() function will be DOS'ed


Note: This issue was submitted in the [previous contest](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/64) however only the CurveSpell got fixed but not the ConvexSpell.


## Impact

When user's position is about to be liquidated, if the closePositionFarm() function is DOS'ed,users will be unable to repay their debt, resulting in the users losing their funds

## Code Snippet

https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/spell/ConvexSpell.sol#L261-L265

```solidity

260: /// Removes liquidity from the Curve pool for the specified token.
261: ICurvePool(pool).remove_liquidity_one_coin(
262:     amountPosRemove,
263:     int128(tokenIndex),
264:     param.amountOutMin
265: );

```

As you can see, remove_liquidity_one_coin() is called when the user calls closePositionFarm() in the ConvexSpell


https://github.com/curvefi/curve-contract/blob/b0bbf77f8f93c9c5f4e415bce9cd71f0cdee960e/contracts/pools/3pool/StableSwap3Pool.vy#L670-L674

```python
670: def remove_liquidity_one_coin(_token_amount: uint256, i: int128, min_amount: uint256):
671:     """
672:     Remove _amount of liquidity all in a form of coin i
673:     """
674:     assert not self.is_killed  # dev: is killed
```


The problem is that remove_liquidity_one_coin() checks if self.is_killed is true so if the Curve pool is killed then this will revert and the user wont be able to close his position.


## Tool used

Manual Review

## Recommendation

When killed, it is only possible for existing LPs to remove liquidity via remove_liquidity so there should be the same isKilled check in the ConvexSpell like there is in the CurveSpell
