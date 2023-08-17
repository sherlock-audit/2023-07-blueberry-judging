Recumbent Stone Lark

medium

# Oracle Returns Incorrect Price During Flash Crashes
## Summary
Chainlink price feeds have in-built minimum & maximum prices they will return; if during a flash crash, bridge compromise, or depegging event, an asset’s value falls below the price feed’s minimum price, [[the oracle price feed will continue to report the (now incorrect) minimum price
## Vulnerability Detail

## Impact
An attacker could:
- buy that asset using a decentralized exchange at the very low price,
- deposit the asset into a Lending / Borrowing platform using Chainlink’s price feeds,
- borrow against that asset at the minimum price Chainlink’s price feed returns, even though the actual price is far lower.

## Code Snippet
https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/ChainlinkAdapterOracle.sol#L102-L126

## Tool used
Manual Review

## Recommendation
cross-check the returned answer against the minPrice/maxPrice and revert if the answer is outside of these bounds:
```solidity
if (answer >= maxPrice or answer <= minPrice) revert();
```