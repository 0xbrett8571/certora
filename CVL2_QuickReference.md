# CVL 2.0 Quick Reference for Ventuals Protocol Verification

**Purpose**: Essential CVL 2.0 syntax and patterns for implementing formal verification properties

---

## 1. BASIC CVL 2.0 SYNTAX

### 1.1 Rules

```cvl
// CVL 2.0 requires 'rule' keyword
rule ruleName(method f, uint256 param) {
    env e;
    calldataarg args;

    // Pre-conditions
    require condition;

    // State before
    uint256 before = getValue();

    // Function call
    f(e, args);

    // State after
    uint256 after = getValue();

    // Assertions
    assert after >= before, "Value should not decrease";
}
```

### 1.2 Invariants

```cvl
// Basic invariant
invariant invariantName(address user)
    balanceOf(user) <= totalSupply()
    {
        preserved with (env e) {
            require e.msg.sender != currentContract;
        }
    }

// Invariant with requireInvariant
invariant dependent()
    property2() == true
    {
        preserved {
            requireInvariant independent();
        }
    }
```

### 1.3 Methods Block (CVL 2.0)

```cvl
methods {
    // External function - must include 'external' keyword
    function transfer(address, uint256) external returns(bool) envfree;

    // Internal function
    function _mint(address, uint256) internal;

    // Wildcard - applies to all contracts
    function _.balanceOf(address) external returns(uint256) envfree;

    // Summary for external function
    function _.externalCall(uint256) external => NONDET;

    // Ghost mapping to contract function
    function totalSupply() external returns(uint256) envfree;
}
```

### 1.4 Types

```cvl
// CVL types
mathint x;              // Arbitrary precision integer (no overflow)
uint256 y;              // Standard uint
address user;           // Address
bool flag;              // Boolean
env e;                  // Environment (msg, block, tx)
method f;               // Method variable for parametric rules
calldataarg args;       // Arbitrary calldata
storage initialState;   // Storage snapshot

// Struct types - must be qualified by contract name
StakingVaultManager.Withdraw withdraw;
StakingVaultManager.Batch batch;
```

---

## 2. GHOSTS AND HOOKS

### 2.1 Ghost Variables

```cvl
// Simple ghost
ghost mathint sumOfBalances {
    init_state axiom sumOfBalances == 0;
}

// Ghost mapping
ghost mapping(address => mathint) balanceSum;

// Ghost function
ghost isValidState(uint256) returns bool;
```

### 2.2 Storage Hooks

```cvl
// Sstore hook - triggers on write
hook Sstore balances[KEY address user] uint256 newBalance (uint256 oldBalance) {
    sumOfBalances = sumOfBalances + newBalance - oldBalance;
}

// Sload hook - triggers on read
hook Sload uint256 balance balances[KEY address user] {
    // Can check or record reads
}

// Hook on struct field
hook Sstore withdraws[KEY uint256 id].claimedAt uint256 newTime (uint256 oldTime) {
    // Track when withdrawals are claimed
}

// Hook on array length
hook Sstore batches.length uint256 newLen (uint256 oldLen) {
    // Track array growth
}
```

### 2.3 Opcode Hooks

```cvl
// Hook on all storage writes
hook ALL_SSTORE(uint loc, uint v) uint old_v {
    // Triggered on any storage write
}

// Hook on all storage reads
hook ALL_SLOAD(uint loc) uint v {
    // Triggered on any storage read
}
```

---

## 3. EXPRESSIONS AND OPERATORS

### 3.1 Mathematical Operations

```cvl
// Arithmetic (no overflow on mathint)
mathint sum = a + b;
mathint diff = a - b;
mathint prod = a * b;
mathint quot = a / b;
mathint mod = a % b;

// Comparisons
bool lessThan = a < b;
bool lessEq = a <= b;
bool equal = a == b;
bool notEqual = a != b;

// Logical
bool and = condition1 && condition2;
bool or = condition1 || condition2;
bool not = !condition;
bool implies = condition1 => condition2;  // Implication
bool iff = condition1 <=> condition2;     // If and only if

// Ternary
uint256 result = condition ? valueIfTrue : valueIfFalse;
```

