# Audit Arena Week 3 Findings (From CSV)

Source: `NVN404_Submissions.csv`

## 1. [H1] - the proposals will get stuck forever without execution

- Date: 2026-02-17T12:28:45.049203
- Severity: High
- Scope: audit-arena-week-3.sol:execute()
- Reporter: -

### Details

[H1] - the proposals will get stuck forever without execution 

summary : 
when the external .call happens to fail , the tx is not reverted 

the executed flags will be set true even when the proposals are not actually executed 

root cause : no require statements after the externall .call and also setting the flag true before the call

recommended fix : use require statements to check the success of the low level call and set the flag true after the external call

    (bool success, ) = p.target.call(p.data);
    
    // 2. Require success — reverts whole tx if call failed or returned false
    require(success, "Proposal execution failed");

    // 3. Effects only after confirmed success
    p.executed = true;

---

## 2. [M1] - anyone can call the create proposal function

- Date: 2026-02-17T12:38:31.972075
- Severity: Medium
- Scope: audit-arena-week-3.sol:createProposal()
- Reporter: -

### Details

[M1] - anyone can call the create proposal function 

summary : 
anyone can call the create proposal function , which leads to creating spam proposals and enables spam attacks that bloat the `proposals` mapping and increment `nextProposalId` indefinitely, so it will be more gas costly for the real proposals created by the DAO . 

root cause :
missing modifier like onlyowner or admin only modifiers can leads to spam proposals 

attack path :
as the function is external with no additional checks
anyone can create proposals by calling createProposal() function 

recommended : add only owner modifers to prevent others from creating proposals

function createProposal(address _target, bytes calldata _data) external onlyOwner {
    
    }

---

## 3. [M2] - attacker can manipulate the results through flash loans

- Date: 2026-02-17T14:16:56.754542
- Severity: Medium
- Scope: audit-arena-week-3.sol:castVote()
- Reporter: -

### Details

[M2] - attacker can manipulate the results through flash loans 

summary - in cast vote function , the votes are decided by the weights
in which weights are nothing but the token balance of the voting user 

attack path - 
any one can flash loan the tokens and manipulate the proposal with 51% or more than that and the proposals
are manipulated 

root cause - there is no security features implemented in this contract to prevent flash loan attack

recommendation - usage of quadratic voting can prevent the attack , or adding more constraints to vote can prevent this type of attacks

---

## 4. [I1]- informational findings

- Date: 2026-02-17T14:31:31.960172
- Severity: Low
- Scope: -
- Reporter: -

### Details

[I1]- informational findings

after casting vote and proposal execution it is recommended to use events to emit 

and we should delete the proposal id after execution , to avoid increasing the size of the proposols mapping

---
