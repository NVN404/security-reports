# TokenStream.sol — Audit Findings

---

## Finding 1

**Title:** Unrestricted Beneficiary Hijacking via `updateBeneficiary()`

**Severity:** Critical

**Scope:** `updateBeneficiary()`

**Summary / Impact:**
Any external caller can reassign the beneficiary of any stream to an address they control, then immediately call `withdraw()` to drain all vested tokens. Complete and irreversible loss of funds for the legitimate beneficiary.

**Root Cause:**
`updateBeneficiary()` has zero access control. No check that the caller is the owner or the current beneficiary of the stream.

**Attack Path:**
1. Attacker spots an active stream ID on-chain
2. Calls `updateBeneficiary(id, attackerAddress)`
3. Calls `withdraw(id)` and drains all vested tokens

**Recommendation:**
Add `require(msg.sender == owner || msg.sender == s.beneficiary, "Not authorized")` before updating the beneficiary.

---

## Finding 2

**Title:** Owner Steals Vested-But-Unwithdrawn Tokens on Cancel

**Severity:** High

**Scope:** `cancel()`

**Summary / Impact:**
When owner cancels a stream, they receive `totalAmount - withdrawn` which incorrectly includes tokens the beneficiary has already vested but not yet claimed. Even a non-malicious owner calling cancel will cause direct fund loss for the beneficiary with no recovery path.

**Root Cause:**
`cancel()` computes `remaining = totalAmount - withdrawn` and sends it all to the owner. This does not account for the vested amount — tokens earned by the beneficiary are treated as an owner refund.

**Attack Path:**
1. Stream is 60% through its duration, beneficiary has withdrawn 0%
2. Owner calls `cancel()`
3. `remaining = 1000 - 0 = 1000` — owner receives everything
4. Beneficiary loses their 600 vested tokens with no recourse

**Recommendation:**
On cancel, split the refund correctly:
- Send `vested(id) - s.withdrawn` to the beneficiary
- Send `totalAmount - vested(id)` back to the owner

---

## Finding 3

**Title:** Double-Cancel Drains Funds Belonging to Other Streams

**Severity:** Medium

**Scope:** `cancel()`

**Summary / Impact:**
An already-canceled stream can be canceled again. Since the contract holds tokens for all streams in a single pool, the second cancel re-executes the transfer with stale `remaining` value and dips into balances belonging to other streams' beneficiaries.

**Root Cause:**
No `require(!s.canceled)` guard at the top of `cancel()`. `s.withdrawn` is never updated on cancel so the stale `remaining` value persists across calls.

**Attack Path:**
1. Stream 1 has 1000 tokens, Stream 2 has 500 tokens in the same contract
2. Owner cancels Stream 1 — `remaining = 1000`, funds sent out
3. Owner calls `cancel()` on Stream 1 again
4. `remaining = 1000 - 0 = 1000` (stale) — contract pays from Stream 2's balance
5. Stream 2 beneficiary loses their funds

**Recommendation:**
Add `require(!s.canceled, "Already canceled")` at the start of `cancel()`.

---

## Finding 4 

**Title:** Reentrancy in `withdraw()` via CEI Violation

**Severity:** Critical (OOS — requires non-standard ERC20)

**Scope:** `withdraw()`

**Summary / Impact:**
`token.transfer()` is called before `s.withdrawn` is updated. A token with transfer hooks (e.g. ERC777) allows the beneficiary to re-enter `withdraw()` before state updates, draining the stream by repeatedly claiming the same `available` amount.

**Root Cause:**
Violates Checks-Effects-Interactions pattern. State update happens after the external call.

**Attack Path:**
1. Beneficiary is a contract with a token receive hook
2. `withdraw()` triggers the hook mid-execution
3. Hook re-enters `withdraw()` — `s.withdrawn` still old, `available` same
4. Repeat until stream is fully drained

**Recommendation:**
Move `s.withdrawn += available` to before the `token.transfer()` call.

---
the end


