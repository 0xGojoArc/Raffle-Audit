Denial of Service

### [S-#] Looping through players array to check for duplicates in `PuppyRaffle::enterRaffle` is potential Denial of service (DoS) attack, incrementing gas costs for future entrants

**Description:** The `PuppyRaffle::enterRaffle` function loops through `players` array to check for duplicates. However, the longer the `PuppyRaffle::players` is the more checks the new player will have to make. The gas cost for players who enter right when the raffle starts will be significantly lower than those enter later. Every additional address in the `players` array is an additional check the loop will have to make.

```javascript
    function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
        }
```

**Impact:** The gas costs for raffle entrants will greatly increase as more players enter the raffle. Discouraging later users from entering. 

An attackers might make the array `PuppyRaffle::players` so big that no one else can enter, guaranteeing themselves the win.


**Proof of Concept:** 

If we have two sets of 100 players, the gas costs will be as such:
1. 1st 100 players: ~6252128
2. 2nd 100 players: ~18068218

<details>
<summary>PoC</summary>
Place the following test in the `PuppyRaffleTest.t.sol` file.

```javascript
        function test_denialOfService() public {
        vm.txGasPrice(1);

        uint256 playersNum = 100;
        address[] memory players = new address[](playersNum);
        for(uint256 i = 0; i < playersNum; i++){
            players[i] = address(i);
        }

        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);
        uint256 gasEnd = gasleft();
        uint256 gasUsedFirst = (gasStart - gasEnd) * tx.gasprice;
        console.log("the cost costed for first 100 players:", gasUsedFirst);

        // Gas Cost for next 100 players

        address[] memory playersTwo = new address[](playersNum);
        for(uint256 i = 0; i < playersNum; i++){
            playersTwo[i] = address(i + playersNum); // add will be 101, 101, 102
        }

        uint256 gasStartSecond = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(playersTwo);
        uint256 gasEndSecond = gasleft();
        uint256 gasUsedSecond = (gasStartSecond - gasEndSecond) * tx.gasprice;
        console.log("the cost costed for first next 100 players:", gasUsedSecond);

        assert(gasUsedFirst < gasUsedSecond);
    }
````
</details>  

**Recommended Mitigation:** 

1. Users can make multiple wallet address to enter the raffle so checking for duplicates is not so effective, consider removing the check for duplicates.
2. Consider using a mapping to check duplicates. This would allow you to check for duplicates in constant time, rather than linear time. You could have each raffle have a uint256 id, and the mapping would be a player address mapped to the raffle Id. [Because the hash function (it converts the key into a fixed-size hash value, which serves as an index or pointer to the location where the corresponding value is stored.) provides a direct path to the storage location, the lookup time remains constant regardless of the number of key-value pairs in the mapping.]

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
Alternatively, you could use OpenZeppelin's EnumerableSet library.


# Gas

### [G-1] State variables should be declared constant or immutable

Instances: 

- `PuppyRaffle::raffleDuration` should be `immutable`
- `PuppyRaffle::commonImageUri`should be `constant`
- `PuppyRaffle::rareImageUri`should be `constant`
- `PuppyRaffle::legendaryImageUri`should be `constant`

Reading from storage is much expensive than reading from constant or immutable variable.


### [G-2] Storage variables in a loop should be cached

Everytime you call `players.length` you read from storage as opposed to reading from memory using `playersLength`.

```diff
+        uint256 playersLength = players.length;
-        for (uint256 i = 0; i < players.length - 1; i++) {
+        for (uint256 i = 0; i < playersLength - 1; i++) {
-        for (uint256 j = i + 1; j < players.length; j++) {
+        for (uint256 j = i + 1; j < playersLength; j++) {
            require(players[i] != players[j], "PuppyRaffle: Duplicate player");
        }
}
```





### [I-1] Solidity pragma should be specific, not wide

Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.0;`, use `pragma solidity 0.8.0;`

<details><summary>1 Found Instances</summary>

- Found in src/PuppyRaffle.sol [Line: 2](src/PuppyRaffle.sol#L2)

	```solidity
	pragma solidity ^0.7.6;
	```
</details>


### [I-2] Using outdates version of Solidity is not recommended.

solc frequently releases new compiler versions. Using an old version prevents access to new Solidity security checks. We also recommend avoiding complex pragma statement.

**Recommendation:**
Deploy with a recent version of Solidity (at least 0.8.0) with no known severe issues.

Use a simple pragma version that allows any of these versions. Consider using the latest version of Solidity for testing.

Please see [Slither](https://github.com/crytic/slither/wiki/Detector-Documentation#incorrect-versions-of-solidity)




### [I-3] Missing checks for `address(0)` when assigning values to address state variables

Check for `address(0)` when assigning values to address state variables.

<details><summary>2 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 76](src/PuppyRaffle.sol#L76)

	```solidity
	        feeAddress = _feeAddress;
	```

- Found in src/PuppyRaffle.sol [Line: 216](src/PuppyRaffle.sol#L216)

	```solidity
	        feeAddress = newFeeAddress;
	```

</details>

