
# [M-01] Owner Can Rug Pull All Faucet Tokens via Flawed `burnFaucetTokens` Function, Breaking Protocol's Core Promise

| | |
| --- | --- |
| **Finding ID** | M-01 |
| **Contract** | `RaiseBoxFaucet.sol` |
| **Functions** | `burnFaucetTokens()` |
| **Severity** | Medium |

---

## Description

**Protocol's Promise & Invariant:**
The project documentation establishes a clear separation of roles. The `Claimer` is responsible for withdrawing tokens, while the `Owner` has administrative duties. A key limitation for the owner is explicitly stated: they "cannot claim faucet tokens." This implies a core invariant: **tokens held in the faucet contract are exclusively reserved for claimers and are inaccessible to the owner for personal withdrawal.**

**The Issue:**
This core invariant is broken by a logical flaw in the `burnFaucetTokens()` function. Instead of burning tokens directly from the contract's balance, the function first transfers the **entire token balance** of the faucet to the owner (`msg.sender`) and then burns the specified `amountToBurn` from the owner's now-inflated balance.

This creates a backdoor for the owner to bypass the "cannot claim" restriction and drain the entire token supply intended for the community. A call to `burnFaucetTokens(1)` would result in all tokens being rugged from the faucet.

```solidity
// src/RaiseBoxFaucet.sol

function burnFaucetTokens(uint256 amountToBurn) public onlyOwner {
    require(amountToBurn <= balanceOf(address(this)), "Faucet Token Balance: Insufficient");

    // The flaw: This line transfers the ENTIRE contract balance to the owner (msg.sender),
    // breaking the invariant that the owner cannot access faucet funds.
@>  _transfer(address(this), msg.sender, balanceOf(address(this)));

    // The burn operation then occurs on the owner's balance.
    _burn(msg.sender, amountToBurn);
}
```

This behavior directly contradicts the protocol's stated limitations and breaks the trust model where the faucet's funds are supposed to be segregated from the owner's control.

## Risk

**Likelihood**: High

* This flawed logic is triggered **every time** the `burnFaucetTokens()` function is called, regardless of the owner's intent. The function's design does not provide a way for the owner to burn a small amount without first sweeping the entire contract balance. Therefore, the likelihood of the dangerous state change (full fund transfer) occurring is high, as it's the only way the function can be used.

**Impact**: High

* The entire supply of faucet tokens can be drained from the contract, making it impossible for any user to claim them. This completely defeats the purpose of the faucet. Even if the owner's intention is benign (e.g., to burn a small number of tokens), the function forces a full withdrawal of all funds designated for the community, breaking the protocol's core promise and trust model.

## Proof of Concept (PoC)

The following Foundry test demonstrates that the owner can drain all faucet tokens by calling `burnFaucetTokens()` with a minimal amount of `1`, thereby breaking the "owner cannot claim" invariant.

**PoC Test Code (`test/TokenRugPull.t.sol`):**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

import {Test, console} from "forge-std/Test.sol";
import {RaiseBoxFaucet} from "../src/RaiseBoxFaucet.sol";

/// @notice Validates that burnFaucetTokens() transfers the entire contract balance
/// to the owner before burning a small amount, effectively draining faucet tokens.
/// This test uses only owner privileges and mirrors the report scenario.
contract TokenRugPullTest is Test {
    RaiseBoxFaucet internal faucet;
    address internal owner;
    address internal user = address(uint160(0xBEEF));

    function setUp() public {
        // The test contract becomes the owner because the constructor uses Ownable(msg.sender)
        owner = address(this);

        faucet = new RaiseBoxFaucet(
            "raiseboxtoken",
            "RB",
            1000 * 10 ** 18,
            0.005 ether,
            0.5 ether
        );
    }

    function test_OwnerCanDrainAllTokensViaBurnFunction() public {
        console.log("--- Starting Token Rug Pull Scenario ---");

        uint256 initialFaucetBalance = faucet.balanceOf(address(faucet));
        uint256 initialOwnerBalance = faucet.balanceOf(owner);

        console.log("Initial Faucet Token Balance:", initialFaucetBalance);
        console.log("Initial Owner Token Balance:", initialOwnerBalance);
        assertEq(initialOwnerBalance, 0, "Owner should start with zero tokens");
        assertTrue(initialFaucetBalance > 0, "Faucet should have a positive initial balance");

        // The owner calls burnFaucetTokens intending to burn only 1 wei.
        uint256 amountToBurn = 1;
        console.log("Owner calls burnFaucetTokens with amount:", amountToBurn);

        faucet.burnFaucetTokens(amountToBurn);

        uint256 finalFaucetBalance = faucet.balanceOf(address(faucet));
        uint256 finalOwnerBalance = faucet.balanceOf(owner);

        console.log("Final Faucet Token Balance:", finalFaucetBalance);
        console.log("Final Owner Token Balance:", finalOwnerBalance);

        // Assertions to prove the behavior
        assertEq(finalFaucetBalance, 0, "Faucet token balance should be fully drained");
        assertEq(
            finalOwnerBalance,
            initialFaucetBalance - amountToBurn,
            "Owner should now hold initial faucet balance minus burned amount"
        );

        // Additional confirmation: a user cannot claim tokens from an empty faucet
        // Advance time beyond cooldown to avoid false failure on cooldown check
        vm.warp(block.timestamp + 3 days + 1);
        vm.prank(user);
        vm.expectRevert(RaiseBoxFaucet.RaiseBoxFaucet_InsufficientContractBalance.selector);
        faucet.claimFaucetTokens();
        console.log("User claim correctly fails as faucet is empty, confirming impact.");

        console.log("--- Token Rug Pull Scenario Verified ---");
    }
}
```

**Test Execution and Results:**

```bash
forge test --match-path test/TokenRugPull.t.sol -vv

Ran 1 test for test/TokenRugPull.t.sol:TokenRugPullTest
[PASS] test_OwnerCanDrainAllTokensViaBurnFunction() (gas: 99964)
Logs:
  --- Starting Token Rug Pull Scenario ---
  Initial Faucet Token Balance: 1000000000000000000000000000
  Initial Owner Token Balance: 0
  Owner calls burnFaucetTokens with amount: 1
  Final Faucet Token Balance: 0
  Final Owner Token Balance: 999999999999999999999999999
  User claim correctly fails as faucet is empty, confirming impact.
  --- Token Rug Pull Scenario Verified ---
```

The test will pass. The logs show the faucet's balance dropping to zero, while the owner's balance increases by the full amount, proving the rug pull capability and the violation of the protocol's core promise.

## Recommended Mitigation

The `burnFaucetTokens` function should not involve transferring tokens to the owner. It should burn tokens directly from the faucet contract's own balance. The standard `_burn` function in OpenZeppelin's ERC20 implementation accepts the account from which to burn as a parameter.

The corrected implementation should burn from `address(this)`.

```diff
// src/RaiseBoxFaucet.sol

function burnFaucetTokens(uint256 amountToBurn) public onlyOwner {
    require(amountToBurn <= balanceOf(address(this)), "Faucet Token Balance: Insufficient");

-   // transfer faucet balance to owner first before burning
-   // ensures owner has a balance before _burn (owner only function) can be called successfully
-   _transfer(address(this), msg.sender, balanceOf(address(this)));
-
-   _burn(msg.sender, amountToBurn);
+   // Correctly burn tokens directly from the faucet's balance (address(this))
+   _burn(address(this), amountToBurn);
}
```

This change aligns the function's behavior with its name and purpose, prevents the owner from draining the faucet's funds, and restores the protocol's trust model.
