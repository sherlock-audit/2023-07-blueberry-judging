Virtual Hazelnut Seal

high

# Potential flash loan attack vulnerability in `getPrice` function of CurveStableOracle
## Summary
During a security review of the `getPrice` function in the `CurveStableOracle`, a potential flash loan attack vulnerability was identified.

## Vulnerability Detail
The `getPrice` function retrieves the current price of each token in a Curve LP pool, calculates the minimum price among them, and multiplies it by the virtual price of the LP token to determine the USD value of the LP token. If the price of one or more tokens in the pool is manipulated, this can cause the minimum price calculation to be skewed, leading to an incorrect USD value for the LP token. This can be exploited by attackers to make a profit at the expense of other users.

## Impact
This vulnerability could potentially allow attackers to manipulate the price of tokens in Curve LP pools and profit at the expense of other users. If exploited, this vulnerability could result in significant financial losses for affected users.

## Code Snippet
https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/oracle/CurveStableOracle.sol#L58-L70

## Tool used

Manual Review

## Recommendation
use TWAP to determine the prices of the underlying assets in the pool.
