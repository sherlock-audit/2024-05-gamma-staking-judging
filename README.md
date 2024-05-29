# Issue M-1: Loss of rewards for all users due to mismatching reward multipliers for re-lock-ups 

Source: https://github.com/sherlock-audit/2024-05-gamma-staking-judging/issues/10 

The protocol has acknowledged this issue.

## Found by 
adamidarrha, guhu95, jokr, wildflowerzx, zraxx
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



## Discussion

**bjp333**

For this recommendation, we didn't mention in our design, but it is intended to keep those users at the higher multiplier even when they auto-relock at the shorter defaultRelockTime.  

It still fulfills its function in that the user is incentivized to stay locked to maintain his higher multiplier.  Yes, he does have the option to leave within the shorter timeframe while still earning at a higher multiplier, but he is still incentivized to stay locked because if he exits, he will lose his multiplier for good and will have to wait a whole year to obtain that status again.

What we don't want to happen is that a user who staked for a year gets relocked for another year or our investors/team who are locked for multiple years to get relocked automatically at multiple years.  

**bjp333**

<img width="1066" alt="image" src="https://github.com/sherlock-audit/2024-05-gamma-staking-judging/assets/91488941/6ee304fd-f748-47c6-8e75-af7f585d5630">
Please see here for how it would look in practice.  

**santipu03**

Considering that this "intended design" wasn't mentioned during the time of the audit, I believe this issue may remain open because it broke a core principle of the `Lock` contract, which is that users should have the right multipliers according to their lock periods. 

# Issue M-2: `earlyExitById()` and `exitLateById()` calls near the end of `lockPeriod` are vulnerable to attacks. 

Source: https://github.com/sherlock-audit/2024-05-gamma-staking-judging/issues/65 

## Found by 
0xRajkumar, 0xreadyplayer1, EgisSecurity, HChang26, KupiaSec, T\_F\_E, denzi\_, emrekocak, h2134, kennedy1030, no, petro1912, pkqs90, samuraii77, yamato, zraxx
## Summary
`earlyExitById()` and `exitLateById()` calls near the end of `lockPeriod` are vulnerable to attacks.
## Vulnerability Detail
Locks are automatically re-locked at the end of the `lockPeriod` for the lesser of `lockPeriod` or `defaultRelockTime`.

The protocol offers 2 methods for stakers to unstake, `earlyExitById()` and `exitLateById()`. Both methods use `calcRemainUnlockPeriod()` to calculate `unlockTime`.

```solidity
    function calcRemainUnlockPeriod(LockedBalance memory userLock) public view returns (uint256) {
        uint256 lockTime = userLock.lockTime;
        uint256 lockPeriod = userLock.lockPeriod;

        if (lockPeriod <= defaultRelockTime || (block.timestamp - lockTime) < lockPeriod) {
            return lockPeriod - (block.timestamp - lockTime) % lockPeriod;
        } else {
            return defaultRelockTime - (block.timestamp - lockTime) % defaultRelockTime;
        }
    }
```

`exitLateById()` is the default method, where the lock `id` stops accumulating rewards immediately. The `unlockTime`/cool-down period is calculated by `calcRemainUnlockPeriod()`. Funds are only available for withdrawal after `unlockTime`.

For example:
The `unlockTime` is dictated by modulo logic in function `calcRemainUnlockPeriod()`. So if the lock time were 30 days, and the user staked for 50 days, he would have been deemed to lock for a full cycle of 30 days, followed by 20 days into the second cycle of 30 days, and thus will have 10 days left before he can withdraw his funds.

Based on this design, it is in stakers' best interest to invoke `exitLateById()` towards the end of a `lockPeriod` when there are only a few seconds left before it auto re-locks for another 30 days. This maximizes rewards and minimizes the cool-down period.

If a staker invokes `exitLateById()` 1 minute before auto re-lock, two scenarios can occur depending on when the transaction is mined:
1. If the transaction is mined before the auto re-lock, the staker's funds become available after a 1-minute cool-down period.
2. If the transaction is mined after the auto re-lock, the staker's funds will not be available for another 30 days.

