Nice Ash Mustang

high

# The `withdrawLend`  function appear to be failing to transfer tokens to the `pos.owner`
## Summary
"The `withdrawLend`  function within `BlueBerryBank.sol` is observed to transfer tokens to the bank contract itself, rather than effectuating transfers to the designated `pos.owner`."
## Vulnerability Detail
The `execute` function accepts three parameters: `positionId`, `spell`, and `data`. When a user intends to withdraw their lent tokens to the bank, they specify the `spell` parameter as the address of the `BlueBerryBank` contract when calling the `execute` function. Subsequently, the `execute` function triggers the `withdrawLend` function. It is crucial to observe that the `msg.sender` executing the `withdrawLend` function is the `BlueBerryBank` contract itself. Consequently, the tokens transferred by the `withdrawLend` function are directed to the `msg.sender`, i.e., the `BlueBerryBank` contract, whereas the intended course is for the tokens to be transferred to the designated `pos.owner`.

## Impact
Lenders encounter difficulties in withdrawing their lent tokens
## Code Snippet
https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/BlueBerryBank.sol#L629-L668


https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/BlueBerryBank.sol#L745



## Tool used

Manual Review

## Recommendation

` - IERC20Upgradeable(token).safeTransfer(msg.sender, wAmount);`
` + IERC20Upgradeable(token).safeTransfer(pos.owner, wAmount);`
