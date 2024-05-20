Damaged Iris Antelope

high

# The calculation of the remaining unlock period is incorrect.

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