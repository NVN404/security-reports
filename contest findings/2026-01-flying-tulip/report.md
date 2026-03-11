# Contest Finding Report — Flying Tulip

**Auditor:** [NVN404](https://github.com/NVN404)  
**Severity:** Medium  
**Finding ID:** M-1  

---

## M-1: Users will DoS new investments by exploiting `collateralSupply` accounting mismatch

### Summary
The missing reduction of `collateralSupply` in `withdrawFT()` will cause a denial of service for new investors. Existing PUT holders returning their FT tokens create "ghost collateral" that blocks the cap. This trapped space can only be slowly cleared by the multisig through the Circuit Breaker-limited `withdrawDivestedCapital()` function.

### Root Cause
In [`PutManager.sol#L442`](https://github.com/sherlock-audit/2026-01-flying-tulip-NVN404/blob/017f57b193641009c7b7933202b138d7948953d2/ftPUT/contracts/PutManager.sol#L436C9-L437C54), the `withdrawFT()` function increments `capitalDivesting` without reducing `collateralSupply`. 

The cap check at `PutManager.sol#L382-L384` uses raw `collateralSupply` without accounting for pending divested capital:

```solidity
// withdrawFT() - Line 442
capitalDivesting[token] += _capitalDivesting;
// collateralSupply[token] is NOT reduced
```

### Pre-conditions

#### Internal Pre-conditions
- Admin calls `setCollateralCap()` to set `collateralCap[token]` to be at least equal to or less than current `collateralSupply[token]` (e.g., cap set to 10M USDC).
- `collateralSupply[token]` is at or near `collateralCap[token]` from a previous sale round (e.g., 10M USDC invested).
- Admin calls `enableTransferable()` to allow users to call `withdrawFT()`.
- Users call `withdrawFT()`, queuing significant capital in `capitalDivesting[token]` (e.g., 5M USDC).

#### External Pre-conditions
- N/A

### Attack Path
When `withdrawFT` is called:
1. User returns FT tokens, which are burned from their position.
2. `ftAllocated` decreases and the PUT position is closed.
3. `capitalDivesting[token]` increases.
4. Capital stays in the vault (still earning yield).
5. `collateralSupply[token]` remains unchanged, even though the position is closed and the option is forfeited.

### Impact
- The protocol suffers a denial of service, preventing new investments for extended periods.
- New investment rounds are blocked indefinitely until queued withdrawals are cleared by the multisig, potentially halting protocol operations.

### Mitigation
Update `withdrawFT()` to reduce `collateralSupply` when capital is moved to the divesting state.

```solidity
function withdrawFT(...) external {
    // ... existing code ...
    
    collateralSupply[token] -= _capitalDivesting;  // ADD THIS LINE
    capitalDivesting[token] += _capitalDivesting;
    
    // ... rest of function ...
}
```
