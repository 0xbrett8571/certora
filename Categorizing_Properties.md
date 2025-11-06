# TYPES OF PROPERTIES

## INTRODUCTION

To make the process of property creation easier, we can look at
the system from different points of view. In our case, these will
be our types of properties.
We split properties into 5 main types:

- Valid States
- State Transitions
- Variable Transitions
- High-Level Properties
- Demonstrating real impact — theft, freezing funds, governance takeover, etc. A "promise" is a statement about the protocol that must always be true for it to be secure. If we can prove it's false, we've found an impactful vulnerability, because we're not hunting for a bug, we're hunting for an impact that break the contract critical promise.

Thought of the day:

In order to create a good property, you need to think like a good property - - - creator. This means you need to think like an attacker. 

## VALID STATES

Systems can often be thought of as state machines. The states
define the possible values that the system's variables can take.
We call these Valid States.
For example, a system can be in one of the following states:

- doesn't exist
- was created
- is active
- is finished 

## VALID STATES

Developers often rely on the valid states implicitly or explicitly
in the system's workflow. Therefore, unintentional behavior can
occur if the system can break out of the intended valid states,
potentially resulting in devastating bugs.

Usually, there can be only one valid state at any given time.
Thus, we also check that a system must always be in exactly
one of its valid states.

## STATE TRANSITIONS

We also verify the correctness of transitions between valid
states. Firstly, we confirm that the valid states change
according to their correct order in the state machine. Then,
we verify that the transitions only occur under the right
conditions, like calls to specific functions or time elapsing.

## VARIABLE TRANSITIONS

Additionally, we can verify the change and validity of variables.
Some variables are designed to change in a certain way
through the entire system’s life cycle (monotonicity), while
others can change in any direction.

As a simple example, If we had a variable that counts the
number of transactions that have ever took place in a system, it
would’ve had to change in a non-decreasing manner
throughout the system’s life.

## HIGH-LEVEL PROPERTIES

Probably the most powerful type of properties is the
high-level property.

It doesn’t cover any tangible part of the system like
the aforementioned types (state, variable, or
transition). However, it does try to cover the whole
system from the users’ point of view. 

## HIGH-LEVEL PROPERTIES EXAMPLE

Let's take Bank as an example. We can think of the system as an
interaction between a client and the bank itself.
A high-level property for a client would be:

“If a client makes any operation within the bank system
(currency conversion, transfer between accounts, etc.), the
total balance of all clients' accounts should remain the same”.

This property makes sure that no assets are disappearing or
being created out of nowhere (unintentionally). We often call
this property solvency. 

## DEMONSTRATING REAL IMPACT

Demonstrating real impact — theft, freezing funds, governance takeover, etc. A "promise" is a statement about the protocol that must always be true for it to be secure. If we can prove it's false, we've found an impactful vulnerability, because we're not hunting for a bug, we're hunting for an impact that break the contract critical promise.

We're no longer hunting for a mistake in the code; you are hunting for a flaw in the logic. The code can be 100% correct by itself, but the system it creates is exploitable.

The impacts we care about include:

- Direct theft of funds
- Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results
- Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
- Permanent freezing of funds
- Protocol insolvency
- Theft of unclaimed yield
- Theft of unclaimed royalties
- Permanent freezing of unclaimed yield
- Temporary freezing of funds
- Smart contract unable to operate due to lack of token funds
- Block stuffing
- Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
- Unbounded gas consumption
- Contract fails to deliver promised returns, but doesn't lose value

Explore the code by adopting an attacker's mindset and asking a specific set of "what if" questions. You're no longer a developer reading for correctness; you are an attacker looking for an assumption to break.

Here is the tactical process for how to "query" the codebase to find those impacts.

The Methodology: Impact-Driven formal verification Interrogation

