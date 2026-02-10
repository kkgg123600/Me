
# Unchecked arithmetic in `_mint` allows supply overflow wraparound

## Root + Impact

`_mint` uses Yul `add` without overflow checks for `totalSupply` and account balance. When `supply + value` exceeds `type(uint256).max`, it wraps modulo 2^256, setting `totalSupply` to an incorrect value (e.g., 0). This breaks core accounting invariants and can desynchronize integrators.

## Description

- Normal behavior: Minting increases `totalSupply` and the recipient balance without altering existing supply invariants.
- Issue: `_mint` uses unchecked addition; in Yul arithmetic, overflows wrap silently.

```solidity
// src/helpers/ERC20Internals.sol
134: function _mint(address account, uint256 value) internal {
...
146:     let supply := sload(supplySlot)
@>147:     sstore(supplySlot, add(supply, value))
...
@>154:     sstore(accountBalanceSlot, add(accountBalance, value))
}
```

## Risk

**Likelihood**:

- Occurs when a derived contract exposes minting to users and large `value` is passed, or supply is already high and further mint causes wrap.

**Impact**:

- Corrupts `totalSupply` and per-account balances, breaking downstream integrations and economic assumptions.

## Proof of Concept

```solidity
// test/PocMintOverflow.t.sol
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

contract PocMintOverflow is Test {
    TestTokenHarness private token;
    address private a = address(0xA1);
    address private b = address(0xB1);

    function setUp() public {
        token = new TestTokenHarness();
    }

    function test_MintOverflowWrapsTotalSupply() public {
        token.exposedMint(a, 1);
        token.exposedMint(b, type(uint256).max);

        assertEq(token.totalSupply(), 0, "supply wrapped to 0 after overflow");
        assertEq(token.balanceOf(b), type(uint256).max, "recipient received max amount");
    }
}
```

## Test result

```bash
➜  2025-12-token-0x git:(main) ✗ forge test --match-path  test/PocMintOverflow.t.sol -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/PocMintOverflow.t.sol:PocMintOverflow
[PASS] test_MintOverflowWrapsTotalSupply() (gas: 62015)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.25ms (186.09µs CPU time)

Ran 1 test suite in 30.37ms (1.25ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Explanation:

- Step 1: We mint `1` token to address `a`. This sets an initial non-zero supply.
- Step 2: We mint `type(uint256).max` tokens to address `b`.
- Because `_mint` uses unchecked addition in Yul, `totalSupply = 1 + type(uint256).max` wraps modulo 2^256 to `0`.
- The recipient `b`'s balance becomes `type(uint256).max` (also via unchecked addition), confirming balance and supply corruption.
- The assertions verify the wrapped `totalSupply` and the maximized recipient balance, matching the expected overflow behavior of the current implementation.

## Recommended Mitigation

```diff
 function _mint(address account, uint256 value) internal {
     assembly ("memory-safe") {
         if iszero(account) { /* revert InvalidReceiver */ }
         let ptr := mload(0x40)
         let balanceSlot := _balances.slot
         let supplySlot := _totalSupply.slot
         let supply := sload(supplySlot)
-        sstore(supplySlot, add(supply, value))
+        let newSupply := add(supply, value)
+        if lt(newSupply, supply) { revert(0, 0) }
+        sstore(supplySlot, newSupply)
         mstore(ptr, account)
         mstore(add(ptr, 0x20), balanceSlot)
         let accountBalanceSlot := keccak256(ptr, 0x40)
         let accountBalance := sload(accountBalanceSlot)
-        sstore(accountBalanceSlot, add(accountBalance, value))
+        let newBal := add(accountBalance, value)
+        if lt(newBal, accountBalance) { revert(0, 0) }
+        sstore(accountBalanceSlot, newBal)
+        mstore(ptr, value)
+        log3(ptr, 0x20, 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef, 0, account)
     }
 }
```
