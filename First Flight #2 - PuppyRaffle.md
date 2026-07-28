# PuppyRaffle - Audit Report

## Business Idea

- The protocol provides a lottery(raffle) platform
- Users can enter into the lottery with entry fees (minimum table fees)
- Protocol earns from the winner's rewards by taking a cut as platform fees
- Winner of the lottery wins reward pool and an NFT

## Expected flow assumptions

- users can enter into lottery
- users should not be able to enter raffle more than once
- users should be able to retreat themselves before raffle actually start if they wish to.
- On a valid retreat (lottery withdrawal), users should be refunded their fees payment (protocol might take some charges on a valid retreat? could totally be a design choice though)
- Lottery must be fair enough
- Winner should be able to withdraw their rewards
- Owner should clean all the dust ever contaminated
- Owner should not be able to pause an ongoing lottery, they can only pause lottery before start and after end
- There should have only one winner at a time
- Reward calculation must be correct.
- After a winner announcement lottery should be resetted wisely.
- There should have a blacklist mechanism(honeypot) for actors who ever found flipping the lottery's fairness.
- There should have a fair maths based unbrokenable mechanism for winner selection.
- Owner should be an outcome of multisig or DAO to avoid centralization risk.
- Owner should not be able to enter into the raffle (they can rekt the lottery).

## Expected Bad circumstances

- DoS on raffle entry
- Same user(EOA) entered into raffle more than once.
- Users can't exit once they entered even if the lottery hasn't started yet.
- Users exit from raffle before it starts but without their funds.
- Winner is unable to withdraw their rewards pool funds.
- Dust remains in the lottery causes DoS on consecutive lotteries turn.
- Owner is able to pause the lottery when it's online.
- DoS on after winner announcement and reset.
- Broken winner selection mechanism, can be frontrun or manipulated.
- Owner can enter into the raffle
- Zero addresses allowed into raffle
- non-existent winner announcement Freezes the lottery forever.
- forgot to award NFT on a fair win.
- Bad initialization causes DoS
- Users and winner withdrawal may be vulnerable to reentrancy threat.
- Winner selection method may attain liveness through external resources i.e., chainlink, may have wrong winner announcements.

## Invariants

- There could have only one winner at once
- Lottery pool must be equals to the number of players into the lottery * entranceFee
- Winner can't ever withdraw full rewards, platform charges its fee
- Lottery can only be spinned after raffleDuration

## Reality

- Yes, there's a `entranceFee` (immutable)
- Players array
- RaffleDuration (raffle has an expiry time, great choice)
- Protocol snaps raffle start time (raffleStartTime)
- Protocol maintains a record of previous winner
- A fee address (feeAddress) for owner to send total collected fees.
- Tracks total Fees (totalFees)

------------------ NFT --------------------

- Mapping tokenId -> Rarity (tokenIdToRarity) to track the rarity of puppy
- Mapping rarity -> Uri (rarityToUri) to track the uri of puppy
- Mapping rarity -> Name (rarityToName) to track the name of puppy

-------------- CONSTANT REALITY --------------
- Not so rare common puppy image uri: commonImageUri
- Rare puppy image uri: rareImageUri
- Rarest puppy image uri: legendaryImageUri

- Common puppy rarity score: COMMON_RARITY
- Rare puppy rarity score: RARE_RARITY
- Rarest puppy rarity score: LEGENDARY_RARITY

- Common puppy name: COMMON
- Rare puppy name: RARE
- Rarest puppy name: LEGENDARY

- event for raffle entry
- event for raffle refund
- event for fee address change

--------------- Functions ---------------

- Initialization -> initialized following
    - entranceFee
    - feeAddress
    - raffleDuration
    - raffleStartTime
    - rarityToUri
    - rarityToName
    - NFT: "Puppy Raffle" <- Name, "PR" <- Symbol

