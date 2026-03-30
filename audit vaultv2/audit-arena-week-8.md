# Audit Arena Week 8 Findings (From CSV)

Source: `NVN404_Submissions.csv`

## 1. [c2]-Reentrancy in withdraw() via CEI Violation

- Date: 2026-03-23T11:27:25.838803
- Severity: Critical
- Scope: -
- Reporter: @yun0hu

### Details

[c2]-Reentrancy in withdraw() via CEI Violation
A malicious token with transfer hooks lets the beneficiary re-enter withdraw() before state updates, claiming the same amount repeatedly until the stream is drained.

Root Cause: token.transfer() is called before s.withdrawn += available, violating CEI.

Attack Path:
Beneficiary is a contract with a token receive hook
withdraw() triggers the hook mid execution
Hook re enters withdraw()  functions s.withdrawn 
Repeat until drained

Recommendation: Move s.withdrawn += available to before the token.transfer() call.

---

## 2. Title: Owner Steals Vested Tokens on Cancel

- Date: 2026-03-23T11:43:45.769435
- Severity: High
- Scope: -
- Reporter: @yun0hu

### Details

Title: Owner Steals Vested Tokens on Cancel 
Owner receives all unwithdrawm tokens when cancels including what the beneficiary already vested. Beneficiary loses earned tokens. 

Root Cause: Cancel uses totalAmount - withdrawn instead of accounting for vested amount separately.

Attack Path:
1. Stream is 60% vested, beneficiary withdrawn 0%
2. Owner cancels — takes 100%
3. Beneficiary loses their 60% vested share

Recommendation: On cancel, send vested(id) - s.withdrawn to beneficiary, then refund totalAmount - vested(id) to owner.

---

## 3. [H]-Double-Cancel Executes Stale Transfer

- Date: 2026-03-23T11:48:25.403416
- Severity: High
- Scope: -
- Reporter: yun0hu

### Details

[H]-Double-Cancel Executes Stale Transfer

Already canceled stream can be canceled again, reexecuting transfer logic on stale state. 

Root Cause: No require(!s.canceled) guard at top of cancel().

 Attack Path:
1. Owner cancels — funds sent out
2. Owner calls cancel() again on same id -> logic re-executes

Recommendation: Add require(!s.canceled, "Already canceled") at the start of cancel().

---

## 4. No Input Validation in updateBeneficiary

- Date: 2026-03-24T03:20:20.532244
- Severity: High
- Scope: -
- Reporter: @yun0hu

### Details

No Input Validation in updateBeneficiary

Doesn't validate the stream ID exists
Doesn't prevent changing beneficiary to address(0) (breaks future withdrawals)

root cause : no input validation 

attack path :the owener can set the address of the benfeciaries to address(0)
which leads to loss of users funds

recommendation fix : 
add validation 
equire(streams[id].beneficiary != address(0), "Stream does not exist");
require(newBeneficiary != address(0), "Invalid address");

---
