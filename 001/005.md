Rough Lava Penguin

medium

# `Lock::notifyUnseenReward` Reverts Due to Array Out-Of-Bounds Access When Notifying a Subset of Reward Tokens

## Summary

When notifying a subset of `rewardTokens`, `Lock::notifyUnseenReward` uses the length of `rewardTokens` instead of the length of `_rewardTokens` which results in an array out-of-bounds access.

## Vulnerability Detail

When notifying a subset of `rewardTokens`, `Lock::notifyUnseenReward` uses the length of `rewardTokens` instead of the length of `_rewardTokens`.

## Impact

Pointed out in the previous audit, notifying a subset a tokens prevents a DOS due to intentionally reverting tokens (such as in the case of Euler vault tokens). However, as currently implemented, the function attempts to notify tokens the length of `rewardTokens`, not length of `_rewardTokens`.

For example, if there are five tokens that are a part of `rewardTokens` and one is maliciously upgraded to always revert, when notifying a subset of four tokens, the function will try grab a fifth token from `_rewardTokens` resulting in an array out-of-bounds access.

## Code Snippet

```solidity
    function notifyUnseenReward(address[] memory _rewardTokens) external {
        uint256 length = rewardTokens.length; // Gets the number of reward tokens currently recognized by the contract.
        for (uint256 i = 0; i < length; ++i) {
            if (rewardTokenAdded[_rewardTokens[i]]) {
                _notifyUnseenReward(_rewardTokens[i]); // Processes each token to update any unseen rewards.
            }
        }
    }
```

[Perma](https://github.com/sherlock-audit/2024-05-gamma-staking/blob/703fd3604069489937037f20490ec8c492c0508e/StakingV2/src/Lock.sol#L504-L511)

## Tool used

Manual Review

## Recommendation

Use `_rewardTokens`'s length instead to prevent out-of-bounds access.

```diff
    function notifyUnseenReward(address[] memory _rewardTokens) external {
-       uint256 length = rewardTokens.length; // Gets the number of reward tokens currently recognized by the contract.
+       uint256 length = _rewardTokens.length; // Gets the number of provided reward tokens.
        for (uint256 i = 0; i < length; ++i) {
            if (rewardTokenAdded[_rewardTokens[i]]) {
                _notifyUnseenReward(_rewardTokens[i]); // Processes each token to update any unseen rewards.
            }
        }
    }
```