# CollateralBorrowBank — Security Audit Report

---

## [CRITICAL-01] `repay()` assigns debt instead of subtracting — positions can never be fully closed

### Summary
The repay function contains an assignment operator where subtraction is required. This corrupts all debt accounting: users cannot fully repay loans, and in some cases the protocol silently forgives debt it should retain.

### Root Cause

```solidity
function repay() external payable {
    require(debtOf[msg.sender] >= msg.value, "Too much");
    debtOf[msg.sender] = msg.value; // ❌ should be -= msg.value
}
```

`debtOf[msg.sender] = msg.value` sets the debt *to* the repaid amount rather than reducing it by that amount.

### Attack Path

A user owing 1 ETH can call `repay{value: 1 wei}` and have their debt set to 1 wei, draining the protocol of nearly all owed ETH. Full repayment is also impossible — a user who sends the full 1 ETH has their debt set back to 1 ETH, meaning collateral can never be cleanly withdrawn.

### Recommended Fix

```solidity
debtOf[msg.sender] -= msg.value;
```

---

## [CRITICAL-02] Oracle decimals never consumed — borrow limit inflated by up to `10^8`

### Summary
`IPriceOracle` exposes a `decimals()` function that is declared but never called. Raw price is multiplied directly against collateral balance with no normalization, producing a `collateralValue` that is orders of magnitude larger than intended. This allows borrowers to drain the entire ETH reserve with minimal collateral.

### Root Cause

```solidity
function maxBorrow(address user) public view returns (uint256) {
    uint256 price = oracle.latestPrice();            // e.g. 2000e8 (Chainlink 8-dec)
    uint256 collateralValue = collateralOf[user] * price; // 1e18 * 2000e8 = 2e29
    return (collateralValue * LTV_BPS) / BPS;        // returns ~1e29 wei — 1e11 ETH
}
```

For a standard Chainlink feed (`decimals = 8`) and a standard 18-decimal ERC20, the borrow limit is inflated by `10^8`, effectively making it unlimited relative to any realistic ETH balance in the contract.

**Concrete example** — user deposits 1 WETH (1e18) as collateral, oracle reports $2000/ETH (2000e8):

```
// Buggy
collateralValue = 1e18 * 2000e8 = 2e29
maxBorrow       = (2e29 * 5000) / 10000 = 1e29 wei = 100,000,000,000 ETH

// Correct (after decimal normalization)
collateralValue = (1e18 * 2000e8 * 1e18) / 1e8 / 1e18 = 2000e18
maxBorrow       = (2000e18 * 5000) / 10000 = 1000e18 wei = 1 ETH  ✓
```

One deposit of 1 WETH lets the attacker claim a borrow limit of 100 billion ETH — total ETH in existence is ~120 million. The entire contract balance drains in one call.

### Attack Path

1. Deposit 1 wei of collateral token.
2. `maxBorrow` returns a number vastly larger than the contract's ETH balance.
3. Call `borrow(address(this).balance)` — full drain in one transaction.

### Recommended Fix

```solidity
function maxBorrow(address user) public view returns (uint256) {
    uint256 price = oracle.latestPrice();
    uint8 priceDecimals = oracle.decimals();
    uint8 tokenDecimals = IERC20Metadata(address(collateralToken)).decimals();

    // Normalize: collateralValue in 18-decimal ETH wei
    uint256 collateralValue = (collateralOf[user] * price * 1e18)
        / (10 ** priceDecimals)
        / (10 ** tokenDecimals);

    return (collateralValue * LTV_BPS) / BPS;
}
```

---

## [HIGH-01] Liquidation seizes 100% of collateral for any repayment amount — full theft via 1 wei

### Summary
The liquidation function allows a caller to repay an arbitrarily small amount of a user's debt while seizing their *entire* collateral balance. There is no proportional seizure calculation.

### Root Cause

```solidity
function liquidate(address user) external payable {
    require(debtOf[user] > maxBorrow(user), "Healthy");
    require(msg.value <= debtOf[user], "Too much");

    debtOf[user] -= msg.value;

    uint256 seized = collateralOf[user]; // ❌ always entire balance
    collateralOf[user] = 0;
    require(collateralToken.transfer(msg.sender, seized), "Transfer failed");
}
```

`seized` is unconditionally set to the full collateral balance regardless of how much debt was repaid.

### Attack Path

1. User has 10,000 USDC collateral and 4,000 ETH-equivalent debt. Price drops slightly — position becomes unhealthy.
2. Attacker calls `liquidate{value: 1}(user)` — repays 1 wei.
3. Attacker receives all 10,000 USDC. Net profit is essentially the entire collateral.

### Recommended Fix

Seize collateral proportional to debt repaid, plus a liquidation bonus:

```solidity
uint256 constant LIQUIDATION_BONUS_BPS = 500; // 5%

uint256 price = oracle.latestPrice();
uint8 priceDecimals = oracle.decimals();

// Convert repaid ETH to equivalent collateral units
uint256 seizable = (msg.value * (10 ** priceDecimals) * (BPS + LIQUIDATION_BONUS_BPS))
    / (price * BPS);

uint256 seized = seizable > collateralOf[user] ? collateralOf[user] : seizable;
collateralOf[user] -= seized;
debtOf[user] -= msg.value;
```

---

## [MEDIUM-01] No oracle staleness check — stale price enables bad loans or blocked liquidations

### Summary
`maxBorrow` and `liquidate` consume oracle price with no validation of when it was last updated. A stale or frozen oracle can be exploited to either over-borrow against an asset that has since devalued, or prevent legitimate liquidations of underwater positions.

### Root Cause
`IPriceOracle.latestPrice()` returns no timestamp. There is no heartbeat check anywhere in the contract.

### Attack Path

1. Oracle goes stale while collateral price has dropped 60%.
2. `maxBorrow` still uses the old inflated price.
3. User borrows at 50% LTV of the stale value — actually 80%+ of real value.
4. Oracle updates, position is immediately insolvent with no time for liquidators to respond.

### Recommended Fix

Extend the interface to return a timestamp and enforce a maximum staleness window:

```solidity
interface IPriceOracle {
    function latestPrice() external view returns (uint256 price, uint256 updatedAt);
}

uint256 public constant MAX_STALENESS = 1 hours;

// in maxBorrow and liquidate:
(uint256 price, uint256 updatedAt) = oracle.latestPrice();
require(block.timestamp - updatedAt <= MAX_STALENESS, "Stale price");
```

---

## [LOW-01] `borrow()` does not check contract ETH balance — misleading revert on underfunded state

### Summary
`borrow()` calls `payable(msg.sender).transfer(amountWei)` without first verifying `address(this).balance >= amountWei`. If the contract is underfunded, the transfer reverts with a generic low-level error rather than a clear protocol message.

### Recommended Fix

```solidity
require(address(this).balance >= amountWei, "Insufficient liquidity");
```