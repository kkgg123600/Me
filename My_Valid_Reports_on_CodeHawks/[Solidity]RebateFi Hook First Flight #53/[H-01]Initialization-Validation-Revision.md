# Vulnerability Report #1 (Revision): Broken Pool Initialization Validation Only Checks `currency1`

## Root + Impact

The `_beforeInitialize` validation in `RebateFiHook` checks `currency1` twice and completely ignores `currency0`. This effectively reduces the intended check
“ReFi must be present in either `currency0` OR `currency1`” to “ReFi must be `currency1`”. As a result:

- Valid pools where ReFi is `currency0` are incorrectly rejected (DoS on initialization).
- Pools where ReFi is `currency1` pass (as intended).
- Pools with neither token being ReFi still revert (which is correct but incidental).

This breaks the protocol invariant that ReFi presence should be accepted regardless of side and makes initialization behavior asymmetric and misleading.

## Description

Intended behavior: prevent pool initialization unless ReFi is present as either `currency0` or `currency1`.

Actual code in `src/RebateFiHook.sol:122-128`:

```solidity
function _beforeInitialize(address, PoolKey calldata key, uint160) internal view override returns (bytes4) {
    if (Currency.unwrap(key.currency1) != ReFi && 
        Currency.unwrap(key.currency1) != ReFi) {  // @> BUG: checks currency1 twice, ignores currency0
        revert ReFiNotInPool();
    }
    
    return BaseHook.beforeInitialize.selector;
}
```

Implications (truth table):

- `currency0 == ReFi`, `currency1 != ReFi` → condition true → reverts → WRONG (should allow).
- `currency0 != ReFi`, `currency1 == ReFi` → condition false → allows → RIGHT.
- `currency0 != ReFi`, `currency1 != ReFi` → condition true → reverts → RIGHT (but not due to proper dual-side check).

Some earlier descriptions claimed “reversion will NEVER occur” due to the duplicated check. That is incorrect. The duplicated check simplifies to `if (currency1 != ReFi)`, which still reverts whenever `currency1` is not ReFi—including the valid case where ReFi sits in `currency0`.

## Risk

**Likelihood:**

- High: Triggered on every pool initialization. Any setup where ReFi resolves to `currency0` will fail systematically.

**Impact:**

- Medium to High: Causes denial-of-service for valid pool initializations (ReFi as `currency0`), breaks the invariant that ReFi presence on either side must be accepted, confuses integrators and deployment tooling. It does not broadly “bypass validation to allow pools without ReFi”, but it does make validation one-sided and wrong.

## Proof of Concept

The goal of this PoC is to demonstrate that the initialization validation only checks `currency1` and ignores `currency0`. We show three realistic cases:

- Case 1: ReFi is `currency0` → initialization wrongly reverts (should allow).
- Case 2: Neither side is ReFi → initialization reverts (as intended, but coincidental).
- Case 3: ReFi is `currency1` → initialization allows (as intended).
This proves the check is one-sided and breaks the invariant “ReFi can be on either side”.

