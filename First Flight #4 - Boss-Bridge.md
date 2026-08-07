## Business Logic

- Protocol providing facility to bridge ERC20 tokens from L1 to L2.
- The L2 part of the bridge is still under construction so it's currently out-of-scope
- Bridge allows to deposit tokens.
- Tokens are held into a vault.

                          users
                            |
                         Deposit (users deposit tokens only ERC20)
                            |
                          vault (vault keeps ERC20 tokens deposited by users)
                            |
                          event (successful deposit emits event)
                            .
                            .
                            .
                     off-chain worker (off-chain worker/mechanism picks up the event signal)
                            |
                    parsing depositions
                            |
                            L2 (triggers L2 mint logic)
                            |
                          mint (mints corresponding tokens)
                            |
                          users (users get their tokens on L2)

- V1 security considerations and mechanisms
    - pause (exclusive to owner) for emergency situations
    - deposit threshold(upper bound) (only limited deposits are allowed)
    - withdrawals must be approved by bridge operator.


- Withdraw Process:
                              users
                                |
                             withdraw (users send withdraw request)
                                |
                          bridge operator
                                |
                         signing requests (bridge operator confirms, validates and signs withdrawal requests) (Bridge, L1 service validates the withdrawal requests and confirms that the requests are first originated a successful deposit)
                                |
                            L2 component (confirmed requests submitted on the L2 component of the bridge) (out of scope)



- Actors/Roles
    * Bridge Owner: A centralized bridge owner who can:

        * pause/unpause the bridge in the event of an emergency

        * set `Signers` (see below)

    * Signer: Users who can "send" a token from L2 -> L1. (L2 role is out-of-Scope)

    * Vault: The contract owned by the bridge that holds the tokens.

    * Users: Users mainly only call `depositTokensToL2`, when they want to send tokens from L1 -> L2.


## Known Issues

- We are aware the bridge is centralized and owned by a single user, aka it is centralized.

- We are missing some zero address checks/input validation intentionally to save gas.

- We have magic numbers defined as literals that should be constants.

- Assume the `deployToken` will always correctly have an L1Token.sol copy, and not some [weird erc20](https://github.com/d-xo/weird-erc20)


## Possible Attack vectors

- Owner can be compromised (permanent freeze and vault sweep)

- Vault could have accounting inconsistencies

- Vault ownership could be obscure or hijacked

- Users deposit tokens however tokens may get burned and never minted on L2

- Users deposit tokens to L2 may get very few tokens on L2

- Malicious users may be able to mint infinite tokens on L2

- Users deposit tokens on L2 may never get their tokens on L2 due to missing event emission on L1 or uncertain cause on event mechanism and never able to trigger mint on L2

- Users may be able to deposit vast amount of tokens exceeding deposit threshold

- Bridge operator may fail to detect duplicate replaying requests

- Users may be able to withdraw infinite tokens due to some logical inconsistencies

- Malicious users may be able to bypass bridge operator and able to withdraw tokens

- Protocol may get pause once and has no capability to unpause forever

- Magical numbers aka literals may influence accounting and computation which may emerge undesired consequences

- There's a contract called `TokenFactory.sol::TokenFactory` which might be using `CREATE2` to deploy `Token`(s) on chains (chains in scope: `Ethereum mainnet`, `ZkSync Era`), Specifically on `ZkSync Era` Deployment on different chains may not support `CREATE2`

- Tokens value on different chains could be different i.e., 10 USDC == 1 ETH on Ethereum mainnet, On the other hand, 10 USDC == 2 ETH on ZkSync Era, is it possible? requires a research...

## Reconnaissance

- `L1Token.sol::L1Token`

```solidity
constructor() ERC20("BossBridgeToken", "BBT") {
    // @info: minting on deployment
    // @info: msg.sender is not sanitized
    // 10_000_000_e18 or 10 million erc20 with 18 decimals of precision
    _mint(msg.sender, INITIAL_SUPPLY * 10 ** decimals());
}
```

- `L1Vault.sol::L1Vault`

```solidity
function approveTo(address target, uint256 amount) external onlyOwner {
    // @info: owner is approving spender to spend
    // @issue: if target is a zero address amount lost forever
    // @question: they already know about zero addresses, is it really a valid issue?
    token.approve(target, amount);
}
```

- `TokenFactory.sol::TokenFactory`

```solidity
// @info: Missing Indexed keyword
event TokenDeployed(string symbol, address addr);

//...

function deployToken(string memory symbol, bytes memory contractBytecode) public onlyOwner returns (address addr) {
    // @info: using create (CREATE2 EVM OPCODE)
    // @issue: might not compatible with zksync era
    assembly {
        addr := create(0, add(contractBytecode, 0x20), mload(contractBytecode))
    }
    s_tokenToAddress[symbol] = addr;
    emit TokenDeployed(symbol, addr);
}
```

- `BossBridge.sol::BossBridge`

```solidity
// @info: missing indexed keyword
event Deposit(address from, address to, uint256 amount);

constructor(IERC20 _token) Ownable(msg.sender) {
    token = _token;
    vault = new L1Vault(token);
    // Allows the bridge to move tokens out of the vault to facilitate withdrawals
    // @info: maximum of uint256 is allowed to this contract (the boss bridge) to spend from vault
    vault.approveTo(address(this), type(uint256).max);
}

function depositTokensToL2(address from, address l2Recipient, uint256 amount) external whenNotPaused {
    // @info: due to DEPOSIT_LIMIT, a race condition could occur among users
    // @issue: front-running, there should have a deposit limit per user instead
    if (token.balanceOf(address(vault)) + amount > DEPOSIT_LIMIT) {
        revert L1BossBridge__DepositLimitReached();
    }
    // @info: from can be any arbitrary address targeted by attacker (msg.sender) after approval phishing
    token.safeTransferFrom(from, address(vault), amount);

    // Our off-chain service picks up this event and mints the corresponding tokens on L2
    emit Deposit(from, l2Recipient, amount);
}

//...

function withdrawTokensToL1(address to, uint256 amount, uint8 v, bytes32 r, bytes32 s) external {
    sendToL1(
        v,
        r,
        s,
        abi.encode(
            address(token),
            0, // value
            abi.encodeCall(IERC20.transferFrom, (address(vault), to, amount))
        )
    );

    // @info: missing withdraw event
}

```





<br/>
<br/>
<br/>
<br/>
<br/>

##### Auditor|Sec-Res|White-Hat: *theirrationalone*