### 3.2 Casting

```cvl
// Safe cast - adds assertion
uint256 x = assert_uint256(mathIntValue);  // Asserts value fits in uint256

// Unsafe cast - adds requirement (DANGEROUS!)
uint256 y = require_uint256(mathIntValue); // Requires value fits, ignores violations

// Explicit cast
mathint m = to_mathint(uint256Value);      // Always safe
```

### 3.3 Quantifiers

```cvl
// Universal quantifier - forall
assert forall address user. balanceOf(user) <= totalSupply();

// Existential quantifier - exists
satisfy exists address user. balanceOf(user) > 0;

// Multiple quantifiers
assert forall address user. forall address other.
    user != other => balanceOf(user) + balanceOf(other) <= totalSupply();
```

---

## 4. STATEMENTS

### 4.1 Require and Assert

```cvl
// Require - filters out models that don't satisfy
require balance > 0, "Balance must be positive";

// Assert - checks property holds
assert balanceAfter >= balanceBefore, "Balance should not decrease";

// Satisfy - checks if property is achievable
satisfy balanceOf(user) > 1000;
```

### 4.2 Function Calls

```cvl
// Call with env
deposit(e, amount);

// Call with revert handling
deposit@withrevert(e, amount);
bool didRevert = lastReverted;

// Call at specific storage state
uint256 bal = balanceOf(e, user) at initialState;
```

### 4.3 Havoc

```cvl
// Havoc with assumption
havoc x assuming x > 0 && x < 100;

// Havoc ghost assuming condition
havoc ghostVar assuming forall address a. ghostVar[a] <= totalSupply();
```

---

## 5. COMMON PATTERNS FOR VENTUALS

### 5.1 Tracking Sum of Balances

```cvl
ghost mathint sumVhypeBalances {
    init_state axiom sumVhypeBalances == 0;
}

hook Sstore vHYPE.balances[KEY address user] uint256 newBal (uint256 oldBal) {
    sumVhypeBalances = sumVhypeBalances + to_mathint(newBal) - to_mathint(oldBal);
}

invariant sumEqualsSupply()
    sumVhypeBalances == to_mathint(vHYPE.totalSupply());
```

### 5.2 Parametric Rule for State Changes

```cvl
rule noUnexpectedBalanceChange(method f, address user)
    filtered { f -> !f.isView }
{
    env e;
    calldataarg args;

    uint256 balanceBefore = balanceOf(user);

    f(e, args);

    uint256 balanceAfter = balanceOf(user);

    assert balanceAfter > balanceBefore =>
        f.selector == sig:deposit().selector ||
        f.selector == sig:transfer(address, uint256).selector,
        "Only deposit and transfer should increase balance";
}
```

### 5.3 Solvency Invariant

```cvl
invariant protocolSolvency()
    totalBalance() >= reservedForWithdrawals()
    {
        preserved with (env e) {
            require e.msg.sender != currentContract;
            requireInvariant totalSupplyBacking();
        }
    }

definition reservedForWithdrawals() returns mathint =
    to_mathint(totalHypeProcessed()) - to_mathint(totalHypeClaimed());
```

### 5.4 Exchange Rate Monotonicity

```cvl
rule exchangeRateNeverDecreases(method f)
    filtered { f -> !f.isView && f.selector != sig:applySlash(uint256,uint256).selector }
{
    env e;
    calldataarg args;

    uint256 rateBefore = exchangeRate();

    f(e, args);

    uint256 rateAfter = exchangeRate();

    assert rateAfter >= rateBefore,
        "Exchange rate should never decrease except during slash";
}
```

### 5.5 State Machine Transitions

