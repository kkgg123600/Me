
# Vulnerability Report #2

## Flawed Auction Extension Logic Grants Unfair Time Advantage

## Description

When a new bid is placed within the final 15 minutes of an auction, the `placeBid` function is intended to extend the auction to ensure there are always 15 minutes remaining. This mechanism is crucial for preventing last-second "sniping".

The issue is that the current implementation adds the `S_AUCTION_EXTENSION_DURATION` (15 minutes) to the existing `listing.auctionEnd` time instead of setting it to `block.timestamp + S_AUCTION_EXTENSION_DURATION`. This means the earlier a bid is placed within the extension window, the longer the total extension will be. For example, a bid placed 14 minutes before the end will result in a total extension of approximately 29 minutes (14 minutes remaining + 15 minutes added), instead of the intended 15 minutes. This gives an unfair advantage to bidders who act earlier in the final moments of the auction.

```solidity
// src/BidBeastsNFTMarketPlace.sol

            uint256 timeLeft = 0;
            if (listing.auctionEnd > block.timestamp) {
                timeLeft = listing.auctionEnd - block.timestamp;
            }
            if (timeLeft < S_AUCTION_EXTENSION_DURATION) {
@>              listing.auctionEnd = listing.auctionEnd + S_AUCTION_EXTENSION_DURATION;
                emit AuctionExtended(tokenId, listing.auctionEnd);
            }
```

## Risk Assessment

**Likelihood**: High

* This vulnerability is triggered every time a bid is placed within the final `S_AUCTION_EXTENSION_DURATION` (15 minutes) of an auction.

**Impact**: Medium

* The flawed logic undermines the fairness of the auction mechanism. Sophisticated bidders can strategically extend auctions for longer than intended, giving them a significant advantage over other participants and disrupting the expected auction dynamics. This compromises the integrity of the marketplace's core functionality.

## Proof of Concept (PoC)

The following test demonstrates that when a bid is placed 5 minutes before the auction's end, the auction is extended by a total of 20 minutes from the time of the bid, instead of the intended 15 minutes.

```solidity
// test/BidBeastsMarketPlaceTest.t.sol

    function test_auctionExtension_vulnerability_analysis() public {
        // Setup: Mint and list NFT
        _mintNFT();
        _listNFT();

        // Place first bid to start the auction
        uint256 firstBidAmount = MIN_PRICE + 0.01 ether;
        vm.prank(BIDDER_1);
        market.placeBid{value: firstBidAmount}(TOKEN_ID);

        uint256 initialAuctionEnd = market.getListing(TOKEN_ID).auctionEnd;
        console.log("Initial auction end time:", initialAuctionEnd);
        console.log("Current block timestamp:", block.timestamp);
        console.log("Extension duration:", market.S_AUCTION_EXTENSION_DURATION());

        // Verify initial auction end is set correctly
        assertEq(initialAuctionEnd, block.timestamp + market.S_AUCTION_EXTENSION_DURATION(), 
                "Initial auction end should be current time + extension duration");

        // Warp time to 5 minutes before auction ends (within extension window)
        uint256 timeBeforeEnd = 5 minutes;
        uint256 newTimestamp = initialAuctionEnd - timeBeforeEnd;
        vm.warp(newTimestamp);

        console.log("Warped to timestamp:", block.timestamp);
        console.log("Time left before extension:", initialAuctionEnd - block.timestamp);

        // Place second bid within extension window
        uint256 secondBidAmount = firstBidAmount * 105 / 100; // 5% increase
        vm.prank(BIDDER_2);
        market.placeBid{value: secondBidAmount}(TOKEN_ID);

        uint256 newAuctionEnd = market.getListing(TOKEN_ID).auctionEnd;
        console.log("New auction end after extension:", newAuctionEnd);

        // Calculate what the correct end time should be (current time + extension)
        uint256 expectedCorrectEnd = block.timestamp + market.S_AUCTION_EXTENSION_DURATION();
        console.log("Expected correct end time:", expectedCorrectEnd);

        // Calculate what the flawed implementation produces (old end + extension)
        uint256 actualFlawedEnd = initialAuctionEnd + market.S_AUCTION_EXTENSION_DURATION();
        console.log("Actual flawed end time:", actualFlawedEnd);

        // Verify the vulnerability exists
        assertEq(newAuctionEnd, actualFlawedEnd, "Auction extension uses flawed logic");
        
        // Calculate the extra time granted due to the flaw
        uint256 extraTime = newAuctionEnd - expectedCorrectEnd;
        console.log("Extra time granted due to flaw (seconds):", extraTime);
        console.log("Extra time granted due to flaw (minutes):", extraTime / 60);

        // The flaw should grant exactly the time that was left before the extension
        assertEq(extraTime, timeBeforeEnd, "Extra time should equal the time left before extension");

        // Verify that the auction is extended longer than intended
        assertTrue(newAuctionEnd > expectedCorrectEnd, "Auction extended longer than intended");
    }
```

**To run the test:**

1. Add the above test to a test file (e.g., `test/BidBeastsMarketPlaceTest.t.sol`).
2. Run the following command: `forge test --match-test test_auctionExtension_vulnerability_analysis -vv`

**Expected Result:**

```bash
➜ forge test --match-test test_auctionExtension_vulnerability_analysis -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/BidBeastsMarketPlaceTest.t.sol:BidBeastsNFTMarketTest
[PASS] test_auctionExtension_vulnerability_analysis() (gas: 343619)
Logs:
  Initial auction end time: 901
  Current block timestamp: 1
  Extension duration: 900
  Warped to timestamp: 601
  Time left before extension: 300
  New auction end after extension: 1801
  Expected correct end time: 1501
  Actual flawed end time: 1801
  Extra time granted due to flaw (seconds): 300
  Extra time granted due to flaw (minutes): 5

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.47ms (1.92ms CPU time)

Ran 1 test suite in 19.85ms (5.47ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The logs clearly show that 300 seconds (5 minutes) of "extra time" were unfairly granted, confirming the vulnerability.

## Recommended Fix

The new `auctionEnd` time should be set to the current `block.timestamp` plus the extension duration, rather than being added to the old end time.

```diff
// src/BidBeastsNFTMarketPlace.sol

            if (timeLeft < S_AUCTION_EXTENSION_DURATION) {
-               listing.auctionEnd = listing.auctionEnd + S_AUCTION_EXTENSION_DURATION;
+               listing.auctionEnd = block.timestamp + S_AUCTION_EXTENSION_DURATION;
                emit AuctionExtended(tokenId, listing.auctionEnd);
            }
```
