## finding


**Week**: 3
**Researcher**: [yunohu](https://github.com/NVN404) + [@yun0hu](https://x.com/yun0hu)
**Severity**: Medium
**Category**: Logic Error / Architectural Weakness
**Affected function**: `init_token` [init_token.rs]

## Description

In the `init_token` instruction, the validation for the `market` account at **Line 121** lacks a PDA `seeds` constraint. While Anchor’s `Account<'info, Market>` check ensures the account belongs to the program and matches the `Market` discriminator, it does not verify if the account address matches the intended version identity. This is a clear case of **Inconsistent Validation**

```rust
// init_token.rs:121
pub market: Account<'info, Market>, // <--- Missing seeds/bump constraint
```

 while the developer correctly implemented this architectural requirement in `buy.rs` and `sell.rs`, they omitted it in the init instruction.

```rust
// COMPARISON: Correct validation in buy.rs:180-183
#[account(
    seeds = [b"market".as_ref(), market_version.to_le_bytes().as_ref()],
    bump
)]
pub market: Account<'info, Market>,
```

Because this check is missing in `init_token`, the instruction will accept **any** previously initialized `Market` account. The provided market's public key is then permanently recorded into the `BondingCurve` state at **Line 37**:

```rust
// init_token.rs:37
ctx.accounts.bonding_curve.market = market.key();
```

this creates a permanent link between the token and the chosen Market version. All future trading activity and the graduation process will derive global parameters (fees, thresholds) from this potentially deprecated or malicious market account.

## Proof of Concept 

**1. Protocol Setup:**
- **Market v1 (Deprecated)**: `trading_fee_bps` = 50 (0.5%).
- **Market v2 (Current)**: `trading_fee_bps = 100` (1.0%).
- The admin intends that all tokens launched after the v2 upgrade must utilize the 1.0% fee structure.

**2. Attack Execution:**
- **Step 1 (Targeting)**: An attacker identifies the on-chain address of the deprecated **Market v1** PDA.
- **Step 2 (The Launch)**: The attacker calls `init_token` to launch a new coin.
- **Step 3 (Unintended Account Selection)**: The attacker passes the address of the deprecated **Market v1** PDA into the instruction instead of the required **Market v2** address.
- **Step 4 (Validation Pass)**: Because `init_token` at Line 121 only checks the account type/owner, it accepts the v1 address as a valid `Market` account.

**3. The Result:**
- the `BondingCurve` state is initialized with `market = Market_V1_Address`.
- **Permanent Bypass**: For the entire lifecycle of this bonding curve, every "Buy" and "Sell" trade will only trigger a **0.5% fee**, regardless of protocol-level upgrades.
- The protocol loses 50% of its intended revenue for this token due to the lack of version enforcement at the point of origin.

## Recommended Fix

Add a `market_version` argument to `init_token` and enforce PDA seed validation in the `Accounts` struct to ensure the provided address is mathematically verified.

```rust
// init_token.rs
#[derive(Accounts)]
#[instruction(params: InitTokenParams, market_version: u16)]
pub struct InitializeAndMint<'info> {
    // ...
    #[account(
        // Mathematically verify the address against the specific version seeds
        seeds = [b"market".as_ref(), market_version.to_le_bytes().as_ref()],
        bump
    )]
    pub market: Account<'info, Market>,
    // ...
}
```

---

## finding


**Week**: 3
**Researcher**: [yunohu](https://github.com/NVN404) + [@yun0hu](https://x.com/yun0hu)
**Severity**: Informational / low
**Category**: Input validation ordering / Compute Optimization
**Affected function**: `buy_tokens`, `sell_tokens`

### Description

Both `buy_tokens` and `sell_tokens` validate zero-amount inputs **after** performing the swap computation:

```rust
// buy.rs lines 31-45
let sol_amount: u128 = sol_amount.into();
let SwapWithoutFeesResult { source_amount_swapped, destination_amount_swapped }
    = swap(sol_amount, sol_reserves, token_reserves).unwrap();  // Swap computed first
let new_sol_amount: u64 = source_amount_swapped.try_into().unwrap();
let token_amount: u64 = destination_amount_swapped.try_into().unwrap();

if sol_amount == 0 {  // Zero check AFTER swap
    return Err(TokenError::TokenAmountZero.into());
}
```

```rust
// sell.rs lines 29-43 — identical pattern
let token_amount: u128 = token_amount.into();
let SwapWithoutFeesResult { ... } = swap(token_amount, token_reserves, sol_reserves).unwrap();
// ...
if token_amount == 0 {  // Zero check AFTER swap
    return Err(TokenError::TokenAmountZero.into());
}
```

While Solana transactions are atomic (the revert prevents state corruption), the swap computation runs unnecessary math, wastes compute units, and the `.unwrap()` on the swap result could panic on edge-case inputs before the validation runs.

### Impact

- **Wasted Compute Units**: Runs complex curve math on every zero-amount call.
- **Defense-in-Depth**: As a best practice, input validation should always precede computation.

### Recommended Fix

Move zero-amount checks to the beginning of the function, before the swap computation:

```rust
pub fn buy_tokens(ctx: Context<BuyTokens>, sol_amount: u64, ...) -> Result<()> {
    if sol_amount == 0 {
        return Err(TokenError::TokenAmountZero.into());
    }
    // ... then proceed with swap
}
```

```rust
pub fn sell(ctx: Context<Sell>, token_amount: u64, ...) -> Result<()> {
    if token_amount == 0 {
        return Err(TokenError::TokenAmountZero.into());
    }
    // ... then proceed with swap
}
```

---


## Finding 

**Week**: 3
**Researcher**: [yunohu](https://github.com/NVN404) + [@yun0hu](https://x.com/yun0hu)
**Severity**: High
**Category**: Missing validation / State machine violation
**Affected functions**: `withdraw_tokens_fee`, `process_completed_curve`

## Description

There are two critical failures that chain together to permanently lock the graduated SOL in the bonding curve PDA:

**1. Arbitrary Token Withdrawal:**
The `withdraw_tokens_fee` instruction accepts an arbitrary `tokens_amount` from the caller:
```rust
pub fn withdraw_tokens_fee(
    ctx: Context<WithdrawTokensFee>,
    tokens_amount: u64, // Unvalidated user input
    _market_version: u16,
) -> Result<()> {
    // ... missing tokens_amount <= market.tokens_fee_amount validation
    transfer_checked(..., tokens_amount, ...)?;
}
```
Because the amount is never validated against `market.tokens_fee_amount`, the market authority can drain **all** remaining tokens from the curve's ATA after graduation, rather than just the reserved protocol fees.

**2. Missing Execution Guard causes "Chained Lock":**
The `process_completed_curve` instruction, which is responsible for migrating the SOL out of the curve PDA, relies on transferring the remaining tokens to the admin ATA in the same transaction. Furthermore, it lacks a `processed` flag.

```rust
// In process_completed_curve
let token_amount = bonding_curve.real_token_reserves.checked_sub(market.tokens_fee_amount).unwrap();
transfer_checked(..., token_amount, ...)?; // Reverts if ATA is empty
// ... SOL lamports drained here
```

## Impact

Because `withdraw_tokens_fee` allows the curve to be drained early, and `process_completed_curve` forces the token transfer to occur alongside the SOL drain, an attacker or careless admin can accidentally **brick the entire curve**.


## Proof of concept


1.  **Target Met**: A bonding curve successfully reaches its SOL target .
2.  **The Trigger (Drain)**: The Market Authority (admin) calls `withdraw_tokens_fee` with a massive amount. Due to a missing validation , they drain the **entire balance** of the Curve's Token Account.
3.  **The Migration Attempt**: The admin later attempts to call `process_completed_curve` to recover the ~85 SOL accumulated in the PDA.
4.  **Transaction Execution**:
    *   **Sub-Instruction A (Tokens)**: The program calculates the `token_amount` based on state (not actual balance). It issues a `transfer_checked` CPI to the Token Program.
    *   **The Failure**: The token Program sees a balance of **0** . It throws an `InsufficientFunds` error.
    *   **The Atomic Revert**: Because the CPI failed, the Anchor instruction returns an error. The Solana runtime **halts execution immediately** and rolls back the transaction.
5.  **The Result (Permanent Lock)**: The code responsible for transferring the SOL is never reached. Because `process_completed_curve` is the only function authorized to move those lamports, and it will always fail at the token step, the SOL is permanently locked .

## Recommended Fix

This requires a two-part fix to secure the state machine:

1. Validate the withdrawal amount in `withdraw_tokens_fee`:
```rust
require!(tokens_amount <= market.tokens_fee_amount, TokenError::InvalidAmount);
```

2. Add a `processed` flag to `BondingCurve` and finalize the state in `process_completed_curve` to untangle the SOL migration from the token balance:
```rust
require!(!bonding_curve.processed, TokenError::AlreadyProcessed);
// ... after transfers
bonding_curve.processed = true;
```


---


## Finding 
---


**Week**: 3
**Researcher**: [yunohu](https://github.com/NVN404) + [@yun0hu](https://x.com/yun0hu)
**Severity**: Medium
**Category**: Logic bug / Fee payment ordering
**Affected function**: `sell_tokens` [sell.rs]

### Description 

The `sell_tokens` instruction contains a logic flaw in its execution sequence. It requires the seller to pay a trading fee from their **own wallet** via a `system_program::transfer` CPI **before** they receive the SOL proceeds from the swap.

  CPI performs an immediate balance check. The execution order here is:

1.  **Line 88–95 (The Bug)**: Pull `trading_fee` from `seller.lamports` -> `treasury.lamports`.
2.  **Line 97–99 (The Proceeds)**: Push `sol_amount` from `bonding_curve.lamports` -> `seller.lamports`.

the transaction will **revert immediately** if `seller.lamports` < `trading_fee`. Selling is meant to be a liquidity-generating event, but this implementation turns it into a liquidity-consuming event.

```rust
// BUG: This CPI fails if the user is "all-in" on tokens and low on SOL
let transfer_fee_cpi_context = CpiContext::new(
    ctx.accounts.system_program.to_account_info(),
    Transfer {
        from: ctx.accounts.seller.to_account_info(), // Pulls from seller wallet
        to: ctx.accounts.trading_fee_treasury.to_account_info(),
    },
);
transfer(transfer_fee_cpi_context, trading_fee)?; // Fails here if seller.lamports < fee

// This proceeds transfer never happens because of the revert above
**bonding_curve.to_account_info().try_borrow_mut_lamports()? -= sol_amount;
**recipient.to_account_info().try_borrow_mut_lamports()? += sol_amount;
```

### Impact

This creates a ** Liquidity Trap**. Users who spend their last bits of SOL to buy tokens  are effectively **locked out** of exiting their position. They cannot sell their tokens because they cannot pay the "pre-exit fee," effectively bricking their ability to recover their SOL.

### Proof of Concept (Transaction Failure Sequence)

1.  **Condition**: A user has 10,000 Tokens but only **5,000 lamports** (barely enough for gas).
2.  **Trade**: User tries to sell his tokens worth **1,000,000 lamports** (1 SOL).
3.  **Calculation**: Market has 1% fee -> **10,000 lamports** fee required.
4.  **Failure**:
    - The program reaches the fee transfer CPI.
    - Environment checks: `Seller Wallet Balance (5,000)` < `Requested Transfer (10,000)`.
    - **Result**: The transaction **reverts immediately**.
5.  **Conclusion**: The user is unable to access their 1,000,000 lamports because they lack the 5,000 lamports "gap" to cover the fee upfront.

### Recommended Fix

The protocol should **deduct** the fee from the proceeds within the same instruction, rather than charging it separately. The `bonding_curve` PDA should be the source for both the proceeds and the fee.


```rust
// Calculate fee
let trading_fee = calculate_fee(sol_amount.into(), trading_fee_bps, 10000).unwrap() as u64;
let net_sol_amount = sol_amount.checked_sub(trading_fee).unwrap();

// Transfer net proceeds to seller from bonding curve
**bonding_curve.to_account_info().try_borrow_mut_lamports()? -= sol_amount;
**recipient.to_account_info().try_borrow_mut_lamports()? += net_sol_amount;

// Transfer fee to treasury from bonding curve (not seller)
**trading_fee_treasury.try_borrow_mut_lamports()? += trading_fee;
```

---


