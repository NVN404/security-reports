# SimplePaymentSplitter — Bug Report

---

## Bug 1 — Incorrect `releasable()` Formula

| Field | Detail |
|-------|--------|
| **Severity** | Critical |
| **Location** | `releasable()` |

### Summary
Uses `address(this).balance` (current balance) instead of total ETH ever received. Every payout shrinks the balance, so later payees always get less than their rightful share.

### Attack Path
1. Contract has 3 payees with equal shares — 30 ETH deposited, 10 ETH each owed
2. Payee A calls `release()` → gets 10 ETH → balance drops to 20 ETH
3. Payee B calls `release()` → `releasable()` now computes against 20 ETH balance → gets ~6.67 ETH instead of 10
4. Payee C gets even less — each payout silently steals from the next in line

### Root Cause
```solidity
// WRONG — balance shrinks after every release()
uint256 gross = (address(this).balance * shares[payee]) / totalShares;
```

### Fix
```solidity
function releasable(address payee) public view returns (uint256) {
    uint256 totalReceived = address(this).balance + totalReleased;
    uint256 gross = (totalReceived * shares[payee]) / totalShares;
    return gross - released[payee];
}
```

---

## Bug 2 — `releaseAll()` DoS via Revert

| Field | Detail |
|-------|--------|
| **Severity** | High |
| **Location** | `releaseAll()` → `release()` |

### Summary
`release()` reverts with `"Nothing due"` if a payee has 0 claimable. A malicious payee can frontrun `releaseAll()`, drain their own share, then cause the entire loop to revert — blocking all remaining payees.

### Attack Path
1. Someone calls `releaseAll()` — tx sits in the mempool
2. Malicious payee sees it, frontruns with their own `release()` — drains their share
3. `releaseAll()` executes — hits the malicious payee in the loop, `releasable()` returns 0
4. `require(payment > 0)` reverts — entire loop rolls back
5. All remaining honest payees get nothing that round
6. Attacker can repeat this indefinitely, permanently griefing `releaseAll()`

### Root Cause
```solidity
// WRONG — single revert kills the entire loop
function releaseAll() external {
    for (uint256 i = 0; i < payees.length; i++) {
        release(payable(payees[i])); // reverts if payment == 0
    }
}
```

### Fix
```solidity
function releaseAll() external {
    for (uint256 i = 0; i < payees.length; i++) {
        if (releasable(payees[i]) > 0) {
            release(payable(payees[i]));
        }
    }
}
```

---

## Bug 3 — No Access Control on `release()`

| Field | Detail |
|-------|--------|
| **Severity** | Medium |
| **Location** | `release()` |

### Summary
Anyone can call `release()` for any payee. Funds still go to the correct address, but it enables griefing — bots can force payouts at bad times, break pull-payment assumptions, and cause unexpected interactions if a payee is a contract.

### Attack Path
1. Payee is a smart contract (e.g. a yield vault) that expects to control when it receives ETH
2. Attacker calls `release(payee)` at a moment the vault's `receive()` isn't ready (e.g. mid-rebalance)
3. Forces an ETH push into the contract at an unexpected time — could trigger unintended logic or accounting errors
4. Even for EOA payees: a bot can drain their share the moment ETH arrives, removing the payee's ability to time their own withdrawal (tax event griefing, etc.)

### Root Cause
```solidity
// WRONG — no restriction on who can trigger a payout
function release(address payable payee) public {
    ...
}
```

### Fix
```solidity
function release(address payable payee) public {
    require(msg.sender == payee, "Only payee can release");
    ...
}
```

---

## Bug 4 — `transfer()` Breaks Contract Payees

| Field | Detail |
|-------|--------|
| **Severity** | Medium |
| **Location** | `release()` |

### Summary
`payee.transfer()` forwards only 2300 gas. Any smart contract payee (multisig, vault, proxy) with non-trivial `receive()` logic will always revert, permanently locking their funds.

### Attack Path
1. Payee is a Gnosis Safe multisig — its `receive()` does a storage write to log the deposit (costs >2300 gas)
2. `release()` is called → `payee.transfer(payment)` → OOG revert
3. `released[payee]` was already incremented before transfer — wait, actually here it's incremented before too... but `transfer` reverting rolls back the whole tx
4. Funds are not sent, but more critically — this payee can **never** be paid out, ever
5. Their share is permanently locked in the contract with no recovery mechanism

### Root Cause
```solidity
// WRONG — 2300 gas stipend too low for contract payees
payee.transfer(payment);
```

### Fix
```solidity
released[payee] += payment;
totalReleased += payment;
(bool ok, ) = payee.call{value: payment}("");
require(ok, "Transfer failed");
```

> **Note:** State is updated before the external call — this already follows CEI, so no new reentrancy surface is introduced.

---

## Bug 5 — Integer Division Dust

| Field | Detail |
|-------|--------|
| **Severity** | Low |
| **Location** | `releasable()` |

### Summary
Integer division truncates remainders. Small amounts of wei accumulate in the contract and are permanently unclaimable.

### Attack Path
Not exploitable — no attacker benefit. Pure fund loss:
1. 10 ETH split among 3 equal payees → each gets `10e18 / 3 = 3333333333333333333` wei
2. Total paid: `3 × 3333333333333333333 = 9999999999999999999` wei
3. `1 wei` is permanently stuck — unclaimable by anyone
4. Scales with deposit frequency and payee count; in high-volume contracts this becomes meaningful

### Root Cause
```solidity
// Remainder from division is silently discarded
uint256 gross = (totalReceived * shares[payee]) / totalShares;
```

### Fix
Add a sweep function so the owner can recover stuck dust after all payees have claimed:
```solidity
function sweep(address payable to) external onlyOwner {
    uint256 dust = address(this).balance;
    require(dust > 0, "Nothing to sweep");
    (bool ok, ) = to.call{value: dust}("");
    require(ok, "Sweep failed");
}
```

---

## Summary Table

| # | Bug | Severity |
|---|-----|----------|
| 1 | Wrong balance in `releasable()` — later payees underpaid | **Critical** |
| 2 | `releaseAll()` reverts if any payee has 0 due — full DoS | **High** |
| 3 | No access control on `release()` — anyone can trigger payouts | **Medium** |
| 4 | `transfer()` used — contract payees locked out permanently | **Medium** |
| 5 | Integer division leaves wei dust stuck in contract | **Low** |