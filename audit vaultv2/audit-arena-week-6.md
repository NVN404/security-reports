# SimpleStaking Contract — Security Audit Report

**Auditor:** yunohu
**Date:** 2026-03-09
**Scope:** [`audit-arena-week-6.sol`  ](https://github.com/radcipher/auditvault/blob/main/competitions/audit-arena-week-6.sol)
**Language:** Solidity ^0.8.20  

---

## Summary

| ID  | Title                                                                 | Severity |
|-----|-----------------------------------------------------------------------|----------|
| H-1 | Reward calculation ignores staked balance — anyone can drain the pool | High     |
| H-2 | Missing access control on `updateRate`                                | High     |
| H-3 | Incorrect `stakeTokenBalance` returns ETH balance instead of ERC20   | High     |
| H-4 | incorrect calculation of principal and rewards leads to loss of user deposits        | High     |
| H-5 | Reentrancy via CEI violation in `_claimRewards`                       | High     |
| H-6 | sudden reward rate change destroys previously accrued user yield        | High     |
| M-1 | `tx.origin` used for access control — phishing vector                 | Medium   |

---

## H-1 — Reward Calculation Ignores Staked Balance, Anyone Can Drain the Pool

### Summary / Impact

The `_claimRewards` function calculates a user's reward by multiplying elapsed time by `rewardRatePerSecond`, completely ignoring the user's staked balance. Any address — even one with **zero tokens staked** — can call `claimRewards()`, initialize a timer, wait, and then claim rewards proportional to the time elapsed. This allows an attacker to drain the entire reward pool without locking any capital.

### Root Cause

In `_claimRewards`, the calculation:

```solidity
uint256 reward = elapsed * rewardRatePerSecond;
```

fails to account for the user's share of the pool (`staked[user]`).

### Attack Path

Attacker calls `claimRewards()` with 0 tokens staked. `lastUpdate[msg.sender]` is initialized to `block.timestamp`.
Attacker waits for an arbitrary time `T`.
Attacker calls `claimRewards()` again.
Contract computes `reward = T * rewardRatePerSecond`.
Contract transfers the reward amount to the attacker despite 0 tokens being staked.

### Recommended Fix

Multiply reward by the user's staked weight:

```solidity
uint256 reward = (elapsed * staked[user] * rewardRatePerSecond) / PRECISION;
```

---

## H-2 — Missing Access Control on `updateRate` Allows Anyone to Manipulate Reward Logic

### Summary / Impact

The `updateRate` function is `external` but contains no access control check. Any address can set `rewardRatePerSecond` to an enormous value, wait one second, and drain the entire reward pool. Alternatively, it can be set to zero to grief all existing stakers.

### Root Cause

```solidity
function updateRate(uint256 newRate) external {
    // ← No require(msg.sender == owner) check!
    rewardRatePerSecond = newRate;
}
```

### Attack Path

Attacker calls `updateRate(1_000_000_000 ether)`.
Attacker waits 1 second.
Attacker calls `claimRewards()` and drains the entire contract balance.

### Recommended Fix

```solidity
function updateRate(uint256 newRate) external {
    require(msg.sender == owner, "Not owner");
    rewardRatePerSecond = newRate;
}
```

---

## H-3 — `stakeTokenBalance` Returns Native ETH Balance Instead of ERC20 Token Balance

### Summary / Impact

The `stakeTokenBalance` function returns `address(stakeToken).balance`, which is the **native ETH balance** of the token contract, not the contract's ERC20 token balance. This value is almost always `0`. As a result, `emergencyWithdraw` will always transfer `0` tokens to the owner, making the recovery mechanism completely non-functional.

### Root Cause

```solidity
function stakeTokenBalance() public view returns (uint256) {
    return address(stakeToken).balance; // ← Returns ETH balance, not ERC20 balance!
}
```

### Attack Path

Owner calls `emergencyWithdraw()` to recover funds.
`stakeTokenBalance()` returns `0` (ETH balance of the ERC20 contract).

`stakeToken.transfer(owner, 0)` executes — no funds are recovered.
All staked tokens and rewards remain permanently locked if the normal flow breaks.

### Recommended Fix

```solidity
function stakeTokenBalance() public view returns (uint256) {
    return stakeToken.balanceOf(address(this));
}
```

---

## H-4 — incorrect calculation of principal and rewards leads to loss of user deposits

### Summary / Impact

The contract holds both staked deposits and reward tokens in a **single shared balance** with no accounting separation. When `_claimRewards` executes, it transfers tokens from the overall contract balance. If the reward pool is insufficiently funded, the contract silently pays rewards using the **principal deposits of other users**, causing a direct and irreversible loss of staked capital.

### Root Cause

There is no `totalStaked` variable to protect user deposits. The contract calls:

```solidity
stakeToken.transfer(user, reward);
```

without verifying that the transfer amount does not dip into deposited principal.

### Attack Path

Alice and Bob each stake `1000` tokens. Contract balance = `2000`.
Owner never calls `fundRewards()`.
Time passes. Alice accrues `500` tokens in rewards.
Alice calls `claimRewards()`. Contract pays `500` tokens from its `2000` balance.
Contract balance = `1500`, but Alice and Bob still have `1000` each recorded in `staked`.
`500` tokens of Bob's principal have been stolen to pay Alice's reward.
Bob can no longer withdraw his full stake.

### Recommended Fix

Track total staked capital and enforce that rewards only come from surplus:

```solidity
uint256 public totalStaked;

// In stake():
totalStaked += amount;

// In withdraw():
totalStaked -= amount;

// In _claimRewards():
uint256 availableRewards = stakeToken.balanceOf(address(this)) - totalStaked;
require(availableRewards >= reward, "Insufficient reward pool");
stakeToken.transfer(user, reward);
```

---

## H-5 — Reentrancy via CEI Violation in `_claimRewards` Allows Recursive Reward Drain

### Summary / Impact

In `_claimRewards`, the external `stakeToken.transfer` call is made **before** `lastUpdate[user]` is updated. This violates the Checks-Effects-Interactions (CEI) pattern. If the token supports transfer hooks (e.g., ERC777), an attacker can re-enter `claimRewards()` during the transfer. Because `lastUpdate` has not been updated yet, the contract calculates the same reward again and transfers it, allowing recursive draining of the entire contract balance.

### Root Cause

```solidity
stakeToken.transfer(user, reward);     // ← External call first
lastUpdate[user] = block.timestamp;    // ← State update happens AFTER
```

### Attack Path

Attacker stakes tokens and waits for rewards to accrue.
Attacker calls `claimRewards()`.
`_claimRewards` calculates the reward and calls `stakeToken.transfer(attacker, reward)`.
The token triggers a receive hook on the attacker's contract.
Inside the hook, attacker calls `claimRewards()` again.
Since lastUpdate was not yet updated, elapsed is unchanged, and the same reward is sent again.
Loop repeats until the entire contract balance is drained.

### Recommended Fix

Always update state before making external calls (CEI pattern):

```solidity
function _claimRewards(address user) internal {
    if (lastUpdate[user] == 0) {
        lastUpdate[user] = block.timestamp;
        return;
    }
    uint256 elapsed = block.timestamp - lastUpdate[user];
    uint256 reward = elapsed * rewardRatePerSecond;

    // EFFECTS: Update state first
    lastUpdate[user] = block.timestamp;

    // INTERACTIONS: External call last
    if (reward > 0) {
        stakeToken.transfer(user, reward);
    }
}
```

---

## H-6 — sudden reward rate change destroys previously accrued user yield

### Summary / Impact

When `updateRate()` modifies `rewardRatePerSecond`, the new rate is retroactively applied to the **entire timespan** since each user's `lastUpdate` timestamp. The contract's naive formula does not cache rewards earned under the old rate. If the rate is decreased, users suffer a direct mathematical loss of legitimately earned, unclaimed yield.

### Root Cause

The formula:

```solidity
uint256 reward = elapsed * rewardRatePerSecond;
```

assumes the rate has been constant for the entire `elapsed` duration, which is mathematically incorrect after any rate change.

### Attack Path

Users stake tokens for 6 months at `rewardRatePerSecond = 10`.
Owner calls `updateRate(5)` to reduce inflation.
The contract now calculates the past 6 months of rewards using rate `5` instead of `10`.
All users immediately — and silently — lose **half** of the yield they legitimately earned before the change.

### Recommended Fix

Before updating the rate, trigger a global checkpoint to lock in rewards earned under the old rate. Alternatively, adopt the Synthetix `rewardPerTokenStored` accumulator architecture so that rate changes only affect time elapsed **after** the update.

---

## M-1 — `tx.origin` Used for Authorization Enables Phishing Attack on `emergencyWithdraw`

### Summary / Impact

The `emergencyWithdraw` function uses `tx.origin == owner` for authorization instead of `msg.sender`. `tx.origin` refers to the original EOA that started the transaction chain, not the immediate caller. A malicious contract can trick the owner into calling it, which then calls `SimpleStaking.emergencyWithdraw()`. The check passes because `tx.origin` is still the owner, allowing the attacker to forcibly trigger a full withdrawal and break the protocol for all users.

### Root Cause

```solidity
require(tx.origin == owner, "Not owner"); // ← tx.origin is phishing-vulnerable
```

### Attack Path

Attacker deploys a malicious contract with a hidden call to `SimpleStaking.emergencyWithdraw()`.
Attacker socially engineers the `owner` into calling a function on the malicious contract.
The malicious contract internally calls `emergencyWithdraw()`.
`tx.origin == owner` evaluates to `true` since the owner's EOA initiated the full call chain.
All staked deposits and reward tokens are transferred to the `owner` address, permanently breaking the protocol for all stakers.

### Recommended Fix

Always use `msg.sender` for authorization:

```solidity
require(msg.sender == owner, "Not owner");
```

---

*End of Report*
