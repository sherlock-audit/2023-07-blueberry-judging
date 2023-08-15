Sunny Iris Elephant

medium

# Using block.timestamp for the deadline is not effective
## Summary

In IchiSpell.sol when swapping the block.timestamp is used for the deadline. However when that tx is included in a block, it will be valid at that time, since block.timestamp will be the current timestamp so this deadline will be ineffective and users txs can be executed at a later point.

## Vulnerability Detail

Because Front-running is a key aspect of AMM design, deadline is a useful tool to ensure that your tx cannot be “saved for later”.

Because block.timestamp is used and the protection does not work here, it may be more profitable for a miner to deny the transaction from being mined until the transaction incurs the maximum amount of slippage.

## Impact

Without a deadline, the transaction might be left hanging in the mempool and be executed way later than the user wanted.

That could lead to users getting a worse price, because a miner can just hold onto the transaction. And when it does get around to putting the transaction in a block, it'll be block.timestamp, so this protection doesnt work here.

## Code Snippet

https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/spell/IchiSpell.sol#L235

```solidity

229: IUniswapV3Router.ExactInputSingleParams
230:     memory params = IUniswapV3Router.ExactInputSingleParams({
231:         tokenIn: swapPath[0],
232:         tokenOut: swapPath[1],
233:         fee: IUniswapV3Pool(vault.pool()).fee(),
234:         recipient: address(this),
235:         deadline: block.timestamp,
236:         amountIn: amountIn,
237:         amountOutMinimum: param.amountOutMin,
238:         sqrtPriceLimitX96: 0
239:  });

```
As you can see here block.timestamp is used for the deadline which means that whenever the miner decides to include the txn in a block, it will be valid at that time, since block.timestamp will be the current timestamp.

## Tool used

Manual Review

## Recommendation

Make the user pass in the deadline that he wants
