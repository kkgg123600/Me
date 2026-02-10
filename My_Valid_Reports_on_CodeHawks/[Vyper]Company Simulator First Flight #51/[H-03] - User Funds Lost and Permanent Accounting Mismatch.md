
# Finding Title

Payment Lost and Unaccounted on Failed Sale in `sell_to_customer` (Stuck ETH)

## [H-03] - User Funds Lost and Permanent Accounting Mismatch

## Description

The decentralized system relies on the `CustomerEngine` to manage customer payments and the `Cyfrin_Hub` to fulfill the sale.

* **Normal Behavior:** The `CustomerEngine` forwards the exact payment (`requested * ITEM_PRICE`) to the hub using a `raw_call` configured to revert on failure (`revert_on_failure=True`). This establishes a clear expectation: the Hub must either successfully complete the sale (and record revenue) or explicitly revert.

* **Issue:** In `Cyfrin_Hub.vy`, when the `if` condition fails due to insufficient inventory, the code enters the `else` block. The function is marked `@payable`, so it successfully accepts the incoming ETH into the contract's physical address balance. However, the `else` block only updates `reputation` and emits an event; it does *not* add the funds to the internal `self.company_balance` nor does it trigger a `revert`. This causes the `raw_call` from the `CustomerEngine` to "succeed" from a transaction standpoint, permanently losing the customer's payment. The ETH becomes **stuck** in the contract's physical balance, invisible and unusable for internal operations or investor payouts.

```python
# Cyfrin_Hub.vy

@external
@payable
def sell_to_customer(requested: uint256):
    assert msg.sender == self.CUSTOMER_ENGINE, "Not the customer engine!!!"
    assert not self._is_bankrupt(), "Company is bankrupt!!!"
    self._apply_holding_cost()

    if self.inventory >= requested:
        # ... success: self.company_balance += revenue
    else:
        @> self.reputation = min(max(self.reputation - REPUTATION_PENALTY, 0), 100)
        @> log ReputationChanged(new_reputation=self.reputation)
        // @> NO REVERT: ETH is accepted but remains unrecorded and stuck.
```

## Risk

**Likelihood**: High

* The vulnerability is triggered by a common operational event: customer demand exceeding available inventory. Since demand is pseudo-randomly calculated, this will occur frequently if the company is not actively producing.
* The attack requires only a **"Customer" (public user)** role, making it easily exploitable.

**Impact**: High

* **User Fund Loss:** Customers lose the ETH paid for the items with no recovery path.
* **Accounting Corruption:** It permanently breaks the critical economic invariant that the contract's physical ETH balance must match its internal accounting balance (`address(this).balance == self.company_balance + holding_debt`). This leads to incorrect calculation of the company's `net_worth` and misrepresents the funds available for operations and investor withdrawals.

## Proof of Concept

This PoC uses the provided test environment to call the vulnerable function when inventory is zero, confirming that the internal balance remains static while the actual contract balance increases by the payment amount.

