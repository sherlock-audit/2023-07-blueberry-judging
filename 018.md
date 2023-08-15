Modern Brunette Raccoon

medium

# `getPrice` in `WeightedBPTOracle.sol`  uses `totalSupply` for price calculations which can lead to wrong results
## Summary
`getPrice` is used to calculate the price in USD of a given balancer LP, and it respects the recommendations  of Balancer docs by calculating the invariant and using it to protect from manipulations, but it uses `totalSupply` for every LP calculated which can lead to wrong results and assumptions.
## Vulnerability Detail
In the Balancer docs it is specified that `There are three potential functions to query when determining the BPT supply depending on pool type.` https://docs.balancer.fi/concepts/advanced/valuing-bpt/valuing-bpt.html#getting-bpt-supply
- `getActualSupply` : which is used by the most recent Weighted and Stable Pools and it accounts for pre-minted BPT as well as due protocol fees:
- `getVirtualSupply` : which is used by Linear Pools and "legacy" Stable Phantom Pools and it accounts just for pre-minted BPT
- `totalSupply` : which makes sense to be called only by older `legacy` pools since those doesn't have pre-minted BPT
The `getPrice` uses every time `totalSupply`
https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/WeightedBPTOracle.sol#L71-L73
which in the case of most recent pools can lead to very wrong calculations because of all the pre-minted BPT.
## Impact
Impact is a medium one, since it can lead to wrong price assumptions
## Code Snippet
https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/WeightedBPTOracle.sol#L71-L73
## Tool used

Manual Review

## Recommendation
Consider implementing the more recent `getActualSupply` even if older pools doesn't have this functions , because it can lead to wrong assumptions and calculations.