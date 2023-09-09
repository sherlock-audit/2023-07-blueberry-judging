# Issue H-1: Stable BPT valuation is incorrect and can be exploited to cause protocol insolvency 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/97 

## Found by 
0x52

The current methodology for valuing Stable BPT is incorrect and can lead to significant over valuation of the stable BPT.

## Vulnerability Detail

[StableBPTOracle.sol#L48-L53](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/StableBPTOracle.sol#L48-L53)

        uint256 minPrice = base.getPrice(tokens[0]);
        for(uint256 i = 1; i != length; ++i) {
            uint256 price = base.getPrice(tokens[i]);
            minPrice = (price < minPrice) ? price : minPrice;
        }
        return minPrice.mulWadDown(pool.getRate());

The above block is used to calculate the price. Finding the min price of all assets in the pool then multiplying by the current rate of the pool. This is nearly identical to how stable curve LP is priced. Balancer pools are a bit different and this methodology is incorrect for them. Lets look at a current mainnet pool to see the problem. Take the wstETH/aETHc pool. Currently getRate() = 1.006. The lowest price is aETHc at 2,073.23. This values the LP at 2,085.66. The issue is that the LPs actual value is 1,870.67 (nearly 12% overvalued) which can be checked [here](https://app.apy.vision/pools/balancerv2_eth-wstETH-ankrETH-0xdfe6e7e18f6cc65fa13c8d8966013d4fda74b6ba).

Overvaluing the LP as such can cause protocol insolvency as the borrower can overborrow against the LP, leaving the protocol with bad debt.

## Impact

Protocol insolvency due to overborrowing

## Code Snippet

https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/StableBPTOracle.sol#L37-L54

## Tool used

Manual Review

## Recommendation

Stable BPT oracles need to use a new pricing methodology




## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**0xyPhilic** commented:
> invalid because there is no sufficient data/explanations to support the explained issue

**Kral01** commented:
> only an issue if the protocol uses this LP pair



**IAm0x52**

Escalate

This is not a dupe of #100. Though it focuses on a similar area of the code, the underlying issue is completely different. StableBPT is value highly incorrectly for some pools and it will cause significant damage to the protocol.

**sherlock-admin2**

 > Escalate
> 
> This is not a dupe of #100. Though it focuses on a similar area of the code, the underlying issue is completely different. StableBPT is value highly incorrectly for some pools and it will cause significant damage to the protocol.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Shogoki**

> ing the LP as such can cause protocol insolvency as the borrower can overborrow against the LP, leaving the protocol with bad debt.

Yes, not a duplicate of #100 
@Gornutz can you take a look at this?

**Gornutz**

Confirm this is not a duplicate of #100  

**hrishibhat**

Result:
High 
Unique
Considering this a valid high issue as the wrong price is calculated and returned

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [IAm0x52](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/97/#issuecomment-1694746948): accepted

# Issue H-2: CurveTricryptoOracle incorrectly assumes that WETH is always the last token in the pool which leads to bad LP pricing 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/98 

## Found by 
0x52, Vagner

CurveTricryptoOracle assumes that WETH is always the last token in the pool (`tokens[2]`). This is incorrect for a majority of tricrypto pools and will lead to LP being highly overvalued.

## Vulnerability Detail

[CurveTricryptoOracle.sol#L53-L63](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/CurveTricryptoOracle.sol#L53-L63)

        if (tokens.length == 3) {
            /// tokens[2] is WETH
            uint256 ethPrice = base.getPrice(tokens[2]);
            return
                (lpPrice(
                    virtualPrice,
                    base.getPrice(tokens[1]),
                    ethPrice,
                    base.getPrice(tokens[0])
                ) * 1e18) / ethPrice;
        }

When calculating LP prices, CurveTricryptoOracle#getPrice always assumes that WETH is the second token in the pool. This isn't the case which will cause the LP to be massively overvalued.

There are 6 tricrypto pools currently deployed on mainnet. Half of these pools have an asset other than WETH as token[2]:

        0x4ebdf703948ddcea3b11f675b4d1fba9d2414a14 - CRV
        0x5426178799ee0a0181a89b4f57efddfab49941ec - INV
        0x2889302a794da87fbf1d6db415c1492194663d13 - wstETH

## Impact

LP will be massively overvalued leading to overborrowing and protocol insolvency

## Code Snippet

[CurveTricryptoOracle.sol#L48-L65](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/CurveTricryptoOracle.sol#L48-L65)

## Tool used

Manual Review

## Recommendation

There is no need to assume that WETH is the last token. Simply pull the price for each asset and input it into lpPrice.





## Discussion

**IAm0x52**

Escalate

This is not a dupe of #105. This will cause a large number of tricrypto pools to be overvalued which presents a serious risk to the protocol.

**sherlock-admin2**

 > Escalate
> 
> This is not a dupe of #105. This will cause a large number of tricrypto pools to be overvalued which presents a serious risk to the protocol.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Shogoki**

Agree. 
This is not a duplicate of #105 
This can become its own main report and #20 is a duplicate of it.

There were some issues with (de)duplication. I would resolve like this.
#98 is the duplicate with #20 
#105 is duplicate with #42 

**hrishibhat**

Result:
High 
Has duplicates
This is a valid high issue based on the description

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [IAm0x52](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/98/#issuecomment-1694746548): accepted

# Issue H-3: CurveTricryptoOracle#getPrice contains math error that causes LP to be priced completely wrong 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/100 

## Found by 
0x52, Kow

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



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**0xyPhilic** commented:
> invalid because lpPrice is considered in ETH so dividing by ETH/USD price returns the final result in USD

**Kral01** commented:
> there is a precision value



**Gornutz**

Judges accurately state why the division by ETH/USD is required to return the proper USD value.

**Shogoki**

Closing in regards to other judges and sponsors comments.

**IAm0x52**

This is valid. The code being used here was borrowed from the Sentiment [CurveTriCryptoOracle](https://arbiscan.io/address/0x4e828a117ddc3e4dd919b46c90d4e04678a05504#code), which is specifically meant to return the price in terms of ETH. This oracle is meant to return the valuation in USD which means the division by the price of ETH needs to be dropped.

**Shogoki**

> This is valid. The code being used here was borrowed from the Sentiment [CurveTriCryptoOracle](https://arbiscan.io/address/0x4e828a117ddc3e4dd919b46c90d4e04678a05504#code), which is specifically meant to return the price in terms of ETH. This oracle is meant to return the valuation in USD which means the division by the price of ETH needs to be dropped.

Maybe was a bit quick in closing. Will reopen it and we will take a deeper look at it.

# Issue H-4: CVX/AURA distribution calculation is incorrect and will lead to loss of rewards at the end of each cliff 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/109 

## Found by 
0x52

When calculating the amount of pending AURA owed to a user _getAuraPendingReward uses the current values for supply. This leads to incorrect calculation across cliffs which leads to loss of rewards for users.

## Vulnerability Detail

[WAuraPools.sol#L233-L248](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L233-L248)

        if (cliff < totalCliffs) {
            /// e.g. (new) reduction = (500 - 100) * 2.5 + 700 = 1700;
            /// e.g. (new) reduction = (500 - 250) * 2.5 + 700 = 1325;
            /// e.g. (new) reduction = (500 - 400) * 2.5 + 700 = 950;
            uint256 reduction = ((totalCliffs - cliff) * 5) / 2 + 700;
            /// e.g. (new) amount = 1e19 * 1700 / 500 =  34e18;
            /// e.g. (new) amount = 1e19 * 1325 / 500 =  26.5e18;
            /// e.g. (new) amount = 1e19 * 950 / 500  =  19e17;
            mintAmount = (mintRequestAmount * reduction) / totalCliffs;

            /// e.g. amtTillMax = 5e25 - 1e25 = 4e25
            uint256 amtTillMax = emissionMaxSupply - emissionsMinted;
            if (mintAmount > amtTillMax) {
                mintAmount = amtTillMax;
            }
        }

The above code is used to calculate the amount of AURA owed to the user. This calculation is perfectly accurate if the AURA hasn't been minted yet. The problem is that each time a user withdraws, AURA is claimed for ALL vault participants. This means that the rewards will be realized for a majority of users before they themselves withdraw. Since the emissions decrease with each cliff, there will be loss of funds at the end of each cliff.

Example:
Assume for simplicity there are only 2 cliffs. User A deposits LP to WAuraPools. After some time User B deposits as well. Before the end of the first cliff User A withdraw. This claims all tokens owed to both users A and B which is now sitting in the contract. Assume both users are owed 10 tokens. Now User B waits for the second cliff to end before withdrawing. When calculating his rewards it will give him no rewards since all cliffs have ended. The issue is that the 10 tokens they are owed is already sitting in the contract waiting to be claimed.

## Impact

All users will lose rewards at the end of each cliff due to miscalculation

## Code Snippet

[WAuraPools.sol#L209-L249](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L209-L249)

[WConvexPools.sol#L149-L172](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WConvexPools.sol#L149-L172)

## Tool used

Manual Review

## Recommendation

I would recommend a hybrid approach. When rewards are claimed upon withdrawal, the reward per token should be cached to prevent loss of tokens that have already been received by the contract. Only unminted AURA should be handled this way.



## Discussion

**IAm0x52**

Escalate

This was wrongly excluded and causes a significant problem. Whenever a position is burned it calls the following line:

https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WAuraPools.sol#L379

        IAuraRewarder(auraRewarder).withdraw(amount, true);

Which withdraws the LP and claims rewards because of the `true` input. Digging into the rewarder contract:

https://etherscan.io/address/0x1204f5060be8b716f5a62b4df4ce32acd01a69f5#code#F18#L229

    function withdraw(uint256 amount, bool claim)
        public
        updateReward(msg.sender)
        returns(bool)
    {
        require(amount > 0, 'RewardPool : Cannot withdraw 0');

        //also withdraw from linked rewards
        for(uint i=0; i < extraRewards.length; i++){
            IRewards(extraRewards[i]).withdraw(msg.sender, amount);
        }

        _totalSupply = _totalSupply.sub(amount);
        _balances[msg.sender] = _balances[msg.sender].sub(amount);

        stakingToken.safeTransfer(msg.sender, amount);
        emit Withdrawn(msg.sender, amount);
     
        if(claim){
            getReward(msg.sender,true);
        }

        emit Transfer(msg.sender, address(0), amount);

        return true;
    }

Here we see that if claim == true then we call `getReward`

https://etherscan.io/address/0x1204f5060be8b716f5a62b4df4ce32acd01a69f5#code#F18#L296

    function getReward(address _account, bool _claimExtras) public updateReward(_account) returns(bool){
        uint256 reward = earned(_account);
        if (reward > 0) {
            rewards[_account] = 0;
            rewardToken.safeTransfer(_account, reward);
            IDeposit(operator).rewardClaimed(pid, _account, reward);
            emit RewardPaid(_account, reward);
        }

        //also get rewards from linked rewards
        if(_claimExtras){
            for(uint i=0; i < extraRewards.length; i++){
                IRewards(extraRewards[i]).getReward(_account);
            }
        }
        return true;
    }

Here we see that the ENTIRE reward balance is claimed for the vault. This presents the problem as outlined in my issue above. Let's assume that the reward for the current round is 1:1 (i.e. that each claimed BAL gives 1 AURA). Assume we have 2 users, each with half the pool. Now 100 tokens are claimed in this round, which means each user is owed 50 AURA. After some amount of rounds, the reward is now 1:0.5 (i.e. that each claimed BAL gives 1 AURA). Now when user A withdraws instead of being paid 100 AURA for that round they will instead only be paid 50 AURA and they will lose the other 50 since it won't be claimable. This is because it always uses the most recent exchange rate instead of the rate at which it was claimed.



**sherlock-admin2**

 > Escalate
> 
> This was wrongly excluded and causes a significant problem. Whenever a position is burned it calls the following line:
> 
> https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/wrapper/WAuraPools.sol#L379
> 
>         IAuraRewarder(auraRewarder).withdraw(amount, true);
> 
> Which withdraws the LP and claims rewards because of the `true` input. Digging into the rewarder contract:
> 
> https://etherscan.io/address/0x1204f5060be8b716f5a62b4df4ce32acd01a69f5#code#F18#L229
> 
>     function withdraw(uint256 amount, bool claim)
>         public
>         updateReward(msg.sender)
>         returns(bool)
>     {
>         require(amount > 0, 'RewardPool : Cannot withdraw 0');
> 
>         //also withdraw from linked rewards
>         for(uint i=0; i < extraRewards.length; i++){
>             IRewards(extraRewards[i]).withdraw(msg.sender, amount);
>         }
> 
>         _totalSupply = _totalSupply.sub(amount);
>         _balances[msg.sender] = _balances[msg.sender].sub(amount);
> 
>         stakingToken.safeTransfer(msg.sender, amount);
>         emit Withdrawn(msg.sender, amount);
>      
>         if(claim){
>             getReward(msg.sender,true);
>         }
> 
>         emit Transfer(msg.sender, address(0), amount);
> 
>         return true;
>     }
> 
> Here we see that if claim == true then we call `getReward`
> 
> https://etherscan.io/address/0x1204f5060be8b716f5a62b4df4ce32acd01a69f5#code#F18#L296
> 
>     function getReward(address _account, bool _claimExtras) public updateReward(_account) returns(bool){
>         uint256 reward = earned(_account);
>         if (reward > 0) {
>             rewards[_account] = 0;
>             rewardToken.safeTransfer(_account, reward);
>             IDeposit(operator).rewardClaimed(pid, _account, reward);
>             emit RewardPaid(_account, reward);
>         }
> 
>         //also get rewards from linked rewards
>         if(_claimExtras){
>             for(uint i=0; i < extraRewards.length; i++){
>                 IRewards(extraRewards[i]).getReward(_account);
>             }
>         }
>         return true;
>     }
> 
> Here we see that the ENTIRE reward balance is claimed for the vault. This presents the problem as outlined in my issue above. Let's assume that the reward for the current round is 1:1 (i.e. that each claimed BAL gives 1 AURA). Assume we have 2 users, each with half the pool. Now 100 tokens are claimed in this round, which means each user is owed 50 AURA. After some amount of rounds, the reward is now 1:0.5 (i.e. that each claimed BAL gives 1 AURA). Now when user A withdraws instead of being paid 100 AURA for that round they will instead only be paid 50 AURA and they will lose the other 50 since it won't be claimable. This is because it always uses the most recent exchange rate instead of the rate at which it was claimed.
> 
> 

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Shogoki**

@Gornutz What do you think about it? 
I think it is a valid Medium.

**Gornutz**

Seems directionally correct and valid from the parts of the POC provided. 

**IAm0x52**

The current AURA APR for pools are high double digit to triple digit returns. Additionally AURA is distributed over [~8 years](https://docs.aura.finance/aura/usdaura/distribution) and there are [500 cliffs](https://etherscan.io/address/0xC0c293ce456fF0ED870ADd98a0828Dd4d2903DBF#code#F1#L25) over that period. This makes the average cliff ~6 days. Unless every single user withdraws and redeposits each 6 day period (which is prohibitively expensive due to gas costs) this loss cannot be avoided. Given the almost certainty of the loss and the substantial amount these users stand to lose due to the high APRs and leveraged nature of the system, I believe this should be high risk rather than medium.

**hrishibhat**

Result:
High
Unique 
After further review considering this to be a high based on the comments above 

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [IAm0x52](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/109/#issuecomment-1694743790): accepted

# Issue H-5: wrong bToken's exchangeRateStored used for calculate ColleteralValue 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/117 

## Found by 
mert\_eren
BlueBerryBank.sol calculate position's risk value with `getPositionRisk()` . This function used for is position can be liquadatable or not.
```solidity
    function getPositionRisk(uint256 positionId) public returns (uint256 risk) {
        uint256 pv = getPositionValue(positionId);
        uint256 ov = getDebtValue(positionId);
        uint256 cv = getIsolatedCollateralValue(positionId);


        if (
            (cv == 0 && pv == 0 && ov == 0) || pv >= ov /// Closed position or Overcollateralized position
        ) {
            risk = 0;
        } else if (cv == 0) {
            /// Sth bad happened to isolated underlying token
            risk = Constants.DENOMINATOR;
        } else {
            risk = ((ov - pv) * Constants.DENOMINATOR) / cv;
        }
    }
```

`getIsolatedColleteralValue()` calculate lended value in position.
```solidity
    function getIsolatedCollateralValue(
        uint256 positionId
    ) public override returns (uint256 icollValue) {
        Position memory pos = positions[positionId];
        /// NOTE: exchangeRateStored has 18 decimals.
        uint256 underlyingAmount;
        if (_isSoftVault(pos.underlyingToken)) {
            underlyingAmount =
                (ICErc20(banks[pos.debtToken].bToken).exchangeRateStored() *
                    pos.underlyingVaultShare) /
                Constants.PRICE_PRECISION;
        } else {
            underlyingAmount = pos.underlyingVaultShare;
        }
        icollValue = oracle.getTokenValue(
            pos.underlyingToken,
            underlyingAmount
        );
    }
```
For calculate total value it should calculate first `underlyingAmount`. Due to calculate this, it should first multiply `pos.underlyingVaultShare` with ```exchangeRateStored()``` of this token.Because when stored `underlyingVaultShare` in `lend` function, it not stored directly lended amount but the amount minted in softVault after this tokens lended to it. 
```solidity
    function lend(
        address token,
        uint256 amount
    ) external override inExec poke(token) onlyWhitelistedToken(token) {
        ...


        if (_isSoftVault(token)) {
            _ensureApprove(token, bank.softVault, amount);
            pos.underlyingVaultShare += ISoftVault(bank.softVault).deposit(
                amount
            );
        } else {
            _ensureApprove(token, bank.hardVault, amount);
            pos.underlyingVaultShare += IHardVault(bank.hardVault).deposit(
                token,
                amount
            );
        }

```
SoftVault's `deposit` function call `mint` function of blueBerry bToken (which work same as compound cToken).And return minted amount of bToken. bToken mintAmount is `amount/exchangeRateStored` . So when calculate `underlyingAmount` , `pos.UnderlyinVaultShare` must be multiplied with btoken of underlyingToken's `exchangeRateStored`. However as seen above of the `getIsolatedColleteralValue()` code snippet it multiply with `debtToken`'s bToken not `underlyingToken`'s. Due to all bTokens' has different `exchangeRate`, `underlyingAmount` will be calculate wrong. 

 
## Vulnerability Detail

## Impact
misCalculation of `getIsolatedColleteralValue()` so `cv` in `getPositionRisk`.This will lead some protocols can be 
## Code Snippet
https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/BlueBerryBank.sol#L493
## Tool used

Manual Review

## Recommendation
use underlying token's bToken exchangeRate in `getIsolatedColleteralValue()` not debtToken's.





## Discussion

**merteren1234**

Escalate
I think this issue is not duplicate of #16. Because this issue indicate differeant root cause than #16. This issue's root cause is , wrong bToken's exchangeRate used in getIsolatedColleteral().Not about old exchangeRate used or not. Moreover without fixing this issue , #16 is not valid because pos.debtToken is updated with poke modifier in liquidate function and its exchangeRate used in getIsolatedColleteralValue() .
This issue is about getIsolatedColleteralValue() use debtToken's exchangeRate which is wrong because this functions should use underlyingToken's exchangeRate. So even though #16 is fixed  (there will be no impact without fix this issue.) this issue still remain and changing `ICErc20(banks[pos.debtToken].bToken).exchangeRateStored()` with `ICErc20(banks[pos.underlyingToken].bToken).exchangeRateStored()`   is mandatory to take correct getIsolatedColleteralValue().


**sherlock-admin2**

 > Escalate
> I think this issue is not duplicate of #16. Because this issue indicate differeant root cause than #16. This issue's root cause is , wrong bToken's exchangeRate used in getIsolatedColleteral().Not about old exchangeRate used or not. Moreover without fixing this issue , #16 is not valid because pos.debtToken is updated with poke modifier in liquidate function and its exchangeRate used in getIsolatedColleteralValue() .
> This issue is about getIsolatedColleteralValue() use debtToken's exchangeRate which is wrong because this functions should use underlyingToken's exchangeRate. So even though #16 is fixed  (there will be no impact without fix this issue.) this issue still remain and changing `ICErc20(banks[pos.debtToken].bToken).exchangeRateStored()` with `ICErc20(banks[pos.underlyingToken].bToken).exchangeRateStored()`   is mandatory to take correct getIsolatedColleteralValue().
> 

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Shogoki**

Not a duplicate of #16 

**hrishibhat**

@Shogoki is this a valid issue on its own?

**nevillehuang**

> @Shogoki is this a valid issue on its own?

@Gornutz 

**hrishibhat**

Result:
High
Unique
After further review and discussions considering this a valid high issue as the exchange rate can be affected significantly by the tokens decimal and this is not a duplicate of #16 

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [merteren1234](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/117/#issuecomment-1693539478): accepted

# Issue M-1: Users will fail to close their Convex position if the Curve pool is killed 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/15 

## Found by 
Arz

If the Curve pool that the ConvexSpell.sol uses becomes killed, users will be unable to close their position because remove_liquidity_one_coin() will revert. Users will be unable to repay their debt so their assets will be liquidated.

## Vulnerability Detail

ConvexSpell.sol::closePositionFarm() is used to close an existing liquidity position. After the collateral is taken and the rewards are swapped, _removeLiquidity() is called which removes liquidity from a Curve pool by calling remove_liquidity_one_coin(). 


The problem arises when the Curve pool is killed(paused) so if self.is_killed in the curve pool contract is true, calling remove_liquidity_one_coin() function will always revert and closePositionFarm() function will be DOS'ed


Note: This issue was submitted in the [previous contest](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/64) however only the CurveSpell got fixed but not the ConvexSpell.


## Impact

When user's position is about to be liquidated, if the closePositionFarm() function is DOS'ed,users will be unable to repay their debt, resulting in the users losing their funds

## Code Snippet

https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/spell/ConvexSpell.sol#L261-L265

```solidity

260: /// Removes liquidity from the Curve pool for the specified token.
261: ICurvePool(pool).remove_liquidity_one_coin(
262:     amountPosRemove,
263:     int128(tokenIndex),
264:     param.amountOutMin
265: );

```

As you can see, remove_liquidity_one_coin() is called when the user calls closePositionFarm() in the ConvexSpell


https://github.com/curvefi/curve-contract/blob/b0bbf77f8f93c9c5f4e415bce9cd71f0cdee960e/contracts/pools/3pool/StableSwap3Pool.vy#L670-L674

```python
670: def remove_liquidity_one_coin(_token_amount: uint256, i: int128, min_amount: uint256):
671:     """
672:     Remove _amount of liquidity all in a form of coin i
673:     """
674:     assert not self.is_killed  # dev: is killed
```


The problem is that remove_liquidity_one_coin() checks if self.is_killed is true so if the Curve pool is killed then this will revert and the user wont be able to close his position.


## Tool used

Manual Review

## Recommendation

When killed, it is only possible for existing LPs to remove liquidity via remove_liquidity so there should be the same isKilled check in the ConvexSpell like there is in the CurveSpell



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**0xyPhilic** commented:
> invalid because the case where a Curve pool is killed is handled by withdrawing liquidity using remove_liquidity function

**darkart** commented:
>  If its already been accepted its up to the DEV to decide if he wants to fix it or no



# Issue M-2: `getPrice` in `WeightedBPTOracle.sol`  uses `totalSupply` for price calculations which can lead to wrong results 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/18 

## Found by 
Vagner
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**YakuzaKiawe** commented:
>  invalid as this could be a factor to exploit the pool in future



# Issue M-3: ConvexSpell/CurveSpell.openPositionFarm will revert in some cases 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/60 

## Found by 
nobody2018

Before calling `ICurvePool#add_liquidity` to deposit tokens, each token needs to be approved to the pool. If the approved amount is smaller than the parameters passed into `ICurvePool#add_liquidity`, it will cause `erc20.transferFrom` revert inside the function. In this way, `openPositionFarm` will also revert.

In previous contest, [[#47](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/47)](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/47) was not fixed correctly. **This issue is still exist**.

## Vulnerability Detail

The fix for this issue from this contest is as following:

```solidity
File: blueberry-core\contracts\spell\CurveSpell.sol
098:         // 2. Borrow specific amounts
099:         uint256 borrowBalance = _doBorrow(
100:             param.borrowToken,
101:             param.borrowAmount
102:         );
103: 
104:         // 3. Add liquidity on curve
105:         address borrowToken = param.borrowToken;
106:         _ensureApprove(param.borrowToken, pool, borrowBalance);
107:         if (tokens.length == 2) {
108:             uint256[2] memory suppliedAmts;
109:             for (uint256 i = 0; i < 2; i++) {
                     //this 'if' check is the fix from the previous contest
110:->               if (tokens[i] == borrowToken) {
111:                     suppliedAmts[i] = IERC20Upgradeable(tokens[i]).balanceOf(
112:                         address(this)
113:                     );
114:                     break;
115:                 }
116:             }
117:             ICurvePool(pool).add_liquidity(suppliedAmts, minLPMint);
118:         } else if (tokens.length == 3) {
```

**The key to this issue is that `borrowBalance` may be smaller than `IERC20Upgradeable(borrowToken).balanceOf(address(this))`**. For simplicity, assume that CurveSpell supports an lptoken which contains two tokens : A and B.

**Bob transferred 1wei of A and B to the CurveSpell contract**. Alice opens a position by calling `BlueBerryBank#execute`, and the flow is as follows:

1.  enter `CurveSpell#openPositionFarm`.
2.  call `_doLend` to deposit isolated collaterals.
3.  call `_doBorrow` to borrow 100e18 A token. borrowBalance = 100e18.
4.  `A.approve(pool, 100e18)`.
5.  `suppliedAmts[0] = A.balance(address(this)) = 100e18+1wei`, `suppliedAmts[1] = 0`.
6.  call `ICurvePool(pool).add_liquidity(suppliedAmts, minLPMint)`, then revert because the approved amount is not enough.

Therefore, no one can successfully open a position.

Of course, bob can also transfer 1wei of `borrowToken` to contract by front-running `openPositionFarm` for a specific user or all users.

In ConvexSpell, the issue lies in [[the same code](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/ConvexSpell.sol#L110-L144)](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/ConvexSpell.sol#L110-L144) as in previous games.

## Impact

`ConvexSpell/CurveSpell.openPositionFarm` will revert due to this issue.

## Code Snippet

https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/CurveSpell.sol#L99-L140

https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/ConvexSpell.sol#L110-L144

## Tool used

Manual Review

## Recommendation

**The following fix is for CurveSpell, but please don't forget ConvexSpell**.

Two ways for fix it:

```fix
--- a/blueberry-core/contracts/spell/CurveSpell.sol
+++ b/blueberry-core/contracts/spell/CurveSpell.sol
@@ -108,9 +108,7 @@ contract CurveSpell is BasicSpell {
             uint256[2] memory suppliedAmts;
             for (uint256 i = 0; i < 2; i++) {
                 if (tokens[i] == borrowToken) {
-                    suppliedAmts[i] = IERC20Upgradeable(tokens[i]).balanceOf(
-                        address(this)
-                    );
+                    suppliedAmts[i] = borrowBalance;
                     break;
                 }
             }
@@ -119,9 +117,7 @@ contract CurveSpell is BasicSpell {
             uint256[3] memory suppliedAmts;
             for (uint256 i = 0; i < 3; i++) {
                 if (tokens[i] == borrowToken) {
-                    suppliedAmts[i] = IERC20Upgradeable(tokens[i]).balanceOf(
-                        address(this)
-                    );
+                    suppliedAmts[i] = borrowBalance;
                     break;
                 }
             }
@@ -130,9 +126,7 @@ contract CurveSpell is BasicSpell {
             uint256[4] memory suppliedAmts;
             for (uint256 i = 0; i < 4; i++) {
                 if (tokens[i] == borrowToken) {
-                    suppliedAmts[i] = IERC20Upgradeable(tokens[i]).balanceOf(
-                        address(this)
-                    );
+                    suppliedAmts[i] = borrowBalance;
                     break;
                 }
             }
```

```fix
--- a/blueberry-core/contracts/spell/CurveSpell.sol
+++ b/blueberry-core/contracts/spell/CurveSpell.sol
@@ -103,7 +103,8 @@ contract CurveSpell is BasicSpell {

         // 3. Add liquidity on curve
         address borrowToken = param.borrowToken;
-        _ensureApprove(param.borrowToken, pool, borrowBalance);
+        require(borrowBalance <= IERC20Upgradeable(borrowToken).balanceOf(address(this)), "impossible");
+        _ensureApprove(param.borrowToken, pool, IERC20Upgradeable(borrowToken).balanceOf(address(this)));
         if (tokens.length == 2) {
             uint256[2] memory suppliedAmts;
             for (uint256 i = 0; i < 2; i++) {
```



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**0xyPhilic** commented:
> invalid because the borrowAmount can't change mid function execution

**darkart** commented:
>  If already approven its up to Developrs to chose if they will implement it



# Issue M-4: Mainnet oracles are incompatible with wstETH causing many popular yields strategies to be broken 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/96 

## Found by 
0x52

Mainnet oracles are incompatible with wstETH causing many popular yields strategies to be broken. Chainlink and Band do not have wstETH oracles and using Uniswap LP pairs would be very dangerous given their low liquidity. 

## Vulnerability Detail

[ChainlinkAdapterOracle.sol#L111-L125](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/ChainlinkAdapterOracle.sol#L111-L125)

        uint256 decimals = registry.decimals(token, USD);
        (
            uint80 roundID,
            int256 answer,
            ,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = registry.latestRoundData(token, USD);
        if (updatedAt < block.timestamp - maxDelayTime)
            revert Errors.PRICE_OUTDATED(token_);
        if (answer <= 0) revert Errors.PRICE_NEGATIVE(token_);
        if (answeredInRound < roundID) revert Errors.PRICE_OUTDATED(token_);

        return
            (answer.toUint256() * Constants.PRICE_PRECISION) / 10 ** decimals;

ChainlinkAdapterOracle only supports single asset price data. This makes it completely incompatible with wstETH because chainlink doesn't have a wstETH oracle on mainnet. Additionally Band protocol doesn't offer a wstETH oracle either. This only leaves Uniswap oracles which are highly dangerous given their low liquidity.

## Impact

Mainnet oracles are incompatible with wstETH causing many popular yields strategies to be broken

## Code Snippet

[ChainlinkAdapterOracle.sol#L102-L126](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/oracle/ChainlinkAdapterOracle.sol#L102-L126)

## Tool used

Manual Review

## Recommendation

Create a special bypass specifically for wstETH utilizing the stETH oracle and it's current exchange rate. 



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**0xyPhilic** commented:
> invalid because this issue can be considered informational or at best low - tokens used are whitelisted

**Kral01** commented:
> low severity



**IAm0x52**

Escalate

This was wrongly excluded. Protocol is meant to be compatible with these pools but can't work with them. I believe this is a valid medium because the protocol is nonfunctional in this area.

**sherlock-admin2**

 > Escalate
> 
> This was wrongly excluded. Protocol is meant to be compatible with these pools but can't work with them. I believe this is a valid medium because the protocol is nonfunctional in this area.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Shogoki**

> Escalate
> 
> This was wrongly excluded. Protocol is meant to be compatible with these pools but can't work with them. I believe this is a valid medium because the protocol is nonfunctional in this area.

Not sure were it says that the protcol is meant to be compatible with `WSTETH` pools on mainnet. 

If that is the case it can be a valid issue, i guess. 
However if it is not, i think the Whitelisting of tokens would count for invalidating it.

**IAm0x52**

Protocol is meant to be compatible with Aura/Convex. wstETH is a component of many highly attractive pools. Not being able to support wstETH as an underlying asset will break support for these.

**hrishibhat**

@Gornutz 

**Gornutz**

Confirm this is valid. 

**hrishibhat**

Result:
Medium
Unique
Considering this issue a valid medium


**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [IAm0x52](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/96/#issuecomment-1694747143): accepted

# Issue M-5: AuraSpell#closePositionFarm exits pool with single token and without any slippage protection 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/102 

## Found by 
0x52, Breeje, Oxhunter526, Vagner, bitsurfer, nobody2018

When exiting the balancer pool, vault#exitPool is called with an empty array for minAmountsOut causing the position to be exited with no slippage protection. Typically it is not an issue to exit off axis but since it is exiting to a single token this can cause massive loss.

## Vulnerability Detail

[AuraSpell.sol#L221-L236](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L221-L236)

                (
                    uint256[] memory minAmountsOut,
                    address[] memory tokens,
                    uint256 borrowTokenIndex
                ) = _getExitPoolParams(param.borrowToken, lpToken);

                wAuraPools.getVault(lpToken).exitPool(
                    IBalancerPool(lpToken).getPoolId(),
                    address(this),
                    address(this),
                    IBalancerVault.ExitPoolRequest(
                        tokens,
                        minAmountsOut,
                        abi.encode(0, amountPosRemove, borrowTokenIndex),
                        false
                    )

When exiting a the balancer vault, closePositionFarm makes a subcall to _getExitPoolParams which is used to set minAmountsOut.

[AuraSpell.sol#L358-L361](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L358-L361)

        (address[] memory tokens, , ) = wAuraPools.getPoolTokens(lpToken);

        uint256 length = tokens.length;
        uint256[] memory minAmountsOut = new uint256[](length);

Inside _getExitPoolParams we see that minAmountsOut are always an empty array. This means that the user has no slippage protection and can be sandwich attacked, suffering massive losses.

## Impact

Exits can be sandwich attacked causing massive loss to the user

## Code Snippet

[AuraSpell.sol#L184-L286](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L184-L286)

## Tool used

Manual Review

## Recommendation

Allow user to specify min amount received from exit



## Discussion

**securitygrid**

Escalate:
Historically, lacking slippage protection is H.


**sherlock-admin2**

 > Escalate:
> Historically, lacking slippage protection is H.
> 

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Shogoki**

> Escalate: Historically, lacking slippage protection is H.

Can be a high issue

**IAm0x52**

Why would lack of slippage be considered high in this scenario? Even with zero slippage protection sandwich attack profitability is always contingent on a few external factors such as: liquidity of underlying pool, the fee of the underlying pool, the amount being swapped and the current gas prices. The higher the liquidity and fee of the pool the higher the cost to push the pool price then pull it back. The lower the amount being swapped, the less the attacker can steal. If this cost exceeds the gain from the attack then the attack isn't profitable and won't happen.

**VagnerAndrei26**

I think no slippage is considered high most of the time cause sandwich attacks are easily doable in the space, especially for those experienced in doing it, and even if there are factors to consider even one successful can occur loss of funds for the protocol in a pretty easy manner. So considering that and also past contests I think it should be considered also a high.

**Nabeel-javaid**

as far as I know 98% of the times slippage issues are considered as Medium severity

**hrishibhat**

Result:
Medium
Has duplicates 
This is a valid medium issue. Agree with the Lead Watson's comments here:
https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/102#issuecomment-1699649100


**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [securitygrid](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/102/#issuecomment-1693593255): rejected

# Issue M-6: AuraSpell#closePositionFarm will take reward fees on underlying tokens when borrow token is also a reward 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/103 

## Found by 
0x52

AuraSpell#AuraSpell#closePositionFarm will take reward fees on underlying tokens when borrow token is also a reward. This is because the BLP is burned before taking the reward cut.

## Vulnerability Detail

[AuraSpell.sol#L227-L247](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L227-L247)

                wAuraPools.getVault(lpToken).exitPool(
                    IBalancerPool(lpToken).getPoolId(),
                    address(this),
                    address(this),
                    IBalancerVault.ExitPoolRequest(
                        tokens,
                        minAmountsOut,
                        abi.encode(0, amountPosRemove, borrowTokenIndex),
                        false
                    )
                );
            }
        }

        /// 4. Swap each reward token for the debt token
        uint256 rewardTokensLength = rewardTokens.length;
        for (uint256 i; i != rewardTokensLength; ) {
            address sellToken = rewardTokens[i];
            if (sellToken == STASH_AURA) sellToken = AURA;

            _doCutRewardsFee(sellToken);

We can see above that closePositionFarm redeems the BLP before it takes the reward cut. This can cause serious issues. If there is any overlap between the reward tokens and the borrow token then _doCutRewardsFee will take a cut of the underlying liquidity. This causes loss to the user as too many fees are taken from them.

## Impact

User will lose funds due to incorrect fees

## Code Snippet

[AuraSpell.sol#L184-L265](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L184-L265)

## Tool used

Manual Review

## Recommendation

Use the same order as ConvexSpell and sell rewards BEFORE burning BLP

# Issue M-7: Adversary can abuse hanging approvals left by PSwapLib.swap to bypass reward fees 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/104 

## Found by 
0x52

PSwapLib.swap can be manipulated to leave hanging allowances via the expectedRewards input. These can then be abused to swap reward tokens out of order allowing them to bypass fees.

## Vulnerability Detail

[AuraSpell.sol#L247-L257](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L247-L257)

            _doCutRewardsFee(sellToken);
            if (
                expectedRewards[i] != 0 &&
                !PSwapLib.swap(
                    augustusSwapper,
                    tokenTransferProxy,
                    sellToken,
                    expectedRewards[i],
                    swapDatas[i]
                )
            ) revert Errors.SWAP_FAILED(sellToken);

AuraSpell#closePositionFarm allows the user to specify any expectedRewards they wish. This allows the user to approve any amount, even if the amount is much larger than they would otherwise use. The can abuse these hanging approvals to swap tokens out of order and avoid paying reward fees.

Example:
Assume there are two rewards, token A and token B. Over time a user's position accumulates 100 rewards for each token. Normally the user would have to pay fees on those rewards. However they can bypass it by first creating hanging approvals. The user would start by redeeming a very small amount of LP and setting expectedRewards to uint256.max. They wouldn't sell the small amount leaving a very large approval left for both tokens. Now the user withdraws the rest of their position. This time they specify the swap data to swap token B first. The user still has to pay fees on token A but now they have traded token B before any fees can be taken on it. 

## Impact

User can bypass reward fees

## Code Snippet

[AuraSpell.sol#L184-L286](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L184-L286)

[ConvexSpell.sol#L179-L229](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/ConvexSpell.sol#L179-L229)

## Tool used

Manual Review

## Recommendation

After the swap reset allowances to 0



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**0xyPhilic** commented:
> invalid because fees are cut before the supplied check in _doCutFees function

**Kral01** commented:
> Not how approval works



**IAm0x52**

Escalate

This was wrongly excluded. The judging comments above are incorrect. The code allows the user to make any swap in whatever order they want. This is what allows the user to bypass the fees. The user is meant to swap in the order that the tokens are listed but by swapping in a different order they can bypass the cut. Assume we have two tokens to claim. Normally the flow would be:

RewardCut A > Swap A > RewardCut B > Swap B

Using the methodology I've shown above the user can instead:

RewardCut A > Swap B > RewardCutB > Swap A

Here we see that B can be swapped before the reward cut allowing the fee to be bypassed.

**sherlock-admin2**

 > Escalate
> 
> This was wrongly excluded. The judging comments above are incorrect. The code allows the user to make any swap in whatever order they want. This is what allows the user to bypass the fees. The user is meant to swap in the order that the tokens are listed but by swapping in a different order they can bypass the cut. Assume we have two tokens to claim. Normally the flow would be:
> 
> RewardCut A > Swap A > RewardCut B > Swap B
> 
> Using the methodology I've shown above the user can instead:
> 
> RewardCut A > Swap B > RewardCutB > Swap A
> 
> Here we see that B can be swapped before the reward cut allowing the fee to be bypassed.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Shogoki**

> Escalate
> 
> This was wrongly excluded. The judging comments above are incorrect. The code allows the user to make any swap in whatever order they want. This is what allows the user to bypass the fees. The user is meant to swap in the order that the tokens are listed but by swapping in a different order they can bypass the cut. Assume we have two tokens to claim. Normally the flow would be:
> 
> RewardCut A > Swap A > RewardCut B > Swap B
> 
> Using the methodology I've shown above the user can instead:
> 
> RewardCut A > Swap B > RewardCutB > Swap A
> 
> Here we see that B can be swapped before the reward cut allowing the fee to be bypassed.

Agree with escalation. As the user can specify any `expectedRewards` and any `callData` in any order, this is a valid attack vector.

**hrishibhat**

@Gornutz 

**Gornutz**

The paraswap lib is already resetting the token approval to zero prior to allocating the allowance for each swap - https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/libraries/Paraswap/PSwapLib.sol#L15 


**Shogoki**

> The paraswap lib is already resetting the token approval to zero prior to allocating the allowance for each swap - https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/libraries/Paraswap/PSwapLib.sol#L15

I do not think this helps preventing this issue. As the approval is resetted before the swap, there still can be a hanging approval.
Issue is that the approval is set/reset for the specified sellToken, but the user can swap any other token by specifying different calldatas. Therefore, i think the stated attack path is still possible

**Gornutz**

Gotcha, then yes seems valid 

**hrishibhat**

Result:
Medium
Unique
This is a valid medium issue

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [IAm0x52](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/104/#issuecomment-1694746004): accepted

# Issue M-8: ConvexSpell is completely broken for any curve LP that utilizes native ETH 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/105 

## Found by 
0x52, 0xjoseph

When a Curve pool utilizes native ETH it uses the address `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee`. This is problematic because it will try to call balanceOf on this address which will always revert.

## Vulnerability Detail

[ConvexSpell.sol#L120-L127](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/ConvexSpell.sol#L120-L127)

        if (tokens.length == 2) {
            uint256[2] memory suppliedAmts;
            for (uint256 i; i != 2; ++i) {
                suppliedAmts[i] = IERC20Upgradeable(tokens[i]).balanceOf(
                    address(this)
                );
            }
            ICurvePool(pool).add_liquidity(suppliedAmts, minLPMint);

ConvexSpell#openPositionFarm attempts to call balanceOf on each component of the LP. Since native ETH uses the `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` this call will always revert. This breaks compatibility with EVERY curve pool that uses native ETH which make most of the highest volume pools on the platfrom.

## Impact

ConvexSpell is completely incompatible with a majority of Curve pools

## Code Snippet

[ConvexSpell.sol#L92-L173](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/ConvexSpell.sol#L92-L173)

## Tool used

Manual Review

## Recommendation

I would recommend conversion between native ETH and wETH to prevent this issue.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**0xyPhilic** commented:
> invalid because the design of the protocol does not show desire for ETH compatibility so the issue can be classified as informational



# Issue M-9: Issue #47 from Update #1 is still present in ConvexSpell 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/106 

## Found by 
0x52

Issue [47](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/47) from update contest 1 was only fixed for CurveSpell but wasn't fixed for ConvexSpell.

## Vulnerability Detail

See issue [47](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/47)

## Impact

See issue [47](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/47)

## Code Snippet

[ConvexSpell.sol#L92-L173](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/ConvexSpell.sol#L92-L173)

## Tool used

Manual Review

## Recommendation

See issue [47](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/47)




## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**darkart** commented:
>  Its up to Devs if they wanna fix it



**IAm0x52**

Escalate

This is a completely different issue than #42. This was the same issue from the previous contest. They dev simply missed the occurrence in convex spell. This is a valid med and needs to be fixed.

**sherlock-admin2**

 > Escalate
> 
> This is a completely different issue than #42. This was the same issue from the previous contest. They dev simply missed the occurrence in convex spell. This is a valid med and needs to be fixed.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Shogoki**

> Escalate
> 
> This is a completely different issue than #42. This was the same issue from the previous contest. They dev simply missed the occurrence in convex spell. This is a valid med and needs to be fixed.

Agree, not a duplicate #42. This is an issue from the previous contest that was fixed, but is here found at a different place. Can be a valid finding on its own.

**IAm0x52**

Missed that this should be a dupe of #60 

**hrishibhat**

Result:
Medium
Unique

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [IAm0x52](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/106/#issuecomment-1694745204): accepted

**hrishibhat**

Considering this issue a valid issue on its own. #60 talks of the fix that was implemented in the previous contest, while this issue points to the original underlying issue that still existed in one of the contracts. 
This is a unique situation, hence both the issues are treated separately 

# Issue M-10: WAuraPools doesn't correctly account for AuraStash causing all deposits to be permanently lost 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/108 

## Found by 
0x52

Some Aura pools have two sources of AURA. First from the booster but also as a secondary reward. This secondary reward is stash AURA that doesn't behave like regular AURA. Although properly accounted for in AuraSpell, it is not properly accounted for in WAuraPools, resulting in all deposits being unrecoverable. 

## Vulnerability Detail

[WAuraPools.sol#L413-L418](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L413-L418)

        uint256 rewardTokensLength = rewardTokens.length;
        for (uint256 i; i != rewardTokensLength; ) {
            IERC20Upgradeable(rewardTokens[i]).safeTransfer(
                msg.sender,
                rewards[i]
            );

When burning the wrapped LP token, it attempts to transfer each token to msg.sender. The problem is that stash AURA cannot be transferred like an regular ERC20 token and any transfers will revert. Since this will be called on every attempted withdraw, all deposits will be permanently unrecoverable.

## Impact

All deposits will be permanently unrecoverable

## Code Snippet

[WAuraPools.sol#L360-L424](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L360-L424)

## Tool used

Manual Review

## Recommendation

Check if reward is stash AURA and send regular AURA instead similar to what is done in AuraSpell.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**Kral01** commented:
> Needs PoC



**IAm0x52**

Escalate

This was incorrectly excluded. 

https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/spell/AuraSpell.sol#L243-L257

        for (uint256 i; i != rewardTokensLength; ) {
            address sellToken = rewardTokens[i];
            if (sellToken == STASH_AURA) sellToken = AURA;

            _doCutRewardsFee(sellToken);
            if (
                expectedRewards[i] != 0 &&
                !PSwapLib.swap(
                    augustusSwapper,
                    tokenTransferProxy,
                    sellToken,
                    expectedRewards[i],
                    swapDatas[i]
                )
            ) revert Errors.SWAP_FAILED(sellToken);

            /// Refund rest (dust) amount to owner
            _doRefund(sellToken);

            unchecked {
                ++i;
            }

AuraSpell requires the above code to prevent this. WAuraPools uses the exact same reward list and needs the same protections. The result is that funds will be permanently trapped, because the transfer will always fail.

**sherlock-admin2**

 > Escalate
> 
> This was incorrectly excluded. 
> 
> https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/spell/AuraSpell.sol#L243-L257
> 
>         for (uint256 i; i != rewardTokensLength; ) {
>             address sellToken = rewardTokens[i];
>             if (sellToken == STASH_AURA) sellToken = AURA;
> 
>             _doCutRewardsFee(sellToken);
>             if (
>                 expectedRewards[i] != 0 &&
>                 !PSwapLib.swap(
>                     augustusSwapper,
>                     tokenTransferProxy,
>                     sellToken,
>                     expectedRewards[i],
>                     swapDatas[i]
>                 )
>             ) revert Errors.SWAP_FAILED(sellToken);
> 
>             /// Refund rest (dust) amount to owner
>             _doRefund(sellToken);
> 
>             unchecked {
>                 ++i;
>             }
> 
> AuraSpell requires the above code to prevent this. WAuraPools uses the exact same reward list and needs the same protections. The result is that funds will be permanently trapped, because the transfer will always fail.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Shogoki**

Seems valid
STASH AURA seems to be a valid `extraRewardToken` but transfer can only be called from the pool, which will actually transfer AURA instead. 
So if the WAURAPool tries to call transfer it will revert.

**hrishibhat**

Result:
Medium
Unique 
Considering this issue a valid medium

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [IAm0x52](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/108/#issuecomment-1694744574): accepted

# Issue M-11: WConvexPool.sol will be broken on Arbitrum due to improper integration with Convex Arbitrum contracts 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/119 

## Found by 
Kow
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

# Issue M-12: AuraSpell `openPositionFarm` will revert when the tokens contains `lpToken` 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/125 

## Found by 
bitsurfer, nobody2018, twcctop

AuraSpell `openPositionFarm` will revert when the tokens contains lpToken due to array length mismatch

## Vulnerability Detail

In AuraSpell, the `openPositionFarm` will call `joinPool` in Balancer's vault. But when analyzing the `JoinPoolRequest` struct, we see issue on `maxAmountsIn` and `amountsIn` which can be in different length, thus this will be reverted since in Balancer's vault, this two array should be in the same length.

```js
File: AuraSpell.sol
088:     function openPositionFarm(
089:         OpenPosParam calldata param,
090:         uint256 minimumBPT
091:     )
...
095:     {
...
110:         /// 3. Add liquidity to the Balancer pool and receive BPT in return.
111:         {
...
128:             if (poolAmountOut != 0) {
129:                 vault.joinPool(
130:                     wAuraPools.getBPTPoolId(lpToken),
131:                     address(this),
132:                     address(this),
133:                     IBalancerVault.JoinPoolRequest({
134:                         assets: tokens,
135:                         maxAmountsIn: maxAmountsIn,
136:                         userData: abi.encode(1, amountsIn, _minimumBPT),
137:                         fromInternalBalance: false
138:                     })
139:                 );
140:             }
141:         }
...
178:     }
...
296:     function _getJoinPoolParamsAndApprove(
297:         address vault,
298:         address[] memory tokens,
299:         uint256[] memory balances,
300:         address lpToken
301:     ) internal returns (uint256[] memory, uint256[] memory, uint256) {
...
304:         uint256 length = tokens.length;
305:         uint256[] memory maxAmountsIn = new uint256[](length);
306:         uint256[] memory amountsIn = new uint256[](length);
307:         bool isLPIncluded;
308:
309:         for (i; i != length; ) {
310:             if (tokens[i] != lpToken) {
311:                 amountsIn[j] = IERC20(tokens[i]).balanceOf(address(this));
312:                 if (amountsIn[j] > 0) {
313:                     _ensureApprove(tokens[i], vault, amountsIn[j]);
314:                 }
315:                 ++j;
316:             } else isLPIncluded = true;
317:
318:             maxAmountsIn[i] = IERC20(tokens[i]).balanceOf(address(this));
319:
320:             unchecked {
321:                 ++i;
322:             }
323:         }
324:
325:         if (isLPIncluded) {
326:             assembly {
327:                 mstore(amountsIn, sub(mload(amountsIn), 1))
328:             }
329:         }
...
345:         return (maxAmountsIn, amountsIn, poolAmountOut);
346:     }
```

these `maxAmountsIn` and `amountsIn` are coming from `_getJoinPoolParamsAndApprove`. And by seeing the function, we can see that there is possible issue when the `tokens[i] == lpToken`.

When `tokens[i] == lpToken`, the flag `isLPIncluded` will be true. And will enter this block,

```js
325:         if (isLPIncluded) {
326:             assembly {
327:                 mstore(amountsIn, sub(mload(amountsIn), 1))
328:             }
329:         }
```

this will decrease the `amountsIn` length. Thus, `amountsIn` and `maxAmountsIn` will be in different length.

In Balancer's `JoinPoolRequest` struct, the `maxAmountsIn`, and `userData` second decoded bytes (`amountsIn`) should be the same array length, because it will be checked in Balancer.

```js
133:                     IBalancerVault.JoinPoolRequest({
134:                         assets: tokens,
135:                         maxAmountsIn: maxAmountsIn,
136:                         userData: abi.encode(1, amountsIn, _minimumBPT),
137:                         fromInternalBalance: false
138:                     })
```

Therefore, in this situation, it will be reverted.

## Impact

User can't open position on AuraSpell when `tokens` contains `lpToken`

## Code Snippet

https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/spell/AuraSpell.sol#L326-L328

## Tool used

Manual Review

## Recommendation

Remove the assembly code where it will decrease the `amountsIn` length when `isLPIncluded` is true to make sure the array length are same.

# Issue M-13: approve() call with incorrect function signature will make any SoftVault deployed with USDT as the underlying token unusable 

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/135 

## Found by 
n1punp, trauki

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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**0xyPhilic** commented:
> invalid because the IERC20 interface would fit also USDT as it has an approve function - no function signature is used anywhere



**Gornutz**

Item discussed in the previous competition full context is given here - https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/15#issuecomment-1542727941 

**Shogoki**

> Item discussed in the previous competition full context is given here - [sherlock-audit/2023-04-blueberry-judging#15 (comment)](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/15#issuecomment-1542727941)

Actually i do not understand why the issue was closed in the last contest with the comment: 

`USDT only can call approve function when the allowance is zero or to set the allowance to zero first.`

This is not related to the stated issue, that the approval will revert because the generic IERC20 interface is used, which is expecting a bool return data.

**nevillehuang**

Escalate:

Referring to @Gornutz comment, this was marked as No-Fix and Non-Reward in previous contest, and should be invalid according to sherlock's [rules here](https://docs.sherlock.xyz/audits/judging/judging):

> In an update contest, issues from the previous contest with wont fix labels are not considered valid.

In addition, `EnsureApprove.sol` uses `_ensureApprove()` directly to approve tokens  (for e.g. USDT) for contracts inheriting it, so the issue above of needing an interface is not a problem.

**sherlock-admin2**

 > Escalate:
> 
> Referring to @Gornutz comment, this was marked as No-Fix and Non-Reward in previous contest, and should be invalid according to sherlock's [rules here](https://docs.sherlock.xyz/audits/judging/judging):
> 
> > In an update contest, issues from the previous contest with wont fix labels are not considered valid.
> 
> In addition, `EnsureApprove.sol` uses `_ensureApprove()` directly to approve tokens  (for e.g. USDT) for contracts inheriting it, so the issue above of needing an interface is not a problem.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Shogoki**

> Escalate:
> 
> Referring to @Gornutz comment, this was marked as No-Fix and Non-Reward in previous contest, and should be invalid according to sherlock's [rules here](https://docs.sherlock.xyz/audits/judging/judging):
> 
> > In an update contest, issues from the previous contest with wont fix labels are not considered valid.
> 

Hmm, yep that is kind of an interesting case. I agree, that it is probably fair in terms of rewards to consider this invalid.
However as stated in my comment above, i do not understand the reasoning for closing this in the initial contest.

**securitygrid**

The function signature only needs the function name and parameters, and does not need a return value.

**Shogoki**

> The function signature only needs the function name and parameters, and does not need a return value.

While this is true for the call, it is still a problem as the transaction will revert because Solidity checks for the expected return size and compares it to the actual returned data size. 


**nevillehuang**

> > The function signature only needs the function name and parameters, and does not need a return value.
> 
> While this is true for the call, it is still a problem as the transaction will revert because Solidity checks for the expected return size and compares it to the actual returned data size.

As @securitygrid said, there will be no reverts, given allowance is approved to zero first. The interface only requires the function selector to match and call the function in the USDT contract [check this link here](https://medium.com/coinmonks/function-selectors-in-solidity-understanding-and-working-with-them-25e07755e976#:~:text=The%20function%20signature%20is%20derived,myFunction(address%2Cuint256)%20.)

> When a contract is called, the EVM (Ethereum Virtual Machine) reads the first four bytes of the provided data to determine the function selector. The EVM uses this selector to match it with the correct function within the contract. If a match is found, the function is executed. If no match is found, the function call fails.

For example, function signature of `approve()` here would be `bytes4(keccak256("approve(address,uint)"));`

Unless there are two approve functions with different number of arguments, this submission seems like only a valid QA/low finding since there is only 1 approve function exposed in USDT contract

**securitygrid**

@nevillehuang 
This problem should be considered from the compiled code:
The compiled code of `IERC20(token).approve(spender, 0)` is similar to the following:
```solidity
(bool ret, bytes data) = token.call(abi.encodeWithSignature("approve(address,uint256)", spender, 0);
if (ret) {
     if (data.length != 1) // since usdt.approve has no return value, so data.length = 0
     {
            revert;
     }
     return abi.decode(data, (bool));
} else {
     revert;
}
```


**Shogoki**

> @nevillehuang This problem should be considered from the compiled code: The compiled code of `IERC20(token).approve(spender, 0)` is similar to the following:
> 
> ```solidity
> (bool ret, bytes data) = token.call(abi.encodeWithSignature("approve(address,uint256)", spender, 0);
> if (ret) {
>      if (data.length != 1) // since usdt.approve has no return vault, so data.length = 0
>      {
>             revert;
>      }
>      return abi.decode(data, (bool));
> } else {
>      revert;
> }
> ```

That is correct. 

To demonstrate the issue we can use the following litle PoC.

1. Creating a Contract `EnsureApproveTest.sol` 

```solidity
// SPDX-License-Identifier: MIT

pragma solidity 0.8.16;

import {EnsureApprove} from "./utils/EnsureApprove.sol" ;

contract EnsureApproveTest is EnsureApprove {

    function approveMe(address token, uint256 amount) external {
        _ensureApprove(token, msg.sender, amount);
    }
}
```

2. Creating a Test `ensureApprove.ts` 

```typescript
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import { ethers, upgrades } from "hardhat";
import chai, { expect } from "chai";
import { EnsureApproveTest, IERC20 } from "../typechain-types";
import { roughlyNear } from "./assertions/roughlyNear";
import { near } from "./assertions/near";
import { Contract } from "ethers";

chai.use(roughlyNear);
chai.use(near);

const USDT_ADDRESS = "0xdac17f958d2ee523a2206206994597c13d831ec7";
const USDC_ADDRESS = "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48";

describe("Ensure Approve", () => {
  let admin: SignerWithAddress;
  let alice: SignerWithAddress;
  let testContract: EnsureApproveTest;
  let usdt: Contract;
  let usdc: Contract;

  before(async () => {
    [admin, alice] = await ethers.getSigners();
  });

  beforeEach(async () => {
    const EnsureApproveTest = await ethers.getContractFactory("EnsureApproveTest");
    testContract = await EnsureApproveTest.deploy();
    usdt = await   ethers.getContractAt("@openzeppelin/contracts/token/ERC20/IERC20.sol:IERC20", USDT_ADDRESS);
    usdc = await   ethers.getContractAt("@openzeppelin/contracts/token/ERC20/IERC20.sol:IERC20", USDC_ADDRESS);
  });
  describe("IERC20 Interface", () => {
    it("TEST USDC APPROVAL", async () => {
        const approvalBefore = await usdc.allowance(testContract.address, alice.address);
        console.log("Allowance Before", approvalBefore);
        await testContract.connect(alice).approveMe(usdc.address, 1000);
        const approvalAfter = await usdc.allowance(testContract.address, alice.address);
        console.log("Allowance After", approvalAfter);
       expect(approvalAfter).to.equal(1000);
    });
    
    it("TEST USDT APPROVAL", async () => {
        const approvalBefore = await usdt.allowance(testContract.address, alice.address);
        console.log("Allowance Before", approvalBefore);
        await testContract.connect(alice).approveMe(usdt.address, 1000);
        const approvalAfter = await usdt.allowance(testContract.address, alice.address);
        console.log("Allowance After", approvalAfter);
        expect(approvalAfter).to.equal(1000);
    });
    
});

});
```

Running this will result in the 2nd test to fail, because of the mismatching return data length

```
1 failing

  1) Ensure Approve
       IERC20 Interface
         TEST USDT APPROVAL:
     Error: Transaction reverted: function returned an unexpected amount of data
```

**hrishibhat**

Result:
Medium
Has duplicates
Considering this a value medium issue based on the discussions above, this was a valid issue that was not acknowledged in the previous contests, but now with additional information is considered a valid medium. 
I agree the `Wont fix` rule might need to be more clear of the conditions to which they are applied

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [nevillehuang](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/135/#issuecomment-1693509481): accepted

