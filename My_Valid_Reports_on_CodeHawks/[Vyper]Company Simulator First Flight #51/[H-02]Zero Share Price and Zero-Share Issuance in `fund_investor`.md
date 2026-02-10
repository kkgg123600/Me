# Finding Title

[H-02]Zero Share Price and Zero-Share Issuance in `fund_investor`

## Root + Impact

`fund_investor` computes share price based on net worth and `issued_shares`. In early states or edge cases, investors can pay ETH yet receive `0` shares, or shares become mispriced due to integer division, leading to unfair issuance and cap/economic inconsistencies.

## Description

- Normal behavior: Investors send ETH; the contract calculates `share_price = net_worth / issued_shares` (or `INITIAL_SHARE_PRICE` when no shares issued) and mints `msg.value // share_price`.
- Issues:
  1) When `issued_shares` is large and `net_worth` is small, `share_price` can collapse to `0` due to integer division. The subsequent `new_shares = msg.value // share_price` risks division by zero (if `issued_shares > 0` and net_worth == 0) or can become extremely large when `share_price` is tiny, destabilizing the cap.
  2) Even with `INITIAL_SHARE_PRICE`, the function allows `new_shares` to be reduced to `0` by cap trimming or tiny `msg.value`, still accepting the ETH and increasing `company_balance` while issuing `0` shares.

```vy
@internal
@payable
def fund_investor():
    # Preconditions
    assert msg.value > 0
    assert self.issued_shares <= self.public_shares_cap
    assert (self.company_balance > self.holding_debt)

    # Net worth
    net_worth: uint256 = 0
    if self.company_balance > self.holding_debt:
        net_worth = self.company_balance - self.holding_debt

    # Share price calc
    share_price: uint256 = (
        net_worth // max(self.issued_shares, 1)
        if self.issued_shares > 0
        else INITIAL_SHARE_PRICE
    )
    @> new_shares: uint256 = msg.value // share_price
    available: uint256 = self.public_shares_cap - self.issued_shares
    if new_shares > available:
        @> new_shares = available  // can become 0 if cap is full or near full

    @> self.shares[msg.sender] += new_shares
    @> self.issued_shares += new_shares
    @> self.company_balance += msg.value
```

## Risk

**Likelihood**:

- Early phases or after debt accrual when net worth is low relative to shares.
- Near cap limits where `available` is small or zero.

**Impact**:

- Investors can pay for `0` shares, violating economic fairness and expectations.
- Mispriced shares can inflate supply or create rounding arbitrage opportunities.
- Distorts `get_share_price`, withdrawals, and insolvency checks via skewed `issued_shares` and balance.

## Proof of Concept

- Setup (public state only):
  1) Owner calls `fund_cyfrin(0)` with 100 ETH (raises company balance, no shares).
  2) Investor calls `fund_cyfrin(1)` with 0.001 ETH, issuing exactly 1 share.
  3) Share Price computed externally from public variables: `share_price = (get_balance() - holding_debt) // issued_shares` ≈ 100 ETH.
  4) Investor sends `share_price - 1000 wei` → zero shares issued; full amount added to company balance.
  
```python
# File: tests/unit/test_vulnerabilities_poc_high_loss.py

import boa
import pytest
from eth_utils import to_wei


def test_AP02_large_absolute_loss_high_share_price(industry_contract, OWNER, PATRICK):
    """
    AP-02 High-Loss PoC: Demonstrates a single-transaction loss close to one full share price
    when share price is engineered to be ~100 ETH by issuing exactly 1 share.

    Steps:
    1) Owner funds 100 ETH via fund_cyfrin(0) to raise company_balance.
    2) Investor (PATRICK) issues exactly 1 share by investing 0.001 ETH at INITIAL_SHARE_PRICE.
    3) Compute share_price externally: net_worth // issued_shares.
    4) Investor sends (share_price - 1000 wei); zero shares are issued, but full amount is added to balance.
    """

    # Ensure test accounts have sufficient ETH
    boa.env.set_balance(OWNER, to_wei(200, "ether"))
    boa.env.set_balance(PATRICK, to_wei(300, "ether"))

    # 1) Owner funds 100 ETH without issuing shares
    with boa.env.prank(OWNER):
        industry_contract.fund_cyfrin(0, value=to_wei(100, "ether"))

    # 2) Investor issues exactly 1 share at INITIAL_SHARE_PRICE (0.001 ETH)
    #    When issued_shares == 0, share_price = INITIAL_SHARE_PRICE = 1e15 wei
    #    new_shares = 1e15 // 1e15 = 1
    with boa.env.prank(PATRICK):
        industry_contract.fund_cyfrin(1, value=to_wei(0.001, "ether"))

    # 3) Compute share price externally from public state
    net_worth = industry_contract.get_balance() - industry_contract.holding_debt()
    issued = industry_contract.issued_shares()
    share_price = net_worth // issued

    # Sanity: share price should be >= 100 ETH
    assert share_price >= to_wei(100, "ether"), "Share price should be at least 100 ETH"

    epsilon = 1000  # 1000 wei under the share price
    lost_amount = share_price - epsilon

    balance_before = industry_contract.get_balance()
    shares_before = industry_contract.get_my_shares(caller=PATRICK)

    # 4) Investor sends just under the share price -> zero shares issued, ETH accepted
    with boa.env.prank(PATRICK):
        industry_contract.fund_cyfrin(1, value=lost_amount)

    shares_after = industry_contract.get_my_shares(caller=PATRICK)
    balance_after = industry_contract.get_balance()

    # Assertions: no shares issued, and company_balance increases by lost_amount
    assert shares_after == shares_before
    assert balance_after == balance_before + lost_amount

    # Output for appeal verification
    print(f"Share Price: {share_price} Wei")
    print(f"Lost Amount: {lost_amount} Wei")
    print("CONFIRMED: Lost amount is substantial and zero shares were issued.")
```

**Verified Test Output:**

```bash
uv run mox test tests/unit/test_vulnerabilities_poc_high_loss.py -v -s

tests/unit/test_vulnerabilities_poc_high_loss.py::test_AP02_large_absolute_loss_high_share_price Cyfrin Industry deployed at 0xC6Acb7D16D51f72eAA659668F30A40d87E2E0551
Share Price: 100001000000000000000 Wei
Lost Amount: 100000999999999999000 Wei
CONFIRMED: Lost amount is substantial and zero shares were issued.
PASSED
```

## Recommended Mitigation

```diff
- share_price: uint256 = (
-     net_worth // max(self.issued_shares, 1)
-     if self.issued_shares > 0
-     else INITIAL_SHARE_PRICE
- )
+ share_price: uint256 = (
+     INITIAL_SHARE_PRICE if self.issued_shares == 0 else max(net_worth // self.issued_shares, 1)
+ )
+
+ # Ensure minimum shares or revert if below cap but rounding yields zero
+ new_shares: uint256 = msg.value // share_price
+ if new_shares == 0:
+     raise "Contribution too small for a single share!!!"
+
+ available: uint256 = self.public_shares_cap - self.issued_shares
+ if available == 0:
+     raise "Share cap reached!!!"  # reject ETH when cap is full
+ if new_shares > available:
+     new_shares = available
```

---
Cross-file or other-code mitigation check: There is no other code path that refunds ETH when investor receives `0` shares. Tests currently expect shares to be minted (see `tests/unit/test_Industry.py`), reinforcing that `0`-share issuance would break economic assumptions.
