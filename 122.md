Icy Shamrock Bobcat

medium

# Potential for user positions to fall below `minPositionSize` when partially closing or reducing positions
## Summary

`_validatePosSize` didn't check on withdrawal or closePositionFarm when user decrease or partial close their position can make user's position under `minPositionSize`

## Vulnerability Detail

This current Blueberry update introduce a `minPositionSize` on its Strategy, (before it only have `maxPositionSize`).

```js
File: BasicSpell.sol
39:     /// @dev Defines strategies for Blueberry Protocol.
40:     /// @param vault Address of the vault where assets are held.
41:     /// @param minPositionSize Minimum size of the position in USD.
42:     /// @param maxPositionSize Maximum size of the position in USD.
43:     struct Strategy {
44:         address vault;
45:         uint256 minPositionSize;
46:         uint256 maxPositionSize;
47:     }
```

In current codebase, the function `_validatePosSize` only invoked during the processes of `deposit` or `openPositionFarm`. However, with the addition of the `minPositionSize`, the verification for position size needs to be extended to include the `withdrawal` process or `closePositionFarm` operation as well. This ensures that the position size remains within acceptable boundaries, avoiding falling below the minimum threshold.

```js
File: BasicSpell.sol
273:     function _validatePosSize(uint256 strategyId) internal {
...
295:         // Check if position size is within bounds
296:         if (prevPosSize + addedPosSize > strategy.maxPositionSize)
297:             revert Errors.EXCEED_MAX_POS_SIZE(strategyId);
298:         if (prevPosSize + addedPosSize < strategy.minPositionSize)
299:             revert Errors.EXCEED_MIN_POS_SIZE(strategyId);
300:     }
```

If the user decrease their position or partial close, they can be in position where their position is under the `minPositionSize` (either it was purposely or unknowingly).

The Blueberry protocol introduce the minPositionSize for any reasons. By imposing a minimum position size, the protocol might aims to balance between resource utilization, risk mitigation, and user engagement. The update extends the position size validation to withdrawal and position closure processes to ensure that positions remain within acceptable boundaries, preventing them from becoming too small to be meaningful within the protocol's operations.

## Impact

User's position might be under minimum size position when they decrease their position or partial `closePositionFarm` which is not expected by the protocol

## Code Snippet

https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/BasicSpell.sol#L298-L299

## Tool used

Manual Review

## Recommendation

`minPositionSize` should be checked on `closePositionFarm` or withdrawal when decreasing position or partial closing by calling `_validatePosSize` 