```python
# File: tests/unit/test_vulnerabilities_poc.py
import boa
import pytest
from eth_utils import to_wei

# Constants from the contracts
ITEM_PRICE = 2 * 10**16  # 0.02 ETH per item
INITIAL_SHARE_PRICE = 1 * 10**15  # 0.001 ETH
LOCKUP_PERIOD = 30 * 86400  # 30 days in seconds


def test_AP01_payment_lost_on_failed_sale(industry_contract, customer_engine_contract, OWNER, PATRICK):
    """
    Test AP-01: Payment Lost and Unaccounted on Failed Sale
    
    This test verifies that when sell_to_customer fails due to insufficient inventory,
    the ETH sent by the customer is accepted but not refunded or added to company_balance,
    causing a permanent loss of user funds and accounting mismatch.
    """
    print("Testing AP-01: Payment Lost on Failed Sale")
    
    # Arrange: Ensure inventory is 0 for guaranteed failed sale
    assert industry_contract.inventory() == 0, "Inventory must be 0 for guaranteed failed sale"
    
    # Fund the company with initial balance to establish baseline
    initial_balance = to_wei(1, "ether")
    with boa.env.prank(OWNER):
        industry_contract.fund_cyfrin(0, value=initial_balance)
    
    # Record starting balances
    starting_balance_internal = industry_contract.get_balance()
    starting_balance_actual = boa.env.get_balance(industry_contract.address)
    
    print(f"Starting internal balance: {starting_balance_internal} Wei")
    print(f"Starting actual balance: {starting_balance_actual} Wei")
    
    assert starting_balance_internal == initial_balance
    assert starting_balance_actual == initial_balance
    
    # Act: PATRICK triggers demand which will fail due to zero inventory
    sale_amount = to_wei(0.1, "ether")  # Max 5 items * 0.02 ETH = 0.1 ETH
    
    with boa.env.prank(PATRICK):
        # This should succeed without revert, but inventory check will fail
        customer_engine_contract.trigger_demand(value=sale_amount)
    
    # Assert: Check the vulnerability
    final_balance_internal = industry_contract.get_balance()
    final_balance_actual = boa.env.get_balance(industry_contract.address)
    
    print(f"Final internal balance: {final_balance_internal} Wei")
    print(f"Final actual balance: {final_balance_actual} Wei")
    
    # The vulnerability: Internal balance unchanged, actual balance increased
    assert final_balance_internal == starting_balance_internal, "Internal balance should not change on failed sale"
    assert final_balance_actual > starting_balance_actual, "Actual balance increased due to stuck ETH"
    
    stuck_eth = final_balance_actual - final_balance_internal
    print(f"Stuck ETH amount: {stuck_eth} Wei")
    
    # Verify the exact amount of stuck ETH
    assert stuck_eth > 0, "ETH should be stuck in the contract"
    print("CONFIRMED: User ETH is lost and stuck, causing accounting mismatch")

```

**Verified Test Output:**

```bash
mox test tests/unit/test_vulnerabilities_poc.py -v -s

tests/unit/test_vulnerabilities_poc.py::test_AP01_payment_lost_on_failed_sale Cyfrin Industry deployed at 0xC6Acb7D16D51f72eAA659668F30A40d87E2E0551
Customer Engine deployed at 0x3d06E92f20305D9a2D71a1D479E9EE22690Ae7E4
Testing AP-01: Payment Lost on Failed Sale
Starting internal balance: 1000000000000000000 Wei
Starting actual balance: 1000000000000000000 Wei
Final internal balance: 1000000000000000000 Wei
Final actual balance: 1020000000000000000 Wei
Stuck ETH amount: 20000000000000000 Wei
CONFIRMED: User ETH is lost and stuck, causing accounting mismatch
PASSED
```

## Recommended Mitigation

The `sell_to_customer` function must explicitly `raise` (revert) when a sale cannot be fulfilled. This relies on the established contract interaction model where `CustomerEngine`'s `raw_call` automatically refunds the ETH upon a revert.

```diff
// src/Cyfrin_Hub.vy

@external
@payable
def sell_to_customer(requested: uint256):
    assert msg.sender == self.CUSTOMER_ENGINE, "Not the customer engine!!!"
    assert not self._is_bankrupt(), "Company is bankrupt!!!"
    self._apply_holding_cost()

    if self.inventory >= requested:
        self.inventory -= requested
        revenue: uint256 = requested * SALE_PRICE
        self.company_balance += revenue
        if self.reputation < 100:
            # Increase reputation for successful sale
            self.reputation = min(self.reputation + REPUTATION_REWARD, 100)
        else:
            # Maintain reputation if already at max
            self.reputation = 100
        log Sold(amount=requested, revenue=revenue)
    else:
        # Revert explicitly to trigger the automatic refund via CustomerEngine's raw_call
+       raise "Insufficient inventory for sale!!!" 
        self.reputation = min(max(self.reputation - REPUTATION_PENALTY, 0), 100)

        log ReputationChanged(new_reputation=self.reputation)
```
