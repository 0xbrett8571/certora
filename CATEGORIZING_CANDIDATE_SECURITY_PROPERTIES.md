# CATEGORIZING CANDIDATE SECURITY PROPERTIES

*(Pre-Spec, Syntax-Free Phase)*

## Purpose

This phase enumerates **candidate security properties** that may later become invariants or rules.
At this stage, we **do not** decide:

* whether the property is an invariant or a rule
* how it is encoded in CVL
* what assumptions or constraints are required

We are only answering:

> **“If this property were violated, would it cause real harm?”**

---
## Integration with Workflow

| Phase | Document | What Happens |
|-------|----------|--------------|
| Phase 0 / -1 | `{contract}_spec_authoring.md` | Execution reality mapped |
| **Phase 2** | **This document** | **Properties discovered & categorized** |
| Phase 2.5 | `{contract}_spec_authoring.md` | INVARIANT vs RULE decision |
| Phase 3.5 | `{contract}_causal_validation.md` | Mutation paths validated |

> ⚠️ **Prerequisite:** Phase 0 and Phase -1 MUST be complete before using this document.
> You need the ownership table to know which truths each property depends on.

---

## Property Documentation Template

For EACH candidate property, document:

```markdown
### [ID]. [Short Name]
**Plain English:** [One sentence describing the law]
**Impact if Violated:** [Theft / Insolvency / Privilege Escalation / Griefing / Protocol Corruption]
**Category:** [Valid State / State Transition / System-Level / Threat-Driven]
**Variables Involved:** [List storage variables and their owners]
**External Truths Needed:** [List any external contract state this depends on]
**Aggregate/History Required?:** [Yes/No - flags if ghost may be needed later]
```

> 📌 The "External Truths Needed" field connects back to your Phase -1 ownership table.

---
## 1. VALID STATES — “Laws of Existence”

**Concept:**
These describe states that should **never exist** at any point in execution.

If the system can ever *be* in such a state, the protocol is already broken.

**Candidate questions to ask:**

* Can the contract be insolvent?
* Can two mutually exclusive flags be true at once?
* Can a user’s position exist in a logically impossible configuration?
* Can an enum variable hold an invalid value?
* Can a relationship between two variables be violated?

**Examples (Conceptual, Not Formal):**

* "The protocol must never owe more assets than it holds."
* "A loan must never be undercollateralized outside liquidation."
* "An account must not be both frozen and active simultaneously."
* "State enum must always be 0, 1, or 2."

**Likely Outcome in Phase 2.5:** Often becomes INVARIANT

> These properties describe **what the world is allowed to look like**, not how it changes.

---

## 2. STATE TRANSITIONS — “Laws of Motion”

**Concept:**
These describe **how state is allowed to change** between two moments in time.

They do **not** assert what is always true — they assert **what must not happen during a transition**.

**Candidate questions to ask:**

* What variables should only increase or only decrease?
* What state changes must be tied to specific actions?
* What must remain unchanged if the caller lacks authority?
* What transitions between states are forbidden?
* What is the expected effect of calling function X?

**Examples (Conceptual):**

* "A user's debt can only increase if they explicitly borrow."
* "Configuration values must not change when called by a non-admin."
* "A nonce must never decrease or repeat."
* "State can only go 0→1→2, never backward."
* "Calling deposit() must increase the user's balance."

**Likely Outcome in Phase 2.5:** Often becomes RULE

> These properties describe **movement**, not existence.

---

## 3. SYSTEM-LEVEL PROPERTIES — “Global Accounting Truths”

**Concept:**
Some security properties depend on **aggregate or historical truth** that the contract does not explicitly store.

These properties are about **system-wide conservation**, not individual variables.

**Candidate questions to ask:**

* Is there value conservation across the system?
* Does supply only change through explicit mechanisms?
* Does reward distribution depend on real input (time, deposits)?
* Does the sum of all user balances equal the total?
* Are there cross-user relationships that must hold?

**Examples (Conceptual):**

* "Total issued claims must correspond to real assets."
* "Rewards must not appear without passage of time or contribution."
* "Global balances must reconcile with external token balances."
* "Sum of all individual locked shares must equal total locked shares."

**🚨 Ghost Flag:** Properties in this category often require **ghosts** for aggregation.
Mark "Aggregate/History Required?: Yes" in your property template.

**Likely Outcome in Phase 2.5:** Often becomes INVARIANT (but requires ghost infrastructure)

> These properties often require *auxiliary tracking*, but that decision comes later.

---

## 4. THREAT-DRIVEN PROPERTIES — “Anti-Exploit Laws”

**Concept:**
Instead of starting from code structure, start from **attacker impact**.

Ask:

> *“If I were malicious, what outcome would I want?”*

Then forbid that outcome.

---

### A. Theft / Insolvency

**Candidate questions:**

* Can funds leave without authorization?
* Can accounting be inflated temporarily?
* Can value be created from nothing?

**Examples:**

* “Funds must not leave unless the protocol explicitly allows it.”
* “User balances must not increase without corresponding backing.”

---

### B. Privilege Escalation / Governance Abuse

**Candidate questions:**

* Can voting power be faked?
* Can time delays be bypassed?
* Can privileged actions be triggered early?

**Examples:**

* “Execution must respect governance delays.”
* “Voting power must reflect real ownership, not flash liquidity.”

---

### C. Griefing / Freezing

**Candidate questions:**

* Can a malicious user permanently block progress?
* Can liquidation or withdrawal be made impossible?
* Can gas usage be weaponized?

