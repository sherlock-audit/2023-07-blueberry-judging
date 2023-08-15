Spare Syrup Dragonfly

medium

# The WAuraPools/WConvexPools.extraRewards array does not delete items, which may cause OOG in the long run
## Summary

Both WAuraPools/WConvexPools have a state variable called [[extraRewards](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L58)](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L58), which is an array that stores the addresses of all extra rewards contracts. In [[mint](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L339-L351)](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L339-L351)/[[burn](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L386-L395)](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L386-L395), [[_syncExtraReward](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L435-L440)](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L435-L440) will be called to synchronize this array. The `extraRewards` array is shared by the contract which can support multiple pools. A pool corresponds to an `auraRewarder`, which can have up to [[12](https://github.com/aurafinance/convex-platform/blob/816cbfd551a80bb4768f9168144dadbd3e35bd13/contracts/contracts/BaseRewardPool.sol#L130)](https://github.com/aurafinance/convex-platform/blob/816cbfd551a80bb4768f9168144dadbd3e35bd13/contracts/contracts/BaseRewardPool.sol#L130) extra reward tokens. **However, this array will not delete item. In this way, it may grow larger and larger over time**. The [[burn](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L360-L363)](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L360-L363) function will traverse `extraRewards` internally, where call [[_getPendingReward](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L296)](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L296) for every rewarder to calculate the amount of rewards. If `extraRewards.length` is big enough, tx will revert due to out of gas.

## Vulnerability Detail

When the user opens a position in AuraSpell, `WAuraPools.mint` will be called. If a new `extraRewarder` appears, it will be pushed to the `extraRewards` array.

```solidity
File: blueberry-core\contracts\wrapper\WAuraPools.sol
314:     function mint(
315:         uint256 pid,
316:         uint256 amount
317:     ) external nonReentrant returns (uint256 id) {
......
336:         /// Store extra rewards info
337:         uint256 extraRewardsCount = IAuraRewarder(auraRewarder)
338:             .extraRewardsLength();
339:         for (uint256 i; i != extraRewardsCount; ) {
340:             address extraRewarder = IAuraRewarder(auraRewarder).extraRewards(i);
......
347:->           _syncExtraReward(extraRewarder);
348: 
349:             unchecked {
350:                 ++i;
351:             }
352:         }
353:     }

435:     function _syncExtraReward(address extraReward) private {
436:         if (extraRewardsIdx[extraReward] == 0) {
                 //@audit if extraReward is new one, it will be pushed into array
437:->           extraRewards.push(extraReward);
438:             extraRewardsIdx[extraReward] = extraRewards.length;
439:         }
440:     }
```

By look at the code of WAuraPools/WConvexPools contracts, we can know that the `extraRewarder` array only pushes item and does not pop item. **As time goes by, this array gets bigger and bigger**.

The `burn` function will call `pendingRewards` to get all `rewardTokens` and `rewards`. `pendingRewards` will traverse `extraRewards` to calculate the reward that each rewarder should distribute to the caller.

```solidity
File: blueberry-core\contracts\wrapper\WAuraPools.sol
257:     function pendingRewards(
258:         uint256 tokenId,
259:         uint256 amount
260:     )
......
271:->       uint256 extraRewardsCount = extraRewards.length;
272:         tokens = new address[](extraRewardsCount + 2);
273:         rewards = new uint256[](extraRewardsCount + 2);
......
288:         /// Additional rewards
289:->       for (uint256 i; i != extraRewardsCount; ) {
290:->           address rewarder = extraRewards[i];
291:             uint256 stRewardPerShare = accExtPerShare[tokenId][rewarder];
292:             tokens[i + 2] = IAuraRewarder(rewarder).rewardToken();
293:             if (stRewardPerShare == 0) {
294:                 rewards[i + 2] = 0;
295:             } else {
296:->               rewards[i + 2] = _getPendingReward(
297:                     stRewardPerShare == type(uint).max ? 0 : stRewardPerShare,
298:                     rewarder,
299:                     amount,
300:                     lpDecimals
301:                 );
302:             }
303: 
304:             unchecked {
305:                 ++i;
306:             }
307:         }
308:     }
```

From L271, `extraRewardsCount` is set to `extraRewards.length`.

From L289-306, The `for` loop processes each `extraRewards[i]`, although some `extraRewards[i]` do not belong to the pool specified by the caller.

After the `pendingRewards` function returns, there is another piece of code that traverses the array again.

```solidity
File: blueberry-core\contracts\wrapper\WAuraPools.sol
386:->       uint256 extraRewardsCount = IAuraRewarder(auraRewarder)
387:             .extraRewardsLength();
......
396:         uint256 storedExtraRewardLength = extraRewards.length;
397:->       bool hasDiffExtraRewards = extraRewardsCount != storedExtraRewardLength;
398: 
399:         /// Transfer Reward Tokens
400:         (rewardTokens, rewards) = pendingRewards(id, amount);//loop extraRewards first time
401: 
402:         /// Withdraw manually
403:         if (hasDiffExtraRewards) {
404:->           for (uint256 i; i != storedExtraRewardLength; ) {//loop extraRewards second time
405:                 IAuraExtraRewarder(extraRewards[i]).getReward();
406: 
407:                 unchecked {
408:                     ++i;
409:                 }
410:             }
411:         }
```

L386, `extraRewardsCount` is the `extraRewardsLength` of the pool specified by the current `burn`, and it does not exceed 12.

L397, when `extraRewards` grows over time, it's length will be greater than 12. So `hasDiffExtraRewards` is true.

Therefore, L404-410, the `for` loop will be executed. This is the second traversal of `extraRewards`.

## Impact

If `extraRewards.length` is big enough, WAuraPools/WConvexPools.burn will revert due to out of gas. This will result in users being unable to close or increase existing position.

## Code Snippet

https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L289-L302

https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WConvexPools.sol#L210-L223

## Tool used

Manual Review

## Recommendation

It is recommended to change `address[] public extraRewards` to `mapping(uint256 => addresss[]) public extraRewards`, the purpose is Mapping from token id to extra rewards addresses, which is safe. The reasons are as following:

1.  The token id is different for different pools. In this way, the `extraRewards[tokenId]` array can only be less than 12.
2.  For the same pool, but different mint time, the [[balRewardPerToken](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WConvexPools.sol#L251-L252)](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WConvexPools.sol#L251-L252) will be different, therefore, the token id will also be different. Then, the `extraRewards[tokenId]` array can only be less than 12.