The process is:
 * Pick an Impact (e.g., "Direct theft of funds").
 * Formulate a "Query" (e.g., "Where does this contract approve other contracts to spend its tokens?").
 * Find the Code: For keywords related to your query (e.g., search for .approve().
 * Analyze the Result: Look at the lines of code the search returned. This is your answer. Ask: "Is this safe? Can I control the spender or amount?"

1. How to Query for: Theft of Funds / Insolvency
These impacts are about moving value. Your questions should hunt for any function that moves or values assets.
Search Keywords: e.g, transfer, transferFrom, approve, send, call, delegatecall, swap, mint, burn, oracle, getPrice
Your "Queries" (Questions to Ask the Code):
 * Q: Where can this contract's funds leave?
   * Find: Search for token.transfer(...) and token.transferFrom(...).
   * Analyze: Who can call this function? What are the conditions? Is the require statement strong enough? Can I bypass it?
 * Q: Does this contract give away its spending power?
   * Find: Search for token.approve(address spender, ...)
   * Analyze: Is the spender a hardcoded, trusted contract, or is it a variable? If it's a variable, can I control what it's set to (e.g., approve(myAttackerContract, ...)? This is a classic vulnerability.
 * Q: How does this contract value its assets or my collateral?
   * Find: Search for e.g, oracle, getPrice, get_price, or getQuote.
   * Analyze: What is the oracle? Is it a single Uniswap V2/V3 pool? (This is a huge red flag). Can I use a flash loan to manipulate that pool's price, then call this contract (e.g., borrow() or liquidate()) while the price is fake?
 * Q: What happens if I send amount = 0?
   * Find: Look at all deposit() and withdraw() functions.
   * Analyze: Does the code check require(amount > 0)? If not, I might be able to trigger reward calculations or other state changes for free, or even take out a "flash loan" from the protocol itself.
 * Q: Can I delegatecall into a contract I control?
   * Find: Search for delegatecall.
   * Analyze: Is the target address of the delegatecall a variable? If I can control that variable, I have full control of the contract's state, storage, and funds. This is a critical bug.
2. How to Query for: Manipulation of Governance
This is about faking power or breaking the voting process.
Search Keywords: vote, delegate, getVotes, _getVotingPower, snapshot, block.timestamp, block.number
Your "Queries":
 * Q: How is my voting power calculated?
   * Find: Search for getVotes or _getVotingPower.
   * Analyze: Does it read my balance right now? If so, I can take a flash loan for millions of tokens, delegate() to myself, vote() on a proposal, and repay the loan, all in one transaction. The voting power must come from a past snapshot.
 * Q: When is the snapshot taken?
   * Find: Search for snapshot().
   * Analyze: Who can call this? Can I call it after a proposal is created but before I've bought my tokens? The timing of the snapshot is critical.
 * Q: Can a proposal execute something I control?
   * Find: Look at the executeProposal function.
   * Analyze: What are the targets and calldata? If the proposal can make an arbitrary call to any address, I can create a proposal to call(myTokenAddress, "transfer(governanceAddress, 1_000_000)") and drain the treasury.
3. How to Query for: Griefing / Unbounded Gas / Freezing Funds
These are "economic" attacks. You're looking for ways to make the protocol unusable or expensive for other users.
Search Keywords: for (, while (, array.push(, address[], mapping(address => ...)
Your "Queries":
 * Q: Can I make this array infinitely large?
   * Find: Search for any public function that takes an array (e.g., airdrop(address[] users)) or lets me add items to an array (e.g., addUser()).
   * Analyze: Now, find the function that loops over that array (e.g., distributeRewards()). If I can add 10,000 addresses to the users array, the for loop in distributeRewards() will consume all the gas in a block, making it impossible to call ever again. The rewards are now permanently frozen.
 * Q: Can I block someone else's action?
   * Find: Look for any logic that checks if (user.isLocked) or if (order.isFilled).
   * Analyze: Can I, as an attacker, front-run a legitimate user and fill their order? Can I call a function that sets user.isLocked = true and prevent them from withdrawing their funds?
 * Q: Can I make this function cost a lot?
   * Find: Look for SSTORE (writing to storage) inside a loop.
   * Analyze: If I call a function that loops 100 times and each loop writes to a new storage slot, the gas cost will be enormous. If I can make other users pay that gas, it's a griefing attack.
By asking these targeted questions, you force the code to "confess" its weaknesses. You're no longer just reading; you're actively hunting for the specific lines that break the protocol's promises. This mindset shift is crucial for finding impactful vulnerabilities.
