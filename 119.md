Alert Lipstick Tarantula

medium

# WConvexPool.sol will be broken on Arbitrum due to improper integration with Convex Arbitrum contracts
## Summary
WConvexPool (and consequently ConvexSpell) will be completely broken on Arbitrum due to not accounting for implementation differences in Convex Arbitrum contracts.

## Vulnerability Detail
WConvexPool.sol is the contract expected to be deployed to both mainnet and Arbitrum (sponsor confirmed). While it does integrate with mainnet Convex contracts, it does not account for implementation differences in Arbitrum Convex contracts (briefly mentioned in their [docs](https://docs.convexfinance.com/convexfinanceintegration/side-chain-implemention) with the Arbitrum version of the Booster contract that Blueberry integrates with found [here](https://arbiscan.io/address/0xF403C135812408BFbE8713b5A23a04b3D48AAE31#readContract)). The primary issue is the change in arguments returned from ``cvxPools.poolInfo()`` when calling [``getPoolInfoFromPoolId``](https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L103-L126). Since the return signature expects 6 values (through destructuring) while the Arbitrum Booster contract's getter only returns 5 (due to a slight change in the [PoolInfo struct](https://github.com/convex-eth/sidechain-platform/blob/b9525005549c8f2d364d092bfd902b8eb05d7079/contracts/contracts/Booster.sol#L36-L43) - mainnet struct [here](https://github.com/convex-eth/platform/blob/a5da3f127a321467a97a684c57970d2586520172/contracts/contracts/Booster.sol#L48-L55)), all attempts to call it will revert. This is called in [``pendingRewards``](https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L179-L191), [``mint``](https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L235-L241), and [``burn``](https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L276-L292), 3 of the main functions in WConvexPool, and will consequently disable any interaction with it (this includes ConvexSpell which also uses this function and also attempts to call mint/burn for position management).

Other issues include the use of ``cvxPools.deposit(...)`` and ``cvxPools.withdraw(...)`` which do not match any function signatures in the Arbitrum Booster contract (``withdraw`` has been changed to ``withdrawTo`` with different arguments and ``deposit`` no longer includes the 3rd argument).

## Impact
Convex integration will be completely broken on Arbitrum due to unaccounted for implementation differences.

## Code Snippet
https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L103-L126
https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L179-L191
https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L235-L241
https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WConvexPools.sol#L276-L292

## Tool used

Manual Review

## Recommendation
Consider creating a separate contract for WConvexPools for Arbitrum that correctly accounts for the Convex Booster implementation changes.