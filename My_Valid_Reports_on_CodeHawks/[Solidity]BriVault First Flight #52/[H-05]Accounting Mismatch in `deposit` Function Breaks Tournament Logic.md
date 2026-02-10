
# Finding Title

Accounting Mismatch in `deposit` Function Breaks Tournament Logic and User Participation Path

## Root + Impact

- **Root:** The `deposit` function mints ERC4626 shares to `msg.sender` while assigning the participation right (`stakedAsset` record) to the `receiver` address.
- **Impact:** This fundamental accounting split makes it impossible for a user to complete the intended "deposit -> join -> withdraw" lifecycle correctly. It breaks the core application logic, leads to failed transactions and confusing states, and forces users to rely on unintended bypass mechanisms (standard ERC4626 functions) to recover funds, defeating the purpose of the tournament's rules.

## Description

The intended user flow of the protocol requires a participant to first deposit assets to receive shares, and then use their credited stake to join the tournament by selecting a team. A critical invariant for this to work is that the address holding the shares must be the same address credited with the stake.

The `deposit` function violates this invariant by allowing `msg.sender` and `receiver` to be different addresses. It correctly mints shares to the depositor (`msg.sender`) but incorrectly assigns the stake record—the prerequisite for joining the event—to the `receiver`.

This creates a broken state with the following consequences:

1. **Depositor (`msg.sender`) Cannot Participate:** The user who funded the deposit and holds the shares cannot call `joinEvent` because the contract does not recognize their stake (`stakedAsset[msg.sender]` is zero).
2. **Receiver's Participation is Ineffective:** The `receiver` can call `joinEvent`, but since they hold no shares (`balanceOf(receiver)` is zero), their participation is meaningless. They are registered as joining with zero contribution.

While funds are not permanently locked due to the presence of un-overridden ERC4626 `withdraw` and `redeem` functions, the primary application logic is unequivocally broken. Users are forced into a confusing situation where the only way to recover funds is to use a backdoor mechanism that bypasses all tournament rules (fees, time-locks, winner conditions).

```solidity
// src/briVault.sol

function deposit(uint256 assets, address receiver) public override returns (uint256) {
    // ... input validation and fee calculation ...
    uint256 stakeAsset = assets - fee;

    // @> The right to join is recorded for the `receiver`.
    stakedAsset[receiver] = stakeAsset;

    // ... asset transfers are performed ...
    uint256 participantShares = _convertToShares(stakeAsset);

    // @> But the shares (economic value) are minted to `msg.sender`.
    _mint(msg.sender, participantShares);

    emit deposited(receiver, stakeAsset);
    return participantShares;
}

function joinEvent(uint256 countryId) public {
    // This check fails for `msg.sender` if they deposited for a `receiver`.
    if (stakedAsset[msg.sender] == 0) {
        revert noDeposit();
    }
    // If the `receiver` calls this, `balanceOf(msg.sender)` will be 0,
    // making their participation ineffective.
    uint256 participantShares = balanceOf(msg.sender);
    userSharesToCountry[msg.sender][countryId] = participantShares;
}
```

## Risk

**Likelihood**: Medium

* This scenario occurs whenever the `deposit` function is called with a `receiver` address different from `msg.sender`. This is a standard pattern in many token vaults, making accidental triggering by users or frontends plausible.

**Impact**: Medium

* **Broken Core Application Logic:** The primary path for participation is broken, preventing users from joining the tournament as intended. This undermines the main purpose of the contract.
* **Unfairness and User Confusion:** It creates a confusing state where neither party can proceed as expected. It forces users to discover and use non-obvious, unintended recovery paths (the standard ERC4626 functions), which also allows them to bypass tournament rules like fees and time-locks.
* **Failure to Adhere to Protocol Rules:** The vulnerability effectively renders the tournament's custom logic optional, as the underlying ERC4626 functions provide a loophole.

## Proof of Concept

The following test demonstrates the broken logic. It shows that when `Alice` (`user1`) deposits for `Bob` (`user2`), neither can complete the intended participation flow, and their interaction with the tournament logic fails.

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
/// PoC: Demonstrates that the deposit-receiver mismatch breaks the participation logic.
function test_poc_depositReceiverMismatch_breaksLogic() public {
    address alice = user1; // msg.sender
    address bob = user2;   // receiver

    uint256 depositAmount = 5 ether;

    // 1. Alice deposits for Bob's benefit.
    vm.startPrank(alice);
    mockToken.approve(address(briVault), depositAmount);
    uint256 sharesMinted = briVault.deposit(depositAmount, bob);
    vm.stopPrank();

    // Assert the split accounting state
    assertEq(briVault.balanceOf(alice), sharesMinted, "Alice (sender) holds the shares");
    assertEq(briVault.balanceOf(bob), 0, "Bob (receiver) has no shares");
    assertGt(briVault.stakedAsset(bob), 0, "Bob (receiver) has the stake record");
    assertEq(briVault.stakedAsset(alice), 0, "Alice (sender) has no stake record");

    // 2. Alice, who holds the shares, is blocked from joining the event.
    vm.startPrank(alice);
    vm.expectRevert(BriVault.noDeposit.selector);
    briVault.joinEvent(10);
    vm.stopPrank();

    // 3. Bob, who has the stake record, joins with an ineffective zero-share contribution.
    vm.startPrank(bob);
    briVault.joinEvent(10);
    vm.stopPrank();

    uint256 bobShareSnapshot = briVault.userSharesToCountry(bob, 10);
    assertEq(bobShareSnapshot, 0, "Bob's participation is recorded with zero shares");
    
    // Conclusion: The intended flow is broken. Alice cannot join. Bob's join is useless.
    // To recover funds, Alice must use the standard ERC4626 `withdraw` function,
    // bypassing the tournament's `withdraw` logic entirely.
}
}
```

**Test result**

```bash
forge test --match-test test_poc_depositReceiverMismatch_breaksLogic -vv

Ran 1 test for test/AuditPoC.t.sol:AuditPoCTest
[PASS] test_poc_depositReceiverMismatch_breaksLogic() (gas: 268318)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.79ms (301.62µs CPU time)
```

## Recommended Mitigation

The contract's accounting must be consistent. The address that receives the shares must be the same one that is credited with the stake.

**Recommended Fix: Enforce `receiver == msg.sender`**
This is the simplest and safest solution. It disables the problematic feature and ensures the integrity of the protocol's logic.

```diff
// src/briVault.sol

function deposit(uint256 assets, address receiver) public override returns (uint256) {
    require(receiver != address(0));
+   require(receiver == msg.sender, "BriVault: receiver must be sender");

    // ... rest of the function ...
}
```

**Alternative Fix: Align All Logic to `receiver`**
If supporting deposits for others is a required feature, then all internal accounting, including share minting, must be consistently keyed to the `receiver`.

```diff
// src/briVault.sol

function deposit(uint256 assets, address receiver) public override returns (uint256) {
    // ...
    stakedAsset[receiver] = stakeAsset;
    // ...
    IERC20(asset()).safeTransferFrom(msg.sender, address(this), stakeAsset);

-   _mint(msg.sender, participantShares);
+   _mint(receiver, participantShares);

    emit deposited(receiver, stakeAsset);
    return participantShares;
}
```
