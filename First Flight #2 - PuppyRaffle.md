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