Howling Bronze Chicken

high

# Loss of rewards for all users due to mismatching reward multipliers for re-lock-ups

## Summary

After the first lock period expires, users who were initially locked for a long duration continue to earn rewards at the highest multiplier while being effectively locked only for the default (short) duration, causing a mismatch between lock-up duration and reward multiplier. 

This excessively high multipler causes other users to earn a much lower share of rewards due to their much lower (but correct) multiplier. This results in loss of reward for the users, disincentivises them from participating, and overpays rewards at high rates for short lock-up durations.

## Vulnerability Detail


The remaining unlock period is calculated in [`calcRemainUnlockPeriod`](https://github.com/sherlock-audit/2024-05-gamma-staking/blob/main/StakingV2/src/Lock.sol#L604-L605): once the original lock period has passed, the function starts using the default relock time to calculate the remaining unlock period:

```solidity
        } else {
            // If the current time exceeds the adjusted lock period, return the remaining time based on the default relock time.
            return defaultRelockTime - (block.timestamp - lockTime) % defaultRelockTime;
        }
```

However, while the user is now locked for a much shorter duration, **their rewards multiplier is not updated**, and they continue earning new rewards at much higher share than other users that are locked for similar durations.

Consider this example, using simple numbers for clarity:
- The default duration and the shortest duration are both 1 day. The longest lock duration is 10 days. 
- The multiplier for the 1 day lock-up is 1x and for 10 days is 10x.

1. **User A** completes one full duration of 10 days. Beyond that point, they are implicitly locked continuously for **1 day** (the default duration) while still earning rewards at the **10x multiplier**, [until they call `exitLateById`](https://github.com/sherlock-audit/2024-05-gamma-staking/blob/main/StakingV2/src/Lock.sol#L355) (which removes their balance).
2. **User B** who starts locking at the default 1 day duration is locked in the same way - can only withdraw tokens after **1 day**, but earns rewards at a much lower **1x multiplier** - one tenth of User A's multiplier.

If A and B are the only users, with same balances, from that point onward **A will earn 90.9% of rewards, and B only 9%**, despite having the exact same lock-up and balance.

While the **duration** calculation is intended design choice, and matches the documentation and README, the lack of adjustment of the **reward multipliers** to the duration logic is not documented and results in losses for new users. 

## Impact

This results in certain loss of rewards for all other users due to lower share of rewards relative to their lock-up.

In addition to the direct loss of rewards, it can significantly disincentivize new users from participating in the lock-staking contract. This is because the contract overpays the users with expired long locks, without receiving the benefit of a long lock-up period in return.

Additionally, since the contract's main purpose is staking at different reward rates for different lock-up durations, this breaks core functionality, due to the mismatch between durations and reward multipliers.

## Code Snippet

https://github.com/sherlock-audit/2024-05-gamma-staking/blob/main/StakingV2/src/Lock.sol#L604-L605

## Tool used

Manual Review

## Recommendation

```diff
-        if (lockPeriod <= defaultRelockTime || (block.timestamp - lockTime) < lockPeriod) {
            return lockPeriod - (block.timestamp - lockTime) % lockPeriod;
-        } else {
-            return defaultRelockTime - (block.timestamp - lockTime) % defaultRelockTime;
-        }
```

Avoid implicitly updating to the default lock period in the calculations. Instead, assume the user continues relocking at their original lock period and multiplier. This maintains fairness for all users based on their original lock choices and matches user expectations since if the user has chosen a long lock period, they similarly have a long period in which to call `exitLateById` before the auto-relock rollover without any negative consequences. 
