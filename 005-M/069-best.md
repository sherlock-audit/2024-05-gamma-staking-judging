Skinny Rosewood Pangolin

medium

# User may create a lock with an unexpected lock type

## Summary
User may create a lock with an unexpected lock type.

## Vulnerability Detail
When a user calls [stake(...)](https://github.com/sherlock-audit/2024-05-gamma-staking/blob/main/StakingV2/src/Lock.sol#L245-L249) function to create a lock, `typeIndex` is passed to specify the lock type, and different lock types have different multiplier and lock period.
```solidity
        uint256 multiplier = rewardMultipliers[typeIndex];
        ...

        locklist.addToList(
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
```
Lock type can also be changed by the owner through [setLockTypeInfo(...)](https://github.com/sherlock-audit/2024-05-gamma-staking/blob/main/StakingV2/src/Lock.sol#L128-L131) function. For example, the lock types are:
```diff
0:  10 days
1:  30 days

0:  1x multiplier
1:  2x multiplier
```
The owner submit an transaction to change to:
```diff
0:  10 days
1:  60 days

0:  1x multiplier
1:  1x multiplier
```
Before the transaction is executed, a user calls **stake(...)** function with `typeIndex` is 1, they expect to create a lock for 30 days with 2x multiplier, however, the owner's transaction gets executed first, results in user's funds being locked for 60 days with 1x multiplier.

## Impact
User may lock funds with higher period but lower multiplier than expected.

## Code Snippet
https://github.com/sherlock-audit/2024-05-gamma-staking/blob/main/StakingV2/src/Lock.sol#L245-L249

## Tool used
Manual Review

## Recommendation
Add parameters to **stake(...)** function to ensure user won't create a wrong lock:
```diff
    function _stake(
        uint256 amount,
        address onBehalfOf,
        uint256 typeIndex,
+       uint256 expectedLockPeriod,
+       uint256 expectedMultiplier,
        bool isRelock           
    ) internal whenNotPaused {
        ...

        uint256 multiplier = rewardMultipliers[typeIndex];
+       uint256 lockPeriod = lockPeriod[typeIndex];
+       if (multiplier != expectedMultiplier || lockPeriod != expectedLockPeriod) revert();

        ...
    }
```