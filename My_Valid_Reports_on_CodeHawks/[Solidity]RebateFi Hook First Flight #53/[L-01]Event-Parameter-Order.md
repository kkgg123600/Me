# Vulnerability Report #4: Incorrect Event Parameter Order in TokensWithdrawn

## Root + Impact

The `TokensWithdrawn` event is emitted with parameters in the wrong order, swapping the `token` and `to` addresses. This causes off-chain monitoring systems and indexers to incorrectly record withdrawal transactions.

## Description

The `TokensWithdrawn` event is defined with parameters in the order `(token, to, amount)`:

```solidity
// Line 44
event TokensWithdrawn(address indexed token, address indexed to, uint256 amount);
```

However, when the event is emitted in the `withdrawTokens` function, the parameters are provided in the wrong order `(to, token, amount)`:

```solidity
// Lines 73-76
function withdrawTokens(address token, address to, uint256 amount) external onlyOwner {
    IERC20(token).transfer(to, amount);
    emit TokensWithdrawn(to, token , amount);  // @> BUG: Parameters swapped (to, token)
}
```

This causes:

- The `token` field to contain the recipient address
- The `to` field to contain the token address
- Off-chain systems monitoring `TokensWithdrawn` events will have incorrect data

## Risk

**Likelihood**: HIGH

- This affects EVERY token withdrawal
- The bug executes every time `withdrawTokens()` is called
- There's no runtime check that would catch this

**Impact**: LOW to MEDIUM

While this doesn't cause direct loss of funds, it has several negative consequences:

- **Broken off-chain monitoring**: Any system tracking withdrawals will have inverted addresses
- **Incorrect audit trails**: Event logs won't accurately reflect what was withdrawn and where
- **UX/Dashboard issues**: Frontend applications showing withdrawal history will display wrong information
- **Investigation difficulties**: When debugging issues, event logs will be misleading
- **Indexer/Subgraph errors**: TheGraph or similar indexers will store incorrect data

## Proof of Concept

The goal of this PoC is to demonstrate that `TokensWithdrawn` is emitted with parameters in the wrong order. We:

- Mint tokens to the hook and call `withdrawTokens(token, to, amount)`.
- Expect and verify the emitted event uses `(to, token, amount)` instead of `(token, to, amount)`.
This proves off-chain indexers will record incorrect fields for token and recipient.

