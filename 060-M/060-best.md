Spare Syrup Dragonfly

medium

# ConvexSpell/CurveSpell.openPositionFarm will revert in some cases
## Summary

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