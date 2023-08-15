Magnificent Mercurial Swift

high

# CurveTricryptoOracle#getPrice contains math error that causes LP to be priced completely wrong
## Summary

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