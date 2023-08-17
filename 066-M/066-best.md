Spare Syrup Dragonfly

medium

# WAuraPools/WConvexPools.burn may revert if one of extraRewardsTokens does not support 0 value transfer
## Summary

Both WAuraPools/WConvexPools have a state variable called [[extraRewards](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L58)](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L58), which is an array that stores the addresses of all extra rewards contracts. In [[mint](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L339-L351)](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L339-L351)/[[burn](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L386-L395)](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L386-L395), [[_syncExtraReward](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L435-L440)](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L435-L440) will be called to synchronize this array. **However, this array will not delete item**. In this way, when the old extra rewards contract stops distributing rewards, that is to say, the value returned by `IAuraRewarder(rewarder).rewardPerToken()` will no longer increase. This will cause the reward of the extra token to be 0. The extra reward token cannot be controlled by blueberry. If one of these tokens does not support zero-value transfers, then `burn` will revert.

## Vulnerability Detail

This issue is described below in WAuraPools. The `extraRewards` array is shared by the WAuraPools contract which can support multiple pools. A pool corresponds to an `auraRewarder`, which can have up to [[12](https://github.com/aurafinance/convex-platform/blob/816cbfd551a80bb4768f9168144dadbd3e35bd13/contracts/contracts/BaseRewardPool.sol#L130)](https://github.com/aurafinance/convex-platform/blob/816cbfd551a80bb4768f9168144dadbd3e35bd13/contracts/contracts/BaseRewardPool.sol#L130) extra reward tokens. `extraRewards` does not pop item, so the old extra reward will not be removed. Even if an extra reward no longer distributes rewards, it still exists in `extraRewards`. For new user, this will inevitably cause the `rewardPerShare` recorded at `mint` to be the same as the `rewardPerShare` obtained at `burn`.

```solidity
File: blueberry-core\contracts\wrapper\WAuraPools.sol
//@audit `rewardPerShare` recorded at `mint`
339:         for (uint256 i; i != extraRewardsCount; ) {
340:             address extraRewarder = IAuraRewarder(auraRewarder).extraRewards(i);
341:->           uint256 rewardPerToken = IAuraRewarder(extraRewarder)
342:                 .rewardPerToken();
343:->           accExtPerShare[id][extraRewarder] = rewardPerToken == 0
344:                 ? type(uint).max
345:                 : rewardPerToken;
346: 
347:             _syncExtraReward(extraRewarder);
348: 
349:             unchecked {
350:                 ++i;
351:             }
352:         }

//@audit `rewardPerShare` recorded at `burn`
File: blueberry-core\contracts\wrapper\WAuraPools.sol
289:         for (uint256 i; i != extraRewardsCount; ) {
290:             address rewarder = extraRewards[i];
             //@audit `rewardPerShare` recorded at `mint`
291:->           uint256 stRewardPerShare = accExtPerShare[tokenId][rewarder];
292:             tokens[i + 2] = IAuraRewarder(rewarder).rewardToken();
293:             if (stRewardPerShare == 0) {
294:                 rewards[i + 2] = 0;
295:             } else {
                     //@audit this function will retrieve current reward per token from rewarder
296:                 rewards[i + 2] = _getPendingReward(
297:->                   stRewardPerShare == type(uint).max ? 0 : stRewardPerShare,
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
```

If one of these extra reward tokens does not support zero value transfers, `burn` will revert at the following code.

```solidity
File: blueberry-core\contracts\wrapper\WAuraPools.sol
360:     function burn(
361:         uint256 id,
362:         uint256 amount
363:     )
......
399:         /// Transfer Reward Tokens
400:         (rewardTokens, rewards) = pendingRewards(id, amount);
......
413:         uint256 rewardTokensLength = rewardTokens.length;
414:         for (uint256 i; i != rewardTokensLength; ) {
415:             IERC20Upgradeable(rewardTokens[i]).safeTransfer(
416:                 msg.sender,
417:->               rewards[i]
418:             );
419: 
420:             unchecked {
421:                 ++i;
422:             }
423:         }
424:     }
```

## Impact

if one of extraRewardsTokens does not support 0 value transfer, `WAuraPools/WConvexPools.burn` will revert. This will result in users being unable to close position or add position on existing position. Users suffer fund loss.

## Code Snippet

https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L415-L418

https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WConvexPools.sol#L329-L332

## Tool used

Manual Review

## Recommendation

```fix
--- a/blueberry-core/contracts/wrapper/WAuraPools.sol
+++ b/blueberry-core/contracts/wrapper/WAuraPools.sol
@@ -412,10 +412,12 @@ contract WAuraPools is

         uint256 rewardTokensLength = rewardTokens.length;
         for (uint256 i; i != rewardTokensLength; ) {
-            IERC20Upgradeable(rewardTokens[i]).safeTransfer(
-                msg.sender,
-                rewards[i]
-            );
+            if (rewards[i] > 0) {
+                IERC20Upgradeable(rewardTokens[i]).safeTransfer(
+                    msg.sender,
+                    rewards[i]
+                );
+            }

             unchecked {
                 ++i;

--- a/blueberry-core/contracts/wrapper/WConvexPools.sol
+++ b/blueberry-core/contracts/wrapper/WConvexPools.sol
@@ -326,10 +326,12 @@ contract WConvexPools is

         uint rewardLen = rewardTokens.length;
         for (uint i; i < rewardLen; ) {
-            IERC20Upgradeable(rewardTokens[i]).safeTransfer(
-                msg.sender,
-                rewards[i]
-            );
+            if (rewards[i] > 0) {
+                IERC20Upgradeable(rewardTokens[i]).safeTransfer(
+                    msg.sender,
+                    rewards[i]
+                );
+            }

             unchecked {
                 ++i;
```