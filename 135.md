Best Glass Worm

medium

# approve() call with incorrect function signature will make any SoftVault deployed with USDT as the underlying token unusable
RohanNero

medium

# approve() call with incorrect function signature will make any SoftVault deployed with USDT as the underlying token unusable

## Summary
Open Zeppelin's `safeTransfer()` is used throughout the blueberry contracts, this ensures that calls to contracts without return values don't fail, however, the `EnsureApprove` contract uses a normal IERC20 interface and a normal `approve()` function call. Since USDT doesn't return a boolean as expected by the interface, this would leave the contract unusable. 

## Vulnerability Detail
If a SoftVault is deployed using USDT as the `uToken`,  the contract's main functions won't work as intended since the `ensureApprove` function call will fail. 

[USDT](https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7#code) approve signature: `function approve(address spender, uint value) public;`

[OpenZeppelin ERC20](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/IERC20.sol) approve signature: `function approve(address spender, uint256 value) external returns (bool);`
## Impact
This will cause loss of funds because the cost to deploy the contract will essentially have been wasted. The developers explicitly stated that they intend to use USDT as an underlying token in the `SoftVault`'s NatSpec:
![image](https://github.com/sherlock-audit/2023-07-blueberry-RohanNero/assets/100052099/9baf59fa-9de8-4406-9499-aaf45489601e)


## Code Snippet
https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/utils/EnsureApprove.sol#L27
## Tool used

Manual Review

## Recommendation
Add support for USDT by importing another interface with ERC20 functions that don't return values.