```cvl
// Define states as definitions
definition isQueued(StakingVaultManager.Withdraw w) returns bool =
    w.batchIndex == max_uint256 && w.cancelledAt == 0 && w.claimedAt == 0;

definition isProcessed(StakingVaultManager.Withdraw w) returns bool =
    w.batchIndex != max_uint256 && w.cancelledAt == 0 && w.claimedAt == 0;

definition isCancelled(StakingVaultManager.Withdraw w) returns bool =
    w.cancelledAt != 0;

definition isClaimed(StakingVaultManager.Withdraw w) returns bool =
    w.claimedAt != 0;

// Verify only one state at a time
invariant withdrawalStateExclusive(uint256 withdrawId)
    (isQueued(getWithdraw(withdrawId)) && !isProcessed(getWithdraw(withdrawId)) &&
     !isCancelled(getWithdraw(withdrawId)) && !isClaimed(getWithdraw(withdrawId))) ||
    (!isQueued(getWithdraw(withdrawId)) && isProcessed(getWithdraw(withdrawId)) &&
     !isCancelled(getWithdraw(withdrawId)) && !isClaimed(getWithdraw(withdrawId))) ||
    (!isQueued(getWithdraw(withdrawId)) && !isProcessed(getWithdraw(withdrawId)) &&
     isCancelled(getWithdraw(withdrawId)) && !isClaimed(getWithdraw(withdrawId))) ||
    (!isQueued(getWithdraw(withdrawId)) && !isProcessed(getWithdraw(withdrawId)) &&
     !isCancelled(getWithdraw(withdrawId)) && isClaimed(getWithdraw(withdrawId)));
```

### 5.6 Time-Based Properties

```cvl
rule claimWindowEnforced(uint256 withdrawId) {
    env e;

    StakingVaultManager.Withdraw w = getWithdraw(withdrawId);
    StakingVaultManager.Batch b = getBatch(w.batchIndex);

    require w.batchIndex != max_uint256;
    require b.finalizedAt > 0;

    // Try to claim before window
    require e.block.timestamp <= b.finalizedAt + 7 days + claimWindowBuffer();

    claimWithdraw@withrevert(e, withdrawId, e.msg.sender);

    assert lastReverted, "Should not be able to claim before claim window";
}
```

### 5.7 Access Control

```cvl
rule onlyManagerCanMint(method f, address user, uint256 amount)
    filtered { f -> f.selector == sig:mint(address,uint256).selector }
{
    env e;

    require !hasRole(MANAGER_ROLE(), e.msg.sender);

    vHYPE.mint@withrevert(e, user, amount);

    assert lastReverted, "Only MANAGER should be able to mint";
}
```

---

## 6. ADVANCED FEATURES

### 6.1 Two-State Contexts

```cvl
ghost mapping(address => mathint) balancesBefore;
ghost mapping(address => mathint) balancesAfter;

// Use @old and @new in two-state context
definition balanceIncreased(address user) returns bool =
    balancesAfter@new[user] > balancesBefore@old[user];
```

### 6.2 Storage Snapshots

```cvl
rule snapshotExample() {
    env e;
    storage initialStorage = lastStorage;

    deposit(e, 100);

    uint256 balanceAfterDeposit = balanceOf(e.msg.sender);

    // Revert to initial storage
    uint256 balanceInitial = balanceOf(e.msg.sender) at initialStorage;

    assert balanceAfterDeposit > balanceInitial;
}
```

### 6.3 Method Filters

```cvl
// Filter by selector
rule onlySpecificMethods(method f)
    filtered {
        f -> f.selector == sig:deposit().selector ||
             f.selector == sig:withdraw(uint256).selector
    }
{
    // ...
}

// Filter by view/pure
rule onlyStateMutating(method f)
    filtered { f -> !f.isView }
{
    // ...
}

// Filter by contract
rule onlyVaultMethods(method f)
    filtered { f -> f.contract == stakingVault }
{
    // ...
}
```

### 6.4 Multi-Contract Verification

```cvl
using StakingVault as vault;
using VHYPE as token;
using StakingVaultManager as manager;

rule crossContractInvariant() {
    // Access multiple contracts
    uint256 vaultBalance = vault.evmBalance();
    uint256 tokenSupply = token.totalSupply();
    uint256 exchangeRate = manager.exchangeRate();

    assert to_mathint(vaultBalance) * 1e18 >= to_mathint(tokenSupply) * to_mathint(exchangeRate);
}
```

---

## 7. DEBUGGING AND OPTIMIZATION

