# Audit Arena Week 7 Findings (From CSV)

Source: `NVN404_Submissions.csv`

## 1. [H]-Seller receives 0 ETH on withdrawal and locking the user funds forever

- Date: 2026-03-16T08:40:58.664916
- Severity: High
- Scope: audit-arena-week-7.sol:withdraw()
- Reporter: yun0hu

### Details

[H]-Seller receives 0 ETH on withdrawal and locking the user funds forever 

In withdraw() function , the seller's payment amount is set to zero 
before being used in the ETH transfer. This causes the seller to receive 
0 ETH despite a successful delivery, and permanently locks the user 
funds in the contract with no recovery path.

Root cause :
o.amount is zeroed out before being passed to .call{value: o.amount}
Since Solidity evaluates o.amount at call time, the transfer sends 0 ETH.

Attack path :
Buyer calls createOrder(seller) with 1 ETH -> orders[1].amount = 1 ETH
Seller calls markShipped(1)
Buyer calls confirmDelivery(1)
Seller calls withdraw(1)
o.amount set to 0
.call{value: 0} executes -> seller receives nothing
ETH remains locked in contract forever
No function exists to recover it


Recommended fix : 
function withdraw(uint256 id) external {
    Order storage o = orders[id];
    require(o.status == Status.Delivered, "Not delivered");
    require(msg.sender == o.seller, "Not seller");

    uint256 payment = o.amount;  // cache first by storing it in different variable 
    o.amount = 0; // then set to 0
    o.status = Status.Refunded;
    (bool ok, ) = o.seller.call{value: payment}("");  // send cached value
    require(ok, "Pay failed");
}

---

## 2. [M] Anyone can call markShipped() on any existing order

- Date: 2026-03-16T08:55:24.804573
- Severity: Medium
- Scope: -
- Reporter: yun0hu

### Details

[M] Anyone can call markShipped() on any existing order

markShipped() has no caller validation. Any address can mark any order as shipped, even if the actual seller has not shipped anything. This allows the buyer themselves or any third party to manipulate order state, 
potentially trapping funds or forcing premature dispute resolution. 

Root cause :
No msg.sender check exists in markShipped() .The function only validates 
order status, not caller identity.

Attack path :
Buyer calls createOrder(seller) with 1 ETH
Seller has not shipped anything yet
Any address calls markShipped(1) -> succeeds
Order is now in Shipped state
Buyer cannot get a refund via disputeRefund until owner intervenes.
Buyer is pressured to either confirm delivery for unshipped goods
or wait indefinitely for owner dispute resolution

Recommended fix : 
function markShipped(uint256 id) external {
    Order storage o = orders[id];
    require(o.status == Status.Created, "Bad status");
    require(msg.sender == o.seller, "Not seller");  // add this
    o.status = Status.Shipped;
}

---

## 3. [H] Owner can drain funds to arbitrary address via buyerOverride in disputeRefund()

- Date: 2026-03-16T09:52:24.733668
- Severity: High
- Scope: -
- Reporter: yun0hu

### Details

[H] Owner can drain funds to arbitrary address via buyerOverride in disputeRefund()


disputeRefund() accepts a buyerOverride parameter with no validation 
tying it to the original buyer. Owner can pass any address, including 
their own wallet and drain ETH from any order in Shipped state.

Root Cause
No check enforces that buyerOverride == o.buyer. The recipient is 
entirely caller-controlled.

Attack path : 
Buyer calls createOrder(seller) with 1 ETH
Anyone calls markShipped(1) 
Owner calls disputeRefund(1, owner_wallet)
ETH sent to owner's wallet instead of buyer
Buyer loses funds with no option

Recommended path : 
function disputeRefund(uint256 id, bool favorBuyer) external {
    Order storage o = orders[id];
    require(o.status == Status.Shipped, "Not shipped");
    require(msg.sender == owner, "Not owner");

    uint256 amount = o.amount;
    o.amount = 0;
    o.status = Status.Refunded;

    address recipient = favorBuyer ? o.buyer : o.seller;
    (bool ok, ) = recipient.call{value: amount}("");
    require(ok, "Refund failed");
}

This is framed to either seller or buyer not a third unknown address

---

## 4. [M] seller = address(0) in createOrder() permanently locks buyer funds

- Date: 2026-03-16T10:14:19.168788
- Severity: Medium
- Scope: -
- Reporter: yun0hu

### Details

[M] seller = address(0) in createOrder() permanently locks buyer funds

createOrder() has no validation on the `seller` parameter. A buyer can 
create an order with `seller = address(0)`, depositing ETH that becomes 
permanently unrecoverable.

Root cause 
No zero address check on `seller` at order creation time.

Attack path :
Buyer calls createOrder(address(0)) with 1 ETH it succeeds
During markShipped address(0) cannot call, order stays Created
In withdraw address(0) cannot call

In disputeRefund  
attempts .call to address(0), fails or burns ETH
ETH permanently locked, no recovery path

Recommended:
function createOrder(address seller) external payable returns (uint256 id) {
    require(msg.value > 0, "No ETH");
    require(seller != address(0), "Invalid seller");  // add this
    ...
}

---

## 5. [L] withdraw() sets order status to Refunded instead of a Completed state

- Date: 2026-03-16T10:21:11.501977
- Severity: Low
- Scope: -
- Reporter: yun0hu

### Details

[L] withdraw() sets order status to Refunded instead of a Completed state

After a successful seller withdrawal, o.status is set to Status.Refunded.
This is semantically incorrect , the order was fulfilled and paid, not 
refunded. This corrupts the state machine and creates a logical inconsistency 
 Root Cause
Wrong enum value used in the withdraw state transition.


Attack Path
No direct fund loss, but:
Completed orders and refunded orders share the same status
Off-chain indexers or integrations reading order status will
misinterpret successful sales as buyer refunds
Future contract logic built on top of this state machine will
inherit corrupted semantics

Recommended Fix
Add a dedicated enum value for completed orders:
enum Status { None, Created, Shipped, Delivered, Withdrawn, Refunded }

// in withdraw()
o.status = Status.Withdrawn;

---
