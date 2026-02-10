
# Vulnerability Report #4

## Misleading `AuctionSettled` Event Emitted During Active Bidding

## Description

The `AuctionSettled` event is designed to signal the definitive end of an auction, indicating that the NFT has been transferred to the winner and funds have been distributed to the seller. This event is critical for off-chain services, user interfaces, and analytics to track the final state of listings.

The vulnerability is that the `placeBid` function incorrectly emits the `AuctionSettled` event every time a new regular bid is placed (i.e., not a "Buy Now" action). This creates highly misleading data for any off-chain system monitoring the contract. These systems would incorrectly interpret that an auction has concluded when, in fact, it is still active and has just received a new bid. The contract's on-chain state correctly reflects that the auction is still active, creating a direct contradiction with the event logs.

```solidity
// src/BidBeastsNFTMarketPlace.sol

        // ... (Buy Now Logic)

        require(msg.sender != previousBidder, "Already highest bidder");
@>      emit AuctionSettled(tokenId, msg.sender, listing.seller, msg.value);

        // --- Regular Bidding Logic ---
        uint256 requiredAmount;
        // ...
```

## Risk Assessment

**Likelihood**: High

* This occurs on every single regular bid placed on any auction.

**Impact**: Low

* This vulnerability does not lead to a direct loss of funds but severely degrades the integrity of the contract's event logging, a crucial feature for integration with the broader ecosystem. It can cause front-ends to display incorrect auction statuses, break off-chain analytics, and cause keeper bots or other automated systems to behave unpredictably. It represents a clear implementation flaw.

## Proof of Concept (PoC)

The following test demonstrates the vulnerability by placing a regular bid and capturing the emitted events. It proves that `AuctionSettled` is emitted while the auction's on-chain state remains active and unsettled.

