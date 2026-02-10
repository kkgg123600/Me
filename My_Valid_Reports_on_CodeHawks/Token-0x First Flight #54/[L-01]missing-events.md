# Mint/Burn do not emit `Transfer` events

## Root + Impact

Absence of `Transfer` events in `_mint` and `_burn` breaks ERC-20 event semantics. Indexers, analytics, and protocols relying on events cannot track supply changes, leading to desynchronized accounting and potential misreporting.

## Description

- Normal behavior: `_mint` should emit `Transfer(address(0), to, value)`; `_burn` should emit `Transfer(from, address(0), value)`.
- Issue: Both functions update storage without emitting events.

```solidity
// src/helpers/ERC20Internals.sol
134: function _mint(address account, uint256 value) internal {
...
146:     let supply := sload(supplySlot)
147:     sstore(supplySlot, add(supply, value))
...
154:     sstore(accountBalanceSlot, add(accountBalance, value))
// @> No log3 Transfer event emitted for mint

158: function _burn(address account, uint256 value) internal {
...
171:     sstore(supplySlot, sub(supply, value))
...
178:     sstore(accountBalanceSlot, sub(accountBalance, value))
// @> No log3 Transfer event emitted for burn
```

## Risk

**Likelihood**:

- Occurs on every mint or burn executed by derived tokens.

**Impact**:

- Off-chain systems fail to detect mints/burns; protocols that rely on events for supply tracking may misbehave.

## Proof of Concept

```solidity
// test/PocMissingEvents.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {Vm} from "forge-std/Vm.sol";
import {ERC20} from "../src/ERC20.sol";

contract TestTokenHarness is ERC20 {
    constructor() ERC20("Test", "TST") {}

    function exposedMint(address to, uint256 value) external {
        _mint(to, value);
    }

    function exposedBurn(address from, uint256 value) external {
        _burn(from, value);
    }
}

contract PocMissingEvents is Test {
    TestTokenHarness private token;
    address private to = address(0xA0);

    function setUp() public {
        token = new TestTokenHarness();
    }

    function test_MintDoesNotEmitTransferEvent() public {
        vm.recordLogs();
        token.exposedMint(to, 123);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bool foundTransfer = false;
        bytes32 transferTopic = keccak256("Transfer(address,address,uint256)");
        for (uint256 i = 0; i < logs.length; i++) {
            if (logs[i].topics.length > 0 && logs[i].topics[0] == transferTopic) {
                foundTransfer = true;
                break;
            }
        }
        assertFalse(foundTransfer, "expected no Transfer event on mint");
    }

    function test_BurnDoesNotEmitTransferEvent() public {
        token.exposedMint(to, 123);
        vm.recordLogs();
        token.exposedBurn(to, 100);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bool foundTransfer = false;
        bytes32 transferTopic = keccak256("Transfer(address,address,uint256)");
        for (uint256 i = 0; i < logs.length; i++) {
            if (logs[i].topics.length > 0 && logs[i].topics[0] == transferTopic) {
                foundTransfer = true;
                break;
            }
        }
        assertFalse(foundTransfer, "expected no Transfer event on burn");
    }
}
```

## Test result

```bash
➜  2025-12-token-0x git:(main) ✗ forge test --match-path  test/PocMissingEvents.t.sol -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 2 tests for test/PocMissingEvents.t.sol:PocMissingEvents
[PASS] test_BurnDoesNotEmitTransferEvent() (gas: 59216)
[PASS] test_MintDoesNotEmitTransferEvent() (gas: 56850)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 3.66ms (1.08ms CPU time)

Ran 1 test suite in 29.46ms (3.66ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

## Recommended Mitigation

```diff
 function _mint(address account, uint256 value) internal {
     assembly ("memory-safe") {
         // existing updates
         sstore(supplySlot, add(supply, value))
         sstore(accountBalanceSlot, add(accountBalance, value))
+        mstore(ptr, value)
+        log3(ptr, 0x20, 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef, 0, account)
     }
 }

 function _burn(address account, uint256 value) internal {
     assembly ("memory-safe") {
         // existing checks and updates
         sstore(supplySlot, sub(supply, value))
         sstore(accountBalanceSlot, sub(accountBalance, value))
+        mstore(ptr, value)
+        log3(ptr, 0x20, 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef, account, 0)
     }
 }
```
