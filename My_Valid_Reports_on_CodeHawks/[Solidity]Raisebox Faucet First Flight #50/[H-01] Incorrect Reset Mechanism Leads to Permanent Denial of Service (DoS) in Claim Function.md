
# **[H-01] Incorrect Reset Mechanism Leads to Permanent Denial of Service (DoS) in Claim Function**

| | |
| --- | --- |
| **Finding ID** | H-01 |
| **Contract** | `RaiseBoxFaucet.sol` |
| **Functions** | `claimFaucetTokens()` |
| **Severity** | High |

---

## **Description**

**Normal Behavior:**
The `RaiseBoxFaucet` contract is designed to allow users to claim testnet tokens once every 3 days. To control the number of claims per day, the contract implements a `dailyClaimLimit` and a counter, `dailyClaimCount`. This counter is intended to reset to zero every 24 hours, allowing a new batch of claims to occur the next day.

**The Issue:**
The logic to reset the `dailyClaimCount` is located inside the `claimFaucetTokens()` function, but it is placed *after* the check that enforces the `dailyClaimLimit`.

If the `dailyClaimLimit` (defaulting to 100) is reached on any given day, any subsequent call to `claimFaucetTokens()` will revert with the error `RaiseBoxFaucet_DailyClaimLimitReached()`. Because this check occurs at the beginning of the function, the code block responsible for resetting the counter (`dailyClaimCount = 0;`) becomes permanently unreachable.

As a result, `dailyClaimCount` will remain stuck at its maximum value forever, preventing any user from ever claiming tokens again. This constitutes a Permanent Denial of Service (DoS) that renders the faucet's core functionality unusable.

```solidity
// src/RaiseBoxFaucet.sol

function claimFaucetTokens() public {
    // ... (other checks)

@>  if (dailyClaimCount >= dailyClaimLimit) {
@>      revert RaiseBoxFaucet_DailyClaimLimitReached();
    }

    // ... (ETH drip logic)

    /**
     *
     * @param lastFaucetDripDay tracks the last day a claim was made
     * @notice resets the @param dailyClaimCount every 24 hours
     */
@>  if (block.timestamp > lastFaucetDripDay + 1 days) {
@>      lastFaucetDripDay = block.timestamp;
@>      dailyClaimCount = 0;
    }

    // Effects
    lastClaimTime[faucetClaimer] = block.timestamp;
@>  dailyClaimCount++;

    // Interactions
    _transfer(address(this), faucetClaimer, faucetDrip);

    emit Claimed(msg.sender, faucetDrip);
}
```

As shown, if `dailyClaimCount >= dailyClaimLimit`, the function reverts before the reset logic block `if (block.timestamp > lastFaucetDripDay + 1 days)` can ever be executed.

### **Risk**

**Likelihood**: High

* An attacker or a group of users can easily call the function 100 times (the default limit) using different addresses within a short period. On a testnet, this is trivial and has a low gas cost.

**Impact**: Critical

* This vulnerability permanently disables the primary function of the contract. No user will be able to claim faucet tokens ever again, rendering the entire faucet useless. The documentation explicitly mentions a "daily" limit, implying a recurring functionality, which this bug breaks permanently.

### **Proof of Concept (PoC)**

The following Foundry test demonstrates how a series of claims within one day can permanently lock the contract, preventing any future claims even after time has passed.

**PoC Test Code (`test/PermanentDos.t.sol`):**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

import {Test, console} from "forge-std/Test.sol";
import {RaiseBoxFaucet} from "../src/RaiseBoxFaucet.sol";