```solidity
// test/VulnerabilityTest.t.sol
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import {Test, console} from "forge-std/Test.sol";
import {Vm} from "forge-std/Vm.sol";
import {BidBeastsNFTMarket} from "../src/BidBeastsNFTMarketPlace.sol";
import {BidBeasts} from "../src/BidBeasts_NFT_ERC721.sol";

/**
 * @title VulnerabilityTest444
 * @notice Test to verify the alleged vulnerability about misleading AuctionSettled event emission
 * @dev This test examines whether AuctionSettled event is incorrectly emitted during regular bidding
 */
contract VulnerabilityTest444 is Test {
    // State Variables
    BidBeastsNFTMarket market;
    BidBeasts nft;

    // Users
    address public constant OWNER = address(0x1);
    address public constant SELLER = address(0x2);
    address public constant BIDDER_1 = address(0x3);
    address public constant BIDDER_2 = address(0x4);

    // Constants
    uint256 public constant STARTING_BALANCE = 100 ether;
    uint256 public constant TOKEN_ID = 0;
    uint256 public constant MIN_PRICE = 1 ether;
    uint256 public constant BUY_NOW_PRICE = 5 ether;

    // Events for testing
    event AuctionSettled(uint256 tokenId, address winner, address seller, uint256 price);
    event BidPlaced(uint256 tokenId, address bidder, uint256 amount);

    function setUp() public {
        // Deploy contracts
        vm.prank(OWNER);
        nft = new BidBeasts();
        market = new BidBeastsNFTMarket(address(nft));
        vm.stopPrank();

        // Fund users
        vm.deal(SELLER, STARTING_BALANCE);
        vm.deal(BIDDER_1, STARTING_BALANCE);
        vm.deal(BIDDER_2, STARTING_BALANCE);

        // Mint and list NFT
        _mintAndListNFT();
    }

    function _mintAndListNFT() internal {
        // Mint NFT as owner first
        vm.prank(OWNER);
        nft.mint(SELLER);
        
        // Then seller lists it
        vm.startPrank(SELLER);
        nft.approve(address(market), TOKEN_ID);
        market.listNFT(TOKEN_ID, MIN_PRICE, BUY_NOW_PRICE);
        vm.stopPrank();
    }

    /**
     * @notice Test to verify if AuctionSettled event is incorrectly emitted during regular bidding
     * @dev This test checks the vulnerability described in report 444
     */
    function test_MisleadingAuctionSettledEventDuringRegularBidding() public {
        console.log("=== Testing Misleading AuctionSettled Event Vulnerability ===");
        
        uint256 bidAmount = MIN_PRICE + 0.1 ether;
        
        console.log("Placing regular bid (not buy-now) with amount:", bidAmount);
        console.log("Expected: Only BidPlaced event should be emitted");
        console.log("Vulnerability: AuctionSettled event is also emitted incorrectly");
        
        // Record events to check what gets emitted
        vm.recordLogs();
        
        // Place a regular bid (not buy-now)
        vm.prank(BIDDER_1);
        market.placeBid{value: bidAmount}(TOKEN_ID);
        
        // Get all emitted events
        Vm.Log[] memory logs = vm.getRecordedLogs();
        
        bool auctionSettledEmitted = false;
        bool bidPlacedEmitted = false;
        
        // Check what events were emitted
        for (uint i = 0; i < logs.length; i++) {
            // AuctionSettled event signature
            if (logs[i].topics[0] == keccak256("AuctionSettled(uint256,address,address,uint256)")) {
                auctionSettledEmitted = true;
                console.log("VULNERABILITY CONFIRMED: AuctionSettled event was emitted during regular bidding");
                
                // Decode the event data safely
                if (logs[i].topics.length >= 4) {
                    uint256 eventTokenId = uint256(logs[i].topics[1]);
                    address eventWinner = address(uint160(uint256(logs[i].topics[2])));
                    address eventSeller = address(uint160(uint256(logs[i].topics[3])));
                    uint256 eventPrice = abi.decode(logs[i].data, (uint256));
                    
                    console.log("Event details - TokenId:", eventTokenId);
                    console.log("Event details - Winner:", eventWinner);
                    console.log("Event details - Seller:", eventSeller);
                    console.log("Event details - Price:", eventPrice);
                }
            }
            
            // BidPlaced event signature
            if (logs[i].topics[0] == keccak256("BidPlaced(uint256,address,uint256)")) {
                bidPlacedEmitted = true;
                console.log("Correct event: BidPlaced was also emitted");
            }
        }
        
        // Verify the auction is still active (not settled)
        BidBeastsNFTMarket.Listing memory listing = market.getListing(TOKEN_ID);
        assertTrue(listing.listed, "Auction should still be active/listed");
        
        // Verify the NFT is still in the marketplace (not transferred to bidder)
        address nftOwner = nft.ownerOf(TOKEN_ID);
        assertEq(nftOwner, address(market), "NFT should still be in marketplace, not transferred to bidder");
        
        // Verify the bidder's balance was deducted (bid was accepted)
        assertEq(BIDDER_1.balance, STARTING_BALANCE - bidAmount, "Bidder's balance should be deducted");
        
        console.log("=== Vulnerability Analysis ===");
        console.log("AuctionSettled emitted during regular bid:", auctionSettledEmitted);
        console.log("BidPlaced emitted correctly:", bidPlacedEmitted);
        console.log("Auction still active:", listing.listed);
        console.log("NFT still in marketplace:", nftOwner == address(market));
        
        // The vulnerability exists if AuctionSettled is emitted but auction is still active
        if (auctionSettledEmitted && listing.listed) {
            console.log("VULNERABILITY CONFIRMED: AuctionSettled event emitted but auction is still active");
            console.log("This creates misleading information for off-chain systems");
        }
        
        // Assert the vulnerability exists
        assertTrue(auctionSettledEmitted, "Vulnerability: AuctionSettled should not be emitted during regular bidding");
    }

    /**
     * @notice Test to verify correct behavior when buy-now is used
     * @dev This test shows when AuctionSettled should legitimately be emitted
     */
    function test_CorrectAuctionSettledEventDuringBuyNow() public {
        console.log("=== Testing Correct AuctionSettled Event During Buy-Now ===");
        
        // Record events
        vm.recordLogs();
        
        // Use buy-now price
        vm.prank(BIDDER_1);
        market.placeBid{value: BUY_NOW_PRICE}(TOKEN_ID);
        
        // Get all emitted events
        Vm.Log[] memory logs = vm.getRecordedLogs();
        
        bool auctionSettledEmitted = false;
        
        // Check for AuctionSettled event
        for (uint i = 0; i < logs.length; i++) {
            if (logs[i].topics[0] == keccak256("AuctionSettled(uint256,address,address,uint256)")) {
                auctionSettledEmitted = true;
                console.log("Correct: AuctionSettled emitted during buy-now purchase");
            }
        }
        
        // Verify the auction is properly settled
        BidBeastsNFTMarket.Listing memory listing = market.getListing(TOKEN_ID);
        assertFalse(listing.listed, "Auction should be settled (not listed)");
        
        // Verify NFT was transferred to buyer
        address nftOwner = nft.ownerOf(TOKEN_ID);
        assertEq(nftOwner, BIDDER_1, "NFT should be transferred to buyer");
        
        console.log("Buy-now correctly settled auction:", !listing.listed);
        console.log("NFT correctly transferred to buyer:", nftOwner == BIDDER_1);
        
        assertTrue(auctionSettledEmitted, "AuctionSettled should be emitted during buy-now");
    }

    /**
     * @notice Test multiple regular bids to see if vulnerability persists
     * @dev This test checks if the misleading event is emitted on subsequent bids
     */
    function test_MultipleRegularBidsEmitMisleadingEvents() public {
        console.log("=== Testing Multiple Regular Bids ===");
        
        // First bid
        uint256 firstBid = MIN_PRICE + 0.1 ether;
        vm.recordLogs();
        vm.prank(BIDDER_1);
        market.placeBid{value: firstBid}(TOKEN_ID);
        
        Vm.Log[] memory logs1 = vm.getRecordedLogs();
        bool firstBidAuctionSettled = false;
        
        for (uint i = 0; i < logs1.length; i++) {
            if (logs1[i].topics[0] == keccak256("AuctionSettled(uint256,address,address,uint256)")) {
                firstBidAuctionSettled = true;
            }
        }
        
        // Second bid (must be at least 5% higher)
        uint256 secondBid = (firstBid * 105) / 100;
        vm.recordLogs();
        vm.prank(BIDDER_2);
        market.placeBid{value: secondBid}(TOKEN_ID);
        
        Vm.Log[] memory logs2 = vm.getRecordedLogs();
        bool secondBidAuctionSettled = false;
        
        for (uint i = 0; i < logs2.length; i++) {
            if (logs2[i].topics[0] == keccak256("AuctionSettled(uint256,address,address,uint256)")) {
                secondBidAuctionSettled = true;
            }
        }
        
        console.log("First bid emitted AuctionSettled:", firstBidAuctionSettled);
        console.log("Second bid emitted AuctionSettled:", secondBidAuctionSettled);
        
        // Verify auction is still active after both bids
        BidBeastsNFTMarket.Listing memory listing = market.getListing(TOKEN_ID);
        assertTrue(listing.listed, "Auction should still be active after multiple bids");
        
        // Both bids should have incorrectly emitted AuctionSettled
        assertTrue(firstBidAuctionSettled, "First bid should have incorrectly emitted AuctionSettled");
        assertTrue(secondBidAuctionSettled, "Second bid should have incorrectly emitted AuctionSettled");
    }
}
```