Attackers can grief honest stakers by front-running `exitLateById()` with a series of dummy transactions to fill up the block. If `exitLateById()` is mined after the auto re-lock, the staker must wait another 30 days. The only way to avoid this is to call `exitLateById()` well before the end of the `lockPeriod` when block stuffing is not feasible, reducing the potential reward earned by the staker.

```solidity
    function exitLateById(uint256 id) external {
        _updateReward(msg.sender);

        LockedBalance memory lockedBalance = locklist.getLockById(msg.sender, id); // Retrieves the lock details from the lock list as a storage reference to modify.

     ->uint256 coolDownSecs = calcRemainUnlockPeriod(lockedBalance);
        locklist.updateUnlockTime(msg.sender, id, block.timestamp + coolDownSecs);

        uint256 multiplierBalance = lockedBalance.amount * lockedBalance.multiplier;
        lockedSupplyWithMultiplier -= multiplierBalance;
        lockedSupply -= lockedBalance.amount;
        Balances storage bal = balances[msg.sender];
        bal.lockedWithMultiplier -= multiplierBalance;
        bal.locked -= lockedBalance.amount;

        locklist.setExitedLateToTrue(msg.sender, id);

        _updateRewardDebt(msg.sender); // Recalculates reward debt after changing the locked balance.

        emit ExitLateById(id, msg.sender, lockedBalance.amount); // Emits an event logging the details of the late exit.
    }
```



`earlyExitById()` is the second method to unstake. In this function, the staker pays a penalty but can access funds immediately. The penalty consists of a base penalty plus a time penalty, which decreases linearly over time. The current configuration sets the minimum penalty at 15% and the maximum penalty at 50%.
```solidity
    function earlyExitById(uint256 lockId) external whenNotPaused {
        if (isEarlyExitDisabled) {
            revert EarlyExitDisabled();
        }
        _updateReward(msg.sender);
        LockedBalance memory lock = locklist.getLockById(msg.sender, lockId);

        if (lock.unlockTime != 0)
            revert InvalidLockId();

        uint256 coolDownSecs = calcRemainUnlockPeriod(lock);
        lock.unlockTime = block.timestamp + coolDownSecs;
        uint256 penaltyAmount = calcPenaltyAmount(lock);
        locklist.removeFromList(msg.sender, lockId);
        Balances storage bal = balances[msg.sender];
        lockedSupplyWithMultiplier -= lock.amount * lock.multiplier;
        lockedSupply -= lock.amount;
        bal.locked -= lock.amount;
        bal.lockedWithMultiplier -= lock.amount * lock.multiplier;
        _updateRewardDebt(msg.sender);

        if (lock.amount > penaltyAmount) {
            IERC20(stakingToken).safeTransfer(msg.sender, lock.amount - penaltyAmount);
            IERC20(stakingToken).safeTransfer(treasury, penaltyAmount);
            emit EarlyExitById(lockId, msg.sender, lock.amount - penaltyAmount, penaltyAmount);
        } else {
            IERC20(stakingToken).safeTransfer(treasury, lock.amount);
        emit EarlyExitById(lockId, msg.sender, 0, penaltyAmount);
        }
    }
```
The penalty is calculated in `calcPenaltyAmount()`, using `unlockTime` from `calcRemainUnlockPeriod()`.
```solidity
    function calcPenaltyAmount(LockedBalance memory userLock) public view returns (uint256 penaltyAmount) {
        if (userLock.amount == 0) return 0; // Return zero if there is no amount locked to avoid unnecessary calculations.
        uint256 unlockTime = userLock.unlockTime;
        uint256 lockPeriod = userLock.lockPeriod;
        uint256 penaltyFactor;


        if (lockPeriod <= defaultRelockTime || (block.timestamp - userLock.lockTime) < lockPeriod) {

            penaltyFactor = (unlockTime - block.timestamp) * timePenaltyFraction / lockPeriod + basePenaltyPercentage;
        }
        else {
            penaltyFactor = (unlockTime - block.timestamp) * timePenaltyFraction / defaultRelockTime + basePenaltyPercentage;
        }

        // Apply the calculated penalty factor to the locked amount.
        penaltyAmount = userLock.amount * penaltyFactor / WHOLE;
    }
```
Using the sample example above, if a user invokes `earlyExitById()` on day 50, the penalty is as follows:

