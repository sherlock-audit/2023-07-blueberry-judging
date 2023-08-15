Prehistoric Jade Pigeon

medium

# WeightedBPTOracle does not support tokens whose decimal is greater than 18
## Summary
WeightedBPTOracle does not support tokens whose decimal is greater than 18.

## Vulnerability Detail
getPrice() in the WeightedBPTOracle contract does not support tokens whose decimal is greater than 18. That means if these tokens interact with this function, will revert because function cant proceed this line: `(18 - IERC20Metadata(tokens[i]).decimals())`

```solidity
 uint256 length = weights.length;
        uint256 temp = 1e18;
        uint256 invariant = 1e18;
        for(uint256 i; i < length; i++) {
            temp = temp.mulDown(
                (base.getPrice(tokens[i]).divDown(weights[i]))
                .powDown(weights[i])
            );
            invariant = invariant.mulDown(
                (balances[i] * 10 ** (18 - IERC20Metadata(tokens[i]).decimals()))
                .powDown(weights[i])
            );
        }
```

## Impact
Tokens whose decimal is greater than 18 are not supported and if used in the future will be problematic in the `WeightedBPTOracle` contract.

## Code Snippet
```solidity
 function getPrice(address token) external override returns (uint256) {
        IBalancerPool pool = IBalancerPool(token);
        IBalancerVault vault = IBalancerVault(pool.getVault());

        // Reentrancy guard to prevent flashloan attack
        checkReentrancy(vault);

        (address[] memory tokens, uint256[] memory balances, ) = vault
            .getPoolTokens(pool.getPoolId());

        uint256[] memory weights = pool.getNormalizedWeights();

        uint256 length = weights.length;
        uint256 temp = 1e18;
        uint256 invariant = 1e18;
        for(uint256 i; i < length; i++) {
            temp = temp.mulDown(
                (base.getPrice(tokens[i]).divDown(weights[i]))
                .powDown(weights[i])
            );
            invariant = invariant.mulDown(
                (balances[i] * 10 ** (18 - IERC20Metadata(tokens[i]).decimals()))
                .powDown(weights[i])
            );
        }
        return invariant
            .mulDown(temp)
            .divDown(IBalancerPool(token).totalSupply());
    }
```
https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/oracle/WeightedBPTOracle.sol#L46-L74

## Tool used
Manual Review

## Recommendation
Consider 36 decimal instead of 18.
