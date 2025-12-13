# CATEGORIZING PROPERTIES FOR FORMAL VERIFICATION

## INTRODUCTION

To break a protocol using Formal Verification (FV), we must define the "Laws of Physics" that the system is supposed to obey. If the Prover can find a path that breaks these laws, we have found an exploit.

We split properties into 4 main types, mapped directly to Certora CVL concepts:

1.  **Valid States** (Invariants)
2.  **State Transitions** (Rules)
3.  **System-Level Properties** (Ghosts)
4.  **Threat Model Properties** (Anti-Exploit Logic)

> **The Prover Mindset:**
> You are not testing if the code "works." You are defining constraints. If you write a property that says "User funds can NEVER decrease," and the Prover finds a trace where they do (e.g., via a reentrancy bug you missed), you have found a critical vulnerability.

-----

## 1\. VALID STATES (CVL: `invariant`)

**Concept:** The "Laws of Existence." These are conditions that must be true for **every single state** of the blockchain, before and after every transaction.

**What to verify:**

  * **Single-Variable Constraints:** "`isPaused` can only be true or false."
  * **Relational Constraints:** "`loanAmount` must always be less than or equal to `collateralValue` \* `LTV`."
  * **State Exclusivity:** "A user cannot be both `blacklisted` and `whitelisted` at the same time."

**Example Logic:**

```solidity
invariant solvency_check()
    contract.balance >= total_deposits
```

-----

## 2\. STATE TRANSITIONS (CVL: `rule`)

**Concept:** The "Laws of Motion." These verify how variables are allowed to change from one state to the next. We compare `current` state vs `next` state.

**What to verify:**

  * **Monotonicity:** "A `nonce` can only increase."
  * **Access Control:** "If `msg.sender` is not the Owner, the `config` variable must remain `unchanged`."
  * **Zero-Sum:** "If User A's balance increases, User B's balance must decrease, OR `totalSupply` must increase."

**Example Logic:**

```solidity
rule nonce_increases() {
    uint256 nonce_before = currentContract.nonce;
    
    method f;
    env e;
    calldataarg args;
    f(e, args); // Call any function
    
    assert currentContract.nonce > nonce_before;
}
```

-----

## 3\. HIGH-LEVEL SYSTEM PROPERTIES (CVL: `ghosts`)

**Concept:** The "Meta-Accounting." Often, the contract does not store the global data you need to check (e.g., it tracks individual balances but not the "sum of all user balances"). We use **Ghosts** (auxiliary variables) to track this "hidden" state.

**What to verify:**

  * **Global Solvency:** "The sum of all user balances (tracked by ghost) must equal the token balance of the contract."
  * **Conservation of Funds:** "The total supply of internal tokens should only change when `deposit` or `withdraw` is explicitly called."

**Example Logic:**

  * **Ghost:** Update `ghost_sum` whenever `deposit()` or `withdraw()` is called.
  * **Invariant:** `assert ghost_sum == token.balanceOf(address(this))`

-----

## 4\. THREAT MODEL PROPERTIES (The "Anti-Exploit")

This is the offensive security section. Instead of asking "Does this work?", we pick a specific **Impact** (Theft, Griefing) and write a property that **forbids** it.

### A. Impact: Direct Theft of Funds / Insolvency

**Goal:** Prove funds cannot leave the contract without authorization.

  * **Property: Fund Retention**
      * *Logic:* "For any function `f`, if `f` is NOT `withdraw`, then `contract.balance` must remain `>=` `old(contract.balance)`."
      * *Catches:* Hidden transfers, reentrancy drains, logic errors in fee calculations.
  * **Property: Spending Power Safety**
      * *Logic:* "The `allowance` of this contract to any spender must be `0` unless specific `approve()` logic is triggered."
      * *Catches:* Unauthorized `approve` calls, arbitrary external calls.

### B. Impact: Manipulation of Governance

**Goal:** Prove voting power cannot be flash-loaned or faked.

  * **Property: Voting Power Stability**
      * *Logic:* "A user's voting power in block `N` must depend ONLY on their balance in block `N-1` (Checkpointing)."
      * *Catches:* Flash loan attacks where an attacker borrows tokens, votes, and repays in the same block.
  * **Property: Proposal Integrity**
      * *Logic:* "If a proposal is `Executed`, it must have been `Queued` for at least `timelock_delay` seconds."
      * *Catches:* Bypassing timelocks, privilege escalation.

### C. Impact: Griefing & Freezing

**Goal:** Prove that the system cannot be locked up by a malicious user.

  * **Property: Always Liquidatable**
      * *Logic:* "If `health_factor < threshold`, the `liquidate()` function MUST NOT revert."
      * *Catches:* DoS attacks where a user makes their account un-liquidatable (e.g., by causing a revert in the hook).
  * **Property: Unbounded Operation Prevention**
      * *Logic:* (Conceptual) "The gas cost of `distributeRewards` should not depend on the number of users."
      * *Catches:* Block stuffing, array-loop DoS.

-----

### SUMMARY CHECKLIST: The "Interrogation" Method

When you look at a new contract, fill in these blanks to generate your property list:

1.  **The Treasure:** What variable holds the money? $\rightarrow$ Write a **Solvency Invariant**.
2.  **The Keys:** Who can move the money? $\rightarrow$ Write **Access Control Rules**.
3.  **The Clock:** Does time matter (vesting, voting)? $\rightarrow$ Write **Monotonicity Rules** for timestamps/blocks.
4.  **The Attack:** If I wanted to steal, I would try to decrement my balance *after* withdrawing. $\rightarrow$ Write a **State Transition Rule** proving `balance` updates *before* or *during* the transfer.
5.  **The Ghost:** Is there any global state I need to track? $\rightarrow$ Define **Ghost Variables** to track totals.