Time penalty = (10 / 30) * 35%
Base penalty = 15%
Total penalty = 26.66%

However, an issue arises when `earlyExitById()` is triggered near the end of the `lockPeriod`. Depending on when the transaction is mined, two scenarios can occur(extreme numbers were used to demonstrate impact):

1. If `earlyExitById()` is mined at 59 days, 23 hours, 59 minutes, and 59 seconds, the time penalty is essentially 0%.
2. If `earlyExitById()` is mined exactly at 60 days, this results in 100% of the time penalty.

The 1-second difference can mean the difference between the minimum penalty and the maximum penalty. An unexpectedly high penalty can occur if the transaction is sent with lower-than-average gas, causing it not to be picked up immediately. Attackers can cause stakers to incur the maximum penalty by block stuffing and delaying their transaction.

## Impact
Funds may be locked for longer than expected in `exitLateById()`
Penalty may be greater than expected in `earlyExitById()` 
## Code Snippet
https://github.com/sherlock-audit/2024-05-gamma-staking/blob/main/StakingV2/src/Lock.sol#L313
https://github.com/sherlock-audit/2024-05-gamma-staking/blob/main/StakingV2/src/Lock.sol#L349
https://github.com/sherlock-audit/2024-05-gamma-staking/blob/main/StakingV2/src/Lock.sol#L596
https://github.com/sherlock-audit/2024-05-gamma-staking/blob/main/StakingV2/src/Lock.sol#L569
## Tool used

Manual Review

## Recommendation
No recommendation for `exitLateById()` since the optimal way to use this function is near the end of a `lockPeriod`.

Consider adding slippage protection to `earlyExitById()`. 
```solidity
-   function earlyExitById(uint256 lockId) external whenNotPaused {
+   function earlyExitById(uint256 lockId, uint256 expectedAmount) external whenNotPaused {
        if (isEarlyExitDisabled) {
            revert EarlyExitDisabled();
        }
        _updateReward(msg.sender);
        LockedBalance memory lock = locklist.getLockById(msg.sender, lockId);

        if (lock.unlockTime != 0)
            revert InvalidLockId();

        uint256 coolDownSecs = calcRemainUnlockPeriod(lock);
        lock.unlockTime = block.timestamp + coolDownSecs;
        uint256 penaltyAmount = calcPenaltyAmount(lock);
        locklist.removeFromList(msg.sender, lockId);
        Balances storage bal = balances[msg.sender];
        lockedSupplyWithMultiplier -= lock.amount * lock.multiplier;
        lockedSupply -= lock.amount;
        bal.locked -= lock.amount;
        bal.lockedWithMultiplier -= lock.amount * lock.multiplier;
        _updateRewardDebt(msg.sender);

+       require(expectedAmount >= lock.amount - penaltyAmount);
        if (lock.amount > penaltyAmount) {
            IERC20(stakingToken).safeTransfer(msg.sender, lock.amount - penaltyAmount);
            IERC20(stakingToken).safeTransfer(treasury, penaltyAmount);
            emit EarlyExitById(lockId, msg.sender, lock.amount - penaltyAmount, penaltyAmount);
        } else {
            IERC20(stakingToken).safeTransfer(treasury, lock.amount);
        emit EarlyExitById(lockId, msg.sender, 0, penaltyAmount);
        }
    }
```



## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/GammaStrategies/StakingV2/commit/001c056122874fe0e3fa6ece8383f2eafaf12cf5


