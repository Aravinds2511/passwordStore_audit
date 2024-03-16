### [H-1] Storing password onchain is not private and anyone can see it.

**Description:** All data stored on-chain is visible to anyone and can be read directly from the blockchain. The `PasswordStore::s_password` variable is intended to be private and only accessed through `PasswordStore::getPassword` function, which is intended to be only call by the owner of the contract.

**Impact:** Anyone can read the private password severly affecting the functionality of the protocol 

**Proof of Concept:**(Proof of Code)

The below test shows how anyone can read the password directly from the blockchain.

1. create local running chain 
```bash
anvil
```

2. Deploy the contract to the chain 
```
make deploy
```

3. Run the storage tool (slot 1 - where password is stored )

```
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 1 --rpc-url http://127.0.0.1:8545
```
you get - `0x6d7950617373776f726400000000000000000000000000000000000000000014` in bytes formate 

4. Convert to string 
```
cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
```
you get - `myPassword` - which is the password we stored.


**Recommended Mitigation:** Due to this the overall architecture of the contract should be rethought. One could encrypt the password off-chain and then store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the password. However, you'd also likely want to remove the view function as you wouldn't want the user to accidentally send a transaction with the password that decrypts your password.


### [H-2] `PasswordStore::setPassword` has no access control, anyone can set the password.

**Description:** The `PasswordStore::setPassword` is an external function where anyone can set the password, Howerver the what the contract intended was `This function allows only the owner to set a new password`

```javascript
function setPassword(string memory newPassword) external {
        //@audit - There is no access control
        s_password = newPassword;
        emit SetNetPassword();
    }
```

**Impact:** Anyone can set or change the password severely violating the contracts intended functionality.

**Proof of Concept:** Add the following to `PasswordStore.t.sol` test file.

<details>
<summary>Code</summary>

```javascript
function test_anyone_can_set_password(address rand_address) public {
        vm.assume(rand_address != owner);
        vm.startPrank(rand_address);
        string memory password_set = "My_Password";
        passwordStore.setPassword(password_set);

        vm.startPrank(owner);
        string memory actual_password = passwordStore.getPassword();
        assertEq(password_set, actual_password);
    }
```
</details>


**Recommended Mitigation:** Add an access control conditional to the `setPassword` function.

```javascript
if(msg.sender != s_owner){
    revert PasswordStore__NotOwner();
}
```

### [I-1] The `PasswordStore::getPassword` natspec indicates a parameter that doesn't exist, making the natspec incorrect.

**Description:** 
```javascript
 /*
     * @notice This allows only the owner to retrieve the password.
     //@audit no newPassword param
     * @param newPassword The new password to set.
     */
    function getPassword() external view returns (string memory)
```
The `PasswordStore::getPassword` function signature is `getPassword` but natspec says `getPassword(string)` which is incorrect.

**Impact:** The natspec is incorrect.

**Recommended Mitigation:** Remove the incorrect natspec line.

```diff
-   * @param newPassword The new password to set.
```