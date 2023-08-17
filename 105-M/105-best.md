Magnificent Mercurial Swift

medium

# ConvexSpell is completely broken for any curve LP that utilizes native ETH
## Summary

When a Curve pool utilizes native ETH it uses the address `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee`. This is problematic because it will try to call balanceOf on this address which will always revert.

## Vulnerability Detail

[ConvexSpell.sol#L120-L127](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/ConvexSpell.sol#L120-L127)

        if (tokens.length == 2) {
            uint256[2] memory suppliedAmts;
            for (uint256 i; i != 2; ++i) {
                suppliedAmts[i] = IERC20Upgradeable(tokens[i]).balanceOf(
                    address(this)
                );
            }
            ICurvePool(pool).add_liquidity(suppliedAmts, minLPMint);

ConvexSpell#openPositionFarm attempts to call balanceOf on each component of the LP. Since native ETH uses the `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` this call will always revert. This breaks compatibility with EVERY curve pool that uses native ETH which make most of the highest volume pools on the platfrom.

## Impact

ConvexSpell is completely incompatible with a majority of Curve pools

## Code Snippet

[ConvexSpell.sol#L92-L173](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/ConvexSpell.sol#L92-L173)

## Tool used

Manual Review

## Recommendation

I would recommend conversion between native ETH and wETH to prevent this issue.