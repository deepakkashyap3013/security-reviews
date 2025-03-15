### [H-1] Storing the password on-chain makes it visible to anyone!

**<ins>Description:<ins>** 
- All data stored on-chain is visible to anyone, no matter the solidity visibility keyword meaning the password is not actually a private password. The `PasswordStore::s_password` variable is intended to be a private variable and only accessed through `PasswordStore::getPassword()` function, which is intended to be only called by the owner of the contract.

- We show one such method of reading any data off chain below.

**<ins>Impact:</ins>** 
- Anyone can read the private password, thus severly breaking the functionality of the protocol.

**<ins>Proof of Concept:</ins>**
- Below test case shows anyone can read the password directly from the blockchain.

1. Create a locally running chain
```sh
make anvil
```

2. Deploy the contract to the chain
```sh
make deploy
```

3. Run the storage tool

- We use `1` here because, that's the storage slot for `s_password` in the contract.

```sh
cast storage <deployed-contract-address> 1 --rpc-url http://127.0.0.1:8545
```

4. Parse bytes32 to string
```sh
cast parse-bytes32-string <bytes32-output-from-prev-step>
```

**<ins>Recommended Mitigation:</ins>** 
- Due to this, the overall architecture of the protocol needs to be rethought. One could encrypt the password off-chain and then store the password on-chain. This would require the user to remember another password off-chain to decrypt the password. 
- However, you'd also likely want to remove the view function as you wouldn't want the user to accidentally send a transaction with the password that decrypts your password.



### [H-2] `PasswordStore::setPassword(string memory)` missing access control, meaning non-owner could change the password

**<ins>Description:</ins>** 
- The `PasswordStore::setPassword(string memory)` has visibility external that means anyone can call this function. Since the expected functionality is that only owner should be able to set the password, which deviates from the fact that now any user can able to set the password.

```js
    function setPassword(string memory newPassword) external {
@>          // @audit - no access modifier here!
            s_password = newPassword;
            emit SetNetPassword();
    }
```

**<ins>Impact:</ins>** 
- Anyone can set/change the password other than the owner, severly breaking the contract intended functionality.

**<ins>Proof of Concept:</ins>**
- Add the following to the `PasswordStore.t.sol`

<Details>

<summary>Code</Summary>

```js
    function test_non_owner_can_set_password(address nonOwner) public {
        vm.assume(nonOwner != owner);

        vm.startPrank(owner);
        string memory owner_password = "ownerpassword";
        passwordStore.setPassword(owner_password);
        assertEq(owner_password, passwordStore.getPassword());
        vm.stopPrank();

        vm.startPrank(nonOwner);
        string memory random_user_password = "anypassword";
        passwordStore.setPassword(random_user_password);
        vm.stopPrank();

        vm.prank(owner);
        assert(keccak256(bytes(owner_password)) != keccak256(bytes(passwordStore.getPassword())));
    }
```

</Details>

**<ins>Recommended Mitigation:</ins>** 
- Add an access control condition to `PasswordStore::setPassword(string memory)` 

```js
@>  if(msg.sender != s_owner){
        revert PasswordStore__NotOwner();
    }
```


### [I-1] `PasswordStore::getPassword()` natspec indicates a paramter that doesn't exists, causing the natspec to be incorrect

**<ins>Description:</ins>** 

- The `PasswordStore::getPassword()` function signature is `getPassword()` which the natspec says it should be `getPassword(string)`.

```js
    /*
     * @notice This allows only the owner to retrieve the password.
@>   * @param newPassword The new password to set.
     */
    function getPassword() external view returns (string memory) {...}
```

**<ins>Impact:</ins>** 
- The natspec is incorrect.

**<ins>Proof of Concept:</ins>**
- N/A

**<ins>Recommended Mitigation:</ins>** 
- Remove the incorrect natspec line.

```diff
-   * @param newPassword The new password to set.
```