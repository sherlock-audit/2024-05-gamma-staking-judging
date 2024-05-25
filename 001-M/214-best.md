Cool Pecan Otter

medium

# Staking dust amount for other users can cause `withdrawAllUnlockedToken()` to DoS

## Summary

The protocol allows staking on behalf of another user, however, it does not limit the minimum amount of tokens. Attackers can perform a grief attack by staking dust amount (1 wei) for other users, and bloat up their locklist, which would brick the user calling `withdrawAllUnlockedToken()`.

## Vulnerability Detail

There are two issues here:

1. Attackers can stake dust amount (e.g. 1 wei) of stakingToken for a victim. This would bloat up their locklist.
2. Users calling `withdrawAllUnlockedToken()` must iterate through ALL existing locks at a O(n^2) complexity, which is a waste of gas, and may easily lead to out-of-gas issues.

Issue 1 is easy to understand. Let's dive into issue 2:

The reason `withdrawAllUnlockedToken()` is a O(n^2) complexity is because the code queries `locklist.getLocks(msg.sender, page, lockCount)` instead of `locklist.getLocks(msg.sender, page, 10)` every time, where `lockCount` is the array length. The `locklist.getLocks(msg.sender, page, lockCount)` will return an array length of `lockCount`, and each element would be iterated every time.

Also, even if `withdrawAllUnlockedToken()` use a correct O(n) implemetation, it still may run into out-of-gas error, as long as attacker sends enough dust amount locks into the locklist of the victim.

Lock.sol
```solidity
    function _stake(
        uint256 amount,
>       address onBehalfOf,
        uint256 typeIndex,
        bool isRelock
    ) internal whenNotPaused {
        if (typeIndex >= lockPeriod.length || amount == 0) revert InvalidAmount();

        _updateReward(onBehalfOf);
        
        Balances storage bal = balances[onBehalfOf];

        bal.locked += amount;
        lockedSupply += amount;

        uint256 multiplier = rewardMultipliers[typeIndex];
        bal.lockedWithMultiplier += amount * multiplier;
        lockedSupplyWithMultiplier += amount * multiplier;
        _updateRewardDebt(onBehalfOf);


>       locklist.addToList(
            onBehalfOf, 
            LockedBalance({
                lockId: 0, // This will be set inside the addToList function
                amount: amount,
                unlockTime: 0, 
                multiplier: multiplier,
                lockTime: block.timestamp,
                lockPeriod: lockPeriod[typeIndex],
                exitedLate: false
            })
        );

        if (!isRelock) {
            IERC20(stakingToken).safeTransferFrom(
                msg.sender,
                address(this),
                amount
            );
        }

        emit Locked(
            onBehalfOf,
            amount,
            balances[onBehalfOf].locked
        );
    }
    ...
    function withdrawAllUnlockedToken() external override nonReentrant {
        uint256 lockCount = locklist.lockCount(msg.sender); // Fetch the total number of locks for the caller.
        uint256 page;
        uint256 limit;
        uint256 totalUnlocked;
        
        while (limit < lockCount) {
>           LockedBalance[] memory lockedBals = locklist.getLocks(msg.sender, page, lockCount); // Retrieves a page of locks for the user.
            for (uint256 i = 0; i < lockedBals.length; i++) {
                if (lockedBals[i].unlockTime != 0 && lockedBals[i].unlockTime < block.timestamp) {
                    totalUnlocked += lockedBals[i].amount; // Adds up the amount from all unlocked balances.
                    locklist.removeFromList(msg.sender, lockedBals[i].lockId); // Removes the lock from the list.
                }
            }

>           limit += 10; // Moves to the next page of locks.
            page++;
        }

        IERC20(stakingToken).safeTransfer(msg.sender, totalUnlocked); // Transfers the total unlocked amount to the user.
        emit WithdrawAllUnlocked(msg.sender, totalUnlocked); // Emits an event logging the withdrawal.
    }
```

Locklist.sol
```solidity
    function getLocks(
        address user,
        uint256 page,
        uint256 limit
    ) public view override returns (LockedBalance[] memory) {
>       LockedBalance[] memory locks = new LockedBalance[](limit);
        uint256 lockIdsLength = lockIndexesByUser[user].length();

        uint256 i = page * limit;
        for (;i < (page + 1) * limit && i < lockIdsLength; i ++) {
            locks[i - page * limit]= lockById[lockIndexesByUser[user].at(i)];
        }
        return locks;
    }
```

## Impact

Users would not be able to call `withdrawAllUnlockedToken()` due to out-of-gas error.

## Code Snippet

- https://github.com/sherlock-audit/2024-05-gamma-staking/blob/main/StakingV2/src/Lock.sol#L260-L307
- https://github.com/sherlock-audit/2024-05-gamma-staking/blob/main/StakingV2/src/libraries/LockList.sol#L97-L110

## Tool used

Manual review

## Recommendation

1. Implement `withdrawAllUnlockedToken()` correctly: change to `locklist.getLocks(msg.sender, page, 10)` and add a `gasleft()` check.
2. Maybe maintain a list of trusted users for each address, and limit only these users can stake on behalf of them.
