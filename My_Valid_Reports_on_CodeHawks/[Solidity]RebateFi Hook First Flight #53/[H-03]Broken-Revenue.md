# Finding Title

Protocol Fails to Collect Revenue Despite Claims and Withdrawal Functionality

## Root + Impact

**Root Cause**: The hook uses `LPFeeLibrary.OVERRIDE_FEE_FLAG` which sends fees to Liquidity Providers (LPs), but does not implement any mechanism to take a cut for the protocol/hook itself.

**Impact**: The protocol generates zero revenue for itself, contrary to the README's claim of "generate protocol revenue". The `withdrawTokens` function is effectively useless for collecting swap fees as no tokens are ever sent to the hook during swaps.

## Description

The README states: "standard (0.3%) or premium fees for selling to discourage dumping and generate protocol revenue." and "can withdraw accumulated tokens from hook contract".

However, the implementation only overrides the LP fee:

```solidity
        return (
            BaseHook.beforeSwap.selector,
            BeforeSwapDeltaLibrary.ZERO_DELTA,
            fee | LPFeeLibrary.OVERRIDE_FEE_FLAG
        );
```

This fee goes entirely to the pool's LPs. The hook contract receives nothing. There is no code in `_beforeSwap` or `afterSwap` to transfer tokens to the hook.

## Risk

**Likelihood**: High (Always occurs).
**Impact**: Medium (Business logic failure - protocol earns no revenue, though LPs benefit).

## Proof of Concept

1. Perform swaps that trigger fees.
2. Check `reFiToken.balanceOf(address(hook))`. It remains 0.
3. Call `withdrawTokens`. It fails or withdraws 0 (unless tokens were manually sent).

```solidity
// File: test/audit/Vulnerability04_BrokenRevenue_PoC.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import {ReFiSwapRebateHook} from "../../src/RebateFiHook.sol";

import {Deployers} from "@uniswap/v4-core/test/utils/Deployers.sol";
import {PoolSwapTest} from "v4-core/test/PoolSwapTest.sol";
import {MockERC20} from "solmate/src/test/utils/mocks/MockERC20.sol";

import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {SwapParams} from "v4-core/types/PoolOperation.sol";
import {ModifyLiquidityParams} from "v4-core/types/PoolOperation.sol";

import {Currency} from "v4-core/types/Currency.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";
import {ERC1155TokenReceiver} from "solmate/src/tokens/ERC1155.sol";

import {Hooks} from "v4-core/libraries/Hooks.sol";
import {TickMath} from "v4-core/libraries/TickMath.sol";
import {LiquidityAmounts} from "@uniswap/v4-core/test/utils/LiquidityAmounts.sol";
import {HookMiner} from "v4-periphery/src/utils/HookMiner.sol";
import {LPFeeLibrary} from "v4-core/libraries/LPFeeLibrary.sol";

contract Vulnerability04_BrokenRevenue_PoC is Test, Deployers, ERC1155TokenReceiver {
    MockERC20 reFiToken;
    ReFiSwapRebateHook public rebateHook;

    Currency ethCurrency = Currency.wrap(address(0));
    Currency reFiCurrency;

    address user1 = address(0x1);

    function setUp() public {
        // Deploy the Uniswap V4 PoolManager
        deployFreshManagerAndRouters();

        // Deploy the ReFi token
        reFiToken = new MockERC20("ReFi Token", "ReFi", 18);
        reFiCurrency = Currency.wrap(address(reFiToken));

        // Mint ReFi to this test contract to satisfy potential liquidity transfers
        reFiToken.mint(address(this), 1_000 ether);
        reFiToken.approve(address(modifyLiquidityRouter), type(uint256).max);

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

        // Initialize pool with ReFi as currency1 (ETH as currency0)
        (PoolKey memory _key, ) = initPool(
            ethCurrency,           // currency0 = ETH
            reFiCurrency,          // currency1 = ReFi
            rebateHook,
            LPFeeLibrary.DYNAMIC_FEE_FLAG,
            SQRT_PRICE_1_1
        );
        key = _key;

        // Add ETH liquidity so swaps can execute
        uint160 sqrtPriceAtTickUpper = TickMath.getSqrtPriceAtTick(60);
        uint256 ethToAdd = 0.1 ether;
        uint128 liquidityDelta = LiquidityAmounts.getLiquidityForAmount0(
            SQRT_PRICE_1_1,
            sqrtPriceAtTickUpper,
            ethToAdd
        );

        modifyLiquidityRouter.modifyLiquidity{value: ethToAdd}(
            key,
            ModifyLiquidityParams({
                tickLower: -60,
                tickUpper: 60,
                liquidityDelta: int256(uint256(liquidityDelta)),
                salt: bytes32(0)
            }),
            ZERO_BYTES
        );
    }

    // Verifies the hook does not accrue tokens from swaps; revenue goes to LPs, not the hook
    function test_HookDoesNotAccrueFees_OnBuysAndSells() public {
        vm.deal(user1, 1 ether);
        vm.startPrank(user1);
        reFiToken.approve(address(swapRouter), type(uint256).max);

        PoolSwapTest.TestSettings memory testSettings = PoolSwapTest.TestSettings({
            takeClaims: false,
            settleUsingBurn: false
        });

        // BUY ReFi (ETH -> ReFi)
        SwapParams memory buyParams = SwapParams({
            zeroForOne: true,
            amountSpecified: -int256(0.005 ether),
            sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
        });
        swapRouter.swap{value: 0.005 ether}(key, buyParams, testSettings, ZERO_BYTES);

        // SELL ReFi (ReFi -> ETH)
        SwapParams memory sellParams = SwapParams({
            zeroForOne: false,
            amountSpecified: -int256(0.002 ether),
            sqrtPriceLimitX96: TickMath.MAX_SQRT_PRICE - 1
        });
        swapRouter.swap(key, sellParams, testSettings, ZERO_BYTES);

        vm.stopPrank();

        uint256 hookReFiBal = reFiToken.balanceOf(address(rebateHook));
        uint256 hookEthBal = address(rebateHook).balance;

        console.log("Hook ReFi balance:", hookReFiBal);
        console.log("Hook ETH balance:", hookEthBal);
        console.log("IMPACT: Hook does not accrue swap fees; revenue remains with LPs");

        assertEq(hookReFiBal, 0, "Hook should not accrue ReFi from swaps");
        assertEq(hookEthBal, 0, "Hook should not accrue ETH from swaps");
    }
}
```

