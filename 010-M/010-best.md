Electric Shadow Dinosaur

medium

# CurveSpell.closePositionFarm's missing _doCutRewardsFee calls before swapping reward tokens for users results in a loss of income for the treasury.
## Summary
`CurveSpell.closePositionFarm`'s missing `_doCutRewardsFee` calls before swapping reward tokens for users result in a loss of income for the treasury.

## Vulnerability Detail
Unlike the implementations of other Spell contracts, the `CurveSpell.closePositionFarm` lacks the equivalent step where the protocol should call the `_doCutRewardsFee` function before performing reward token swaps for users (At line 194 & 226). This missing step leads to the loss of eligible income for the treasury.

```solidity
// Findings are labeled with '<= FOUND'
// File: blueberry-core/contracts/spell/CurveSpell.sol
168:    function closePositionFarm(...)
178:    {
179:        ...
192:        {
193:            // 2. Swap rewards tokens to debt token
194:            _swapOnParaswap(CRV, amounts[0], swapDatas[0]); //<= FOUND: missing _doCutRewardsFee call before swapping reward tokens for user
195:        }
196:
197:        {
198:            ...
222:            if (isKilled) {
223:                uint256 len = tokens.length;
224:                for (uint256 i; i != len; ++i) {
225:                    if (tokens[i] != pos.debtToken) {
226:                        _swapOnParaswap( //<= FOUND: missing _doCutRewardsFee call before swapping reward tokens for user
227:                            tokens[i],
228:                            amounts[i + 1],
229:                            swapDatas[i + 1]
230:                        );
231:                    }
232:                }
233:            }
234:        }
236:        ...
255:    }
299:    function _swapOnParaswap(
300:        ...
315:        // Refund rest amount to owner
316:        _doRefund(token); //<= FOUND: token will be refunded to the position owner instead of the protocol's treasury
317:    }
```

## Impact
The treasury is supposed to receive a dedicated fee from the harvested rewards before they are distributed to users. However, due to the missing `_doCutRewardsFee` calls, this fee is not deducted. The rewards are then refunded to the position's owner via `_doRefund` inside `_swapOnParaswap` (Line 316), resulting in a direct loss of potential income for the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-07-blueberry/tree/main/blueberry-core/contracts/spell/CurveSpell.sol#L194
https://github.com/sherlock-audit/2023-07-blueberry/tree/main/blueberry-core/contracts/spell/CurveSpell.sol#L226

## Tool used
Manual Review

## Recommendation
Add the missing `_doCutRewardsFee` calls before the reward token swap in the `closePositionFarm` functions of the affected spells.

```solidity
// Findings are labeled with '<= FOUND'
// File: blueberry-core/contracts/spell/CurveSpell.sol
168:    function closePositionFarm(...)
178:    {
179:        ...
192:        {
193:            // 2. Swap rewards tokens to debt token
    +           _doCutRewardsFee(CRV);
194:            _swapOnParaswap(CRV, amounts[0], swapDatas[0]); // @audit amounts[0] & swapDatas[0] should already take into account for the reward cut amount
195:        }
196:
197:        {
198:            ...
222:            if (isKilled) {
223:                uint256 len = tokens.length;
224:                for (uint256 i; i != len; ++i) {
225:                    if (tokens[i] != pos.debtToken) {
    +                       _doCutRewardsFee(tokens[i]);
226:                        _swapOnParaswap( // @audit amounts[i + 1] & swapDatas[i + 1] should already take into account for the reward cut amount
227:                            tokens[i],
228:                            amounts[i + 1],
229:                            swapDatas[i + 1]
230:                        );
231:                    }
232:                }
233:            }
234:        }
236:        ...
255:    }
```