### 7.1 Sanity Rules

```cvl
// Sanity check - should always fail
rule sanity(method f) {
    env e;
    calldataarg args;

    f(e, args);

    assert false;
}
```

### 7.2 Rule Descriptions

```cvl
rule withdrawalIntegrity(uint256 withdrawId)
    description "Ensures users can always claim their processed withdrawals"
{
    // ...
}
```

### 7.3 Timeout Prevention

```cvl
// Use filters to reduce parametric rule scope
rule limitedParametricRule(method f)
    filtered {
        f -> !f.isView &&                    // Skip view functions
             f.selector != sig:complexFunction().selector  // Skip complex functions
    }
{
    // ...
}

// Split complex assertions
rule splitRule() {
    require condition1;
    require condition2;

    // Instead of one complex assertion, split into multiple
    assert property1;
    assert property2;
}
```

---

## 8. COMMON PITFALLS TO AVOID

### 8.1 Type Casting Issues

```cvl
// WRONG - can overflow
uint256 result = userBalance + depositAmount;

// CORRECT - use mathint
mathint total = to_mathint(userBalance) + to_mathint(depositAmount);
require total <= max_uint256;
uint256 result = assert_uint256(total);
```

### 8.2 Vacuity from Over-Constraining

```cvl
// DANGEROUS - may be vacuous
require exchangeRate() > 0;
require totalSupply() > 0;
require totalBalance() > 0;
require totalBalance() / totalSupply() == exchangeRate();  // May have no solutions!

// BETTER - derive one from others
require totalSupply() > 0;
uint256 rate = exchangeRate();
require rate > 0;
// Don't add redundant constraint
```

### 8.3 Missing Struct Qualification

```cvl
// WRONG - will be treated as uninterpreted sort
Withdraw w;

// CORRECT - must qualify with contract name
StakingVaultManager.Withdraw w;
```

### 8.4 Forgetting envfree

```cvl
methods {
    // WRONG - will require env parameter in calls
    function totalSupply() external returns(uint256);

    // CORRECT - if function doesn't use msg.sender, block, etc.
    function totalSupply() external returns(uint256) envfree;
}
```

---

## 9. VENTUALS-SPECIFIC HELPERS

### 9.1 Definition Helpers

```cvl
definition MAX_UINT256() returns uint256 = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF;

definition isManagerRole(address account) returns bool =
    hasRole(MANAGER_ROLE(), account);

definition isValidBatchIndex(uint256 index) returns bool =
    index < getBatchesLength();

definition canIncreaseBalance(method f) returns bool =
    f.selector == sig:deposit().selector ||
    f.selector == sig:mint(address,uint256).selector;

definition canDecreaseBalance(method f) returns bool =
    f.selector == sig:queueWithdraw(uint256).selector ||
    f.selector == sig:burn(uint256).selector;
```

### 9.2 Ghost Tracking

```cvl
// Track total vHYPE escrowed in manager
ghost mathint totalEscrowedVhype {
    init_state axiom totalEscrowedVhype == 0;
}

hook Sstore vHYPE.balances[KEY address manager] uint256 newBal (uint256 oldBal) {
    if (manager == currentContract) {
        totalEscrowedVhype = to_mathint(newBal);
    }
}
```

---

## 10. QUICK SYNTAX CHECKLIST

- ✅ All rules start with `rule` keyword
- ✅ Methods block entries start with `function` and end with `;`
- ✅ External functions marked with `external` or `internal`
- ✅ Method literals use `sig:` prefix (e.g., `sig:transfer(address,uint256).selector`)
- ✅ Struct types qualified with contract name (e.g., `StakingVaultManager.Batch`)
- ✅ Use `mathint` for overflow-safe arithmetic
- ✅ Use `to_mathint()` for safe casting to mathint
- ✅ Use `assert_uint256()` for explicit cast assertions
- ✅ Semicolons on `using`, `import`, `invariant` statements
- ✅ `envfree` for functions that don't use env
- ✅ `@withrevert` for calls that might revert

---

**Ready to wage algorithmic warfare! 🛡️⚔️**