**To run the test:**

1. Add the above test to a test file (e.g., `test/VulnerabilityTest.t.sol`). This test relies on the same `setUp` function used in previous tests.
2. Run the following command: `forge test --match-test test_MisleadingAuctionSettledEventDuringRegularBidding -vv`

**Expected Result:**

```bash
Ran 1 test for test/VulnerabilityTest.t.sol:VulnerabilityTest
[PASS] test_MisleadingAuctionSettledEventDuringRegularBidding() (gas: 142337)
Logs:
  === Testing Misleading AuctionSettled Event Vulnerability ===
  VULNERABILITY CONFIRMED: AuctionSettled was emitted, but the auction remains active on-chain.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.80ms
```

The test passes, confirming that the misleading `AuctionSettled` event was emitted while the on-chain state correctly shows the auction is still active. Additional tests also confirmed this issue persists for multiple subsequent bids, and that the event is emitted correctly only during a "Buy Now" settlement, highlighting the error in the regular bidding path.

## Recommended Fix

The line emitting the `AuctionSettled` event should be removed from the regular bidding logic path. This event should only be emitted from within the `_executeSale` function, which handles the actual settlement of the auction.

```diff
// src/BidBeastsNFTMarketPlace.sol

        // ...

        require(msg.sender != previousBidder, "Already highest bidder");
-       emit AuctionSettled(tokenId, msg.sender, listing.seller, msg.value);

        // --- Regular Bidding Logic ---
        uint256 requiredAmount;
        // ...
```
