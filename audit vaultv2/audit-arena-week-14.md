# SimpleTokenGovernance — Security Audit Findings

**Contract:** `SimpleTokenGovernance.sol`
**Solidity Version:** `^0.8.20`
**Audit Scope:** Governance voting logic, token weight accounting, quorum calculation
**Total Findings:** 6

---

## Finding 1 — Flash Loan Vote Manipulation

| Field | Detail |
|---|---|
| **Severity** | Critical |
| **Location** | `castVote()` |

### Description
The `castVote()` function reads a voter's token balance at the time of the vote using `token.balanceOf(msg.sender)`. There is no snapshot mechanism anchoring voting power to a fixed block. An attacker can borrow a massive amount of tokens via a flash loan, cast a vote with inflated weight within the same transaction, and repay the loan — permanently skewing the vote outcome with tokens they never actually owned.

### Impact
A single attacker can single-handedly pass or block any proposal regardless of the actual token holder distribution. The governance system can be fully captured in one transaction with zero capital cost.

### Root Cause
Voting power is derived from a live `balanceOf` query rather than a historical snapshot (e.g., ERC20Votes checkpoints at proposal creation block).

### Attack Path
```
1. Attacker calls a flash loan provider to borrow X tokens (X >> quorum threshold).
2. Within the same transaction, attacker calls castVote(id, true).
3. balanceOf(attacker) returns X → forVotes += X.
4. Attacker repays the flash loan.
5. After endTime, execute() sees forVotes >= quorum → approved[id] = true.
```

### Recommended Fix
Use OpenZeppelin's `ERC20Votes` and snapshot voting power at proposal creation using `token.getPastVotes(account, proposalSnapshot)`. Store `proposalSnapshot = block.number` when the proposal is created and query it at vote time.

```solidity
// In Proposal struct
uint256 snapshotBlock;

// In propose()
proposals[id].snapshotBlock = block.number;

// In castVote()
uint256 weight = token.getPastVotes(msg.sender, proposals[id].snapshotBlock);
```

---

## Finding 2 — Double Voting via Token Transfer

| Field | Detail |
|---|---|
| **Severity** | Critical |
| **Location** | `castVote()` |

### Description
Once a voter casts their vote, nothing prevents them from transferring their tokens to a different address and voting again from that address. The `hasVoted` mapping tracks the voter's address, not the tokens themselves. A single token balance can therefore be used to generate votes multiple times across as many wallets as the attacker controls, causing `forVotes` or `againstVotes` to exceed the actual token supply.

### Impact
Vote totals can be inflated arbitrarily beyond `totalSupply`. A coordinated attacker with tokens spread across wallets (or willing to do repeated transfers) can guarantee any proposal passes quorum without a flash loan.

### Root Cause
No snapshot mechanism. Voting power is live and transferable after it has already been counted.

### Attack Path
```
1. Alice holds 1000 tokens. She calls castVote(id, true) → forVotes += 1000.
2. Alice transfers 1000 tokens to Bob (controlled by Alice).
3. Bob calls castVote(id, true) → forVotes += 1000 again.
4. Repeat with Charlie, Dave, etc.
5. forVotes can reach N * 1000 where N = number of wallets.
```

### Recommended Fix
Same as Finding 1 — snapshot voting power at proposal creation block. Once `getPastVotes` is used, transferring tokens post-snapshot has no effect on recorded voting weight.

---

## Finding 3 — Reentrancy in `castVote` (CEI Violation)

| Field | Detail |
|---|---|
| **Severity** | High |
| **Location** | `castVote()` |

### Description
`castVote()` calls the external function `token.balanceOf(msg.sender)` before writing `hasVoted[id][msg.sender] = true`. This violates the Checks-Effects-Interactions pattern. If the token contract is ERC777-based or contains any transfer/receive hook that re-enters `castVote`, the attacker can vote multiple times before the `hasVoted` flag is set, doubling (or further multiplying) their counted vote weight.

### Impact
An attacker using a token with receive hooks can cast multiple votes from a single address in one transaction, inflating vote counts beyond their actual balance.

### Root Cause
State update (`hasVoted`) occurs after the external interaction (`token.balanceOf`), violating CEI ordering.

### Vulnerable Code
```solidity
uint256 weight = token.balanceOf(msg.sender); // <-- external call
require(weight > 0, "No voting power");
hasVoted[id][msg.sender] = true;              // <-- state written AFTER
```

### Attack Path
```
1. Attacker deploys a malicious ERC777 token registered as governance token.
2. token.balanceOf() triggers a tokensReceived hook on the attacker's contract.
3. Hook re-enters castVote() before hasVoted is set → vote counted again.
4. Recursion depth = attacker's desired vote multiplier.
```

### Recommended Fix
Move the `hasVoted` write before the external call, strictly following CEI:

```solidity
function castVote(uint256 id, bool support) external {
    Proposal storage p = proposals[id];
    require(block.timestamp < p.endTime, "Voting ended");
    require(!hasVoted[id][msg.sender], "Already voted");

    // EFFECT before INTERACTION
    hasVoted[id][msg.sender] = true;

    uint256 weight = token.balanceOf(msg.sender);
    require(weight > 0, "No voting power");

    if (support) {
        p.forVotes += weight;
    } else {
        p.againstVotes += weight;
    }
}
```

