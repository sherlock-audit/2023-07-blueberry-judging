Magnificent Mercurial Swift

medium

# AuraSpell#closePositionFarm will take reward fees on underlying tokens when borrow token is also a reward
## Summary

AuraSpell#AuraSpell#closePositionFarm will take reward fees on underlying tokens when borrow token is also a reward. This is because the BLP is burned before taking the reward cut.

## Vulnerability Detail

[AuraSpell.sol#L227-L247](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L227-L247)

                wAuraPools.getVault(lpToken).exitPool(
                    IBalancerPool(lpToken).getPoolId(),
                    address(this),
                    address(this),
                    IBalancerVault.ExitPoolRequest(
                        tokens,
                        minAmountsOut,
                        abi.encode(0, amountPosRemove, borrowTokenIndex),
                        false
                    )
                );
            }
        }

        /// 4. Swap each reward token for the debt token
        uint256 rewardTokensLength = rewardTokens.length;
        for (uint256 i; i != rewardTokensLength; ) {
            address sellToken = rewardTokens[i];
            if (sellToken == STASH_AURA) sellToken = AURA;

            _doCutRewardsFee(sellToken);

We can see above that closePositionFarm redeems the BLP before it takes the reward cut. This can cause serious issues. If there is any overlap between the reward tokens and the borrow token then _doCutRewardsFee will take a cut of the underlying liquidity. This causes loss to the user as too many fees are taken from them.

## Impact

User will lose funds due to incorrect fees

## Code Snippet

[AuraSpell.sol#L184-L265](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L184-L265)

## Tool used

Manual Review

## Recommendation

Use the same order as ConvexSpell and sell rewards BEFORE burning BLP