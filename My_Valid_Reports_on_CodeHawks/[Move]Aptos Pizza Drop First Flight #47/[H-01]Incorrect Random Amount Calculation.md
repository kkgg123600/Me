# Incorrect Random Amount Calculation Vulnerability

## Root Cause + Impact

The random slice amount calculation is incorrect, causing users to receive significantly less APT than intended due to missing decimal scaling.

## Description

The normal behavior should be to give users a random slice between 100-500 APT as stated in the project description. APT uses 8 decimal places (Octas), so 1 APT = 100,000,000 Octas.

The specific issue is that the `get_random_slice()` function calculates amounts in the range 100-500 without proper decimal scaling, effectively giving users 100-500 Octas instead of 100-500 APT.

```move
// Root cause in the codebase
#[randomness]
entry fun get_random_slice(user_addr: address) acquires ModuleData, State {
    let state = borrow_global_mut<State>(get_resource_address());
    let time = timestamp::now_microseconds();
    let random_val = time % 401;  // 0 to 400
@>  let random_amount = 100 + random_val;  // 100-500 (should be in APT, but calculated as Octas)
@>  // Comment says: "100-500 APT (in Octas: 10^8 smallest unit)" - This is misleading!
    table::add(&mut state.users_claimed_amount, user_addr, random_amount);
}
```

## Risk

**Likelihood**: High

* This error occurs every time a user is registered
* All users will receive incorrect amounts
* The calculation is fundamentally wrong by a factor of 100,000,000

**Impact**: Critical

* Users receive 100-500 Octas instead of 100-500 APT
* This means users get 0.000001 to 0.000005 APT instead of 100-500 APT
* Complete failure of the airdrop mechanism
* Users lose 99.9999% of their intended rewards
* Project reputation damage due to failed token distribution
* Potential legal issues due to misleading reward amounts

### Proof of Concept (PoC)

To practically demonstrate this critical calculation error, a new test case named `test_incorrect_random_amount_calculation_vulnerability` has been created. This test simulates the full lifecycle of funding the contract and registering a user, then asserts that the assigned airdrop amount is in raw Octas (100-500) instead of the properly scaled APT value.

**Setup**

Add the following test case to the end of the `sources/pizza_drop.move` file, alongside the other existing test functions.

**Test Code**

```move
    // WORKING TEST WITH APTOS COIN
    #[test(deployer = @pizza_drop, user = @0x123, framework = @0x1)]
    fun test_incorrect_random_amount_calculation_vulnerability(deployer: &signer, user: &signer, framework: &signer) acquires State, ModuleData {
        use aptos_framework::account;
        use aptos_framework::timestamp;
        use aptos_framework::aptos_coin;

        // Initialize timestamp and APT for testing
        timestamp::set_time_has_started_for_testing(framework);
        let (burn_cap, mint_cap) = aptos_coin::initialize_for_test(framework);

        // Create accounts
        account::create_account_for_test(@pizza_drop);
        account::create_account_for_test(signer::address_of(user));

        debug::print(&b"=== TESTING INCORRECT RANDOM AMOUNT CALCULATION VULNERABILITY ===");

        // Initialize the pizza drop module
        init_module(deployer);
        debug::print(&b"Pizza drop module initialized");

        // According to README.md, users should receive 100-500 APT
        // 1 APT = 100,000,000 Octas (10^8)
        // So minimum should be: 100 * 100,000,000 = 10,000,000,000 Octas
        // And maximum should be: 500 * 100,000,000 = 50,000,000,000 Octas
        
        let expected_min_octas = 100 * 100000000; // 10,000,000,000 Octas (100 APT)
        let expected_max_octas = 500 * 100000000; // 50,000,000,000 Octas (500 APT)
        
        debug::print(&b"Expected minimum amount (100 APT in Octas): ");
        debug::print(&expected_min_octas);
        debug::print(&b"Expected maximum amount (500 APT in Octas): ");
        debug::print(&expected_max_octas);

        // Fund the contract with enough APT to cover maximum possible claims
        let funding_amount = 60000000000; // 600 APT in Octas
        let deployer_coins = coin::mint<AptosCoin>(funding_amount, &mint_cap);
        coin::register<AptosCoin>(deployer);
        coin::deposit<AptosCoin>(@pizza_drop, deployer_coins);

        let deployer_balance = coin::balance<AptosCoin>(@pizza_drop);
        debug::print(&b"Contract funded with (in Octas): ");
        debug::print(&deployer_balance);

        // Fund the contract through the proper function
        fund_pizza_drop(deployer, funding_amount);
        
        let contract_balance_after_funding = get_pizza_pool_balance();
        debug::print(&b"Contract balance after funding: ");
        debug::print(&contract_balance_after_funding);

        // Register user and check the assigned amount
        register_pizza_lover(deployer, signer::address_of(user));
        debug::print(&b"User registered successfully");
        
        // Get the assigned amount for the user
        let assigned_amount = get_claimed_amount(signer::address_of(user));
        debug::print(&b"Assigned amount to user (in Octas): ");
        debug::print(&assigned_amount);
        
        // Convert to APT for readability
        let assigned_amount_in_apt = assigned_amount / 100000000;
        debug::print(&b"Assigned amount to user (in APT): ");
        debug::print(&assigned_amount_in_apt);
        
        // Check if the vulnerability exists
        // If the amount is between 100-500 (Octas), then the vulnerability is confirmed
        // If the amount is between 10,000,000,000-50,000,000,000 (Octas), then it's correct
        
        debug::print(&b"=== VULNERABILITY ANALYSIS ===");
        
        if (assigned_amount >= 100 && assigned_amount <= 500) {
            debug::print(&b"VULNERABILITY CONFIRMED: Amount is in Octas range (100-500)");
            debug::print(&b"User will receive only 0.000001-0.000005 APT instead of 100-500 APT");
            debug::print(&b"This represents a 99.9999% loss of intended rewards");
            
            // Calculate the actual APT amount user will receive
            let actual_apt_received = assigned_amount / 100000000;
            debug::print(&b"Actual APT user will receive: ");
            debug::print(&actual_apt_received);
            
            // This should be 0 due to integer division
            assert!(actual_apt_received == 0, 999); // User gets 0 APT due to rounding
            
        } else if (assigned_amount >= expected_min_octas && assigned_amount <= expected_max_octas) {
            debug::print(&b"NO VULNERABILITY: Amount is correctly scaled to APT");
            debug::print(&b"User will receive the intended 100-500 APT");
        } else {
            debug::print(&b"UNEXPECTED: Amount is outside expected ranges");
            debug::print(&b"This indicates a different issue with the calculation");
        };
        
        debug::print(&b"=== CONCLUSION ===");
        debug::print(&b"The vulnerability report claims users get 100-500 Octas instead of 100-500 APT");
        debug::print(&b"Based on the assigned amount, we can verify this claim");
        
        coin::destroy_burn_cap(burn_cap);
        coin::destroy_mint_cap(mint_cap);
    }
```

