# Issue H-1: Attacker will prevent distribution of USDC to stakers through frequent reward updates 

Source: https://github.com/sherlock-audit/2024-07-kwenta-staking-contracts-judging/issues/22 

## Found by 
aslanbek, eeyore, g, tedox
### Summary

USDC's lower precision of 6 decimals and frequent reward updates will cause stakers to receive 0 of the allotted [10K USDC weekly rewards](https://gov.kwenta.eth.limo/kips/kip-127#specification).

> distributions should be done at the pace of 10k per week until fully distributed

### Root Cause

Using the same [reward-per-token calculation](https://github.com/sherlock-audit/2024-07-kwenta-staking-contracts/blob/main/token/contracts/StakingRewardsV2.sol#L455) for USDC used for KWENTA is a mistake as USDC only has 6 decimals of precision compared to KWENTA's 18 decimals. 

### Internal pre-conditions

1. The[ `rewardsDuration` is 1 week](https://github.com/sherlock-audit/2024-07-kwenta-staking-contracts/blob/main/token/contracts/StakingRewardsV2.sol#L187). This is the default duration.
2. The `StakingRewardsNotifier` [sends](https://github.com/sherlock-audit/2024-07-kwenta-staking-contracts/blob/main/token/contracts/StakingRewardsNotifier.sol#L92) 10_000e6 USDC of rewards to `StakingRewards`. Kwenta Governance will be [distributing 10K USDC per week](https://gov.kwenta.eth.limo/kips/kip-127#specification) as rewards. 
3. There is at least 100_000e18 KWENTA staked in `StakingRewardsV2`. Note that KWENTA's current total supply is at ~631_000e18. 

### External pre-conditions

_No response_

### Attack Path

1. The `StakingRewardsNotifier` sends 10_000e6 USDC to `StakingRewards` via `notifyRewardAmount()`. The [`rewardRateUSDC`](https://github.com/sherlock-audit/2024-07-kwenta-staking-contracts/blob/main/token/contracts/StakingRewardsV2.sol#L652) is now equal 16534 (`10_000e6 / 1 weeks`). This is scheduled to happen every week.
2. Every 1 to 3 blocks, the attacker will trigger an update of rewards by calling stake or unstake in `StakingRewards`. The reward-per-token for USDC is calculated:
```solidity
((lastTimeRewardApplicable() - lastUpdateTime) * rewardRateUSDC * 1e18) / allTokensStaked
```
When 3 blocks have passed, the time since the last update would be 6 seconds because block time in Optimism is 2 seconds per block. 
```solidity
(6 * rewardRateUSDC * 1e18) / allTokensStaked
```
`rewardRateUSDC` is 16534 and `allTokensStaked` is 100_000e18.
```solidity
(6 * 16534 * 1e18) / 100_000e18  // ==> this will return 0
```
Since the reward-per-token for USDC is 0, none of the 10_000e6 USDC will be distributed to the stakers.

Note that the attacker does not need to update rewards every 1-3 blocks if there is organic activity on every block from users interacting with `StakingRewardsV2` and trigger the rewards update.

### Impact

All stakers will receive 0 USDC rewards of the 10_000e6 USDC weekly rewards. 

### PoC

The `test_rewardPerTokenUSDC` test in `StakingRewardsV2.t.sol` can be modified to the following test case.

```solidity
    function test_rewardPerTokenUSDC() public {
        // fund so that staking can succeed
        uint256 stakedAmount = 100_000e18; // @audit 100_000 staked KWENTA
        fundAndApproveAccountV2(address(this), stakedAmount);

        // check reward per token starts as 0
        assertEq(stakingRewardsV2.rewardPerTokenUSDC(), 0);

        // stake
        stakingRewardsV2.stake(stakedAmount);
        assertEq(stakingRewardsV2.totalSupply(), stakedAmount);

        // set rewards
        uint256 reward = 10_000e6; // @audit 10,000 USDC will be distributed every week
        vm.prank(address(rewardsNotifier));
        stakingRewardsV2.notifyRewardAmount(0, reward);

        // ff to end of period
        vm.warp(block.timestamp + 2);

        // check reward per token updated
        assertEq(stakingRewardsV2.rewardPerTokenUSDC(), 1 ether); // @audit this will return 0
    }
```

The test shows that none of the rewards are getting distributed for the given values.

### Mitigation

A possible mitigation is to convert the USDC rewards to an 18-decimal value when computing for `rewardRateUSDC`. When claiming rewards, convert the earned USDC rewards back to 6-decimal precision. If it is 0, store the unclaimed partial values of USDC until they can be claimed whole.