## Test result

```bash
forge test -vvv --via-ir -s Vulnerability04_BrokenRevenue_PoC

Ran 1 test for test/audit/Vulnerability04_BrokenRevenue_PoC.t.sol:Vulnerability04_BrokenRevenue_PoC
[PASS] test_HookDoesNotAccrueFees_OnBuysAndSells() (gas: 501108)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.80s (2.19ms CPU time)
```

The test `test_HookDoesNotAccrueFees_OnBuysAndSells` verifies that the hook does not accrue tokens from swaps. The protocol revenue remains with the LPs, not the hook.

## Recommended Mitigation

If the intention is to collect protocol revenue, the hook must explicitly take a fee.
This can be done by:

1. Using `Hooks.afterSwap` to take a portion of the output token from the user (requires `takeClaims` or similar).
2. Or, if using `OVERRIDE_FEE_FLAG`, the protocol relies on LPs. If the protocol OWNS the liquidity, then it works. But if LPs are public, the protocol gets nothing.
3. To take a fee to the hook:
   - In `beforeSwap`, calculate a fee amount.
   - Take that amount from the user (this is complex in V4 hooks without `accessLock`).
   - Easier: Use `ProtocolFee` if supported by V4 core for hooks (Hook Fees).
   - Or, mint/burn if the token allows it.
   - Or, simply accept that fees go to LPs and update documentation.

   If the goal is explicit revenue to the hook:
   - Implement `afterSwap` to transfer a percentage of the output to the hook.

### Sample Fix (illustrative)

```diff
// File: src/RebateFiHook.sol
 function getHookPermissions() public pure override returns (Hooks.Permissions memory) {
     return Hooks.Permissions({
         beforeInitialize: true,
         afterInitialize: true,
         beforeAddLiquidity: false,
         afterAddLiquidity: false,
         beforeRemoveLiquidity: false,
         afterRemoveLiquidity: false,
         beforeSwap: true,
-        afterSwap: false,
+        afterSwap: true,
         beforeDonate: false,
         afterDonate: false,
         beforeSwapReturnDelta: false,
         afterSwapReturnDelta: false,
         afterAddLiquidityReturnDelta: false,
         afterRemoveLiquidityReturnDelta: false
     });
 }

@@
 // Take a small protocol cut into the hook after swaps
+function _afterSwap(
+    address,
+    PoolKey calldata key,
+    SwapParams calldata,
+    BalanceDelta callerDelta,
+    bytes calldata
+) internal override returns (bytes4, BalanceDelta) {
+    // Example: move 10% of positive deltas to the hook
+    int128 amount0 = callerDelta.amount0();
+    int128 amount1 = callerDelta.amount1();
+
+    uint128 feeAmount0 = amount0 > 0 ? uint128(uint256(int256(amount0)) / 10) : 0;
+    uint128 feeAmount1 = amount1 > 0 ? uint128(uint256(int256(amount1)) / 10) : 0;
+
+    if (feeAmount0 > 0) manager.take(key.currency0, address(this), feeAmount0);
+    if (feeAmount1 > 0) manager.take(key.currency1, address(this), feeAmount1);
+
+    return (BaseHook.afterSwap.selector, callerDelta);
+}
```