```solidity
// File: test/audit/Vulnerability01_InitializationValidation_PoC.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "forge-std/Test.sol";
import {ReFiSwapRebateHook} from "../../src/RebateFiHook.sol";

import {Deployers} from "@uniswap/v4-core/test/utils/Deployers.sol";
import {MockERC20} from "solmate/src/test/utils/mocks/MockERC20.sol";

import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {Currency} from "v4-core/types/Currency.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";

import {Hooks} from "v4-core/libraries/Hooks.sol";
import {HookMiner} from "v4-periphery/src/utils/HookMiner.sol";
import {LPFeeLibrary} from "v4-core/libraries/LPFeeLibrary.sol";
import {TickMath} from "v4-core/libraries/TickMath.sol";

contract Vulnerability01_InitializationValidation_PoC is Test, Deployers {
    MockERC20 tokenA;
    MockERC20 tokenB;
    MockERC20 reFiToken;
    ReFiSwapRebateHook rebateHook;

    Currency currencyA;
    Currency currencyB;
    Currency reFiCurrency;
    Currency ethCurrency;

    function setUp() public {
        deployFreshManagerAndRouters();

        // Deploy ReFi first to make its address lower than subsequently deployed tokens
        reFiToken = new MockERC20("ReFi Token", "ReFi", 18);
        tokenA = new MockERC20("Token A", "TKNA", 18);
        tokenB = new MockERC20("Token B", "TKNB", 18);
        currencyA = Currency.wrap(address(tokenA));
        currencyB = Currency.wrap(address(tokenB));
        reFiCurrency = Currency.wrap(address(reFiToken));
        ethCurrency = Currency.wrap(address(0));

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

    // Case 1: ReFi is currency0, currency1 is NOT ReFi → should ALLOW, but current code REVERTS
    function test_ReFiAsCurrency0_WronglyReverts() public {
        // Ensure ordering: pick a token deployed AFTER ReFi so its address is greater
        Currency nonReFiHigher = Currency.wrap(address(tokenA) > address(reFiToken) ? address(tokenA) : address(tokenB));

        PoolKey memory keyReFi0 = PoolKey({
            currency0: reFiCurrency,      // ReFi as currency0 (address lower)
            currency1: nonReFiHigher,     // Non-ReFi as currency1 (address higher)
            fee: LPFeeLibrary.DYNAMIC_FEE_FLAG,
            tickSpacing: 60,
            hooks: rebateHook
        });

        vm.expectRevert();
        manager.initialize(keyReFi0, TickMath.getSqrtPriceAtTick(0));
    }

    // Case 2: Neither currency is ReFi → should REVERT (and it does)
    function test_NoReFi_Reverts() public {
        // Sort tokens to satisfy currency ordering check
        (Currency low, Currency high) = address(tokenA) < address(tokenB)
            ? (currencyA, currencyB)
            : (currencyB, currencyA);

        PoolKey memory keyNoReFi = PoolKey({
            currency0: low,
            currency1: high,
            fee: LPFeeLibrary.DYNAMIC_FEE_FLAG,
            tickSpacing: 60,
            hooks: rebateHook
        });

        vm.expectRevert();
        manager.initialize(keyNoReFi, TickMath.getSqrtPriceAtTick(0));
    }

    // Case 3: ReFi is currency1 → should ALLOW (and it does)
    function test_ReFiAsCurrency1_Allows() public {
        // Use ETH as currency0 to guarantee ordering (address(0) < contract addresses)
        PoolKey memory keyReFi1 = PoolKey({
            currency0: ethCurrency,
            currency1: reFiCurrency,
            fee: LPFeeLibrary.DYNAMIC_FEE_FLAG,
            tickSpacing: 60,
            hooks: rebateHook
        });

        // No revert expected here
        manager.initialize(keyReFi1, TickMath.getSqrtPriceAtTick(0));
    }
}
```

## test results

```bash
 forge test --via-ir -vv --match-path test/audit/Vulnerability01_InitializationBypass_PoC.t.sol

Ran 3 tests for test/audit/Vulnerability01_InitializationBypass_PoC.t.sol:Vulnerability01_InitializationValidation_PoC
[PASS] test_NoReFi_Reverts() (gas: 51354)
[PASS] test_ReFiAsCurrency0_WronglyReverts() (gas: 49501)
[PASS] test_ReFiAsCurrency1_Allows() (gas: 87751)
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 1.78s (720.71µs CPU time)
```

Expected behavior:

- Case 1 should pass but reverts due to checking only `currency1`.
- Case 2 correctly reverts.
- Case 3 correctly passes.

## Recommended Mitigation

```diff
function _beforeInitialize(address, PoolKey calldata key, uint160) internal view override returns (bytes4) {
-   if (Currency.unwrap(key.currency1) != ReFi && 
-       Currency.unwrap(key.currency1) != ReFi) {
   if (Currency.unwrap(key.currency0) != ReFi && 
       Currency.unwrap(key.currency1) != ReFi) {
        revert ReFiNotInPool();
    }
    
    return BaseHook.beforeInitialize.selector;
}
```

This restores the intended invariant: initialization is only permitted if ReFi is present as either `currency0` or `currency1`.
