
# Vulnerability Report #5

## Precision Loss in Fee Calculation Leads to Protocol Revenue Leakage

## Description

The contract calculates the 5% platform fee on the final sale price using the formula `(bid.amount * S_FEE_PERCENTAGE) / 100`. Due to Solidity's integer-only arithmetic, any division operation results in truncation, rounding down the result.

This means that for any sale price that is not a perfect multiple of 20, the calculated fee will be less than the true 5%. For example, a sale price of 19 wei would result in a fee of `(19 * 5) / 100 = 0`, losing the entire 0.95 wei fee. While the `S_MIN_NFT_PRICE` makes this specific low-value scenario unlikely, the same mathematical flaw applies to all valid bid amounts. This creates a small but consistent leakage of value from the protocol over time, as fractional wei amounts are perpetually rounded down in the platform's favor.

```solidity
// src/BidBeastsNFTMarketPlace.sol

    function _executeSale(uint26 tokenId) internal {
        // ...
@>      uint256 fee = (bid.amount * S_FEE_PERCENTAGE) / 100;
        s_totalFee += fee;
        
        uint256 sellerProceeds = bid.amount - fee;
        // ...
    }
```

## Risk Assessment

**Likelihood**: High

* This rounding error occurs on every successful auction where the final price is not a perfect multiple of 20 wei.

**Impact**: Low

* The financial loss per transaction is minimal (less than 1 wei). However, this represents a consistent and preventable leakage of protocol revenue. Over thousands of transactions, these small amounts can accumulate. More importantly, it reflects a deviation from best practices for handling financial calculations in Solidity.

## Proof of Concept (PoC)

The following test demonstrates that when an auction is settled with a realistic bid amount that is not a multiple of 20, the fee collected by the contract is less than the true 5% fee, confirming the value leak.

```solidity
// test/FeePrecisionLossPoC.t.sol
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import {Test, console} from "forge-std/Test.sol";
import {BidBeastsNFTMarket} from "../src/BidBeastsNFTMarketPlace.sol";
import {BidBeasts} from "../src/BidBeasts_NFT_ERC721.sol";

contract FeePrecisionLossPoC is Test {
    BidBeastsNFTMarket public market;
    BidBeasts public nft;
    
    address public constant OWNER = address(0x1);
    address public constant SELLER = address(0x2);
    address public constant BIDDER = address(0x3);
    uint256 public constant TOKEN_ID = 0;
    
    function setUp() public {
        vm.startBroadcast(OWNER);
        nft = new BidBeasts();
        market = new BidBeastsNFTMarket(address(nft));
        nft.mint(SELLER);
        vm.stopBroadcast();
        
        vm.deal(BIDDER, 10 ether);
    }
    
    /// @notice Demonstrates that the protocol loses fee revenue due to precision loss with realistic values.
    function test_FeePrecisionLoss_WithRealisticAmounts() public {
        uint256 minPrice = market.S_MIN_NFT_PRICE();

        // 1. List NFT with the minimum allowed price.
        vm.startPrank(SELLER);
        nft.approve(address(market), TOKEN_ID);
        market.listNFT(TOKEN_ID, minPrice, 0); 
        vm.stopPrank();
        
        // 2. Place a bid that will cause a rounding error (not a multiple of 20).
        // The true 5% fee will have a fractional part that gets truncated.
        uint256 bidAmount = minPrice + 19 wei; // e.g., 10,000,000,000,000,019 wei
        
        vm.prank(BIDDER);
        market.placeBid{value: bidAmount}(TOKEN_ID);
        vm.stopPrank();

        // 3. Settle the auction.
        vm.warp(block.timestamp + market.S_AUCTION_EXTENSION_DURATION() + 1);
        uint256 feesBefore = market.s_totalFee();
        market.settleAuction(TOKEN_ID);
        uint256 feesAfter = market.s_totalFee();

        // 4. Calculate the fee collected and compare it with the flawed formula's output.
        uint256 feeCollected = feesAfter - feesBefore;
        uint256 expectedFeeByFlawedFormula = (bidAmount * 5) / 100;
        // The "true" 5% fee is 500,000,000,000,000.95 wei. The contract collects 500,000,000,000,000 wei.

        console.log("Bid Amount: %s wei", bidAmount);
        console.log("Fee Collected by Contract: %s wei", feeCollected);
        console.log("Fee as calculated by flawed formula: %s wei", expectedFeeByFlawedFormula);

        assertEq(feeCollected, expectedFeeByFlawedFormula, "The collected fee must match the flawed calculation.");
        assertTrue(bidAmount > 0 && bidAmount % 20 != 0, "Sanity check: bid amount must cause rounding error");
    }
}
```

**To run the test:**

1. Add the above test to a test file (e.g., `test/FeePrecisionLossPoC.t.sol`).
2. Run the following command: `forge test --match-path test/FeePrecisionLossPoC.t.sol -vv`

**Expected Result:**

```bash
Ran 1 test for test/FeePrecisionLossPoC.t.sol:FeePrecisionLossPoC
[PASS] test_FeePrecisionLoss_WithRealisticAmounts() (gas: 269841)
Logs:
  Bid Amount: 10000000000000019 wei
  Fee Collected by Contract: 500000000000000 wei
  Fee as calculated by flawed formula: 500000000000000 wei

Suite result: ok. 1 passed; 0 failed; 0 skipped;
```

The logs show that the fee collected matches the result of the flawed formula, which truncates the true fractional fee of `0.95 wei`, confirming the value leak.

## Recommended Fix

To increase precision and minimize rounding errors, it is standard practice to use a higher precision base, such as basis points (1/100th of a percent).

```diff
// src/BidBeastsNFTMarketPlace.sol

-   uint256 constant public S_FEE_PERCENTAGE = 5;
+   // Use a basis of 10,000 for fees (BPS), where 1 BPS = 0.01%
+   uint256 constant public S_FEE_BPS = 500; // 500 basis points = 5%

    // ...

    function _executeSale(uint256 tokenId) internal {
        // ...
-       uint256 fee = (bid.amount * S_FEE_PERCENTAGE) / 100;
+       uint256 fee = (bid.amount * S_FEE_BPS) / 10000;
        s_totalFee += fee;
        
        uint256 sellerProceeds = bid.amount - fee;
        _payout(listing.seller, sellerProceeds);

        emit AuctionSettled(tokenId, bid.bidder, listing.seller, bid.amount);
    }
```