Alternatively, add a `ReentrancyGuard` modifier from OpenZeppelin.

---

## Finding 4 — Live `totalSupply` Allows Quorum Manipulation

| Field | Detail |
|---|---|
| **Severity** | High |
| **Location** | `quorum()`, `execute()` |

### Description
The `quorum()` function reads `token.totalSupply()` at the moment `execute()` is called, not at proposal creation. If the token supply is mutable (mintable or burnable), an attacker can manipulate the quorum threshold between proposal creation and execution. Burning tokens reduces `totalSupply`, which lowers the quorum threshold — potentially allowing a proposal to pass quorum that would not have passed at creation time.

### Impact
- **Burn attack:** Burn tokens before `execute()` → lower quorum → previously-failing proposal now passes.
- **Mint attack:** Mint tokens before `execute()` → higher quorum → prevent a passing proposal from executing.

### Root Cause
`quorum()` is a live view that queries the current `totalSupply` instead of a snapshot taken at proposal creation.

### Attack Path
```
1. Proposal created when totalSupply = 1,000,000. Quorum = 40,000.
2. forVotes = 35,000 (below quorum).
3. Attacker burns 125,001 tokens → totalSupply = 874,999. Quorum = 34,999.
4. execute() → forVotes (35,000) >= quorum (34,999) → approved.
```

### Recommended Fix
Snapshot `totalSupply` at proposal creation and store it in the `Proposal` struct:

```solidity
struct Proposal {
    ...
    uint256 snapshotSupply;
}

// In propose()
proposals[id].snapshotSupply = token.totalSupply();

// In execute()
uint256 quorumRequired = (p.snapshotSupply * quorumBps) / 10000;
if (p.forVotes > p.againstVotes && p.forVotes >= quorumRequired) {
    approved[id] = true;
}
```

---

## Finding 5 — Unrestricted `propose()` Allows Governance Spam

| Field | Detail |
|---|---|
| **Severity** | Medium |
| **Location** | `propose()` |

### Description
Any address — including those with zero token balance — can call `propose()` with no restrictions. There is no minimum token threshold, no proposer stake, and no limit on active proposals. An attacker can flood the system with thousands of proposals, causing off-chain indexers to become overwhelmed, governance UIs to break, and legitimate voters to be unable to identify real proposals.

### Impact
- Denial-of-service on the governance frontend and indexers.
- Legitimate proposals buried under spam.
- `proposalCount` overflows are not possible due to Solidity 0.8+ checks, but gas costs for iterating proposals off-chain grow unbounded.

### Root Cause
No access control or economic barrier on the `propose()` function.

### Attack Path
```
1. Attacker calls propose() in a loop (via script) with junk descriptions.
2. proposalCount grows to thousands instantly.
3. Off-chain tooling becomes unusable; real proposals are indistinguishable from noise.
```

### Recommended Fix
Require a minimum token balance to propose:

```solidity
uint256 public proposalThresholdBps = 100; // 1%

function propose(...) external {
    uint256 threshold = (token.totalSupply() * proposalThresholdBps) / 10000;
    require(token.balanceOf(msg.sender) >= threshold, "Below proposal threshold");
    ...
}
```

Or use a timelock and a minimum proposal interval per address.

---

## Finding 6 — No Minimum Voting Duration

| Field | Detail |
|---|---|
| **Severity** | Low |
| **Location** | `propose()` |

### Description
The only validation on `durationSeconds` is `require(durationSeconds > 0)`, meaning a proposal with a 1-second voting window is valid. Combined with Finding 5 (unrestricted `propose()`), an attacker who holds significant tokens can create a proposal, immediately vote on it in the same block or the next one, and have it execute before any other holder is aware it existed.

### Impact
Governance can be bypassed by a large holder creating and passing proposals in near-zero time, eliminating the deliberation window that governance systems are designed to provide.

### Root Cause
No minimum duration enforcement in `propose()`.

### Attack Path
```
1. Whale calls propose(description, 1).
2. In the same or next block, calls castVote(id, true).
3. One second later, calls execute(id).
4. No other token holder had time to respond.
```

### Recommended Fix
Enforce a minimum voting period:

```solidity
uint256 public constant MIN_VOTING_PERIOD = 2 days;

function propose(...) external {
    require(durationSeconds >= MIN_VOTING_PERIOD, "Duration too short");
    ...
}
```

---

## Summary Table

| # | Title | Severity | Location |
|---|---|---|---|
| 1 | Flash Loan Vote Manipulation | **Critical** | `castVote()` |
| 2 | Double Voting via Token Transfer | **Critical** | `castVote()` |
| 3 | Reentrancy — CEI Violation | **High** | `castVote()` |
| 4 | Live `totalSupply` Quorum Manipulation | **High** | `quorum()`, `execute()` |
| 5 | Unrestricted `propose()` — Spam DoS | **Medium** | `propose()` |
| 6 | No Minimum Voting Duration | **Low** | `propose()` |

---

*Report generated for educational and audit-practice purposes.*