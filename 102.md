Magnificent Mercurial Swift

medium

# AuraSpell#closePositionFarm exits pool with single token and without any slippage protection
## Summary

When exiting the balancer pool, vault#exitPool is called with an empty array for minAmountsOut causing the position to be exited with no slippage protection. Typically it is not an issue to exit off axis but since it is exiting to a single token this can cause massive loss.

## Vulnerability Detail

[AuraSpell.sol#L221-L236](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L221-L236)

                (
                    uint256[] memory minAmountsOut,
                    address[] memory tokens,
                    uint256 borrowTokenIndex
                ) = _getExitPoolParams(param.borrowToken, lpToken);

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

When exiting a the balancer vault, closePositionFarm makes a subcall to _getExitPoolParams which is used to set minAmountsOut.

[AuraSpell.sol#L358-L361](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L358-L361)

        (address[] memory tokens, , ) = wAuraPools.getPoolTokens(lpToken);

        uint256 length = tokens.length;
        uint256[] memory minAmountsOut = new uint256[](length);

Inside _getExitPoolParams we see that minAmountsOut are always an empty array. This means that the user has no slippage protection and can be sandwich attacked, suffering massive losses.

## Impact

Exits can be sandwich attacked causing massive loss to the user

## Code Snippet

[AuraSpell.sol#L184-L286](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L184-L286)

## Tool used

Manual Review

## Recommendation

Allow user to specify min amount received from exit