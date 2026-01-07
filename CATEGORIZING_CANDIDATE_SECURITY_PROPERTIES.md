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

## 1. VALID STATES — “Laws of Existence”

**Concept:**
These describe states that should **never exist** at any point in execution.

If the system can ever *be* in such a state, the protocol is already broken.

**Candidate questions to ask:**

* Can the contract be insolvent?
* Can two mutually exclusive flags be true at once?
* Can a user’s position exist in a logically impossible configuration?

**Examples (Conceptual, Not Formal):**

* “The protocol must never owe more assets than it holds.”
* “A loan must never be undercollateralized outside liquidation.”
* “An account must not be both frozen and active simultaneously.”

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

**Examples (Conceptual):**

* “A user’s debt can only increase if they explicitly borrow.”
* “Configuration values must not change when called by a non-admin.”
* “A nonce must never decrease or repeat.”

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

**Examples (Conceptual):**

* “Total issued claims must correspond to real assets.”
* “Rewards must not appear without passage of time or contribution.”
* “Global balances must reconcile with external token balances.”

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

---

## Output of This Phase

The output is a **plain-English list of candidate laws**, each with:

* A short name
* A one-sentence description
* The impact if violated

**No CVL. No syntax. No constraints.**

---

## Mental Check (Before Moving On)

You are done with this phase **only if**:

* Every item can be explained to a non-CVL auditor
* Nothing mentions `invariant`, `rule`, `require`, `assert`, or `ghost`
* Each property clearly maps to a *real exploit if broken*

---

### One-Line Litmus Test

> **If it already looks like CVL, you skipped a step.**