# Issue M-3: The calculation of the remaining unlock period is incorrect. 

Source: https://github.com/sherlock-audit/2024-05-gamma-staking-judging/issues/256 

## Found by 
0xRajkumar, Tri-pathi, ZdravkoHr., ggg\_ttt\_hhh, pkqs90, wildflowerzx, yamato, ydlee
## Summary
In `calcRemainUnlockPeriod`, the calculation of the remaining unlock period is incorrect when current timestamp is larger than the lock time. The impacts are:
1. The penalty amount of early exit is wrong, more or less than expected.
2. The resulting unlock time is wrong, meaning that users are unable to unlock their tokens at the expected time.

## Vulnerability Detail
In L605, when `block.time > lockTime`, the first `lockPeriod` should be subtracted before modulo `lockPeriod` or `defaultRelockTime`. For example
```solidity
lockPeriod = 10 Days
defaultRelockTime = 7 Days
block.timestamp - lockTime = 12 Days
reaminUnlockPeriod = 7 - 12 % 7 = 2 Days  (L605)
```
However, the user's tokens are first locked for `10 Days` during the first period (i.e. `lockPeriod`), and then the tokens are then re-locked for `7 Days` during the second period (i.e. `defaultRelockTime`). After `12 Days`, only `2 Days` have passed in the second locking period, so the remaining unlocking period should be `5 Days`, not the `2 Days` calculated above.

```solidity
596:    function calcRemainUnlockPeriod(LockedBalance memory userLock) public view returns (uint256) {
597:        uint256 lockTime = userLock.lockTime;
598:        uint256 lockPeriod = userLock.lockPeriod;
599:        
600:        if (lockPeriod <= defaultRelockTime || (block.timestamp - lockTime) < lockPeriod) {
601:            // If the adjusted lock period is less than or equal to the default relock time, or if the current time is still within the adjusted lock period, return the remaining time based on the adjusted lock period.
602:            return lockPeriod - (block.timestamp - lockTime) % lockPeriod;
603:        } else {
604:            // If the current time exceeds the adjusted lock period, return the remaining time based on the default relock time.
605:@>          return defaultRelockTime - (block.timestamp - lockTime) % defaultRelockTime;
606:        }
607:    }
```
https://github.com/sherlock-audit/2024-05-gamma-staking/blob/main/StakingV2/src/Lock.sol#L596-L607

## Impact
The calculation of the remaining unlock period is incorrect, as the unlock time is used when calculating the penalty amount for early exits and withdrawing unlocked token, so the impacts can be:
1. The penalty amount of early exit is wrong, more or less than expected.
2. The resulting unlock time is wrong, meaning that users are unable to unlock their tokens at the expected time.

## Code Snippet
https://github.com/sherlock-audit/2024-05-gamma-staking/blob/main/StakingV2/src/Lock.sol#L596-L607

## Tool used

Manual Review

## Recommendation
```solidity
    function calcRemainUnlockPeriod(LockedBalance memory userLock) public view returns (uint256) {
        uint256 lockTime = userLock.lockTime;
        uint256 lockPeriod = userLock.lockPeriod;
        
        if (lockPeriod <= defaultRelockTime || (block.timestamp - lockTime) < lockPeriod) {
            // If the adjusted lock period is less than or equal to the default relock time, or if the current time is still within the adjusted lock period, return the remaining time based on the adjusted lock period.
            return lockPeriod - (block.timestamp - lockTime) % lockPeriod;
        } else {
            // If the current time exceeds the adjusted lock period, return the remaining time based on the default relock time.
-            return defaultRelockTime - (block.timestamp - lockTime) % defaultRelockTime;
+            return defaultRelockTime - (block.timestamp - lockTime - lockPeriod) % defaultRelockTime;
        }
    }
```



## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/GammaStrategies/StakingV2/commit/6963d1b628fa2909a04243d4dd6dc873ab01b058


