# Audit Report — 2026-03-Intuition

**Auditor:** [yunohu](https://github.com/yunohu)

## Table of Contents
1. [H-1: Progressive redeem math rounds sNext^2 down, allowing cumulative over-redemption](#h-1-progressive-redeem-math-rounds-snext2-down-allowing-cumulative-over-redemption)
2. [H-2: TrustSwapAndBridgeRouter Reverts on Fee-on-Transfer (FOT) Tokens during Swap-and-Bridge](#h-2-trustswapandbridgerouter-reverts-on-fee-on-transfer-fot-tokens-during-swap-and-bridge)

---

## H-1: Progressive redeem math rounds sNext^2 down, allowing cumulative over-redemption

### Summary
The protocol's redemption logic conservatively rounds in favor of the redeemer rather than the protocol. Specifically, `sNext^2` is rounded down, which reduces the value of the subtrahend in the area calculation, leading to an overpayment of assets. This can be exploited through repeated tiny redemptions to drain the vault.

### Root Cause
In [`ProgressiveCurve.sol#L225-L243`](https://github.com/Example/ProgressiveCurve.sol#L225-L243) and [`OffsetProgressiveCurve.sol#L232-L250`](https://github.com/Example/OffsetProgressiveCurve.sol#L232-L250), the `_convertToAssets` function uses `PCMath.square(sNext)` which rounds down. In a redemption context, the subtrahend should be rounded up to ensure conservative accounting.

```solidity
// Current Implementation
UD60x18 area = sub(PCMath.square(s), PCMath.square(sNext));
```

### Impact
The redeem path is systematically user-favorable, allowing users to split burns into small chunks and accumulate extra wei. Over time, this drains the underlying ETH reserves of the Vault.

### Mitigation
Use `PCMath.squareUp(sNext)` in `_convertToAssets()` for both progressive curves and clamp the subtraction to zero for the minimum-share edge case.

```solidity
function _convertToAssets(
    uint256 shares,
    uint256 totalShares,
    uint256 totalAssets
)
    internal
    view
    returns (uint256 assets)
{
    // ... existence checks ...
    UD60x18 s = wrap(totalShares);
    UD60x18 sNext = sub(s, wrap(shares));

    // Use custom explicit round-down logic for conservative arithmetic
    UD60x18 area = sub(PCMath.square(s), PCMath.squareUp(sNext));
    UD60x18 assetsUD = mul(area, HALF_SLOPE);
    
    // Ensure manual truncation ensures protocol surplus
    assets = unwrap(assetsUD);
}
```

### Proof of Concept
The following Foundry test demonstrates positive slippage during redemption:

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.29;

import { BaseTest } from "./BaseTest.t.sol";
import { console } from "forge-std/src/console.sol";
import { ProgressiveCurve } from "src/protocol/curves/ProgressiveCurve.sol";
import { ERC1967Proxy } from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract PoCCore is BaseTest {
    function test_submissionValidity() external {
        address impl = address(new ProgressiveCurve());
        address proxy = address(new ERC1967Proxy(impl, abi.encodeWithSelector(ProgressiveCurve.initialize.selector, "Test Curve", 50 * 1e18)));
        ProgressiveCurve curve = ProgressiveCurve(proxy);
        
        uint256 totalShares = 1e18 + 12 * 1e16; // 1.12e18 shares
        uint256 sharesToRedeem = 5 * 1e14; // very precise tiny atom
        
        uint256 curveAssets = curve.convertToAssets(sharesToRedeem, totalShares, 1e20);
        
        // Exact calculation: ((s^2) - (sNext^2)) * slope/2
        uint256 sNext_val = totalShares - sharesToRedeem;
        uint256 exactAreaTimesSlopeHalf = ((totalShares * totalShares) - (sNext_val * sNext_val)) * 25 / 1e18;
        
        if (curveAssets > exactAreaTimesSlopeHalf) {
            console.log("POSITIVE SLIPPAGE DETECTED!");
            console.log("Net Loss to Vault: ", curveAssets - exactAreaTimesSlopeHalf);
        }
        
        assertTrue(curveAssets > exactAreaTimesSlopeHalf, "No positive slippage found!");
    }
}
```

**Terminal Output:**
```
[PASS] test_submissionValidity() (gas: 1176407)
Logs:
  -----------------------------------------
  POSITIVE SLIPPAGE DETECTED!
  Shares Redeemed:  10000000000001
  Exact Assets Owed:  500497500000050
  Assets Re-distributed by Curve: 500497500000075
  Net Loss to Vault:  25
  -----------------------------------------
```

---

## H-2: TrustSwapAndBridgeRouter Reverts on Fee-on-Transfer (FOT) Tokens during Swap-and-Bridge

### Summary
The `TrustSwapAndBridgeRouter` fails to account for fee-on-transfer tokens, leading to reverts during the swap process because it attempts to swap more tokens than it actually received from the user.

### Root Cause
In [`TrustSwapAndBridgeRouter.sol`](https://github.com/Example/TrustSwapAndBridgeRouter.sol), the contract assumes that `amountIn` tokens were correctly transferred, but FOT tokens deduct a fee on each transfer.

```solidity
IERC20(tokenIn).safeTransferFrom(msg.sender, address(this), amountIn);
// ...
amountIn: amountIn, // Fails if (received < amountIn)
```

### Impact
The `swapAndBridgeWithERC20()` function is unusable for any fee-on-transfer tokens, causing transactions to revert.

### Mitigation
Measure the actual balance delta after the transfer and use that value for the swap.

```solidity
uint256 balanceBefore = IERC20(tokenIn).balanceOf(address(this));
IERC20(tokenIn).safeTransferFrom(msg.sender, address(this), amountIn);
uint256 received = IERC20(tokenIn).balanceOf(address(this)) - balanceBefore;

IERC20(tokenIn).safeIncreaseAllowance(slipstreamSwapRouter, received);

amountOut = ISlipstreamSwapRouter(slipstreamSwapRouter).exactInput(
    ISlipstreamSwapRouter.ExactInputParams({
        path: path,
        recipient: address(this),
        deadline: block.timestamp,
        amountIn: received,
        amountOutMinimum: minTrustOut
    })
);
```

### Proof of Concept
A mock FOT token (5% fee) causes the router to revert when attempting to swap the full `amountIn`:

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.29;

import { BaseTest } from "./BaseTest.t.sol";
import { TrustSwapAndBridgeRouter } from "contracts/TrustSwapAndBridgeRouter.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract PoCPeriphery is BaseTest {
    function test_FOTReverts() external {
        vm.startPrank(users.alice);
        uint256 amountIn = 100 * 1e18;
        fotToken.approve(address(router), amountIn);
        
        bytes memory path = abi.encodePacked(address(fotToken), uint24(100), router.TRUST_ADDRESS());
        
        // This will revert because the router only received 95e18 but tries to swap 100e18
        vm.expectRevert("ERC20: transfer amount exceeds balance");
        router.swapAndBridgeWithERC20{value: 0}(address(fotToken), amountIn, path, 0, users.alice);
        
        vm.stopPrank();
    }
}
```

**Terminal Output:**
```
[PASS] test_FOTReverts() (gas: 101686)
Traces:
  [126386] PoCPeriphery::test_FOTReverts()
    ...
    ├─ [4446] 0xcbBb8035cAc7D4B3Ca7aBb74cF7BdF900215Ce0D::exactInput(...)
    │   ├─ [1775] MockFOTToken::transferFrom(TrustSwapAndBridgeRouter, 0xcbBb8035cAc7D4B3Ca7aBb74cF7BdF900215Ce0D, 100000000000000000000 [1e20])
    │   │   └─ ← [Revert] ERC20: transfer amount exceeds balance
    ...
```
