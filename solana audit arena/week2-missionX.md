
# [Week 2] [HIGH] Unconditionally Overwrites `trade_status`, Permanently Blocking Migration

## Finding

- **Week:** 2
- **Researcher:** yunohu + @yun0hu
- **Severity:** High
- **Category:** State machine logic / Locked funds
- **Affected function(s):** `accept_missionx`

## Description

In `accept_missionx` (`accept.rs:51`), the trading status is unconditionally forced to `Open`:

```rust
// accept.rs:51
missionx_state.trade_status = MissionxTradeStatus::Open;
```

During the buy instruction (`buy.rs:82-90`), when `reserve0` reaches `migration_threshold`, the program sets `trade_status = MigrationRequired`. This is the trigger that allows the executor to call `migrate`.

When a moderator rejects a player via `complete_missionx(is_successful=false)`, the mission status reverts to `Open` (`complete.rs:159`).

The bug: when a new player then accepts the reopened mission via `accept_missionx`, line 51 unconditionally overwrites `trade_status` from `MigrationRequired` back to `Open`. The migration trigger is permanently erased.

## Impact

All SOL deposited by buyers (`reserve0`) is stuck in the PDA forever. The executor can only call `migrate` when `trade_status == MigrationRequired`, but this state has been overwritten and cannot be restored (the threshold check in `buy` is `<=`, so if `reserve0` already equals the threshold, buying more won't re-trigger it since `effective_sol_spend` would be capped to `0`).

## Proof of Concept

### Step-by-Step Transaction Exploit Sequence

**Initial setup**

- A MissionX is created and approved (`trade_status = 0 (Closed)`).

**Step 1: Player 1 accepts the mission**

- Executor calls `accept_missionx`.
- Resulting state: `trade_status` becomes `1 (Open)` (trading begins).

**Step 2: Trading pushes past `migration_threshold`**

- A series of buys (or one large buy) happens on the AMM.
- The AMM `reserve0` goes past `migration_threshold` (e.g., hits `82.2 SOL`).
- In `buy.rs`, `reserve_threshold_met` evaluates to `true`.
- Resulting state: `trade_status` is updated to `2 (MigrationRequired)`.

**Step 3: The moderator rejects Player 1's submission**

- `complete_missionx(false)` is invoked by the moderator.
- `missionx_status` correctly goes back to `2 (Open)` so a new player can try.
- Crucially, the `82.2 SOL` stays locked in the PDA.

**Step 4: Bug trigger (a new player accepts)**

- Player 2 calls `accept_missionx` to attempt the resumed mission.
- At `accept.rs:51`, the protocol executes this line unconditionally:

```rust
missionx_state.trade_status = TradeStatus::Open;
```

- Resulting state: `trade_status` is overwritten from `2` back to `1`.

**Step 5: Funds become permanently locked**

- The total SOL is already `> migration_threshold`, so future buys never flip the threshold check again (because `effective_sol_spend` is capped to `0` once the threshold is crossed).
- The executor attempts to call `migrate()`, but `migrate.rs:35` strictly checks:

```rust
require2!(missionx_state.trade_status == TradeStatus::MigrationRequired...)
```

- Resulting transaction revert: `MissionxNotMigrationReady`.
- Total damage: `82.2 SOL` is permanently stranded in `Missionx.vault` with no migration path and no withdrawal path.

## Recommended Fix

Only set `trade_status = Open` if it's currently `Closed`:

```rust
if missionx_state.trade_status == MissionxTradeStatus::Closed {
    missionx_state.trade_status = MissionxTradeStatus::Open;
}
```