```solidity
// File: test/audit/Vulnerability04_WrongEventParams_PoC.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import {ReFiSwapRebateHook} from "../../src/RebateFiHook.sol";

import {Deployers} from "@uniswap/v4-core/test/utils/Deployers.sol";
import {MockERC20} from "solmate/src/test/utils/mocks/MockERC20.sol";

import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";

import {Hooks} from "v4-core/libraries/Hooks.sol";
import {HookMiner} from "v4-periphery/src/utils/HookMiner.sol";

contract Vulnerability04_WrongEventParams_PoC is Test, Deployers {
    
    MockERC20 reFiToken;
    MockERC20 otherToken;
    ReFiSwapRebateHook public rebateHook;

    // Event with CORRECT parameter order for testing
    event TokensWithdrawn(address indexed token, address indexed to, uint256 amount);

    function setUp() public {
        // Deploy the Uniswap V4 PoolManager
        deployFreshManagerAndRouters();

        // Deploy tokens
        reFiToken = new MockERC20("ReFi Token", "ReFi", 18);
        otherToken = new MockERC20("Other Token", "OTHER", 18);

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
     * @notice This test demonstrates the event parameter order is wrong
     */
    function test_EventParametersAreSwapped() public {
        // Setup: send tokens to hook
        uint256 withdrawAmount = 100 ether;
        otherToken.mint(address(rebateHook), withdrawAmount);
        
        address tokenAddress = address(otherToken);
        address recipientAddress = address(0xBEEF);
        
        console.log("\n=== Withdrawal Details ===");
        console.log("Token being withdrawn:", tokenAddress);
        console.log("Recipient address:", recipientAddress);
        console.log("Amount:", withdrawAmount);
        
        // Expect event with SWAPPED parameters
        // Event definition: TokensWithdrawn(address indexed token, address indexed to, uint256 amount)
        // Actual emission: TokensWithdrawn(to, token, amount)
        
        // This expectation matches what ACTUALLY gets emitted (wrong order)
        vm.expectEmit(true, true, false, true);
        emit TokensWithdrawn(recipientAddress, tokenAddress, withdrawAmount);
        
        // Perform withdrawal
        rebateHook.withdrawTokens(tokenAddress, recipientAddress, withdrawAmount);
        
        console.log("\n=== Event Analysis ===");
        console.log("Event 'token' field contains:", recipientAddress);
        console.log("Event 'to' field contains:", tokenAddress);
        console.log("\nIMPACT: Event parameters are inverted!");
        console.log("Off-chain systems will record wrong data!");
    }

    /**
     * @notice Show what the CORRECT event would look like
     */
    function test_ShowCorrectEventExpectation() public {
        uint256 withdrawAmount = 50 ether;
        otherToken.mint(address(rebateHook), withdrawAmount);
        
        address tokenAddress = address(otherToken);
        address recipientAddress = address(0xDEAD);
        
        // This is what we SHOULD expect if the code was correct:
        // vm.expectEmit(true, true, false, true);
        // emit TokensWithdrawn(tokenAddress, recipientAddress, withdrawAmount);
        
        // But we have to expect the WRONG order to match actual emission:
        vm.expectEmit(true, true, false, true);
        emit TokensWithdrawn(recipientAddress, tokenAddress, withdrawAmount);
        
        rebateHook.withdrawTokens(tokenAddress, recipientAddress, withdrawAmount);
        
        console.log("\n=== Correct vs Actual ===");
        console.log("SHOULD emit: TokensWithdrawn(token=%s, to=%s, amount)", tokenAddress, recipientAddress);
        console.log("ACTUALLY emits: TokensWithdrawn(token=%s, to=%s, amount)", recipientAddress, tokenAddress);
    }
}
```

**Test Results:**

```bash
forge test --via-ir -vv --match-path test/audit/Vulnerability04_WrongEventParams_PoC.t.sol

Ran 2 tests for test/audit/Vulnerability04_WrongEventParams_PoC.t.sol:Vulnerability04_WrongEventParams_PoC
[PASS] test_EventParametersAreSwapped() (gas: 98290)
Logs:
  
=== Withdrawal Details ===
  Token being withdrawn: 0x212224D2F2d262cd093eE13240ca4873fcCBbA3C
  Recipient address: 0x000000000000000000000000000000000000bEEF
  Amount: 100000000000000000000
  
=== Event Analysis ===
  Event 'token' field contains: 0x000000000000000000000000000000000000bEEF
  Event 'to' field contains: 0x212224D2F2d262cd093eE13240ca4873fcCBbA3C
  
IMPACT: Event parameters are inverted!
  Off-chain systems will record wrong data!

[PASS] test_ShowCorrectEventExpectation() (gas: 90555)
Logs:
  
=== Correct vs Actual ===
  SHOULD emit: TokensWithdrawn(token=0x212224D2F2d262cd093eE13240ca4873fcCBbA3C, to=0x000000000000000000000000000000000000dEaD, amount)
  ACTUALLY emits: TokensWithdrawn(token=0x000000000000000000000000000000000000dEaD, to=0x212224D2F2d262cd093eE13240ca4873fcCBbA3C, amount)

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 1.82s (9.43ms CPU time)
```

The test demonstrates that the event emission has swapped parameters.

## Recommended Mitigation

```diff
function withdrawTokens(address token, address to, uint256 amount) external onlyOwner {
    IERC20(token).transfer(to, amount);
-   emit TokensWithdrawn(to, token , amount);
+   emit TokensWithdrawn(token, to, amount);
}
```

This ensures the emitted event matches its definition, with parameters in the correct order: `(token, to, amount)`.