**Execution Command**

Compile and run the specific test using the following command from the project's root directory:

```bash
aptos move test --filter test_incorrect_random_amount_calculation_vulnerability
```

**Test Results**

The test will pass, and the debug output will confirm the vulnerability by showing the incorrectly calculated amount.

```bash

Running Move unit tests
[debug] 0x3d3d3d2054455354494e4720494e434f52524543542052414e444f4d20414d4f554e542043414c43554c4154494f4e2056554c4e45524142494c495459203d3d3d
[debug] 0x50697a7a612064726f70206d6f64756c6520696e697469616c697a6564
[debug] 0x4578706563746564206d696e696d756d20616d6f756e7420283130302041505420696e204f63746173293a20
[debug] 10000000000
[debug] 0x4578706563746564206d6178696d756d20616d6f756e7420283530302041505420696e204f63746173293a20
[debug] 50000000000
[debug] 0x436f6e74726163742066756e64656420776974682028696e204f63746173293a20
[debug] 60000000000
[debug] 0x436f6e74726163742062616c616e63652061667465722066756e64696e673a20
[debug] 60000000000
[debug] 0x557365722072656769737465726564207375636365737366756c6c79
[debug] 0x41737369676e656420616d6f756e7420746f20757365722028696e204f63746173293a20
[debug] 100
[debug] 0x41737369676e656420616d6f756e7420746f20757365722028696e20415054293a20
[debug] 0
[debug] 0x3d3d3d2056554c4e45524142494c49545920414e414c59534953203d3d3d
[debug] 0x56554c4e45524142494c49545920434f4e4649524d45443a20416d6f756e7420697320696e204f637461732072616e676520283130302d35303029
[debug] 0x557365722077696c6c2072656365697665206f6e6c7920302e3030303030312d302e3030303030352041505420696e7374656164206f66203130302d35303020415054
[debug] 0x5468697320726570726573656e747320612039392e3939393925206c6f7373206f6620696e74656e6465642072657761726473
[debug] 0x41637475616c2041505420757365722077696c6c20726563656976653a20
[debug] 0
[debug] 0x3d3d3d20434f4e434c5553494f4e203d3d3d
[debug] 0x5468652076756c6e65726162696c697479207265706f727420636c61696d7320757365727320676574203130302d353030204f6374617320696e7374656164206f66203130302d35303020415054
[debug] 0x4261736564206f6e207468652061737369676e656420616d6f756e742c2077652063616e20766572696679207468697320636c61696d
[ PASS    ] 0xccc::airdrop::test_incorrect_random_amount_calculation_vulnerability
Test result: OK. Total tests: 1; passed: 1; failed: 0
{
  "Result": "Success"
}
```

The output clearly shows the `Assigned amount to user (in Octas)` is `100`, which is within the 100-500 raw value range, instead of the expected range starting from `10,000,000,000` Octas (100 APT). This confirms the vulnerability and its critical impact.

## Recommended Mitigation

```diff
  #[randomness]
  entry fun get_random_slice(user_addr: address) acquires ModuleData, State {
      let state = borrow_global_mut<State>(get_resource_address());
      let time = timestamp::now_microseconds();
      let random_val = time % 401;  // 0 to 400
-     let random_amount = 100 + random_val;  // 100-500 (incorrect - in Octas)
+     let random_amount = (100 + random_val) * 100000000;  // 100-500 APT in Octas
      table::add(&mut state.users_claimed_amount, user_addr, random_amount);
  }

// Alternative approach using constants for clarity:
+ const APT_DECIMALS: u64 = 100000000;  // 10^8 Octas per APT
+ const MIN_SLICE_APT: u64 = 100;
+ const MAX_SLICE_APT: u64 = 500;

  #[randomness]
  entry fun get_random_slice(user_addr: address) acquires ModuleData, State {
      let state = borrow_global_mut<State>(get_resource_address());
      let time = timestamp::now_microseconds();
-     let random_val = time % 401;
-     let random_amount = 100 + random_val;
+     let random_val = time % (MAX_SLICE_APT - MIN_SLICE_APT + 1);
+     let random_amount_apt = MIN_SLICE_APT + random_val;
+     let random_amount = random_amount_apt * APT_DECIMALS;
      table::add(&mut state.users_claimed_amount, user_addr, random_amount);
  }
```

This mitigation ensures that:

1. Users receive the correct amount in APT as intended
2. The random range is properly scaled to Octas
3. Constants make the code more readable and maintainable
4. The airdrop functions as designed in the project specification
