
**Week**: 4
**Researcher**: [NVN404](https://github.com/NVN404) + [yun0hu](https://x.com/yun0hu)
**Severity**: Low
**Category**: Economic / Resource Leak
**Affected function**: `stake_v2::stake`

## Description

The `UserPosition` PDA in `stake_v2` is created via `init_if_needed` during the first `stake` call, but **there is no instruction to close it**. The `unstake` instruction decrements the position's `franksol_balance` via `saturating_sub` but **never closes the account**.

```rust
// stake.rs:35-42 — created with init_if_needed, never closed
#[account(
    init_if_needed,
    payer = user,
    space = 8 + core::mem::size_of::<UserPosition>(),
    seeds = [USER_POSITION_SEED, user.address().as_ref()],
    bump
)]
pub user_position: Account<UserPosition>,
```

## Impact
The rent-exempt SOL (~0.0011 SOL) for every user who ever interacts with the protocol is permanently locked. For a protocol with high user churn, this results in an ever-growing pool of "dead" capital trapped in accounts that can never be recovered or reused.

## Proof of Concept

1. User stakes 10 SOL. `UserPosition` created, user pays ~0.00111 SOL rent.
2. User unstakes all frankSOL.
3. `user_position.franksol_balance = 0`, `user_position.sol_deposited = 10 SOL`.
4. No instruction exists to close the `UserPosition` and recover rent.
5. Repeat for 10,000 users → ~11.1 SOL permanently locked in unused accounts.

## Recommended Fix

Add a `close_position` instruction:

```rust
#[derive(Accounts)]
pub struct ClosePosition {
    #[account(mut)]
    pub user: Signer,
    #[account(
        mut,
        close = user,
        seeds = [USER_POSITION_SEED, user.address().as_ref()],
        bump = user_position.bump,
        constraint = user_position.franksol_balance.get() == 0 @ StakeError::InvalidAmount
    )]
    pub user_position: Account<UserPosition>,
}

pub fn close_position(_ctx: &mut Context<ClosePosition>) -> Result<()> {
    Ok(())
}

---


**Week**: 4
**Researcher**:  [NVN404](https://github.com/NVN404) + [yun0hu](https://x.com/yun0hu)
**Severity**: Medium 
**Category**: Rent drain / vault PDA destruction
**Affected function**: `yield_generator::withdraw` — lamport mutation at `withdraw.rs:60-77`

## Description

The `withdraw` instruction transfers lamports from the yield vault using direct `set_lamports` mutation rather than a system `Transfer` CPI:

```rust
// withdraw.rs:60-77
let mut vault = *ctx.accounts.yield_vault.account();
require!(
    vault.lamports() >= total_out,
    YieldError::InsufficientVaultBalance
);
vault.set_lamports(
    vault.lamports().checked_sub(total_out).ok_or(YieldError::MathOverflow)?,
);
```

The balance check `vault.lamports() >= total_out` does not account for rent exemption. The yield vault is a zero-data account owned by yield_generator. Its rent-exempt minimum is `Rent::get()?.minimum_balance(0)` ≈ 890,880 lamports.

If `total_out` consumes enough lamports to bring the vault below the rent-exempt threshold, the Solana runtime garbage-collects the vault account. Once destroyed:
- All subsequent `deposit` calls fail (vault PDA no longer exists, seeds check fails).
- `initialize` cannot re-create it because the `state` PDA already exists (Anchor `init` constraint rejects).
- The entire yield_generator is permanently bricked.

## Impact

When an operator withdraws their full `principal + accrued_reward` and the reward portion consumes lamports that were part of the vault's rent exemption balance, the vault account is destroyed. The yield_generator program is permanently bricked. Any remaining positions lose their principal.

## Proof of Concept

1. Yield generator initialized. Vault has ~890,880 lamports (rent-exempt minimum).
2. External funding adds exactly 100 SOL to the vault for yield.
3. Operator deposits 100 SOL. Vault = 200,000,890,880 lamports.
4. After 1 year at 10% APY: reward = 10 SOL. `total_out = 110 SOL`.
5. Vault after = 200,000,890,880 - 110,000,000,000 = 90,000,890,880. Still rent-exempt. OK.
6. Edge case: if vault only had 110,000,100,000 lamports total, `total_out = 110 SOL`, vault after = 100,000. Below rent-exempt (~890,880). Vault is destroyed.

## Recommended Fix

```rust
let rent = Rent::get()?.minimum_balance(0);
require!(
    vault.lamports().checked_sub(total_out).unwrap_or(0) >= rent,
    YieldError::InsufficientVaultBalance
);
```
---

**Week**: 4
**Researcher**: [NVN404](https://github.com/NVN404) + [yun0hu](https://x.com/yun0hu)
**Severity**: Medium
**Category**: Phantom liquidity / accounting based solvency check
**Affected function**: `stake_v2::unstake` — liquidity guard at `unstake.rs:88-89`

## Description

The liquidity guard in `unstake` checks tracked accounting instead of actual vault lamports:

```rust
// unstake.rs:88-89
let liquid_sol = checked_sub_u64(pool.total_sol.get(), pool.deployed_sol.get())?;
require!(sol_out <= liquid_sol, StakeError::InsufficientVaultBalance);
```

When `pool.total_sol` is **inflated** relative to actual vault lamports (which happens via the inverted PnL bug when the strategy has a loss), the liquidity check passes but the subsequent `Transfer` CPI reverts because the vault doesn't actually hold enough SOL. Users see a confusing "transaction failed" error despite passing the protocol's liquidity check.

When `pool.total_sol` is **deflated** relative to actual vault lamports (which happens via the inverted PnL bug when the strategy profits), the liquidity check blocks unstakes that the vault could actually service. Users are needlessly locked out.

The handler performs transfers sequentially , first `user_sol_out` to user, then fee transfers. No aggregate vault balance check is performed. If the vault has enough for the user transfer but not the fees, the user transfer "succeeds" but the fee transfer reverts, rolling back everything.

## Impact

The accounting only liquidity check creates a false sense of solvency.  users can be blocked from unstaking even when the vault has sufficient lamports, or the check can pass while the vault is actually insolvent. In both cases, the UX is broken and user funds may be inaccessible.

## Proof of Concept

1. Pool: `total_sol = 1000`, `deployed_sol = 500`, vault = 500 SOL + rent.
2. Fund manager calls `withdraw_from_yield(principal=500, yield=50)`. Strategy had positive yield.
3. CPI returns 550 SOL to vault. `total_return = 550`, `pnl = +50`.
4. `pool.total_sol -= 50 = 950`. But vault now has 1050 SOL + rent. `deployed_sol = 0`.
5. `liquid_sol = 950 - 0 = 950`. A user trying to unstake 1000 SOL: `sol_out = 1000 > 950`. Blocked.
6. But vault actually has ~1050 SOL. The user should be able to redeem.

## Recommended Fix

Add a real vault balance check alongside the accounting check:

```rust
let liquid_sol = checked_sub_u64(pool.total_sol.get(), pool.deployed_sol.get())?;
require!(sol_out <= liquid_sol, StakeError::InsufficientVaultBalance);

let rent = Rent::get()?.minimum_balance(0);
require!(
    ctx.accounts.vault.lamports().saturating_sub(rent) >= sol_out,
    StakeError::InsufficientVaultBalance
);
```
---

**Week**: 4
**Researcher**:  [NVN404](https://github.com/NVN404) + [yun0hu](https://x.com/yun0hu)
**Severity**: High
**Category**: Missing CPI target validation / arbitrary program substitution
**Affected function**: `stake_v2::withdraw_from_yield` — `yield_generator_program` at `withdraw_from_yield.rs:34-35`

## Description

In `withdraw_from_yield`, the `yield_generator_program` account is declared as an `UncheckedAccount` with **no `address` constraint**:

```rust
// withdraw_from_yield.rs:34-35
/// CHECK: External yield program invoked via CPI.
pub yield_generator_program: UncheckedAccount,
```

Compare to `deploy_to_yield.rs:40-41` which **does** validate the address:

```rust
// deploy_to_yield.rs:40-41
#[account(address = yield_generator::id())]
pub yield_generator_program: UncheckedAccount,
```

The fund manager can substitute a **malicious program** as the CPI target. The `yield_state` and `yield_vault` accounts have `seeds::program = yield_generator_program.address()` constraints, but since `yield_generator_program` itself is unchecked, those seed validations derive against the attacker's fake program — the attacker deploys their own program with matching PDAs.

The malicious program can:
1. Accept the CPI call with the expected instruction layout.
2. Increase the `vault` lamports by an arbitrary amount (transferring lamports from an attacker-funded account into the stake_v2 vault).
3. Return `Ok(())`.

After the CPI, the handler computes PnL from the vault delta:
```rust
let vault_after = ctx.accounts.vault.lamports();
let total_return = checked_sub_u64(vault_after, vault_before)?;
let pnl_i128 = (total_return as i128) - (principal_returned as i128);
```

The attacker controls `total_return` arbitrarily, enabling them to inflate or deflate `pool.total_sol` and thus manipulate the frankSOL price.

## Impact

The fund manager can invoke `withdraw_from_yield` against a fake program that returns arbitrary vault deltas, enabling manipulation of `pool.total_sol`. This can inflate the frankSOL price (allowing the fund manager to unstake at inflated rate) or deflate it (devaluing all other stakers). Direct path to extracting value from the pool.

## Proof of Concept

1. Fund manager deploys `FakeYield` program.
2. `FakeYield::withdraw` transfers 1000 SOL from an attacker account into the stake_v2 vault PDA.
3. Fund manager calls `withdraw_from_yield` with `yield_generator_program = FakeYield`, `principal_returned = 1`.
4. `vault_after - vault_before = 1000`. `pnl_i128 = 999`.
5. With PnL fix: `pool.total_sol += 999`. FrankSOL price inflates massively.
6. Fund manager unstakes their frankSOL at inflated rate, draining other users' SOL.

## Recommended Fix

```rust
#[account(address = yield_generator::id())]
pub yield_generator_program: UncheckedAccount,
```


----


**Week**: 4
**Researcher**:  [NVN404](https://github.com/NVN404) + [yun0hu](https://x.com/yun0hu)
**Severity**: High
**Category**: Accounting desync / position mirror drift via SPL transfers
**Affected function**: `stake_v2::unstake` — position balance tracking at `unstake.rs:135-136`

## Description

`unstake` decrements the on-chain `user_position.franksol_balance` using `saturating_sub`, while the pool-level `pool.franksol_supply` uses strict `checked_sub`. The `user_position.franksol_balance` mirror is **never validated** against the actual SPL burn amount — unstake burns whatever `franksol_in` the user provides, subtracts it from `pool.franksol_supply`, and then silently floors the position balance to zero if `franksol_in` exceeds the tracked `franksol_balance`.

The position balance is a mirror (per PROGRAM_GUIDE.md:104 — "mirror of ATA"), but it is never enforced as a ceiling on redemptions. A user can:

1. Receive frankSOL via SPL `Transfer` from another user (B → A).
2. Call `unstake(franksol_in)` where `franksol_in` is their full ATA balance (which exceeds `user_position.franksol_balance` because the position only tracks mints-to-this-user, not incoming transfers).
3. The SPL burn succeeds (user holds the tokens), `pool.franksol_supply -= franksol_in` succeeds, but `user_position.franksol_balance` uses `saturating_sub` and silently clips to 0.

The inverse problem: user B sent tokens away but their `user_position.franksol_balance` was never decremented. User B retains a phantom balance in their position that cannot be redeemed (no tokens in ATA).

Additionally, `user_position.sol_deposited` (cost basis) is **never decremented on unstake** — confirmed at `unstake.rs:129-136` where only `franksol_balance` is touched. It's a monotonically increasing accumulator with no protocol utility.

```rust
// unstake.rs:135-136 — saturating_sub silently absorbs over-redemptions
user_position.franksol_balance =
    PodU64::from(user_position.franksol_balance.get().saturating_sub(franksol_in));
```

## Impact

The `saturating_sub` means a user who received transferred frankSOL can redeem more than their tracked position without reverting. The position balance mirror drifts permanently. While the SPL burn and pool math are correct at the protocol level, the on-chain position accounting is permanently corrupted. This is the accounting layer that `set_user_blacklist` and off-chain systems rely on. A user who transfers all their frankSOL away retains a phantom `franksol_balance` in their position that can never be cleared.

## Proof of Concept

1. Alice stakes 100 SOL via `stake()`. Gets 100 frankSOL. `alice_position.franksol_balance = 100`.
2. Bob stakes 100 SOL via `stake()`. Gets 100 frankSOL. `bob_position.franksol_balance = 100`.
3. Alice sends 100 frankSOL to Bob via SPL `Transfer` (off-protocol). No position update.
4. State: Alice ATA = 0 frankSOL, `alice_position.franksol_balance = 100` (stale). Bob ATA = 200 frankSOL, `bob_position.franksol_balance = 100`.
5. Bob calls `unstake(200)`. SPL burn burns 200 from Bob's ATA. `pool.franksol_supply -= 200`. Bob's `franksol_balance = saturating_sub(100, 200) = 0`. Bob redeems 200 SOL worth of value.
6. Alice calls `unstake(100)`. SPL burn **fails** — Alice has 0 tokens in her ATA. Alice's position shows balance of 100 forever.
7. Sum of all `franksol_balance` = 100 (Alice's phantom). Actual `pool.franksol_supply` = 0. Permanent desync.

## Recommended Fix

Enforce that `franksol_in` doesn't exceed the tracked position balance, and use `checked_sub`:

```rust
require!(
    user_position.franksol_balance.get() >= franksol_in,
    StakeError::InvalidAmount
);
user_position.franksol_balance =
    PodU64::from(checked_sub_u64(user_position.franksol_balance.get(), franksol_in)?);
```

---