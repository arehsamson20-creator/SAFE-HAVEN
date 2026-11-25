# Ledger-Based Withdrawal Check: Implementation Notes

## Overview

This document provides technical details about the ledger-based deposit withdrawal check implementation in the SAFE-HAVEN contract.

## Requirement

Withdrawals of ledger-based deposits must verify that:
```
current_ledger >= unlock_ledger
```

If this condition is not met, the withdrawal must fail with `FundsStillLocked` error.

## Implementation Location

### File: `contracts/safe-haven/src/contract.rs`

#### 1. Function: `withdraw()` (Line 1355)
```rust
// Try ledger-based deposit.
if let Some(entry) = storage::get_deposit_by_ledger_readonly(&env, &depositor, deposit_id) {
    let current_ledger = env.ledger().sequence();
    if current_ledger < entry.unlock_ledger {  // ← Check here
        return Err(VaultError::FundsStillLocked);
    }
    // ... proceed with withdrawal
}
```

#### 2. Function: `withdraw_to()` (Line 1464)
```rust
// Try ledger-based deposit.
if let Some(entry) = storage::get_deposit_by_ledger_readonly(&env, &depositor, deposit_id) {
    let current_ledger = env.ledger().sequence();
    if current_ledger < entry.unlock_ledger {  // ← Check here
        return Err(VaultError::FundsStillLocked);
    }
    // ... proceed with withdrawal
}
```

## Key Details

### Error Code
- **Error:** `VaultError::FundsStillLocked` (code 4)
- **Defined in:** `contracts/safe-haven/src/errors.rs`
- **Used for:** Both timestamp-based and ledger-based locked deposits

### Ledger Entry Structure
```rust
pub struct LedgerVaultEntry {
    pub token: Address,
    pub amount: i128,
    pub unlock_ledger: u32,  // ← Target ledger for unlocking
    pub depositor: Address,
    pub penalty_bps: u32,
}
```

### Check Logic
The check uses the **less-than** operator for rejection:
- **Rejects if:** `current_ledger < unlock_ledger` → `FundsStillLocked`
- **Allows if:** `current_ledger >= unlock_ledger` → withdrawal proceeds

This is a safe implementation because:
1. It prevents withdrawal at off-by-one (when current is exactly 1 less than target)
2. It allows withdrawal at exactly the unlock ledger (critical edge case)
3. It allows withdrawal after any point past the unlock ledger

### Timing Estimation
The contract provides time estimation for ledger-based deposits:
- Estimated seconds = `(unlock_ledger - current_ledger) × 5`
- Assumes 5 seconds per ledger close (average on Stellar Testnet/Public Network)
- Note: This is an estimate. Actual ledger close times vary ±1-2 seconds.
- The actual withdrawal check is exact and will work correctly regardless of timing variance

## Test Coverage

### New Tests (6 added)

#### 1. Boundary: One Ledger Before
```
test_withdraw_ledger_deposit_one_before_unlock_fails
```
- **Setup:** unlock_ledger = current + 10
- **Action:** Advance to current + 9 (one before unlock)
- **Expected:** Withdrawal fails with `FundsStillLocked`
- **Verifies:** The `-1` boundary condition

#### 2. Boundary: Exact Unlock Ledger
```
test_withdraw_ledger_deposit_at_exact_unlock_succeeds
```
- **Setup:** unlock_ledger = current + 10
- **Action:** Advance to current + 10 (exactly at unlock)
- **Expected:** Withdrawal succeeds, tokens returned
- **Verifies:** The critical edge case at exact unlock

#### 3. Large Values: Many Ledgers After
```
test_withdraw_ledger_deposit_many_ledgers_after_unlock_succeeds
```
- **Setup:** unlock_ledger = current + 5
- **Action:** Advance by 500 ledgers (far past unlock)
- **Expected:** Withdrawal succeeds
- **Verifies:** Works for large current_ledger values

#### 4. withdraw_to at Exact Unlock
```
test_withdraw_to_ledger_deposit_at_exact_unlock_succeeds
```
- **Setup:** unlock_ledger = current + 8
- **Action:** withdraw_to at exactly unlock_ledger
- **Expected:** Funds transferred to recipient, withdrawal succeeds
- **Verifies:** `withdraw_to` enforces same check

#### 5. withdraw_to Before Unlock
```
test_withdraw_to_ledger_deposit_before_unlock_fails
```
- **Setup:** unlock_ledger = current + 8
- **Action:** withdraw_to one ledger before unlock
- **Expected:** Withdrawal fails with `FundsStillLocked`
- **Verifies:** `withdraw_to` rejects early withdrawal

#### 6. Multiple Deposits, Independent Checks
```
test_multiple_ledger_deposits_independent_checks
```
- **Setup:** Three deposits with unlock at +5, +15, +25
- **Actions:** Progressive advancement and withdrawal attempts
- **Expected:** Each deposit unlocks only after its own unlock_ledger
- **Verifies:** Independent checking per deposit

### Related Existing Tests

These tests also verify ledger withdrawal logic:

