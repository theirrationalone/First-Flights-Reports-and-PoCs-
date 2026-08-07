# Boss Bridge Audit Report

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
                         signing requests (bridge operator confirms, validates and signs withdrawal requests) (bridge L1 service validates the withdrawal requests and confirms that the requests are first originated a successful deposit)
                                |
                            L2 component (confirmed requests submitted on the L2 component of the bridge) (out of scope)



- Actors/Roles
    * Bridge Owner: A centralized bridge owner who can:

        * pause/unpause the bridge in the event of an emergency

        * set `Signers` (see below)

    * Signer: Users who can "send" a token from L2 -> L1.

    * Vault: The contract owned by the bridge that holds the tokens.

    * Users: Users mainly only call `depositTokensToL2`, when they want to send tokens from L1 -> L2.


## Attack vectors


                               
