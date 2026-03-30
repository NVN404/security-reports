# Audit Arena Week 5 Findings (From CSV)

Source: `NVN404_Submissions.csv`

## 1. [H1] - Missing access control on setLockDuration allows anyone to change the global lock duration

- Date: 2026-03-02T11:41:25.387731
- Severity: High
- Scope: TimelockVault:setLockDuration()
- Reporter: yun0hu

### Details

[H1] - Missing access control on setLockDuration allows anyone to change the global lock duration

The setLockDuration function lacks any access control mechanism or checks  (such as an onlyOwner modifier or msg.sender == owner ). This allows any user or contract to change the global lockDuration variable. An attacker can set the lock duration to an exceptionally high value (like 100 years), effectively bricking the vault and permanently locking the funds of any user who deposits subsequently.

root cause : There is no check to ensure that msg.sender == owner inside the setLockDuration function or onlyowner modifiers !!!

attack path:
An unsuspecting user calls deposit() to deposit ETH normally as intended .
An attacker calls setLockDuration(type(uint256).max).(this can happen before or after depositing )
The user's unlockAt timestamp is set to block.timestamp + type(uint256).max, permanently locking their funds.

recommended fix
Add a validation check or an onlyOwner modifier so that only the authorized owner can change the lock duration:
or 
function setLockDuration(uint256 _lockDuration) external {
    require(msg.sender == owner, "Not authorized");
    lockDuration = _lockDuration;
}

---

## 2. [C1]- Flawed logic in extendLock allows all users to instantly bypass the timelock mechanism

- Date: 2026-03-02T11:47:44.294399
- Severity: Critical
- Scope: TimelockVault:extendLock()
- Reporter: yun0hu

### Details

[C1]- Flawed logic in extendLock allows all users to instantly bypass the timelock mechanism

The extendLock function is designed to let users increase their own lock time. However, it calculates the new lock time as block.timestamp + extraSeconds. By passing 0 as extraSeconds, a user can overwrite their existing lock time to the current block.timestamp, entirely bypassing the vault's core timelock restrictions and allowing immediate withdrawal.
changes the invarient of locked vault itself .

root cause :
The function adds the extra time to block.timestamp rather than the user's current unlockAt timestamp .

attack path :
An attacker calls deposit() with 10 ETH. Their unlockAt is set to block.timestamp + lockDuration.
The attacker immediately calls extendLock(0).
The contract updates their unlockAt to block.timestamp + 0.
The attacker calls withdraw(10 ether) and successfully retrieves their funds immediately, bypassing the wait time.

recommended fix :
Update the logic to add extraSeconds to the user's existing unlockAt timestamp, and ensure it can only strictly increase the duration like 

   function extendLock(uint256 extraSeconds) external {
        
        unlockAt[msg.sender] = currentLock + extraSeconds; // curent lock instead of block.timestamp
    }

---

## 3. [C1]-Reentrancy vulnerability in withdraw allows complete draining of the vault

- Date: 2026-03-02T11:51:59.501091
- Severity: Critical
- Scope: TimelockVault:withdraw()
- Reporter: yun0hu

### Details

[C1]-Reentrancy vulnerability in withdraw allows complete draining of the vault

The withdraw function suffers from a classic Reentrancy vulnerability. It transfers ETH to the caller via an external call (msg.sender.call) before deducting the withdrawn amount from the user's balance. A malicious smart contract can exploit this to recursively call withdraw() and drain all ETH stored in the vault.

root cause :
The state variable balances[msg.sender] is updated after the external ETH transfer, violating the Checks-Effects-Interactions (CEI) pattern.

attack path :

An attacker deploys a malicious smart contract and calls deposit() with 1 ETH.
The attacker waits for the lockDuration to expire .
The attacker's contract calls withdraw(1 ether).
The Vault sends 1 ETH to the malicious contract. This triggers the attacker's receive() or fallback() function.
In the fallback function, the attacker calls withdraw(1 ether) again. Because their balance has not been reduced yet, the check passes.
The loop continues until the Vault is completely drained of ETH.

recommended fix :
function withdraw(uint256 amount) external {
    require(block.timestamp >= unlockAt[msg.sender], "Locked");
    require(balances[msg.sender] >= amount, "Insufficient");
    // Effects
    balances[msg.sender] -= amount;
    // Interactions
    (bool ok, ) = msg.sender.call{value: amount}("");
    require(ok, "Transfer failed");
}

---
