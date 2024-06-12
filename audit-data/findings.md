### [H-1] Storage variables are visible to everyone resulting in the password not being private

**Description:** All data stored on-chain is visible to everyone. This means that the password stored in 
the storage variable `PasswordStore::s_password` is not private and get be retrieved without using the
`PasswordStore::getPassword()` function.

A PoC is provided in the section below.

**Impact:** Anyone can read the password, severely compromising the security of the protocol.

**Proof of Concept:** Proof of Code
The following steps allow anyone to read the password from storage:

1. Deploy the contract to a local anvil chain

```bash
anvil 
make deploy
```

2. Inspect the storage location where the password is stored. Its the second storage slot. The first one stores the owner.

```bash
cast storage <contract-address> 1
````

3. Cast the encoded hex value into a ascii string

```bash
cast --to-ascii 0x6d7950617373776f726400000000000000000000000000000000000000000014
```

4. See the logged string matches the password

```bash
myPassword
```


**Recommended Mitigation:** The overall architecture of the protocol builds on the assumption that
the password can be saved in storage without being visible to everyone, which is flawed by design.
The password could be encrypted off-chain before being stored in storage, and again decrypted
off-chain when needed. Although this is possible, it does not make much sense since the user would
need to remember a password to encrypt and decrypt the password stored in the contract. 

### [H-2] `PasswordStore::setPassword` has no access control, allowing anyone to set the password

**Description:** The function to set the stored password `PasswordStore::setPassword` has no
access control to restrict invocation. The intended functionality is to allow the deployer to set the
password and deny anyone else from setting it. 

Quote from the NatSpec: 
> This function allows only the owner to set a new password.

**Impact:** Anyone can set the password, compromising the intended functionality of the protocol.

**Proof of Concept:** The following tests proofs that anyone can set the password:
<details>
<Summary>Code Snippet</Summary>

```javascript
function test_anyone_can_set_password(address randomAddress) public {
        vm.assume(randomAddress != owner);
        vm.prank(randomAddress);
        string memory passwordFromAStranger = "youGotPwnd";
        passwordStore.setPassword(passwordFromAStranger);
        vm.prank(owner);
        string memory actualPassword = passwordStore.getPassword();
        assertEq(actualPassword, passwordFromAStranger);
    }
```

</details>

**Recommended Mitigation:** Add an access control conditional or modifier to the `PasswordStore::setPassword` function to restrict invocation to the owner.

<details>
<Summary>Code Snippet</Summary>

```javascript
   function setPassword(string memory newPassword) external {
        if (msg.sender != owner) {
            revert PasswordStore__NotOwner();
        }
        s_password = newPassword;
        emit SetNetPassword();
    }
```

</details>

### [I-1] The NatSpec in `PasswordStore::getPassword` suggests a parameter that is not present in the function signature

**Description:** The NatSpec in `PasswordStore::getPassword` suggests that a newPassword should be passed
to the function, but the function signature does not include a parameter for newPassword.

**Impact:** Misleading documentation can lead to confusion and misunderstanding of the intended functionality.


**Recommended Mitigation:** Remove the parameter from the NatSpec in `PasswordStore::getPassword` to align with the function signature.

```diff
    /*
     * @notice This allows only the owner to retrieve the password.
-    * @param newPassword The new password to set.
     */
    function getPassword() external view returns (string memory) {
```
