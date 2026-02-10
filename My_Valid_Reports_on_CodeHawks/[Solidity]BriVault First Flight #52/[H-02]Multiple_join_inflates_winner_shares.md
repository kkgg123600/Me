# Finding Title

Multiple joinEvent calls by same user inflate totalWinnerShares, reducing payouts

## Root + Impact

- Root: `joinEvent` allows repeated joins and pushes duplicate `msg.sender` entries into `usersAddress` without guard.
- Impact: `totalWinnerShares` is computed by summing `userSharesToCountry[user][winnerCountryId]` across `usersAddress`, counting duplicates multiple times, inflating denominator and underpaying winners.

## Description

- Normal behavior: Each user should join once, and winner share sum should equal the sum of unique winners’ shares.
- Issue: `joinEvent` does not enforce single participation or uniqueness of `usersAddress`. Duplicate entries cause `totalWinnerShares` to be multiplied.

```solidity
// BriVault.sol
function joinEvent(uint256 countryId) public {
    // @> no guard preventing multiple joins
    userToCountry[msg.sender] = teams[countryId];
    uint256 participantShares = balanceOf(msg.sender);
    userSharesToCountry[msg.sender][countryId] = participantShares;
    usersAddress.push(msg.sender); // @> duplicates accumulate
}

function _getWinnerShares () internal returns (uint256) {
    for (uint256 i = 0; i < usersAddress.length; ++i){
        address user = usersAddress[i]; 
        totalWinnerShares += userSharesToCountry[user][winnerCountryId]; // @> duplicates counted
    }
}
```

## Risk

- Likelihood:
  - Occurs whenever a user calls `joinEvent` multiple times (no checks preventing it).
- Impact:
  - Winners receive a fraction of `finalizedVaultAsset` divided by inflated `totalWinnerShares`, leading to lower payouts and unfair distribution.

## Proof of Concept

```solidity
// File: test/AuditPoC.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {BriVault} from "../src/briVault.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {MockERC20} from "./MockErc20.t.sol";

contract AuditPoCTest is Test {
    uint256 public participationFeeBsp;
    uint256 public eventStartDate;
    uint256 public eventEndDate;
    address public participationFeeAddress;
    uint256 public minimumAmount;

    BriVault public briVault;
    MockERC20 public mockToken;

    address owner = makeAddr("owner");
    address user1 = makeAddr("user1");
    address user2 = makeAddr("user2");
    address user3 = makeAddr("user3");

    string[48] countries = [
        "United States", "Canada", "Mexico", "Argentina", "Brazil", "Ecuador",
        "Uruguay", "Colombia", "Peru", "Chile", "Japan", "South Korea",
        "Australia", "Iran", "Saudi Arabia", "Qatar", "Uzbekistan", "Jordan",
        "France", "Germany", "Spain", "Portugal", "England", "Netherlands",
        "Italy", "Croatia", "Belgium", "Switzerland", "Denmark", "Poland",
        "Serbia", "Sweden", "Austria", "Morocco", "Senegal", "Nigeria",
        "Cameroon", "Egypt", "South Africa", "Ghana", "Algeria", "Tunisia",
        "Ivory Coast", "New Zealand", "Costa Rica", "Panama", "United Arab Emirates", "Iraq"
    ];

    function setUp() public {
        participationFeeBsp = 150; // 1.5%
        eventStartDate = block.timestamp + 2 days;
        eventEndDate = eventStartDate + 31 days;
        participationFeeAddress = makeAddr("participationFeeAddress");
        minimumAmount = 0.0002 ether;

        mockToken = new MockERC20("Mock Token", "MTK");

        // Mint balances
        mockToken.mint(owner, 100 ether);
        mockToken.mint(user1, 100 ether);
        mockToken.mint(user2, 100 ether);
        mockToken.mint(user3, 100 ether);

        vm.startPrank(owner);
        briVault = new BriVault(
            IERC20(address(mockToken)),
            participationFeeBsp,
            eventStartDate,
            participationFeeAddress,
            minimumAmount,
            eventEndDate
        );
        vm.stopPrank();
    }


    /// PoC-4: Multiple joinEvent calls inflate totalWinnerShares via duplicate addresses
    function test_poc4_multiple_joins_double_count_winner_shares() public {
        // Set countries and pick a winner index
        vm.prank(owner);
        briVault.setCountry(countries);
        uint256 winnerIdx = 10; // Japan

        // User1 deposits and joins winner multiple times
        vm.startPrank(user1);
        mockToken.approve(address(briVault), type(uint256).max);
        briVault.deposit(5 ether, user1);
        briVault.joinEvent(winnerIdx);
        briVault.joinEvent(winnerIdx);
        briVault.joinEvent(winnerIdx);
        uint256 userShares = briVault.balanceOf(user1);
        vm.stopPrank();

        // End event and set winner
        vm.warp(eventEndDate + 1);
        vm.prank(owner);
        briVault.setWinner(winnerIdx);

        // totalWinnerShares should be 3x userShares due to duplicate entries in usersAddress
        assertEq(briVault.totalWinnerShares(), userShares * 3, "winner shares doubled/tripled by multiple joins");

        // Withdraw gives only 1/3 of finalized vault assets to the sole winner
        uint256 beforeBal = mockToken.balanceOf(user1);
        vm.prank(user1);
        briVault.withdraw();
        uint256 afterBal = mockToken.balanceOf(user1);
        uint256 payout = afterBal - beforeBal;

        // Expect payout to be approx finalizedVaultAsset / 3
        assertEq(payout, briVault.finalizedVaultAsset() / 3, "payout reduced by inflated denominator");
    }
}
```

**Test result**

```bash
forge test --match-test test_poc4_multiple_joins_double_count_winner_shares -vv

Ran 1 test for test/AuditPoC.t.sol:AuditPoCTest
[PASS] test_poc4_multiple_joins_double_count_winner_shares() (gas: 1816936)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.36ms (1.00ms CPU time)
```

## Recommended Mitigation

```diff
- usersAddress.push(msg.sender);
 require(bytes(userToCountry[msg.sender]).length == 0, "Already joined");
 usersAddress.push(msg.sender);
 numberOfParticipants++;
```

Additionally, consider maintaining a `mapping(address => bool) joined` and ensure `userSharesToCountry` updates append rather than overwrite if needed.
