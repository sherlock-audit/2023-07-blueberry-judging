Great Plastic Grasshopper

high

# Add Liquidity to Curve ETH pools will not work
## Summary
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