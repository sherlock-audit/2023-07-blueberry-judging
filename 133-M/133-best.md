Scruffy Shamrock Sardine

medium

# Hardcoded Decimal Precision in `ChainlinkAdapterOracleL2.getPrice()`
## Summary

The BlueBerryBank contract utilizes a hardcoded constant for deriving the price from the Chainlink price feed, which assumes a representation with 8 decimals. This becomes problematic when integrating tokens like AMPL, which Chainlink price feed returns with 18 decimals. Such an approach restricts the system's adaptability to different decimal precisions across various Chainlink price feeds and could cause the protocol to enter a state where user funds can be lost.

## Vulnerability Detail

The contract `ChainlinkAdapterOracleL2.sol` employs a hardcoded constant `Constants.CHAINLINK_PRICE_FEED_PRECISION` to adjust the price returned from the Chainlink price feed, assuming it represents the value with 8 decimals. This approach presupposes that all Chainlink price feeds return values with 8 decimals. However, not all Chainlink feeds conform to this assumption. For instance, the [AMPL/USD Chainlink feed](https://data.chain.link/ethereum/mainnet/crypto-usd/ampl-usd) actually returns values with 18 decimals.

Comparatively, in `ChainlinkAdapterOracle.sol`, the system uses a dynamic approach by dividing with `10 ** decimals`, which automatically adapts to the precision of the returned price feed. The hardcoded approach in `ChainlinkAdapterOracleL2.sol` makes it inflexible and incompatible with any Chainlink feed that doesn't strictly return 8 decimals.

## Impact

- **Inaccuracy**: Any token with a Chainlink feed that doesn't return exactly 8 decimals will have its price inaccurately represented if it were to be integrated.
  
- **Limited Integration**: Tokens such as AMPL, which have Chainlink feeds returning 18 decimals, cannot be integrated into the system without causing potential financial inaccuracies.

- **Future Proofing**: As the ecosystem evolves, more tokens with varying decimal representations may emerge. The hardcoded approach restricts adaptability to future changes.

## Code Snippet

[ChainlinkAdapterOracleL2.sol#L144-L146](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/ChainlinkAdapterOracleL2.sol#L144-L146)

```solidity
return (price.toUint256() * Constants.PRICE_PRECISION) / Constants.CHAINLINK_PRICE_FEED_PRECISION; //@audit-issue AMPL/USD = 18 decimals
```
```solidity
/// @dev Precision factor for Chainlink price feed values.
uint256 constant CHAINLINK_PRICE_FEED_PRECISION = 1e8; //@audit-issue not all XXX/USD Chainlink price feeds return 8 decimals
```

## Tool used

Manual Review

## Recommendation

Replace the hardcoded constant with a dynamic approach that fetches the decimal precision directly from the Chainlink feed for the respective token as it is done in `ChainlinkAdapterOracle.sol`:
```solidity
        /// Get token-USD price
        uint256 decimals = registry.decimals(token, USD);
        (uint80 roundID, int256 answer,, uint256 updatedAt, uint80 answeredInRound) =
            registry.latestRoundData(token, USD);
        if (updatedAt < block.timestamp - maxDelayTime) {
            revert Errors.PRICE_OUTDATED(token_);
        }
        if (answer <= 0) revert Errors.PRICE_NEGATIVE(token_);
        if (answeredInRound < roundID) revert Errors.PRICE_OUTDATED(token_);

        return (answer.toUint256() * Constants.PRICE_PRECISION) / 10 ** decimals;
```