// This test demonstrates a permanent DoS by exhausting the dailyClaimLimit
// and shows that the reset block is never reached because it sits after the revert check.
// Comments are in English and outputs are plain English.
contract PermanentDosTest is Test {
    RaiseBoxFaucet internal faucet;
    uint256 internal dailyLimit;

    function setUp() public {
        faucet = new RaiseBoxFaucet(
            "raiseboxtoken",
            "RB",
            1000 * 10 ** 18, // faucetDrip
            0.005 ether,      // sepEthAmountToDrip
            0.5 ether         // dailySepEthCap
        );

        // Advance time to satisfy first-claim cooldown for fresh addresses
        vm.warp(block.timestamp + 3 days);

        // Fund faucet with a bit of ETH to avoid ETH drip checks interfering
        vm.deal(address(faucet), 1 ether);

        // Read the default daily limit (expected 100)
        dailyLimit = faucet.dailyClaimLimit();
        console.log("Daily claim limit:", dailyLimit);
    }

    function test_PermanentDoS_ByExhaustingDailyClaimLimit() public {
        console.log("--- Starting Permanent DoS scenario ---");

        // Step 1: Exhaust the dailyClaimLimit with unique addresses in the same day
        for (uint256 i = 1; i <= dailyLimit; i++) {
            // Avoid precompile addresses (0x000...01 to 0x000...09) by offsetting
            address claimant = address(uint160(1000 + i));
            vm.prank(claimant);
            faucet.claimFaucetTokens();
        }

        // Verify we reached the cap
        assertEq(
            faucet.dailyClaimCount(),
            dailyLimit,
            "Daily claim count should have reached the limit"
        );
        console.log("Reached max daily claims:", faucet.dailyClaimCount());

        // Step 2: Another user tries to claim in the same day (should fail)
        address lateUser = address(0xBEEF);
        vm.prank(lateUser);
        vm.expectRevert(RaiseBoxFaucet.RaiseBoxFaucet_DailyClaimLimitReached.selector);
        faucet.claimFaucetTokens();
        console.log("Expected failure for extra claim in same day confirmed.");

        // Step 3: Advance time by 2 days
        vm.warp(block.timestamp + 2 days);
        console.log("Advanced time by two days.");

        // Step 4: A legitimate user attempts to claim after two days
        // Due to the placement of the reset block AFTER the revert, this still fails.
        address legitimateUser = address(0xCAFE);
        vm.prank(legitimateUser);
        vm.expectRevert(RaiseBoxFaucet.RaiseBoxFaucet_DailyClaimLimitReached.selector);
        faucet.claimFaucetTokens();
        console.log("Claim after two days still fails due to non-reset.");

        // Final confirmation that the counter did not reset
        assertEq(
            faucet.dailyClaimCount(),
            dailyLimit,
            "Daily claim count did not reset after time advance"
        );
        console.log("--- Permanent DoS confirmed: contract remains locked at daily limit ---");
    }
}
```

**Test Execution and Results:**
Run the test using the following command:

```bash
forge test --match-path test/PermanentDos.t.sol -vv
```

The test will pass, confirming the vulnerability. The output logs clearly show that the `dailyClaimCount` remains at the limit even after two days have passed, and subsequent claims fail as expected.

```bash
forge test --match-path test/PermanentDos.t.sol -vv

[PASS] test_PermanentDoS_ByExhaustingDailyClaimLimit() (gas: 11614201)
Logs:
  Daily claim limit: 100
  --- Starting Permanent DoS scenario ---
  Reached max daily claims: 100
  Expected failure for extra claim in same day confirmed.
  Advanced time by two days.
  Claim after two days still fails due to non-reset.
  --- Permanent DoS confirmed: contract remains locked at daily limit ---

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.44ms
```

### **Recommended Mitigation**

The `dailyClaimCount` reset logic must be moved to the beginning of the `claimFaucetTokens()` function, before any checks that might cause a revert. This ensures that the first transaction on a new day will successfully reset the counter, allowing claim functionality to resume.

Apply the following changes to `src/RaiseBoxFaucet.sol`:

```diff
// src/RaiseBoxFaucet.sol

function claimFaucetTokens() public {
+   /**
+    * @notice Resets the daily claim count every 24 hours.
+    * @dev This check is placed at the beginning to prevent a permanent DoS if the daily limit is reached.
+    */
+   if (block.timestamp > lastFaucetDripDay + 1 days) {
+       lastFaucetDripDay = block.timestamp;
+       dailyClaimCount = 0;
+   }

    // Checks
    faucetClaimer = msg.sender;

    if (block.timestamp < (lastClaimTime[faucetClaimer] + CLAIM_COOLDOWN)) {
        revert RaiseBoxFaucet_ClaimCooldownOn();
    }

    if (faucetClaimer == address(0) || faucetClaimer == address(this) || faucetClaimer == Ownable.owner()) {
        revert RaiseBoxFaucet_OwnerOrZeroOrContractAddressCannotCallClaim();
    }

    if (balanceOf(address(this)) <= faucetDrip) {
        revert RaiseBoxFaucet_InsufficientContractBalance();
    }

    if (dailyClaimCount >= dailyClaimLimit) {
        revert RaiseBoxFaucet_DailyClaimLimitReached();
    }

    // ... (rest of the function, remove the old reset block)

-   /**
-    *
-    * @param lastFaucetDripDay tracks the last day a claim was made
-    * @notice resets the @param dailyClaimCount every 24 hours
-    */
-   if (block.timestamp > lastFaucetDripDay + 1 days) {
-       lastFaucetDripDay = block.timestamp;
-       dailyClaimCount = 0;
-   }

    // Effects
    lastClaimTime[faucetClaimer] = block.timestamp;
    dailyClaimCount++;
    // ...
}
```

This simple reordering guarantees that the reset logic is independent of the current claim count, completely resolving the permanent DoS vulnerability.
