Cheesy Jade Lobster

medium

# Users can call multiple times `exitLateById` for the same lock

## Summary

Users can call multiple times `exitLateById` for the same lock

## Vulnerability Detail

Users are able to call `exitLateById` multiple times for the same lock. Let's say I've two locks first, first lock equal to 100 tokens, and the second one 200 tokens. By mistake I'll `exitLateById` for the first lock twice, those transactions will pass. Then if I try to exit from the second lock the transaction will revert because there is not enough value in the `bal.lockedWithMultiplier` variable. The result is that funds in the second lock, are stuck and cannot be exited. This also breaks protocol invariants because `lockedSupplyWithMultiplier`, and `lockedSupply` no longer track the real amount locked funds in the protocol.

In the `RestakeAfterLateExit.t` add the following test

```solidity
function test_multiple_exit_late_for_the_same_lock() public {
    vm.startPrank(user1);
    lock.stake(100e18, user1, 0); // first stake, lockId 0
    lock.stake(200e18, user1, 0); // second stake, lockId 1

    uint256 firstStakeLockId = 0;

    // we are able to exit late twice for the same lock
    lock.exitLateById(firstStakeLockId);
    lock.exitLateById(firstStakeLockId);

    uint256 secondStakeLockId = 1;

    // because we exited late twice for the same lock
    // exiting late for the second lock revers because of the underflow
    // for bal.lockedWithMultiplier it tries to subtract 200e18 from 100e18
    vm.expectRevert();
    lock.exitLateById(secondStakeLockId);
   
}
```

## Impact

The user's lock funds are stuck in the contract. Protocol invariants are broken.

## Code Snippet

https://github.com/sherlock-audit/2024-05-gamma-staking/blob/main/StakingV2/src/Lock.sol#L349

## Tool used

Manual Review

## Recommendation

Don't allow to call `exitLateById` for the same lock multiple times

```diff
function exitLateById(uint256 id) external {
        _updateReward(msg.sender); // Updates any pending rewards for the caller before proceeding.

        LockedBalance memory lockedBalance = locklist.getLockById(msg.sender, id); // Retrieves the lock details from the lock list as a storage reference to modify.
+       if(lockedBalance.exitedLate) revert InvalidLockId(); // Ensures the lock has not already exited

        // Calculate and set the new unlock time based on the remaining cooldown period.
        uint256 coolDownSecs = calcRemainUnlockPeriod(lockedBalance);
        locklist.updateUnlockTime(msg.sender, id, block.timestamp + coolDownSecs);

        // Reduce the locked supply and the user's locked balance with and without multiplier.
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