- `test_deposit_by_ledger_withdraw_lifecycle` - Full lifecycle including exact unlock
- `test_withdraw_to_ledger_deposit_succeeds` - Withdrawal to recipient (at past unlock)
- `test_withdraw_to_ledger_deposit_still_locked_fails` - Rejection while locked
- `test_cancel_ledger_deposit_zero_penalty` - Cancel before unlock
- `test_cancel_ledger_deposit_with_penalty` - Cancel with penalty
- `test_cancel_ledger_deposit_after_unlock_fails` - Cancel rejects after unlock
- `test_cancel_ledger_deposit_not_found_fails` - Deposit not found error
- `test_emergency_withdraw_ledger_deposit_succeeds` - Admin emergency withdraw
- `test_emergency_withdraw_ledger_deposit_non_admin_fails` - Admin permission check
- `test_emergency_withdraw_ledger_deposit_not_found` - Deposit not found error
- `test_emergency_withdraw_ledger_based_succeeds` - Emergency withdraw variant
- `test_emergency_withdraw_ledger_based_non_admin_fails` - Admin check variant
- `test_emergency_withdraw_ledger_based_not_found` - Deposit not found variant

## Helper Function

The tests use a helper to advance the ledger:

```rust
fn advance_ledger(env: &Env, ledgers: u32) {
    env.ledger().set(LedgerInfo {
        timestamp: env.ledger().timestamp(),
        protocol_version: env.ledger().protocol_version(),
        sequence_number: env.ledger().sequence() + ledgers,  // ← Advances sequence
        network_id: Default::default(),
        base_reserve: 10,
        min_temp_entry_ttl: 16,
        min_persistent_entry_ttl: 4096,
        max_entry_ttl: 33_000_000,
    });
}
```

This simulates ledger progression in tests.

## Validation Logic Flow

```
User calls withdraw(&depositor, deposit_id)
    ↓
Contract retrieves LedgerVaultEntry
    ↓
Get current_ledger = env.ledger().sequence()
    ↓
CHECK: Is current_ledger < entry.unlock_ledger?
    ├─ YES → Return FundsStillLocked error ✗
    └─ NO → Proceed to withdrawal ✓
        ↓
    Remove deposit from storage
    Transfer tokens to depositor
    Emit withdrawal event
    Clean up associated data
    Return Ok(())
```

## Edge Cases Covered

| Case | Current Ledger | Unlock Ledger | Result | Test |
|------|---|---|---|---|
| Before unlock | X | X+1 | ✗ Locked | `test_withdraw_ledger_deposit_one_before_unlock_fails` |
| Exactly at unlock | X | X | ✓ Success | `test_withdraw_ledger_deposit_at_exact_unlock_succeeds` |
| After unlock | X | X-1 | ✓ Success | `test_withdraw_ledger_deposit_many_ledgers_after_unlock_succeeds` |
| Far past unlock | X | X-500 | ✓ Success | `test_withdraw_ledger_deposit_many_ledgers_after_unlock_succeeds` |
| withdraw_to at unlock | X | X | ✓ Success | `test_withdraw_to_ledger_deposit_at_exact_unlock_succeeds` |
| withdraw_to before | X | X+1 | ✗ Locked | `test_withdraw_to_ledger_deposit_before_unlock_fails` |
| Multiple deposits | Varies | Varies | Each independent | `test_multiple_ledger_deposits_independent_checks` |

## Consistency

The check is consistent across:
- ✅ `withdraw()` function
- ✅ `withdraw_to()` function
- ✅ Both use same error code (`FundsStillLocked`)
- ✅ Both have comprehensive test coverage

## Related Features

This check works in conjunction with:

1. **Deposit Validation** - `deposit_by_ledger` enforces minimum gap:
   - `unlock_ledger > current_ledger + MIN_LOCK_LEDGERS`
   - Prevents "already unlocked" deposits

2. **Time Estimation** - `time_remaining` provides ledger-based estimates:
   - Returns `(unlock_ledger - current_ledger) × 5` seconds
   - Useful for UI display and rough scheduling

3. **Cancel Deposit** - `cancel_deposit` works before unlock:
   - Can only cancel before unlock_ledger is reached
   - After unlock, must use withdraw instead

4. **Emergency Withdraw** - Admin override:
   - Bypasses the ledger check
   - Used by administrators in emergency situations

## Security Considerations

✅ **Safe Implementation:**
- Check occurs before any state modification
- Uses appropriate error type (`FundsStillLocked`)
- Applied consistently in both withdrawal functions
- No possibility of off-by-one allowing early withdrawal
- Clear and straightforward logic (easy to audit)

✅ **Prevention of Issues:**
- Cannot withdraw before unlock_ledger
- Cannot re-enter and double-withdraw (state removed)
- Cannot bypass check via `withdraw_to` (same logic applied)
- Cannot withdraw with wrong ledger (each deposit has own target)

---

**Last Updated:** 2024-11-25  
**Implementation Status:** Complete ✅  
**Test Coverage:** 6 new tests + 11+ existing related tests