**Examples:**

* “Unhealthy positions must always be liquidatable.”
* “No user input should permanently lock funds.”

---

## Important Exclusion Rule (Threat Model)

If a candidate property **only fails because a trusted role behaves maliciously**, it must be explicitly marked:

> **OUT OF SCOPE — Trusted Role Behavior**

Do **not** discard it silently.

**Examples of trusted role assumptions:**
* Admin doesn't set malicious parameters
* Governance doesn't pass malicious proposals
* Oracle provides accurate prices
* Bridge relays correct messages

> 📌 **Document WHY it's out of scope.** This becomes part of your threat model documentation.

---

## External Dependency Warning

🚨 **CRITICAL:** Properties that depend on external contract state need special handling.

If your property references:
* ERC20 balances held by another contract
* Oracle prices
* AMM reserves
* Lending protocol state
* Any state owned by a contract you don't control

Then mark it clearly:

> **External Truth Dependency: [Contract] owns [variable]**

This connects to your **Phase -1 Interaction Ownership Table**.

**Why this matters:**
* External state can change unexpectedly (via other callers)
* HAVOC can make these properties appear violated
* May need DISPATCHER or explicit modeling
* Often becomes a RULE rather than INVARIANT in Phase 2.5

---

## Output of This Phase

The output is a **plain-English list of candidate laws**, each with:

* A unique ID (e.g., A1, B2, C1)
* A short name
* A one-sentence description
* The category (Valid State / State Transition / System-Level / Threat-Driven)
* The impact if violated
* Variables involved and their owners
* External truths needed (if any)
* Whether aggregation/history tracking is needed

**No CVL. No syntax. No constraints.**

---

## Example Output Format

```markdown
### A1. ETH Solvency
**Plain English:** The ETH balance of the contract must always be >= total claimable ETH owed to users.
**Impact if Violated:** Insolvency - users cannot withdraw
**Category:** Valid State
**Variables Involved:** 
  - `address(this).balance` (owned by EVM)
  - `_accounting.totalClaimableETH` (owned by this contract)
**External Truths Needed:** None (ETH balance is native)
**Aggregate/History Required?:** No

### C1. Total Equals Sum
**Plain English:** Total locked shares must equal sum of all individual user locked shares.
**Impact if Violated:** Accounting corruption
**Category:** System-Level
**Variables Involved:**
  - `_accounting.stETHTotals.lockedShares` (owned by this contract)
  - `_accounting.assets[user].stETHLockedShares` (owned by this contract, per-user)
**External Truths Needed:** None
**Aggregate/History Required?:** Yes (sum over mapping requires ghost)
```

---

## Mental Check (Before Moving On)

You are done with this phase **only if**:

* Every item can be explained to a non-CVL auditor
* Nothing mentions `invariant`, `rule`, `require`, `assert`, or `ghost`
* Each property clearly maps to a *real exploit if broken*
* Each property has a unique ID for tracking
* External dependencies are explicitly noted
* Aggregate requirements are flagged

---

## Transition to Phase 2.5

After completing this phase, you will take each property to the **INVARIANT vs RULE Decision Gate** in Phase 2.5.

The decision will use:
1. Your property description
2. Your ownership analysis (from Phase -1)
3. Your external truth flags

**Do NOT pre-decide invariant vs rule here.** That's Phase 2.5's job.

Properties marked "Aggregate/History Required?: Yes" will need ghost design in Phase 5.

---

### One-Line Litmus Test

> **If it already looks like CVL, you skipped a step.**

> 🚫 **Anti-Pattern:** Any statement that can be translated directly into CVL without additional modeling decisions is too concrete for this phase.

This guards the boundary.

---

## Category Mapping Reference

How categories here relate to categories in `SPEC AUTHORING (CERTORA).md`:

| This Document | SPEC AUTHORING Phase 2 | Notes |
|---------------|------------------------|-------|
| Valid States | A. State-Inductive Safety | Same concept |
| State Transitions | B. State Transition Properties | Same concept |
| System-Level | C. Global/Cross-User Properties | Same concept |
| Threat-Driven | D. Attack Surface Inversion | Same concept |

> 📌 The categories are semantic (what kind of harm), NOT encoding decisions (invariant vs rule).

---

## Common Mistakes to Avoid

| Mistake | Why It's Wrong | Correct Approach |
|---------|----------------|------------------|
| "invariant: balance >= 0" | Contains CVL keyword | "Balance must never be negative" |
| Skipping external truth analysis | HAVOC will break your spec | Fill in "External Truths Needed" |
| Not flagging aggregates | You'll miss ghost requirements | Mark "Aggregate Required?: Yes" |
| Mixing two truth owners | Creates causally unclosed property | Split into separate properties |
| Assuming trusted roles | Hidden threat model assumption | Mark "OUT OF SCOPE" explicitly |
| No unique IDs | Can't track through phases | Use A1, A2, B1, C1 format |

---

## Quick Checklist

Before leaving this phase:

- [ ] Every property has a unique ID
- [ ] Every property has one-sentence plain English description
- [ ] Every property has impact assessment
- [ ] Every property is categorized (Valid State / Transition / System-Level / Threat)
- [ ] Every property lists variables involved with owners
- [ ] External dependencies are flagged
- [ ] Aggregate requirements are flagged
- [ ] Trusted role assumptions are marked OUT OF SCOPE
- [ ] No CVL syntax anywhere
- [ ] No invariant/rule decision made yet
