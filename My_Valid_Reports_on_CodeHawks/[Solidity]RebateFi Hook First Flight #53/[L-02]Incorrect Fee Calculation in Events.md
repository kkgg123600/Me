# Finding Title

Incorrect Fee Calculation in Events Reports 10x the Actual Fee

## Root + Impact

**Root Cause**: The event fee calculation uses a denominator of `100,000` (10^5) while Uniswap V4 fees are typically in pips (1/1,000,000 or 10^6).
**Impact**: The `ReFiSold` event reports a fee amount that is 10 times larger than the actual fee applied to the pool. This causes misleading off-chain data and accounting errors for indexers or users relying on events.

## Description

In `_beforeSwap`, the fee amount for the event is calculated as:

```solidity
            uint256 feeAmount = (swapAmount * sellFee) / 100000;
            emit ReFiSold(sender, swapAmount, feeAmount);
```

The `sellFee` is initialized to `3000`.

- In Uniswap V4, `3000` usually represents `0.3%` (3000 / 1,000,000).
- The event calculation uses `100,000` as denominator: `3000 / 100,000` = `0.03` = `3%`.

Thus, the event reports a 3% fee, while the pool applies a 0.3% fee.

## Risk

**Likelihood**: High (Always occurs for sells).
**Impact**: Low (Off-chain reporting issue, does not affect on-chain balances directly, but misleading).

## Proof of Concept

If `sellFee` is set to `100000` (10% in pips):

- Pool Fee: 10%.
- Event Fee: `(amount * 100000) / 100000` = `amount` (100%).
- The event reports 100% fee when it should be 10%.

## Recommended Mitigation

Use the correct denominator for pips (1,000,000).

```diff
-           uint256 feeAmount = (swapAmount * sellFee) / 100000;
+           uint256 feeAmount = (swapAmount * sellFee) / 1000000;
```
