# Voting Booth - Audit Report

## Business Logic

Take a look at First-Flights's contest details


## Known Issues

Please head to First-Flights's contests Known-issues section

## Reconnaissance

- `VotingBooth.sol::VotingBooth`

- Resource: [PUSH0 Support](https://docs.arbitrum.io/arbitrum-essentials/arbitrum-vs-ethereum/solidity-support)

```solidity
//...

constructor(address[] memory allowList) payable {
    // @info: now it's 2026 and arbitrum supports PUSH0
    // @note: no issue with PUSH0 on arbitrum
    // @res: https://docs.arbitrum.io/arbitrum-essentials/arbitrum-vs-ethereum/solidity-support
    
    //...

    // odd number of voters required to simplify quorum check
    // @info: possible length: 3, 5, 7, 9
    require(allowListLength % 2 != 0, "DP: Odd number of voters required");

    //...
}

//...

function vote(bool voteInput) external {
    // @info: no way to reset the voting booth
    // After 1st voting completion there's no way to restart the voting for other proposals
    // prevent voting if already completed

    //...
    
    // @Bug: possible front-running due to race for rewards among allowed voters
    if (totalCurrentVotes * 100 / s_totalAllowedVoters >= MIN_QUORUM) {
        // mark voting as having been completed
        s_votingComplete = true;

        // distribute the voting rewards
        _distributeRewards();
    }
}

//...

function _distributeRewards() private {
    //...

    // otherwise the proposal passed so distribute rewards to the `For` voters
    else {
        // @info: total rewards should be distributed among for voters only
        // @issue: rewards distribution calculation divides the rewards among all voters
        // @info: Accounting Bug
        // @danger: after rewards distribution there's no way for creator to withdraw dust or any bounty remained in the protocol(smc) this(contract).
        uint256 rewardPerVoter = totalRewards / totalVotes;

        for (uint256 i; i < totalVotesFor; ++i) {
            // proposal creator is trusted when creating allowed list of voters,
            // findings related to gas griefing attacks or sending eth
            // to an address reverting thereby stopping the reward payouts are
            // invalid. Yes pull is to be preferred to push but this
            // has not been implemented in this simplified version to
            // reduce complexity & help you focus on finding the
            // harder to find bug

            // if at the last voter round up to avoid leaving dust; this means that
            // the last voter can get 1 wei more than the rest - this is not
            // a valid finding, it is simply how we deal with imperfect division
            if (i == totalVotesFor - 1) {
                // @info: same accounting bug here also
                // using totalVotes instead of totalVotesFor
                rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotes, Math.Rounding.Ceil);
            }
            _sendEth(s_votersFor[i], rewardPerVoter);
        }
    }
}

```

## PoCs

- Place below PoCs into `./test/VotingBoothTest.t.sol:VotingBoothTest`

```solidity
function testRewardDistributionCalculationMakesDisproportionatedRewardsShare() public {
    console2.log("balance of 0x1 before rewards: ", address(0x1).balance);
    console2.log("balance of 0x2 before rewards: ", address(0x2).balance);
    console2.log("balance of 0x3 before rewards: ", address(0x3).balance);

    vm.prank(address(0x1));
    booth.vote(true);

    vm.prank(address(0x2));
    booth.vote(false);

    vm.prank(address(0x3));
    booth.vote(true);

    console2.log("------------------------------------");

    console2.log("balance of 0x1 before rewards: ", address(0x1).balance);
    console2.log("balance of 0x2 before rewards: ", address(0x2).balance);
    console2.log("balance of 0x3 before rewards: ", address(0x3).balance);

    assert(!booth.isActive());
}
```

- Open Bash and execute below:

```bash
forge test
```

- For specific PoC Run, Execute below:

```bash
forge test --mt testRewardDistributionCalculationMakesDisproportionatedRewardsShare -vv
```




<br/>
<br/>
<br/>
<br/>
<br/>

#### Auditor|Sec-Res|White-Hat: *theirrationalone*