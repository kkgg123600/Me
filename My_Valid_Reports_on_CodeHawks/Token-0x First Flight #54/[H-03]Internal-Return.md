# Internal Functions Use `RETURN` Opcode, Breaking Control Flow

## Root + Impact

## Description

The internal functions `_balanceOf` and `totalSupply_` use the assembly `return(offset, size)` opcode. In the EVM, the `RETURN` opcode halts the execution of the entire transaction (or the current call context) and returns data to the caller. It does *not* return control to the calling Solidity function.

This means that any function calling `_balanceOf` or `totalSupply_` will immediately terminate, and any code after the call will never be executed.

```solidity
// src/helpers/ERC20Internals.sol
    function _balanceOf(address owner) internal view returns (uint256) {
        assembly {
            // ...
            mstore(ptr, amount)
            mstore(add(ptr, 0x20), 0)
// @> RETURN opcode terminates execution of the entire call
            return(ptr, 0x20)
        }
    }
```

**Real-world impact example:**

```solidity
// A developer extends this base implementation (as README suggests)
contract MyToken is ERC20 {
    function transferIfRich(address to, uint256 amount) public {
        uint256 bal = _balanceOf(msg.sender);
        // ❌ CRITICAL: The following lines will NEVER execute!
        require(bal > 1000, "Insufficient balance");  // Bypassed
        _transfer(msg.sender, to, amount);            // Never called
    }
}
```

In this example, `_balanceOf` returns the balance value but also **terminates the entire function**, causing the `require` check and `_transfer` call to be silently skipped. The Solidity compiler will warn about "unreachable code," but developers may overlook this, leading to critical security vulnerabilities.

## Risk

**Likelihood**:

High. The project's README explicitly states: *"User can use it as ERC20 token like openzeppelin implementation"* (README.md:12), positioning this as a base implementation for developers to extend. Any developer using `_balanceOf` or `totalSupply_` within their own logic (e.g., checking balance before an action, conditional minting, etc.) will encounter this broken control flow.

**Impact**:

Critical. It breaks the expected semantics of internal function calls. Logic placed after these calls will be silently skipped, potentially bypassing critical security checks, state updates, or logic flows. This is especially severe since the project markets itself as a reusable base like OpenZeppelin, meaning derived contracts will unknowingly inherit this dangerous behavior.

## Proof of Concept

```solidity
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

    function burn(address from, uint256 amount) public {
        _burn(from, amount);
    }

    function checkBalanceControlFlow(address user) public view returns (uint256) {
        uint256 bal = _balanceOf(user);
        // If _balanceOf uses RETURN, execution stops here and returns bal.
        // If correct, it continues.
        return bal + 1; // We return balance + 1 to distinguish.
    }
}

contract AuditTest is Test {
    ERC20Harness token;
    address user1 = address(0x1);
    address user2 = address(0x2);

    function setUp() public {
        token = new ERC20Harness();
    }

    function testBalanceOfControlFlow() public {
        token.mint(user1, 100);
        
        // Call wrapper function
        // If _balanceOf returns normally, we get 101.
        // If _balanceOf uses RETURN opcode, we get 100.
        uint256 result = token.checkBalanceControlFlow(user1);
        console.log("Result of checkBalanceControlFlow:", result);
        
        // Expectation: It returns 100 because _balanceOf terminates execution
        assertEq(result, 100);
    }
}
```

## Test result

```bash
➜  2025-12-token-0x git:(main) ✗ forge test --match-path test/Audit.t.sol -vv  

Ran 1 test for test/Audit.t.sol:AuditTest
[PASS] testBalanceOfControlFlow() (gas: 58095)
Logs:
  Result of checkBalanceControlFlow: 100

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 519.88µs (118.56µs CPU time)

Ran 1 test suite in 13.68ms (519.88µs CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

**Analysis:**

- **Expected result (if working correctly):** `101` (bal + 1)
- **Actual result:** `100` (just the balance)

This confirms that `_balanceOf` terminates the calling function immediately, preventing any subsequent code from executing. The compiler warning indicates that it recognizes the code path as unreachable, which developers might overlook when extending this base implementation.

## Recommended Mitigation

Do not use the `return` opcode in assembly for internal functions. Instead, assign the value to the return variable or leave it on the stack (though in inline assembly within Solidity, assigning to the variable is the standard way).

```diff
    function _balanceOf(address owner) internal view returns (uint256 amount) { // Name the return variable
        assembly {
            // ...
            let dataSlot := keccak256(ptr, 0x40)
-           let amount := sload(dataSlot)
+           amount := sload(dataSlot) // Assign to return variable
-           mstore(ptr, amount)
-           mstore(add(ptr, 0x20), 0)
-           return(ptr, 0x20)
        }
    }
```
