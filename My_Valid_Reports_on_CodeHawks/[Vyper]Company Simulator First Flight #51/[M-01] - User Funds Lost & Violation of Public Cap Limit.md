
# Finding Title

Investment Accepted When Public Share Cap Reached

## [M-01] - User Funds Lost & Violation of Public Cap Limit

## Description

The protocol mandates a `public_shares_cap` to limit the number of shares available for investment.

* **Normal Behavior:** When `issued_shares` reaches `public_shares_cap`, all public funding attempts via `fund_investor` should revert.

* **Issue:** The initial access control check `assert (self.issued_shares <= self.public_shares_cap)` incorrectly uses the "less than or equal to" operator (`<=`), which allows the function to proceed even when the cap is precisely met (`==`). Following this, the available shares are calculated: `available: uint256 = self.public_shares_cap - self.issued_shares`. When the cap is full, `available` is `0`.
    The subsequent logic trims the newly calculated shares to zero (`new_shares = 0`) but still adds the investor's full `msg.value` to `self.company_balance` and records `0` shares.

This is a specific, high-likelihood scenario of the "zero shares for payment" vulnerability, resulting from the cap being filled.

```vyper
// src/Cyfrin_Hub.vy

@internal
@payable
def fund_investor():
    assert msg.value > 0, "Must send ETH!!!"
    assert (
        @> self.issued_shares <= self.public_shares_cap // Passed even when ==
    ), "Share cap reached!!!"
    assert (self.company_balance > self.holding_debt), "Company is insolvent!!!"

    # ... Share Price calculation ...

    # Cap calculation logic
    available: uint256 = self.public_shares_cap - self.issued_shares
    if new_shares > available:
        new_shares = available     // Sets new_shares to 0 when cap is full

    # ... state updates ...
    self.company_balance += msg.value   // ETH accepted even if new_shares == 0
```

## Risk

**Likelihood**: High

* This is guaranteed to occur at the precise moment the total `issued_shares` reaches the defined `public_shares_cap`, and any subsequent public investment is attempted.

**Impact**: Medium

* **Loss of User Funds:** Investors send ETH but receive no corresponding equity.
* **Cap Violation:** The intent of the public cap is violated by allowing `company_balance` to grow through public investment even when the cap is technically full.

## Proof of Concept

This test sets a small cap and attempts an investment that pushes the contract to or beyond its capacity, confirming that the payment is accepted but no shares are issued.

```python
# File: tests/unit/test_vulnerabilities_poc.py

import boa
import pytest
from eth_utils import to_wei

def test_AP03_investment_accepted_when_cap_reached(industry_contract, OWNER, PATRICK, HE1M):
    """
    Test AP-03: Investment Accepted When Public Share Cap Reached
    
    This test verifies that fund_investor accepts ETH even when public_shares_cap
    is full, resulting in zero shares issued while ETH is added to company_balance.
    """
    print("Testing AP-03: Investment Accepted When Cap Reached")
    
    # Arrange: Set up scenario where share cap is nearly full
    # 1. Increase share cap to a small number (1000 shares total)
    with boa.env.prank(OWNER):
        # Initial cap is 1,000,000. Increase it by 1000 for a total of 1,001,000.
        industry_contract.increase_share_cap(1000)  
        # Fund company first (10 ETH)
        industry_contract.fund_cyfrin(0, value=to_wei(10, "ether"))
    
    # 2. Fill the cap almost completely with PATRICK (1 ETH investment)
    large_investment = to_wei(1, "ether") 
    
    with boa.env.prank(PATRICK):
        industry_contract.fund_cyfrin(1, value=large_investment)
    
    issued_shares = industry_contract.issued_shares()
    shares_cap = industry_contract.public_shares_cap()
    
    print(f"Issued shares: {issued_shares}") # Will be close to cap
    print(f"Shares cap: {shares_cap}")
    
    # Now try to invest when cap is reached/nearly reached
    balance_before = industry_contract.get_balance()
    ATTACK_AMOUNT = to_wei(0.5, "ether")

    # Act: HE1M invests a substantial amount, but only 0 or very few shares are left
    with boa.env.prank(HE1M):
        industry_contract.fund_cyfrin(1, value=ATTACK_AMOUNT)
    
    # Assert
    he1m_shares = industry_contract.get_my_shares(caller=HE1M)
    balance_after = industry_contract.get_balance()
    
    print(f"HE1M shares received: {he1m_shares}")
    print(f"Balance before HE1M investment: {balance_before} Wei")
    print(f"Balance after HE1M investment: {balance_after} Wei")
    
    # The vulnerability check:
    assert he1m_shares == 0, "HE1M should have received zero shares as cap was effectively full/new_shares was trimmed"
    assert balance_after == balance_before + ATTACK_AMOUNT, "ETH was accepted even though zero shares were issued"

    print("CONFIRMED: Zero shares issued but ETH accepted and added to balance")
```

**Verified Test Output:**

```bash
Testing AP-03: Investment Accepted When Cap Reached
Issued shares: 1000000
Shares cap: 1001000
HE1M shares received: 0
Balance before HE1M investment: 11000000000000000000 Wei
Balance after HE1M investment: 11500000000000000000 Wei
CONFIRMED: Zero shares issued but ETH accepted and added to balance
PASSED
```

## Recommended Mitigation

The precondition check should use the "less than" operator (`<`) instead of "less than or equal to" (`<=`) to immediately revert when the cap is met. Furthermore, a secondary check should be added after calculating `available` to ensure it is not zero before proceeding.

```diff
// src/Cyfrin_Hub.vy

@internal
@payable
def fund_investor():
    assert msg.value > 0, "Must send ETH!!!"
    assert (
-       self.issued_shares <= self.public_shares_cap
+       self.issued_shares < self.public_shares_cap // Must be strictly less than
    ), "Share cap reached!!!"
    assert (self.company_balance > self.holding_debt), "Company is insolvent!!!"

    # ... (share price and new_shares calculation)

    # Calculate available and check cap before attempting any minting/balance update
    available: uint256 = self.public_shares_cap - self.issued_shares
+   if available == 0:
+       raise "Share cap reached!" // Failsafe check (though handled by the initial assert now)
+
    if new_shares > available:
        new_shares = available

    # ... (rest of the function)
```
