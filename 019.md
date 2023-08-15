Modern Brunette Raccoon

medium

# `getPrice` in  can revert 100% of the time for some pools because of `_checkReentrant`
## Summary
The function `getPrice` use `_checkReentrant` to protect itself from reentrancy but this function can revert all the time of some Curve pools which can make calculating the price impossible.
## Vulnerability Detail
The function `_checkReentrant` checks how many number of tokens does the Curve pool have and then tries to call `remove_liquidity` with 0 values
https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/CurveStableOracle.sol#L40-L51
but there are multiple Curve pools which tries to do a subtraction of the `_amount` variable inserted with 1 to **# Make rounding errors favoring other LPs a tiny bit** as can be seen here 
https://etherscan.io/address/0x21d158d95c2e150e144c36fc64e3653b8d6c6267#code#L1013
or here 
https://etherscan.io/address/0xf5f5b97624542d72a9e06f04804bf81baa15e2b4#code#L699
In those cases the subtraction would underflow and revert which will make getting the price of those pools impossible.
## Impact
Impact is medium since calculation of prices would revert for those cases.
## Code Snippet
https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/CurveStableOracle.sol#L40-L51
## Tool used

Manual Review

## Recommendation
Consider using other methods of protections against pools reentrancy since many pools implement this kind of behavior on `remove_liquidity` which would make the oracle not work.