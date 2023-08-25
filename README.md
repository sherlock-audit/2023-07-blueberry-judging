# Issue H-1: CurveTricryptoOracle#getPrice contains math error that causes LP to be priced completely wrong 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/100 

## Found by 
0x52, Kow

CurveTricryptoOracle#getPrice incorrectly divides the LP price by the price of ETH which causes it to return the price of LP in terms of ETH instead of USD

## Vulnerability Detail

[CurveTricryptoOracle.sol#L57-L62](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/CurveTricryptoOracle.sol#L57-L62)

                (lpPrice(
                    virtualPrice,
                    base.getPrice(tokens[1]),
                    ethPrice,
                    base.getPrice(tokens[0])
                ) * 1e18) / ethPrice;

After the LP price has been calculated in USD it is mistakenly divided by the price of ETH causing the contract to return the LP price in terms of ETH rather than USD. This leads to LP that is massively undervalued causing positions which are actually heavily over collateralized to be liquidated.

## Impact

Healthy positions are liquidated due to incorrect LP pricing

## Code Snippet

[CurveTricryptoOracle.sol#L48-L65](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/CurveTricryptoOracle.sol#L48-L65)

## Tool used

Manual Review

## Recommendation

Don't divide the price by the price of ETH



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**0xyPhilic** commented:
> invalid because lpPrice is considered in ETH so dividing by ETH/USD price returns the final result in USD

**Kral01** commented:
> there is a precision value



**Gornutz**

Judges accurately state why the division by ETH/USD is required to return the proper USD value.

**Shogoki**

Closing in regards to other judges and sponsors comments.

**IAm0x52**

This is valid. The code being used here was borrowed from the Sentiment [CurveTriCryptoOracle](https://arbiscan.io/address/0x4e828a117ddc3e4dd919b46c90d4e04678a05504#code), which is specifically meant to return the price in terms of ETH. This oracle is meant to return the valuation in USD which means the division by the price of ETH needs to be dropped.

**Shogoki**

> This is valid. The code being used here was borrowed from the Sentiment [CurveTriCryptoOracle](https://arbiscan.io/address/0x4e828a117ddc3e4dd919b46c90d4e04678a05504#code), which is specifically meant to return the price in terms of ETH. This oracle is meant to return the valuation in USD which means the division by the price of ETH needs to be dropped.

Maybe was a bit quick in closing. Will reopen it and we will take a deeper look at it.

# Issue M-1: Users will fail to close their Convex position if the Curve pool is killed 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/15 

## Found by 
Arz

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



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**0xyPhilic** commented:
> invalid because the case where a Curve pool is killed is handled by withdrawing liquidity using remove_liquidity function

**darkart** commented:
>  If its already been accepted its up to the DEV to decide if he wants to fix it or no



# Issue M-2: getIsolatedCollateralValue() is not using the up-to-date exchange rate 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/16 

## Found by 
Arz, BugBusters, Oxhunter526, RadCet, mert\_eren

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



## Discussion

**Gornutz**

Previous contest item that was discussed thoroughly. See direct link - https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/97#issuecomment-1542716461 

**Shogoki**

> Previous contest item that was discussed thoroughly. See direct link - [sherlock-audit/2023-04-blueberry-judging#97 (comment)](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/97#issuecomment-1542716461)

Included this here as the reasoning in the mentioned item from the previous contest was:

> Since we are using a view function we are unable to use exchangeRateCurrent() we have to use exchangeRateStored()

However as we can see in the code and as the report states:

> This is not a view function so exchangeRateCurrent() should be used.




# Issue M-3: `getPrice` in `WeightedBPTOracle.sol`  uses `totalSupply` for price calculations which can lead to wrong results 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/18 

## Found by 
Vagner
`getPrice` is used to calculate the price in USD of a given balancer LP, and it respects the recommendations  of Balancer docs by calculating the invariant and using it to protect from manipulations, but it uses `totalSupply` for every LP calculated which can lead to wrong results and assumptions.
## Vulnerability Detail
In the Balancer docs it is specified that `There are three potential functions to query when determining the BPT supply depending on pool type.` https://docs.balancer.fi/concepts/advanced/valuing-bpt/valuing-bpt.html#getting-bpt-supply
- `getActualSupply` : which is used by the most recent Weighted and Stable Pools and it accounts for pre-minted BPT as well as due protocol fees:
- `getVirtualSupply` : which is used by Linear Pools and "legacy" Stable Phantom Pools and it accounts just for pre-minted BPT
- `totalSupply` : which makes sense to be called only by older `legacy` pools since those doesn't have pre-minted BPT
The `getPrice` uses every time `totalSupply`
https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/WeightedBPTOracle.sol#L71-L73
which in the case of most recent pools can lead to very wrong calculations because of all the pre-minted BPT.
## Impact
Impact is a medium one, since it can lead to wrong price assumptions
## Code Snippet
https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/WeightedBPTOracle.sol#L71-L73
## Tool used

Manual Review

## Recommendation
Consider implementing the more recent `getActualSupply` even if older pools doesn't have this functions , because it can lead to wrong assumptions and calculations.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**YakuzaKiawe** commented:
>  invalid as this could be a factor to exploit the pool in future



# Issue M-4: Add Liquidity to Curve ETH pools will not work 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/42 

## Found by 
0x52, 0xjoseph
The Curve & Convex spell uses the Curve `add_liquidity` function to provide liquidity to curve pools.

This will only work for non-ETH related Curve pools as ETH curve pools will require you to use the native version of ETH.

## Vulnerability Detail
The Curve `add_liquidity` function was written in a way which assumes that all Curve pools with the same number of tokens uses the same interface for providing liquidity, this is not the case.

Here is how they differ:
1. All ETH related pools are payable functions, which **expects** a msg.value when calling the `add_liquidity` function.

Currently, the Curve & Convex Spell contract can only support non-ETH related pools with an interface like so:
`function add_liquidity(uint256[2] calldata, uint256) external;`
`function add_liquidity(uint256[3] calldata, uint256) external;`
 
 But Curve ETH pools uses payable functions where the `msg.value` is **mandatory** like so:
`function add_liquidity(uint256[3] calldata, uint256) external payable;`

![image](https://github.com/sherlock-audit/2023-07-blueberry-JosephSaw/assets/28586597/385dd122-e476-4cdb-8d64-d564e9a7d997)

Code snippet of the `add_liquidity` function inside the frxEth curve pool: https://etherscan.io/address/0xa1f8a6807c402e4a15ef4eba36528a3fed24e577#code
    
## Impact
BlueBerry will not be able to support **any** Curve ETH LSD related tokens, as the `add_liquidity` function signature are incorrect due to the `payable` keyword.

## Code Snippet
- https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/ConvexSpell.sol#L120-L144
- https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/CurveSpell.sol#L107-L140

## Tool used

Manual Review

## Recommendation
There are 2 ways to fix this, the first is easier and more direct, the second method will require a bit of re-structuring to your code.

Do note that this fix applies to **all** ETH related pools such as frxETH/ETH, stETH/ETH, ETH/USDT/BTC, etc

**First Method:**
i. Create a new mapping `isEthPool` (curveLpAddress -> true/false)
ii. Before checking `if (tokens.length == 2)`, you add another check to see if the curve LP is an ETH pool from the mapping above
iii. If it is, you can add liquidity using code like so: `CURVE_ETH_POOL.add_liquidity{value: amount}([amount, 0], 0);`

**Second Method:**
i. You can either create a mapping to save the function interface or pass it in as an argument.
ii. Encode the data dynamically and call it like so
 `CURVE_ETH_POOL.call{value: amount}{(
            abi.encodeWithSignature("add_liquidity(uint256[],uint256)", amounts, min_mint_amount)
        );`



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**0xyPhilic** commented:
> invalid because this issue can be considered informational as the contest details do not suggest usage of native ETH pools



# Issue M-5: ConvexSpell/CurveSpell.openPositionFarm will revert in some cases 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/60 

## Found by 
nobody2018

Before calling `ICurvePool#add_liquidity` to deposit tokens, each token needs to be approved to the pool. If the approved amount is smaller than the parameters passed into `ICurvePool#add_liquidity`, it will cause `erc20.transferFrom` revert inside the function. In this way, `openPositionFarm` will also revert.

In previous contest, [[#47](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/47)](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/47) was not fixed correctly. **This issue is still exist**.

## Vulnerability Detail

The fix for this issue from this contest is as following:

```solidity
File: blueberry-core\contracts\spell\CurveSpell.sol
098:         // 2. Borrow specific amounts
099:         uint256 borrowBalance = _doBorrow(
100:             param.borrowToken,
101:             param.borrowAmount
102:         );
103: 
104:         // 3. Add liquidity on curve
105:         address borrowToken = param.borrowToken;
106:         _ensureApprove(param.borrowToken, pool, borrowBalance);
107:         if (tokens.length == 2) {
108:             uint256[2] memory suppliedAmts;
109:             for (uint256 i = 0; i < 2; i++) {
                     //this 'if' check is the fix from the previous contest
110:->               if (tokens[i] == borrowToken) {
111:                     suppliedAmts[i] = IERC20Upgradeable(tokens[i]).balanceOf(
112:                         address(this)
113:                     );
114:                     break;
115:                 }
116:             }
117:             ICurvePool(pool).add_liquidity(suppliedAmts, minLPMint);
118:         } else if (tokens.length == 3) {
```

**The key to this issue is that `borrowBalance` may be smaller than `IERC20Upgradeable(borrowToken).balanceOf(address(this))`**. For simplicity, assume that CurveSpell supports an lptoken which contains two tokens : A and B.

**Bob transferred 1wei of A and B to the CurveSpell contract**. Alice opens a position by calling `BlueBerryBank#execute`, and the flow is as follows:

1.  enter `CurveSpell#openPositionFarm`.
2.  call `_doLend` to deposit isolated collaterals.
3.  call `_doBorrow` to borrow 100e18 A token. borrowBalance = 100e18.
4.  `A.approve(pool, 100e18)`.
5.  `suppliedAmts[0] = A.balance(address(this)) = 100e18+1wei`, `suppliedAmts[1] = 0`.
6.  call `ICurvePool(pool).add_liquidity(suppliedAmts, minLPMint)`, then revert because the approved amount is not enough.

Therefore, no one can successfully open a position.

Of course, bob can also transfer 1wei of `borrowToken` to contract by front-running `openPositionFarm` for a specific user or all users.

In ConvexSpell, the issue lies in [[the same code](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/ConvexSpell.sol#L110-L144)](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/ConvexSpell.sol#L110-L144) as in previous games.

## Impact

`ConvexSpell/CurveSpell.openPositionFarm` will revert due to this issue.

## Code Snippet

https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/CurveSpell.sol#L99-L140

https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/ConvexSpell.sol#L110-L144

## Tool used

Manual Review

## Recommendation

**The following fix is for CurveSpell, but please don't forget ConvexSpell**.

Two ways for fix it:

```fix
--- a/blueberry-core/contracts/spell/CurveSpell.sol
+++ b/blueberry-core/contracts/spell/CurveSpell.sol
@@ -108,9 +108,7 @@ contract CurveSpell is BasicSpell {
             uint256[2] memory suppliedAmts;
             for (uint256 i = 0; i < 2; i++) {
                 if (tokens[i] == borrowToken) {
-                    suppliedAmts[i] = IERC20Upgradeable(tokens[i]).balanceOf(
-                        address(this)
-                    );
+                    suppliedAmts[i] = borrowBalance;
                     break;
                 }
             }
@@ -119,9 +117,7 @@ contract CurveSpell is BasicSpell {
             uint256[3] memory suppliedAmts;
             for (uint256 i = 0; i < 3; i++) {
                 if (tokens[i] == borrowToken) {
-                    suppliedAmts[i] = IERC20Upgradeable(tokens[i]).balanceOf(
-                        address(this)
-                    );
+                    suppliedAmts[i] = borrowBalance;
                     break;
                 }
             }
@@ -130,9 +126,7 @@ contract CurveSpell is BasicSpell {
             uint256[4] memory suppliedAmts;
             for (uint256 i = 0; i < 4; i++) {
                 if (tokens[i] == borrowToken) {
-                    suppliedAmts[i] = IERC20Upgradeable(tokens[i]).balanceOf(
-                        address(this)
-                    );
+                    suppliedAmts[i] = borrowBalance;
                     break;
                 }
             }
```

```fix
--- a/blueberry-core/contracts/spell/CurveSpell.sol
+++ b/blueberry-core/contracts/spell/CurveSpell.sol
@@ -103,7 +103,8 @@ contract CurveSpell is BasicSpell {

         // 3. Add liquidity on curve
         address borrowToken = param.borrowToken;
-        _ensureApprove(param.borrowToken, pool, borrowBalance);
+        require(borrowBalance <= IERC20Upgradeable(borrowToken).balanceOf(address(this)), "impossible");
+        _ensureApprove(param.borrowToken, pool, IERC20Upgradeable(borrowToken).balanceOf(address(this)));
         if (tokens.length == 2) {
             uint256[2] memory suppliedAmts;
             for (uint256 i = 0; i < 2; i++) {
```



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**0xyPhilic** commented:
> invalid because the borrowAmount can't change mid function execution

**darkart** commented:
>  If already approven its up to Developrs to chose if they will implement it



# Issue M-6: AuraSpell#closePositionFarm exits pool with single token and without any slippage protection 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/102 

## Found by 
0x52, Breeje, Oxhunter526, Vagner, bitsurfer, nobody2018

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

# Issue M-7: AuraSpell#closePositionFarm will take reward fees on underlying tokens when borrow token is also a reward 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/103 

## Found by 
0x52

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

# Issue M-8: ConvexSpell is completely broken for any curve LP that utilizes native ETH 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/105 

## Found by 
0x52, Vagner

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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**0xyPhilic** commented:
> invalid because the design of the protocol does not show desire for ETH compatibility so the issue can be classified as informational



# Issue M-9: WConvexPool.sol will be broken on Arbitrum due to improper integration with Convex Arbitrum contracts 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/119 

## Found by 
Kow
WConvexPool (and consequently ConvexSpell) will be completely broken on Arbitrum due to not accounting for implementation differences in Convex Arbitrum contracts.

## Vulnerability Detail
WConvexPool.sol is the contract expected to be deployed to both mainnet and Arbitrum (sponsor confirmed). While it does integrate with mainnet Convex contracts, it does not account for implementation differences in Arbitrum Convex contracts (briefly mentioned in their [docs](https://docs.convexfinance.com/convexfinanceintegration/side-chain-implemention) with the Arbitrum version of the Booster contract that Blueberry integrates with found [here](https://arbiscan.io/address/0xF403C135812408BFbE8713b5A23a04b3D48AAE31#readContract)). The primary issue is the change in arguments returned from ``cvxPools.poolInfo()`` when calling [``getPoolInfoFromPoolId``](https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L103-L126). Since the return signature expects 6 values (through destructuring) while the Arbitrum Booster contract's getter only returns 5 (due to a slight change in the [PoolInfo struct](https://github.com/convex-eth/sidechain-platform/blob/b9525005549c8f2d364d092bfd902b8eb05d7079/contracts/contracts/Booster.sol#L36-L43) - mainnet struct [here](https://github.com/convex-eth/platform/blob/a5da3f127a321467a97a684c57970d2586520172/contracts/contracts/Booster.sol#L48-L55)), all attempts to call it will revert. This is called in [``pendingRewards``](https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L179-L191), [``mint``](https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L235-L241), and [``burn``](https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L276-L292), 3 of the main functions in WConvexPool, and will consequently disable any interaction with it (this includes ConvexSpell which also uses this function and also attempts to call mint/burn for position management).

Other issues include the use of ``cvxPools.deposit(...)`` and ``cvxPools.withdraw(...)`` which do not match any function signatures in the Arbitrum Booster contract (``withdraw`` has been changed to ``withdrawTo`` with different arguments and ``deposit`` no longer includes the 3rd argument).

## Impact
Convex integration will be completely broken on Arbitrum due to unaccounted for implementation differences.

## Code Snippet
https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L103-L126
https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L179-L191
https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L235-L241
https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L276-L292

## Tool used

Manual Review

## Recommendation
Consider creating a separate contract for WConvexPools for Arbitrum that correctly accounts for the Convex Booster implementation changes.

# Issue M-10: Potential for user positions to fall below `minPositionSize` when partially closing or reducing positions 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/122 

## Found by 
bitsurfer, chainNue

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

# Issue M-11: AuraSpell `openPositionFarm` will revert when the tokens contains `lpToken` 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/125 

## Found by 
bitsurfer, nobody2018, twcctop

AuraSpell `openPositionFarm` will revert when the tokens contains lpToken due to array length mismatch

## Vulnerability Detail

In AuraSpell, the `openPositionFarm` will call `joinPool` in Balancer's vault. But when analyzing the `JoinPoolRequest` struct, we see issue on `maxAmountsIn` and `amountsIn` which can be in different length, thus this will be reverted since in Balancer's vault, this two array should be in the same length.

```js
File: AuraSpell.sol
088:     function openPositionFarm(
089:         OpenPosParam calldata param,
090:         uint256 minimumBPT
091:     )
...
095:     {
...
110:         /// 3. Add liquidity to the Balancer pool and receive BPT in return.
111:         {
...
128:             if (poolAmountOut != 0) {
129:                 vault.joinPool(
130:                     wAuraPools.getBPTPoolId(lpToken),
131:                     address(this),
132:                     address(this),
133:                     IBalancerVault.JoinPoolRequest({
134:                         assets: tokens,
135:                         maxAmountsIn: maxAmountsIn,
136:                         userData: abi.encode(1, amountsIn, _minimumBPT),
137:                         fromInternalBalance: false
138:                     })
139:                 );
140:             }
141:         }
...
178:     }
...
296:     function _getJoinPoolParamsAndApprove(
297:         address vault,
298:         address[] memory tokens,
299:         uint256[] memory balances,
300:         address lpToken
301:     ) internal returns (uint256[] memory, uint256[] memory, uint256) {
...
304:         uint256 length = tokens.length;
305:         uint256[] memory maxAmountsIn = new uint256[](length);
306:         uint256[] memory amountsIn = new uint256[](length);
307:         bool isLPIncluded;
308:
309:         for (i; i != length; ) {
310:             if (tokens[i] != lpToken) {
311:                 amountsIn[j] = IERC20(tokens[i]).balanceOf(address(this));
312:                 if (amountsIn[j] > 0) {
313:                     _ensureApprove(tokens[i], vault, amountsIn[j]);
314:                 }
315:                 ++j;
316:             } else isLPIncluded = true;
317:
318:             maxAmountsIn[i] = IERC20(tokens[i]).balanceOf(address(this));
319:
320:             unchecked {
321:                 ++i;
322:             }
323:         }
324:
325:         if (isLPIncluded) {
326:             assembly {
327:                 mstore(amountsIn, sub(mload(amountsIn), 1))
328:             }
329:         }
...
345:         return (maxAmountsIn, amountsIn, poolAmountOut);
346:     }
```

these `maxAmountsIn` and `amountsIn` are coming from `_getJoinPoolParamsAndApprove`. And by seeing the function, we can see that there is possible issue when the `tokens[i] == lpToken`.

When `tokens[i] == lpToken`, the flag `isLPIncluded` will be true. And will enter this block,

```js
325:         if (isLPIncluded) {
326:             assembly {
327:                 mstore(amountsIn, sub(mload(amountsIn), 1))
328:             }
329:         }
```

this will decrease the `amountsIn` length. Thus, `amountsIn` and `maxAmountsIn` will be in different length.

In Balancer's `JoinPoolRequest` struct, the `maxAmountsIn`, and `userData` second decoded bytes (`amountsIn`) should be the same array length, because it will be checked in Balancer.

```js
133:                     IBalancerVault.JoinPoolRequest({
134:                         assets: tokens,
135:                         maxAmountsIn: maxAmountsIn,
136:                         userData: abi.encode(1, amountsIn, _minimumBPT),
137:                         fromInternalBalance: false
138:                     })
```

Therefore, in this situation, it will be reverted.

## Impact

User can't open position on AuraSpell when `tokens` contains `lpToken`

## Code Snippet

https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L326-L328

## Tool used

Manual Review

## Recommendation

Remove the assembly code where it will decrease the `amountsIn` length when `isLPIncluded` is true to make sure the array length are same.

# Issue M-12: approve() call with incorrect function signature will make any SoftVault deployed with USDT as the underlying token unusable 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/135 

## Found by 
n1punp, trauki

medium

# approve() call with incorrect function signature will make any SoftVault deployed with USDT as the underlying token unusable

## Summary
Open Zeppelin's `safeTransfer()` is used throughout the blueberry contracts, this ensures that calls to contracts without return values don't fail, however, the `EnsureApprove` contract uses a normal IERC20 interface and a normal `approve()` function call. Since USDT doesn't return a boolean as expected by the interface, this would leave the contract unusable. 

## Vulnerability Detail
If a SoftVault is deployed using USDT as the `uToken`,  the contract's main functions won't work as intended since the `ensureApprove` function call will fail. 

[USDT](https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7#code) approve signature: `function approve(address spender, uint value) public;`

[OpenZeppelin ERC20](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/IERC20.sol) approve signature: `function approve(address spender, uint256 value) external returns (bool);`
## Impact
This will cause loss of funds because the cost to deploy the contract will essentially have been wasted. The developers explicitly stated that they intend to use USDT as an underlying token in the `SoftVault`'s NatSpec:
![image](https://github.com/sherlock-audit/2023-07-blueberry-RohanNero/assets/100052099/9baf59fa-9de8-4406-9499-aaf45489601e)


## Code Snippet
https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/utils/EnsureApprove.sol#L27
## Tool used

Manual Review

## Recommendation
Add support for USDT by importing another interface with ERC20 functions that don't return values.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**0xyPhilic** commented:
> invalid because the IERC20 interface would fit also USDT as it has an approve function - no function signature is used anywhere



**Gornutz**

Item discussed in the previous competition full context is given here - https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/15#issuecomment-1542727941 

**Shogoki**

> Item discussed in the previous competition full context is given here - [sherlock-audit/2023-04-blueberry-judging#15 (comment)](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/15#issuecomment-1542727941)

Actually i do not understand why the issue was closed in the last contest with the comment: 

`USDT only can call approve function when the allowance is zero or to set the allowance to zero first.`

This is not related to the stated issue, that the approval will revert because the generic IERC20 interface is used, which is expecting a bool return data.

