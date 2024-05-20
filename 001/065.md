Dizzy Nylon Tiger

medium

# `earlyExitById()` and `exitLateById()` calls near the end of `lockPeriod` are vulnerable to attacks.

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