Icy Shamrock Bobcat

medium

# AuraSpell `openPositionFarm` will revert when the tokens contains `lpToken`
## Summary

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
