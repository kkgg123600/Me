# Vulnerability Report #3: Unchecked ERC20 Transfer in withdrawTokens

## Root + Impact

The `withdrawTokens` function uses `transfer()` without checking its return value. Some ERC20 tokens return `false` on failure instead of reverting, which means failed withdrawals will silently succeed, potentially locking funds permanently in the contract.

## Description

The `withdrawTokens` function allows the owner to withdraw any ERC20 tokens from the hook contract. However, it uses the unsafe `transfer()` method without checking the return value.

According to the ERC20 standard, `transfer()` should return a boolean indicating success or failure. While many tokens revert on failure, some tokens (especially older or non-standard implementations) return `false` instead of reverting. Examples include:

- USDT on mainnet (returns nothing)
- ZRX (returns false on failure)
- BNB (returns false on failure)

When such tokens are used, a failed transfer will not revert the transaction, causing the function to emit the `TokensWithdrawn` event even though no tokens were actually transferred.

```solidity
// RebateFiHook.sol, lines 73-76
function withdrawTokens(address token, address to, uint256 amount) external onlyOwner {
    IERC20(token).transfer(to, amount);  // @> BUG: Return value not checked
    emit TokensWithdrawn(to, token , amount);
}
```

## Risk

**Likelihood**: MEDIUM

- This vulnerability only affects tokens that return false instead of reverting (not all tokens)
- The protocol is designed to work with "standard ERC20 tokens" per README
- However, USDT is a very common token that could be accumulated via sell fees
- Owner might try to withdraw tokens that returned from an incomplete transfer

**Impact**: MEDIUM

- Failed tokens withdrawals will appear successful (event emitted)
- Tokens remain stuck in the contract
- Owner may think funds were recovered when they weren't
- Affects protocol revenue collection
- Violates INV-15: "After withdrawal, hook balance should decrease by exact withdrawal amount"

## Proof of Concept

The goal of this PoC is to demonstrate that using `IERC20.transfer` without checking its boolean return can cause silent failures. We simulate a token that returns `false` and show:

- The withdrawal call does not revert, and the event is emitted.
- Balances remain unchanged because the transfer failed.
This proves the need to either use `SafeERC20.safeTransfer` or manually require a successful return value.

