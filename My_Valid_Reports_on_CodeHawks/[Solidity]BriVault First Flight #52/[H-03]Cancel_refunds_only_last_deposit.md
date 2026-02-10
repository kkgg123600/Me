# Finding Title

cancelParticipation refunds only last deposit stake, earlier deposits lost

# Root + Impact

- Root: `stakedAsset[receiver] = stakeAsset;` overwrites on each deposit, and `cancelParticipation()` refunds `stakedAsset[msg.sender]` only.
- Impact: Users making multiple deposits before event start receive a refund of only the last deposit’s stake (net of fee). Earlier deposits remain in the vault while all shares are burned, causing loss to the user.

## Description

- Normal behavior: Users should be able to cancel before event start and recover all their staked funds (minus fees) contributed so far.
- Issue: `stakedAsset` tracks only the most recent deposit. On cancel, shares are fully burned and only the last stake is refunded, leaving previous stakes in the vault.

```solidity
// BriVault.sol
function deposit(uint256 assets, address receiver) public override returns (uint256) {
    uint256 stakeAsset = assets - fee;
    stakedAsset[receiver] = stakeAsset; // @> overwrites previous deposits
    _mint(msg.sender, participantShares);
}

function cancelParticipation () public {
    uint256 refundAmount = stakedAsset[msg.sender]; // @> only last value
    stakedAsset[msg.sender] = 0;
    uint256 shares = balanceOf(msg.sender);
    _burn(msg.sender, shares);
    IERC20(asset()).safeTransfer(msg.sender, refundAmount);
}
```

## Risk

- Likelihood: High
  - Occurs when a user makes multiple deposits before cancelling.
- Impact: High
  - Loss of earlier deposit amounts for the user, vault retains funds unfairly.

## Proof of Concept

```solidity
// test/AuditPoC.t.sol::test_poc3_cancelParticipation_refunds_only_last_deposit
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

    /// PoC-3: cancelParticipation refunds only last deposit amount, losing prior deposits
    function test_poc3_cancelParticipation_refunds_only_last_deposit() public {
        vm.startPrank(user1);
        mockToken.approve(address(briVault), type(uint256).max);

        // First deposit 5 ether
        briVault.deposit(5 ether, user1);
        // Second deposit 3 ether
        briVault.deposit(3 ether, user1);

        uint256 userBalBefore = mockToken.balanceOf(user1);
        // Cancel before event start
        briVault.cancelParticipation();
        vm.stopPrank();

        // Refund equals only the last deposit stake (assets - fee)
        uint256 lastFee = (3 ether * participationFeeBsp) / 10000; // 0.045 ether
        uint256 expectedRefund = 3 ether - lastFee; // 2.955 ether
        assertEq(mockToken.balanceOf(user1), userBalBefore + expectedRefund, "only last deposit refunded");
        // Shares are fully burned; earlier deposit funds remain stuck in vault
        assertEq(briVault.balanceOf(user1), 0, "shares burned");
        // Vault should still hold the first stake amount
        uint256 firstFee = (5 ether * participationFeeBsp) / 10000; // 0.075 ether
        uint256 firstStake = 5 ether - firstFee; // 4.925 ether
        // Total vault assets should be close to firstStake (allow small rounding on shares math not relevant here)
        assertEq(mockToken.balanceOf(address(briVault)), firstStake, "first stake remains in vault (loss to user)");
    }
}
```

**Test result**

```bash
forge test --match-test test_poc3_cancelParticipation_refunds_only_last_deposit -vv

Ran 1 test for test/AuditPoC.t.sol:AuditPoCTest
[PASS] test_poc3_cancelParticipation_refunds_only_last_deposit() (gas: 176410)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.72ms (326.52µs CPU time)
```

## Recommended Mitigation

```diff
- stakedAsset[receiver] = stakeAsset;
 stakedAsset[receiver] += stakeAsset; // accumulate total stake

- uint256 refundAmount = stakedAsset[msg.sender];
 uint256 refundAmount = stakedAsset[msg.sender]; // refunds all accumulated stake
```

Also consider tracking per-deposit entries if needed for granular accounting.