- enterRaffle: To let users enter into raffle
- refund: To let uers get their refund (hopefully before raffle start?)
- getActivePlayerIndex: Wow getter(we haven't thought about that), to get an active player index
- selectWinner: Contains winner selection mechanism
- withdrawFees: Owner can withdraw their fees via this function
- changeFeeAddress: facility to change or update fee address (probably only owner can so.)
- _isActivePlayer: Internal helper to confirm if queried player is active
- _baseURI: prefix or base of URI internal helper function
- tokenURI: returns the URI associated to the given tokenID

## Recon

- raffleDuration:

```solidity
// @question: shouldn't the raffle duration be a constant or an immutable one?
// @danger: variable raffleDuration may cause temporal inconsistencies
// @answer: No it shouldn't be and not necessary as well, However, This reality is missing its update capability
// It should have a update function with owner privileges
uint256 public raffleDuration;
```

- commonImageUri, rareImageUri, legendaryImageUri:

```solidity
// @Bug: should be a constant
// @issue: Excessive GAS causing high deployment cost.
string private commonImageUri = "ipfs://QmSsYRx3LpDAb1GZQm7zZ1AuHZjfbPkD6J7s9r41xu1mf8";

// @Bug: should be a constant
// @issue: Excessive GAS causing high deployment cost.
string private rareImageUri = "ipfs://QmUPjADFGEKmfohdTaNcWhp7VGk26h5jXDA7v3VtTnTLcW";

// @Bug: should be a constant
// @issue: Excessive GAS causing high deployment cost.
string private legendaryImageUri = "ipfs://QmYx6GsYAKnNzZ9A6NvEKV9nf1VaDzJrqDR23Y8YSkebLU";
```

- COMMON_RARITY, RARE_RARITY, LEGENDARY_RARITY:

```solidity
// @Question: Why it is public not private???
// @Guess: It might be allowed to be used by everyone.
// @answer: It's public for a automatic getter API availability, although guess was almost right.
uint256 public constant COMMON_RARITY = 70;

// @Question: Why it is public not private???
// @Guess: It might be allowed to be used by everyone.
// @answer: It's public for a automatic getter API availability, although guess was almost right.
uint256 public constant RARE_RARITY = 25;

// @Question: Why it is public not private???
// @Guess: It might be allowed to be used by everyone.
// @answer: It's public for a automatic getter API availability, although guess was almost right.
uint256 public constant LEGENDARY_RARITY = 5;
```

- Events: RaffleRefunded, FeeAddressChanged

```solidity
// @Bug: Forgot to mark player as indexed one
// @issue: Off-chain tools, APIs, and system analytics would get out of sync.
event RaffleRefunded(address player);

// @Bug: Forgot to mark newFeeAddress as indexed one
// @issue: Fee addresse(s) could be frequent. So, Off-chain tools, APIs, and system analytics, would get out of sync.
event FeeAddressChanged(address newFeeAddress);
```

- Constructor:

```solidity
// @info: Missing zero fee check
// @danger: Initially, Providing platform without any economic incentive
entranceFee = _entranceFee;

// @info: Missing zero fee address check
// @danger: fee collection may get lost.
feeAddress = _feeAddress;

// @info: First of all a variable raffle duration is a bit inconvenient itself.
// Moreover, missing raffle duration threshold checks i.e.,
// min duration and|or max duration
// @danger: a raffle may span either merely for a moment or could be a prolong invincible one,
// A DoS and|or unfair obscurity may occur.
raffleDuration = _raffleDuration;

// @Question: Raffle isn't started yet and raffle start time is being initialized here on initialization, is it okay??? or an escalation of temporal bypass????????
// @Answer: Yes, It's indeed shouldn't be initialized here, raffleStartTime with 0 value indicates that the raffle is just initialized and deployed, Moreover to say, Lottery haven't spinned yet.
raffleStartTime = block.timestamp;
```

- enterRaffle:

```solidity
// @BUG: players can enter into raffle after raffle close
// @danger: players may rigged the lottery

// @BUG: newPlayers with zero length can bypass the check
// @issue: emits RaffleEnter event on zero player(s) entry

// @BUG: Adversaries can create gaps into players array and can also pass phantom addresses
// @issue: DoS due to non-existent winner(s), nobody withdraws rewards.
require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");

// ...

// @BUG: DoS could occur on a high stream raffle
// @issue: Let's say a raffle pool got so high, high pool attracted so many players
// Checking duplicates on a very high array increases the complexity exponentially
// Therefore, due to heavy gas consumption eventually out of GAS would hit and DoS occur.
for (uint256 i = 0; i < players.length - 1; i++) {
    for (uint256 j = i + 1; j < players.length; j++) {
        require(players[i] != players[j], "PuppyRaffle: Duplicate player");
    }
}
```

- refund:

```solidity
// @info: ambiguous check
require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

// @BUG: Reentrancy is possible
// @issue: Player isn't removed from players array yet.
payable(msg.sender).sendValue(entranceFee);
```

- getActivePlayerIndex:

```solidity
function getActivePlayerIndex(address player) external view returns (uint256) {
    for (uint256 i = 0; i < players.length; i++) {
        // @BUG: Vulnerable to DoS
        // @danger: iterating over a very big array may lead transaction to out of GAS exception
        if (players[i] == player) {
            return i;
        }
    }
    // @BUG: Returns index 0 for non-existent player
    // @danger: mislead users especially users who're about to retreat and want their refund
    return 0;
}
```

- selectWinner:

```solidity
// @BUG: raffleStartTime should be recorded in `enterRaffle` function when there are at least 4 or more players have entered into raffle.
// @danger: raffle duration may occur too early after players entry.
require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");

// @BUG: blank spots weakens this check
// @danger: there could have so many blank sopts due to refund facility
// So, in these circumstances, the consequence that the players length is greater than or equal to 4 but there're actually less than 4 players, is possible.
require(players.length >= 4, "PuppyRaffle: Need at least 4 players");

// @BUG: global parameters i.e., msg.sender, block.timestamp, block.difficulty, etc can be front-run by validators and MEV Bots
// @danger: Possible front-run attack
uint256 winnerIndex =
    uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))) % players.length;

// @BUG: players array has blank spots so the players length isn't the actual length
// @danger: DoS could occur due to accounting exception
uint256 totalAmountCollected = players.length * entranceFee;

// @BUG: totalFees is of type uint64, heavy accumulation may break the protocol
// @danger: Potential overflow

// @BUG: fee is of type uint256 and then wrapped to uint64 here below, fee may wrap around
// @danger: wrap around behavior may occur
totalFees = totalFees + uint64(fee);

// @BUG: Again, global parameters, not so good choice
// @danger: Vulnerable to front-running
uint256 rarity = uint256(keccak256(abi.encodePacked(msg.sender, block.difficulty))) % 100;

// @BUG: raffleStartTime should reset to 0
// @danger: raffle duration may occur too early after players entry because of this reset just after winner selection
// Therefore, raffleStartTime should update into enterRaffle when there're at least 4 or more players into the raffle.
raffleStartTime = block.timestamp;
```

- withdrawFees:

```solidity
// @BUG: race-condition between players(malicious ones or attackers) and caller(hopfully owner)
// @danger: As we know there's a reentrancy in the refund function, a malicious player or user can deliberately fron-run the caller of this function
// and make the raffle active again each time whenever caller tries the call this function.
// Parallelly, malicious user drains the contract's balance through refund function.
require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```

- _isActivePlayer:

```solidity
function _isActivePlayer() internal view returns (bool) {
    // @BUG: Finding active player in a Big players array is dangerous
    // @danger: Gas exhaustion revert the lookup therefore DoS occurs
    for (uint256 i = 0; i < players.length; i++) {
        if (players[i] == msg.sender) {
            return true;
        }
    }
    return false;
}
```


## PoCs:

### testRaffleDurationCannotBeUpdated:

- input: (Paste this test into: `test/PuppyRaffleTest.t.sol::PuppyRaffleTest`)

```solidity
function testRaffleDurationCannotBeUpdated() public playersEntered {
    vm.warp(block.timestamp + duration + 1);
    vm.roll(block.number + 1);

    uint256 expectedPrizeAmount = ((entranceFee * 4) * 20) / 100;

    puppyRaffle.selectWinner();
    puppyRaffle.withdrawFees();
    assertEq(address(feeAddress).balance, expectedPrizeAmount);

    uint256 newRaffleDuration = 1 weeks;

    vm.expectRevert();
    puppyRaffle.setRaffleDuration(newRaffleDuration);
}
```

- output:

```bash
[⠊] Compiling...
[⠆] Compiling 21 files with Solc 0.7.6
[⠰] Solc 0.7.6 finished in 210.80ms
Error: Compiler run failed:
Warning (2462): Visibility for constructor is ignored. If you want the contract to be non-deployable, making it "abstract" is sufficient.
lib/openzeppelin-contracts/contracts/access/Ownable.sol:26:5: Warning: Visibility for constructor is ignored. If you want the contract to be non-deployable, making it "abstract" is sufficient.
    constructor () internal {
    ^ (Relevant source part starts here and spans across multiple lines).
Warning (2462): Visibility for constructor is ignored. If you want the contract to be non-deployable, making it "abstract" is sufficient.
lib/openzeppelin-contracts/contracts/introspection/ERC165.sol:24:5: Warning: Visibility for constructor is ignored. If you want the contract to be non-deployable, making it "abstract" is sufficient.
    constructor () internal {
    ^ (Relevant source part starts here and spans across multiple lines).
Warning (2462): Visibility for constructor is ignored. If you want the contract to be non-deployable, making it "abstract" is sufficient.
lib/openzeppelin-contracts/contracts/token/ERC721/ERC721.sol:93:5: Warning: Visibility for constructor is ignored. If you want the contract to be non-deployable, making it "abstract" is sufficient.
    constructor (string memory name_, string memory symbol_) public {
    ^ (Relevant source part starts here and spans across multiple lines).
Error (9582): Member "setRaffleDuration" not found or not visible after argument-dependent lookup in contract PuppyRaffle.
test/PuppyRaffleTest.t.sol:226:9: TypeError: Member "setRaffleDuration" not found or not visible after argument-dependent lookup in contract PuppyRaffle.
        puppyRaffle.setRaffleDuration(newRaffleDuration);
        ^---------------------------^
```

### testVariablesCanBeSetAsConstantsCauseExcessiveDeploymentCost:

```solidity
function testVariablesCanBeSetAsConstantsCauseExcessiveDeploymentCost() public {
    // First run : Run this function without any mutation
    // Second run: Run this function after mutating, commonImageUri, rareImageUri, legendaryImageUri, to constants, in src/PuppyRaffle.sol:PuppyRaffle contract
    uint256 gasBeforeDeployment = gasleft();

    vm.txGasPrice(1000000000000000000);
    puppyRaffle = new PuppyRaffle(entranceFee, feeAddress, duration);

    uint256 gasAfterDeployment = gasleft();

    uint256 gasConsumed = gasBeforeDeployment - gasAfterDeployment;
    uint256 gasPrice = tx.gasprice;
    uint256 totalGasCost = gasConsumed * gasPrice;

    console2.log("GAS before deployment: ", gasBeforeDeployment);
    console2.log("GAS after  deployment: ", gasAfterDeployment);
    console2.log("GAS Consumed         : ", gasConsumed);
    console2.log("GAS price            : ", gasPrice);
    console2.log("Total GAS cost       : ", totalGasCost);
}
```
- output:

    - Before Mutation:

    ```bash
    [⠊] Compiling...
    No files changed, compilation skipped

    Ran 1 test for test/PuppyRaffleTest.t.sol:PuppyRaffleTest
    [PASS] testVariablesCanBeSetAsConstantsCauseExcessiveDeploymentCost() (gas: 4498848)
    Logs:
    GAS before deployment:  1073720530
    GAS after  deployment:  1069228792
    GAS Consumed         :  4491738
    GAS price            :  1000000000000000000
    Total GAS cost       :  4491738000000000000000000

    Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 545.83µs (125.60µs CPU time)

    Ran 1 test suite in 3.93ms (545.83µs CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
    ```

    - After Mutation:
    
    ```bash
    [⠊] Compiling...
    No files changed, compilation skipped

    Ran 1 test for test/PuppyRaffleTest.t.sol:PuppyRaffleTest
    [PASS] testVariablesCanBeSetAsConstantsCauseExcessiveDeploymentCost() (gas: 4307371)
    Logs:
    GAS before deployment:  1073720530
    GAS after  deployment:  1069420269
    GAS Consumed         :  4300261
    GAS price            :  1000000000000000000
    Total GAS cost       :  4300261000000000000000000

    Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 611.28µs (155.29µs CPU time)

    Ran 1 test suite in 4.05ms (611.28µs CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
    ```

    - Difference:

    ```bash
    191477
    ```

###






<br/>
<br/>
<br/>
<br/>
<br/>

##### Auditor|Sec-Res|Hacker: *theirrationalone*