```solidity
// File: test/audit/Vulnerability03_UncheckedTransfer_PoC.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import {ReFiSwapRebateHook} from "../../src/RebateFiHook.sol";

import {Deployers} from "@uniswap/v4-core/test/utils/Deployers.sol";
import {MockERC20} from "solmate/src/test/utils/mocks/MockERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";

import {Hooks} from "v4-core/libraries/Hooks.sol";
import {HookMiner} from "v4-periphery/src/utils/HookMiner.sol";

// Mock token that returns false instead of reverting
contract BadERC20 {
    mapping(address => uint256) public balanceOf;
    
    function transfer(address, uint256) external pure returns (bool) {
        return false;  // Always returns false without reverting
    }
    
    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
    }
}

contract Vulnerability03_UncheckedTransfer_PoC is Test, Deployers {
    
    MockERC20 reFiToken;
    BadERC20 badToken;
    ReFiSwapRebateHook public rebateHook;

    function setUp() public {
        // Deploy the Uniswap V4 PoolManager
        deployFreshManagerAndRouters();

        // Deploy tokens
        reFiToken = new MockERC20("ReFi Token", "ReFi", 18);
        badToken = new BadERC20();

        // Deploy hook
        bytes memory creationCode = type(ReFiSwapRebateHook).creationCode;
        bytes memory constructorArgs = abi.encode(manager, address(reFiToken));

        uint160 flags = uint160(
            Hooks.BEFORE_INITIALIZE_FLAG | 
            Hooks.AFTER_INITIALIZE_FLAG | 
            Hooks.BEFORE_SWAP_FLAG
        );

        (address hookAddress, bytes32 salt) = HookMiner.find(
            address(this),
            flags,
            creationCode,
            constructorArgs
        );

        rebateHook = new ReFiSwapRebateHook{salt: salt}(manager, address(reFiToken));
        require(address(rebateHook) == hookAddress, "Hook address mismatch");
    }

    /**
     * @notice This test demonstrates that failed transfers don't revert
     * @dev A token that returns false will cause silent failure
     */
    function test_UncheckedTransfer_SilentFailure() public {
        // Mint bad tokens to the hook
        uint256 amount = 100 ether;
        badToken.mint(address(rebateHook), amount);
        
        uint256 hookBalanceBefore = badToken.balanceOf(address(rebateHook));
        uint256 ownerBalanceBefore = badToken.balanceOf(address(this));
        
        console.log("\n=== Before Withdrawal ===");
        console.log("Hook balance:", hookBalanceBefore);
        console.log("Owner balance:", ownerBalanceBefore);
        
        // Try to withdraw - this will NOT revert even though transfer returns false!
        rebateHook.withdrawTokens(address(badToken), address(this), amount);
        
        uint256 hookBalanceAfter = badToken.balanceOf(address(rebateHook));
        uint256 ownerBalanceAfter = badToken.balanceOf(address(this));
        
        console.log("\n=== After Withdrawal ===");
        console.log("Hook balance:", hookBalanceAfter);
        console.log("Owner balance:", ownerBalanceAfter);
        console.log("\nIMPACT: Withdrawal appeared to succeed but tokens were not transferred!");
        
        // Withdrawal "succeeded" (no revert) but balances didn't change
        assertEq(hookBalanceAfter, hookBalanceBefore, "Hook balance unchanged");
        assertEq(ownerBalanceAfter, ownerBalanceBefore, "Owner balance unchanged");
        
        // Event was emitted even though transfer failed
        // This is extremely misleading for the owner
    }

    /**
     * @notice Compare with safe transfer patterns
     */
    function test_SafeTransferWouldCatch() public {
        uint256 amount = 100 ether;
        badToken.mint(address(rebateHook), amount);
        
        // Simulate what a safe transfer check would do
        bool success = IERC20(address(badToken)).transfer(address(this), amount);
        
        console.log("\n=== Safe Pattern Demonstration ===");
        console.log("Transfer return value:", success);
        console.log("With proper check, this would revert");
        
        assertFalse(success, "Transfer returns false");
        
        // A safe implementation would revert here:
        // require(success, "Transfer failed");
    }
}
```

**Test Results:**

```bash
forge test --via-ir -vv --match-path test/audit/Vulnerability03_UncheckedTransfer_PoC.t.sol

Ran 2 tests for test/audit/Vulnerability03_UncheckedTransfer_PoC.t.sol:Vulnerability03_UncheckedTransfer_PoC
[PASS] test_SafeTransferWouldCatch() (gas: 42451)
Logs:
  
=== Safe Pattern Demonstration ===
  Transfer return value: false
  With proper check, this would revert

[PASS] test_UncheckedTransfer_SilentFailure() (gas: 71072)
Logs:
  
=== Before Withdrawal ===
  Hook balance: 100000000000000000000
  Owner balance: 0
  
=== After Withdrawal ===
  Hook balance: 100000000000000000000
  Owner balance: 0
  
IMPACT: Withdrawal appeared to succeed but tokens were not transferred!

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 1.66s (601.20µs CPU time)
```

Expected output showing the silent failure of token withdrawal.

## Recommended Mitigation

Use OpenZeppelin's `SafeERC20` library:

```diff
+ import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract ReFiSwapRebateHook is BaseHook, Ownable {
+   using SafeERC20 for IERC20;
    
    function withdrawTokens(address token, address to, uint256 amount) external onlyOwner {
-       IERC20(token).transfer(to, amount);
+       IERC20(token).safeTransfer(to, amount);
        emit TokensWithdrawn(to, token , amount);
    }
}
```

**Alternative manual fix:**

```diff
function withdrawTokens(address token, address to, uint256 amount) external onlyOwner {
-   IERC20(token).transfer(to, amount);
+   bool success = IERC20(token).transfer(to, amount);
+   require(success, "Transfer failed");
    emit TokensWithdrawn(to, token , amount);
}
```

Both solutions ensure that failed transfers cause the transaction to revert, preventing silent failures and protecting the withdrawal functionality.
