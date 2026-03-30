# Audit Arena Week 2 Findings (From CSV)

Source: `NVN404_Submissions.csv`

## 1. Reentrancy in withdraw function

- Date: 2026-02-12T02:56:43.241678
- Severity: High
- Scope: audit-arena-week-2.sol:withdraw()
- Reporter: -

### Details

Reentrancy in withdraw function 

It doesn't follow CIE method 


Recommended method 

function withdraw(uint256 amount) external {
    require(userCollateral[msg.sender] >= amount, "Insufficient funds");
    
    uint256 newCollateral = userCollateral[msg.sender] - amount;
    require(userDebt[msg.sender] <= newCollateral / 2, "Debt too high");


    userCollateral[msg.sender] = newCollateral;


    (bool success, ) = msg.sender.call{value: 0}("");
    require(success, "Transfer signal failed");
}

This current implementation follows CIE method

---

## 2. Amount stuck in the contract forever while withdrawal of the collateral by user

- Date: 2026-02-12T03:04:44.633011
- Severity: High
- Scope: audit-arena-week-2.sol:withdraw()
- Reporter: -

### Details

Amount stuck in the contract forever while withdrawal of the collateral by user 

In withdraw function when we make external call the value is set 0 

But the value should be the  amount specified by the user to withdraw and not "0"

Recommended 

function withdraw(uint256 amount) external {
    require(userCollateral[msg.sender] >= amount, "Insufficient funds");
    

    uint256 newCollateral = userCollateral[msg.sender] - amount;
    require(userDebt[msg.sender] <= newCollateral / 2, "Debt too high");


    userCollateral[msg.sender] = newCollateral;


    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "ETH transfer failed");
}

---
