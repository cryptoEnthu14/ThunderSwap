# ThunderSwap — Smart Contract Design

**Version**: 1.0
**Status**: Design Phase — ready for implementation
**Framework**: Anchor 0.31.x (Rust)
**Chain**: Solana
**Related**: [THUNDERSWAP_ARCHITECTURE.md](THUNDERSWAP_ARCHITECTURE.md)

This document is the authoritative specification for the ThunderSwap Anchor program. A Rust developer should be able to implement the full program from this document without making any design decisions.

---

## Table of Contents

1. [Program Structure](#1-program-structure)
2. [Account Structures](#2-account-structures)
3. [PDA Seeds Reference](#3-pda-seeds-reference)
4. [Math Module](#4-math-module)
5. [Instructions](#5-instructions)
6. [Events](#6-events)
7. [Error Codes](#7-error-codes)
8. [Security Invariants](#8-security-invariants)
9. [CPI Integration with ThunderLaunch PLP](#9-cpi-integration-with-thunderlaunch-plp)
10. [Testing Checklist](#10-testing-checklist)

---

## 1. Program Structure

```
programs/thunderswap/
├── Cargo.toml
├── Xargo.toml
└── src/
    ├── lib.rs                              # declare_id!, mod declarations, instruction dispatch
    ├── state.rs                            # GlobalConfig, AmmPool, LpBurnRecord structs
    ├── errors.rs                           # ThunderSwapError enum
    ├── math.rs                             # integer_sqrt, constant_product, TWAP, fee math
    └── instructions/
        ├── mod.rs
        ├── initialize_global_config.rs
        ├── create_pool.rs                  # Public permissionless path
        ├── create_pool_from_graduation.rs  # Privileged graduation path
        ├── swap.rs
        ├── add_liquidity.rs
        ├── remove_liquidity.rs
        ├── claim_creator_fees.rs
        └── update_global_config.rs
```

**Program ID**: Determined at first `anchor keys list` — store in `declare_id!` and `Anchor.toml`.

---

## 2. Account Structures

### 2.1 GlobalConfig

One per program. Controls global fee rates, circuit breaker, and the graduation authority.

**PDA Seeds**: `[b"global-config"]`

```rust
#[account]
pub struct GlobalConfig {
    pub authority: Pubkey,               // Squads 3-of-5 multisig — owns this config
    pub graduation_authority: Pubkey,    // Only this wallet can call create_pool_from_graduation
    pub protocol_fee_vault: Pubkey,      // Receives 15 bps protocol fees (system account)
    pub lp_fee_bps: u16,                 // 20 — stays in pool (auto-compounds)
    pub protocol_fee_bps: u16,           // 15 — extracted to protocol_fee_vault
    pub creator_fee_bps: u16,            // 15 — extracted to creator_fee_vault
    pub min_pool_sol_lamports: u64,      // 5_000_000_000 (5 SOL) — public pool minimum
    pub paused: bool,                    // Global circuit breaker
    pub bump: u8,
}

impl GlobalConfig {
    pub const LEN: usize = 8    // discriminator
        + 32 + 32 + 32          // authority, graduation_authority, protocol_fee_vault
        + 2 + 2 + 2             // fee bps (3 fields)
        + 8                     // min_pool_sol_lamports
        + 1 + 1;                // paused, bump
    // = 122 bytes
}
```

> **Note on `graduation_authority`**: This is the Squads multisig PDA (same as `authority` for v1 launch). It is stored separately so that in v2 the PLP program's graduation authority PDA can be substituted here without changing governance. For v1, set both `authority` and `graduation_authority` to the Squads multisig.

---

### 2.2 AmmPool

One per token. Holds all pool state. Graduated pools have `lp_burned = true` permanently.

**PDA Seeds**: `[b"amm-pool", token_mint.key().as_ref()]`

```rust
#[account]
pub struct AmmPool {
    pub token_mint: Pubkey,                      // SPL token mint address
    pub creator: Pubkey,                          // Token creator (receives creator fees)
    pub sol_reserve: u64,                         // Pool SOL in lamports (mirrors sol_vault balance)
    pub token_reserve: u64,                       // Pool token balance (raw units)
    pub lp_mint: Pubkey,                          // LP token mint address
    pub lp_supply: u64,                           // Total LP tokens in circulation
    pub creator_fees_accumulated: u64,            // Lifetime creator fees earned (lamports)
    pub locked: bool,                             // Per-pool emergency pause
    pub graduated_from_plp: bool,                 // True = seeded via graduation CPI
    pub lp_burned: bool,                          // True = LP tokens permanently burned
    pub lp_burn_tx_sig: [u8; 64],                // LP burn tx signature bytes (audit record)
    // TWAP — slot-weighted cumulative prices (UQ64.64 fixed-point: price × 2^32)
    pub twap_last_slot: u64,
    pub twap_sol_per_token_cumulative: u128,
    pub twap_token_per_sol_cumulative: u128,
    pub bump: u8,
    pub sol_vault_bump: u8,
    pub token_vault_authority_bump: u8,
    pub lp_mint_authority_bump: u8,
    pub creator_fee_vault_bump: u8,
}

impl AmmPool {
    pub const LEN: usize = 8      // discriminator
        + 32 + 32                 // token_mint, creator
        + 8 + 8                   // sol_reserve, token_reserve
        + 32 + 8                  // lp_mint, lp_supply
        + 8                       // creator_fees_accumulated
        + 1 + 1 + 1               // locked, graduated_from_plp, lp_burned
        + 64                      // lp_burn_tx_sig
        + 8 + 16 + 16             // TWAP fields
        + 1 + 1 + 1 + 1 + 1;     // bumps
    // = 260 bytes
}
```

---

### 2.3 LpBurnRecord

Written once per graduated pool in the same transaction that burns LP tokens. On-chain proof of the burn.

**PDA Seeds**: `[b"lp-burn-record", amm_pool.key().as_ref()]`

```rust
#[account]
pub struct LpBurnRecord {
    pub pool: Pubkey,
    pub token_mint: Pubkey,
    pub lp_mint: Pubkey,
    pub amount_burned: u64,
    pub burn_slot: u64,
    pub bump: u8,
}

impl LpBurnRecord {
    pub const LEN: usize = 8 + 32 + 32 + 32 + 8 + 8 + 1; // = 121 bytes
}
```

> The burn transaction signature is stored in `AmmPool.lp_burn_tx_sig` (64 bytes). The indexer reads this to populate `lp_burn_records.tx_sig` in the database.

---

### 2.4 System Accounts (Lamport PDAs)

These accounts hold lamports only. They are system-owned. `invoke_signed` with their seeds transfers SOL out.

| Account | Seeds | Purpose |
|---|---|---|
| `sol_vault` | `[b"sol-vault", token_mint]` | Holds pool SOL reserves |
| `creator_fee_vault` | `[b"creator-fee-vault", token_mint]` | Accumulates creator fees |

### 2.5 SPL Token Accounts

| Account | Owner PDA Seeds | Purpose |
|---|---|---|
| `token_vault` (ATA) | `[b"token-vault-authority", token_mint]` | Holds pool token reserves |
| `lp_mint_authority` | `[b"lp-mint-authority", token_mint]` | Signs LP mint/burn CPIs |

---

## 3. PDA Seeds Reference

| Account | Seeds | Notes |
|---|---|---|
| `GlobalConfig` | `["global-config"]` | One per program |
| `AmmPool` | `["amm-pool", token_mint]` | One per token |
| `LpBurnRecord` | `["lp-burn-record", amm_pool]` | One per graduated pool |
| `sol_vault` | `["sol-vault", token_mint]` | System lamport account |
| `creator_fee_vault` | `["creator-fee-vault", token_mint]` | System lamport account |
| `token_vault_authority` | `["token-vault-authority", token_mint]` | PDA that owns token ATA |
| `lp_mint_authority` | `["lp-mint-authority", token_mint]` | PDA that signs LP mint/burn |

---

## 4. Math Module

All math is in `src/math.rs`. No `f64` or `f32` anywhere in the program. All arithmetic uses `checked_*` operations.

### 4.1 Constants

```rust
/// Minimum LP tokens permanently locked at pool creation.
/// Prevents total LP supply from reaching zero (which would break pro-rata math).
pub const MINIMUM_LIQUIDITY: u64 = 1_000;

/// Basis points denominator.
pub const BPS_DENOMINATOR: u64 = 10_000;
```

### 4.2 Integer Square Root

Used for initial LP minting. Newton-Raphson convergence, no floating point.

```rust
/// Returns floor(sqrt(n)). Safe for all u128 values.
pub fn integer_sqrt(n: u128) -> u64 {
    if n == 0 {
        return 0;
    }
    let mut x: u128 = n;
    let mut y: u128 = (x.saturating_add(1)) >> 1;
    while y < x {
        x = y;
        y = (x + n / x) >> 1;
    }
    x as u64
}

#[cfg(test)]
mod tests {
    use super::integer_sqrt;
    #[test]
    fn test_sqrt() {
        assert_eq!(integer_sqrt(0), 0);
        assert_eq!(integer_sqrt(1), 1);
        assert_eq!(integer_sqrt(4), 2);
        assert_eq!(integer_sqrt(9), 3);
        assert_eq!(integer_sqrt(2), 1);      // floor
        assert_eq!(integer_sqrt(8), 2);      // floor
        assert_eq!(integer_sqrt(u128::MAX), u64::MAX);  // no overflow
    }
}
```

### 4.3 Constant Product — Buy Quote (SOL → Token)

```rust
/// Compute tokens_out for a given sol_net (after fees already deducted).
/// Returns ThunderSwapError::InsufficientLiquidity if pool cannot fill the order.
pub fn quote_buy(
    sol_reserve: u64,
    token_reserve: u64,
    sol_net: u64,
) -> Result<u64> {
    require!(sol_reserve > 0 && token_reserve > 0, ThunderSwapError::EmptyPool);

    let k: u128 = (sol_reserve as u128)
        .checked_mul(token_reserve as u128)
        .ok_or(ThunderSwapError::Overflow)?;

    let new_sol_reserve: u128 = (sol_reserve as u128)
        .checked_add(sol_net as u128)
        .ok_or(ThunderSwapError::Overflow)?;

    let new_token_reserve: u128 = k
        .checked_div(new_sol_reserve)
        .ok_or(ThunderSwapError::DivisionByZero)?;

    let tokens_out: u64 = (token_reserve as u128)
        .checked_sub(new_token_reserve)
        .ok_or(ThunderSwapError::InsufficientLiquidity)? as u64;

    Ok(tokens_out)
}
```

### 4.4 Constant Product — Sell Quote (Token → SOL)

```rust
/// Compute sol_out_gross (before protocol + creator fee deduction) for tokens_in.
pub fn quote_sell(
    sol_reserve: u64,
    token_reserve: u64,
    tokens_in: u64,
) -> Result<u64> {
    require!(sol_reserve > 0 && token_reserve > 0, ThunderSwapError::EmptyPool);

    let k: u128 = (sol_reserve as u128)
        .checked_mul(token_reserve as u128)
        .ok_or(ThunderSwapError::Overflow)?;

    let new_token_reserve: u128 = (token_reserve as u128)
        .checked_add(tokens_in as u128)
        .ok_or(ThunderSwapError::Overflow)?;

    let new_sol_reserve: u128 = k
        .checked_div(new_token_reserve)
        .ok_or(ThunderSwapError::DivisionByZero)?;

    let sol_out_gross: u64 = (sol_reserve as u128)
        .checked_sub(new_sol_reserve)
        .ok_or(ThunderSwapError::InsufficientLiquidity)? as u64;

    Ok(sol_out_gross)
}
```

### 4.5 Fee Calculation

```rust
pub struct FeeAmounts {
    pub protocol_fee: u64,
    pub creator_fee: u64,
    pub lp_fee: u64,      // informational only — stays in pool, never transferred
    pub net_amount: u64,  // amount after all fee deductions (used for AMM calculation on buy)
}

/// Compute fee breakdown for a given gross amount.
/// For BUY: gross = sol_in, net_amount goes into pool for AMM calculation.
/// For SELL: gross = sol_out_gross from AMM, net_amount is what user receives.
pub fn compute_fees(gross: u64, config: &GlobalConfig) -> Result<FeeAmounts> {
    let protocol_fee = gross
        .checked_mul(config.protocol_fee_bps as u64)
        .ok_or(ThunderSwapError::Overflow)?
        / BPS_DENOMINATOR;

    let creator_fee = gross
        .checked_mul(config.creator_fee_bps as u64)
        .ok_or(ThunderSwapError::Overflow)?
        / BPS_DENOMINATOR;

    let lp_fee = gross
        .checked_mul(config.lp_fee_bps as u64)
        .ok_or(ThunderSwapError::Overflow)?
        / BPS_DENOMINATOR;

    let net_amount = gross
        .checked_sub(protocol_fee)
        .and_then(|x| x.checked_sub(creator_fee))
        .ok_or(ThunderSwapError::Overflow)?;
    // Note: net_amount includes lp_fee because we only subtract protocol + creator

    Ok(FeeAmounts { protocol_fee, creator_fee, lp_fee, net_amount })
}
```

> **Fee flow summary**:
> - **Buy** (SOL → token): `fees = compute_fees(sol_in)`. Transfer `protocol_fee` to protocol vault, `creator_fee` to creator vault, `net_amount` to sol_vault. Run AMM on `net_amount`. LP fee is the 20 bps inside `net_amount` that stays permanently in `sol_vault`.
> - **Sell** (token → SOL): Run AMM on `tokens_in` to get `sol_out_gross`. `fees = compute_fees(sol_out_gross)`. Transfer `net_amount` to user, `protocol_fee` to protocol vault, `creator_fee` to creator vault. The LP fee (20 bps of gross) was already captured in the AMM calculation (k grows slightly with each trade via integer truncation).

### 4.6 TWAP Update

Uses slot-weighted cumulative prices. Stored as UQ32.32 fixed-point (price multiplied by `2^32`) in `u128` fields.

```rust
/// Update TWAP accumulators. Call at the start of every swap instruction,
/// BEFORE updating reserves (so the price reflects the state prior to this swap).
pub fn update_twap(pool: &mut AmmPool, current_slot: u64) -> Result<()> {
    if pool.sol_reserve == 0 || pool.token_reserve == 0 {
        pool.twap_last_slot = current_slot;
        return Ok(());
    }

    let slots_elapsed = current_slot.saturating_sub(pool.twap_last_slot);
    if slots_elapsed == 0 {
        return Ok(());
    }

    // Price as UQ32.32 fixed point (multiply by 2^32 to preserve decimal precision)
    // sol_per_token = sol_reserve / token_reserve
    let sol_per_token_fixed: u128 = ((pool.sol_reserve as u128) << 32)
        .checked_div(pool.token_reserve as u128)
        .ok_or(ThunderSwapError::DivisionByZero)?;

    let token_per_sol_fixed: u128 = ((pool.token_reserve as u128) << 32)
        .checked_div(pool.sol_reserve as u128)
        .ok_or(ThunderSwapError::DivisionByZero)?;

    pool.twap_sol_per_token_cumulative = pool.twap_sol_per_token_cumulative
        .saturating_add(sol_per_token_fixed.saturating_mul(slots_elapsed as u128));

    pool.twap_token_per_sol_cumulative = pool.twap_token_per_sol_cumulative
        .saturating_add(token_per_sol_fixed.saturating_mul(slots_elapsed as u128));

    pool.twap_last_slot = current_slot;
    Ok(())
}

/// Read TWAP over a window. Returns (sol_per_token_fixed, token_per_sol_fixed)
/// as UQ32.32. Divide by 2^32 to get actual price.
/// Caller provides cumulative values at start and end of window, plus slot counts.
pub fn read_twap(
    cumulative_end: u128,
    cumulative_start: u128,
    slots: u64,
) -> Result<u128> {
    require!(slots > 0, ThunderSwapError::DivisionByZero);
    Ok(cumulative_end
        .saturating_sub(cumulative_start)
        .checked_div(slots as u128)
        .ok_or(ThunderSwapError::DivisionByZero)?)
}
```

---

## 5. Instructions

### 5.1 `initialize_global_config`

Creates the singleton `GlobalConfig` PDA. Called once at program deployment.

**Accounts**:
```rust
#[derive(Accounts)]
pub struct InitializeGlobalConfig<'info> {
    #[account(
        init,
        payer = authority,
        space = GlobalConfig::LEN,
        seeds = [b"global-config"],
        bump
    )]
    pub global_config: Account<'info, GlobalConfig>,

    #[account(mut)]
    pub authority: Signer<'info>,    // Becomes GlobalConfig.authority (transfer to Squads after init)

    pub system_program: Program<'info, System>,
}
```

**Parameters**: `graduation_authority: Pubkey`, `protocol_fee_vault: Pubkey`

**Logic**:
```rust
global_config.authority = ctx.accounts.authority.key();
global_config.graduation_authority = graduation_authority;
global_config.protocol_fee_vault = protocol_fee_vault;
global_config.lp_fee_bps = 20;
global_config.protocol_fee_bps = 15;
global_config.creator_fee_bps = 15;
global_config.min_pool_sol_lamports = 5_000_000_000;  // 5 SOL
global_config.paused = false;
global_config.bump = ctx.bumps.global_config;
```

**Emits**: `GlobalConfigInitializedEvent`

---

### 5.2 `create_pool`

Permissionless pool creation. Anyone with ≥ 5 SOL per side can create a pool.

**Accounts**:
```rust
#[derive(Accounts)]
pub struct CreatePool<'info> {
    #[account(seeds = [b"global-config"], bump = global_config.bump)]
    pub global_config: Account<'info, GlobalConfig>,

    #[account(
        init,
        payer = creator,
        space = AmmPool::LEN,
        seeds = [b"amm-pool", token_mint.key().as_ref()],
        bump
    )]
    pub amm_pool: Account<'info, AmmPool>,

    pub token_mint: Account<'info, Mint>,

    // sol_vault: system account PDA — init via system_program CPI
    /// CHECK: PDA validated by seeds
    #[account(mut, seeds = [b"sol-vault", token_mint.key().as_ref()], bump)]
    pub sol_vault: UncheckedAccount<'info>,

    // creator_fee_vault: system account PDA
    /// CHECK: PDA validated by seeds
    #[account(mut, seeds = [b"creator-fee-vault", token_mint.key().as_ref()], bump)]
    pub creator_fee_vault: UncheckedAccount<'info>,

    // token_vault_authority PDA (owns the ATA)
    /// CHECK: PDA validated by seeds
    #[account(seeds = [b"token-vault-authority", token_mint.key().as_ref()], bump)]
    pub token_vault_authority: UncheckedAccount<'info>,

    // token_vault: ATA of token_vault_authority for token_mint
    #[account(
        init,
        payer = creator,
        associated_token::mint = token_mint,
        associated_token::authority = token_vault_authority,
    )]
    pub token_vault: Account<'info, TokenAccount>,

    // lp_mint_authority PDA
    /// CHECK: PDA validated by seeds
    #[account(seeds = [b"lp-mint-authority", token_mint.key().as_ref()], bump)]
    pub lp_mint_authority: UncheckedAccount<'info>,

    // lp_mint: new SPL mint for LP tokens
    #[account(
        init,
        payer = creator,
        mint::decimals = 6,
        mint::authority = lp_mint_authority,
    )]
    pub lp_mint: Account<'info, Mint>,

    // creator's LP token account (receives LP tokens)
    #[account(
        init_if_needed,
        payer = creator,
        associated_token::mint = lp_mint,
        associated_token::authority = creator,
    )]
    pub creator_lp_account: Account<'info, TokenAccount>,

    // creator's token account (source of initial tokens)
    #[account(
        mut,
        associated_token::mint = token_mint,
        associated_token::authority = creator,
    )]
    pub creator_token_account: Account<'info, TokenAccount>,

    #[account(mut)]
    pub creator: Signer<'info>,

    pub token_program: Program<'info, Token>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub system_program: Program<'info, System>,
    pub rent: Sysvar<'info, Rent>,
}
```

**Parameters**: `sol_amount: u64`, `token_amount: u64`, `min_lp_out: u64`

**Validation**:
```rust
require!(!global_config.paused, ThunderSwapError::ProgramPaused);
require!(sol_amount >= global_config.min_pool_sol_lamports, ThunderSwapError::InsufficientInitialLiquidity);
require!(token_amount > 0, ThunderSwapError::InsufficientInitialLiquidity);
```

**Logic**:
```rust
// 1. Transfer SOL from creator to sol_vault
system_program::transfer(CpiContext::new(...), sol_amount)?;

// 2. Transfer tokens from creator to token_vault
token::transfer(CpiContext::new(...), token_amount)?;

// 3. Calculate initial LP supply
let product = (sol_amount as u128).checked_mul(token_amount as u128).ok_or(ThunderSwapError::Overflow)?;
let lp_raw = integer_sqrt(product);
require!(lp_raw > MINIMUM_LIQUIDITY, ThunderSwapError::InsufficientLiquidityMinted);
let lp_to_creator = lp_raw - MINIMUM_LIQUIDITY;
require!(lp_to_creator >= min_lp_out, ThunderSwapError::SlippageExceeded);

// 4. Mint MINIMUM_LIQUIDITY LP tokens to lp_burn_vault, then immediately burn them
//    (simplest: mint to creator, burn MINIMUM_LIQUIDITY, keep lp_to_creator)
let seeds = &[b"lp-mint-authority", token_mint.key().as_ref(), &[bumps.lp_mint_authority]];
token::mint_to(CpiContext::new_with_signer(..., &[seeds]), lp_raw)?;
token::burn(CpiContext::new(...), MINIMUM_LIQUIDITY)?;

// 5. Store pool state
amm_pool.token_mint = token_mint.key();
amm_pool.creator = creator.key();
amm_pool.sol_reserve = sol_amount;
amm_pool.token_reserve = token_amount;
amm_pool.lp_mint = lp_mint.key();
amm_pool.lp_supply = lp_raw - MINIMUM_LIQUIDITY;  // outstanding (not burned)
amm_pool.creator_fees_accumulated = 0;
amm_pool.locked = false;
amm_pool.graduated_from_plp = false;
amm_pool.lp_burned = false;
amm_pool.lp_burn_tx_sig = [0u8; 64];
amm_pool.twap_last_slot = Clock::get()?.slot;
amm_pool.twap_sol_per_token_cumulative = 0;
amm_pool.twap_token_per_sol_cumulative = 0;
// store bumps
```

**Emits**: `PoolCreatedEvent`

---

### 5.3 `create_pool_from_graduation`

**Privileged path** — only `GlobalConfig.graduation_authority` can call this instruction. Creates a pool AND burns LP tokens in the same transaction.

**Accounts**: Same as `create_pool` but with these changes:
```rust
// graduation_authority replaces creator as signer
#[account(
    constraint = graduation_authority.key() == global_config.graduation_authority
        @ ThunderSwapError::UnauthorizedGraduationCaller
)]
pub graduation_authority: Signer<'info>,

// lp_burn_record is initialized here
#[account(
    init,
    payer = graduation_authority,
    space = LpBurnRecord::LEN,
    seeds = [b"lp-burn-record", amm_pool.key().as_ref()],
    bump
)]
pub lp_burn_record: Account<'info, LpBurnRecord>,
```

**Parameters**: `sol_amount: u64`, `token_amount: u64`, `creator: Pubkey`

**Validation**:
```rust
require!(!global_config.paused, ThunderSwapError::ProgramPaused);
require!(
    graduation_authority.key() == global_config.graduation_authority,
    ThunderSwapError::UnauthorizedGraduationCaller
);
require!(sol_amount > 0 && token_amount > 0, ThunderSwapError::InsufficientInitialLiquidity);
// No min_pool_sol_lamports check — graduation guarantees sufficient liquidity
```

**Logic**:
```rust
// 1. Transfer SOL from graduation_authority to sol_vault
// 2. Transfer tokens from graduation_authority to token_vault
// 3. Calculate LP: integer_sqrt(sol_amount * token_amount)
let lp_total = integer_sqrt((sol_amount as u128) * (token_amount as u128));

// 4. Mint ALL lp_total LP tokens temporarily to graduation_authority's LP account
let mint_auth_seeds = &[b"lp-mint-authority", token_mint.key().as_ref(), &[bumps.lp_mint_authority]];
token::mint_to(CpiContext::new_with_signer(..., &[mint_auth_seeds]), lp_total)?;

// 5. Burn ALL lp_total LP tokens immediately (graduation = permanently locked liquidity)
token::burn(CpiContext::new(...), lp_total)?;

// 6. Revoke LP mint authority (no more LP tokens can ever be minted for this pool)
token::set_authority(
    CpiContext::new_with_signer(..., &[mint_auth_seeds]),
    AuthorityType::MintTokens,
    None,  // revoke — no authority
)?;

// 7. Write LpBurnRecord
lp_burn_record.pool = amm_pool.key();
lp_burn_record.token_mint = token_mint.key();
lp_burn_record.lp_mint = lp_mint.key();
lp_burn_record.amount_burned = lp_total;
lp_burn_record.burn_slot = Clock::get()?.slot;
lp_burn_record.bump = ctx.bumps.lp_burn_record;

// 8. Store pool state (same as create_pool but with graduation flags)
amm_pool.graduated_from_plp = true;
amm_pool.lp_burned = true;
amm_pool.lp_supply = 0;  // All LP burned — no outstanding LP
amm_pool.creator = creator;
// ... remaining fields same as create_pool
// lp_burn_tx_sig: set by indexer reading lp_burn_record;
// on-chain we store the slot, off-chain indexer captures the tx sig
```

**Emits**: `GraduationPoolCreatedEvent`

**Post-condition**: `add_liquidity` and `remove_liquidity` will both fail for this pool permanently (`pool.lp_burned == true`).

---

### 5.4 `swap`

Core AMM swap. Handles both buy (SOL → token) and sell (token → SOL) in one instruction, differentiated by `is_buy: bool`.

**Accounts**:
```rust
#[derive(Accounts)]
pub struct Swap<'info> {
    #[account(seeds = [b"global-config"], bump = global_config.bump)]
    pub global_config: Account<'info, GlobalConfig>,

    #[account(
        mut,
        seeds = [b"amm-pool", amm_pool.token_mint.as_ref()],
        bump = amm_pool.bump,
        constraint = !amm_pool.locked @ ThunderSwapError::PoolLocked,
    )]
    pub amm_pool: Account<'info, AmmPool>,

    /// CHECK: validated against global_config.protocol_fee_vault
    #[account(
        mut,
        constraint = protocol_fee_vault.key() == global_config.protocol_fee_vault
            @ ThunderSwapError::InvalidProtocolFeeVault
    )]
    pub protocol_fee_vault: UncheckedAccount<'info>,

    /// CHECK: PDA validated
    #[account(
        mut,
        seeds = [b"creator-fee-vault", amm_pool.token_mint.as_ref()],
        bump = amm_pool.creator_fee_vault_bump
    )]
    pub creator_fee_vault: UncheckedAccount<'info>,

    /// CHECK: PDA validated
    #[account(
        mut,
        seeds = [b"sol-vault", amm_pool.token_mint.as_ref()],
        bump = amm_pool.sol_vault_bump
    )]
    pub sol_vault: UncheckedAccount<'info>,

    #[account(
        mut,
        seeds = [b"token-vault-authority", amm_pool.token_mint.as_ref()],
        bump = amm_pool.token_vault_authority_bump
    )]
    /// CHECK: PDA validated
    pub token_vault_authority: UncheckedAccount<'info>,

    #[account(
        mut,
        associated_token::mint = amm_pool.token_mint,
        associated_token::authority = token_vault_authority,
    )]
    pub token_vault: Account<'info, TokenAccount>,

    #[account(
        init_if_needed,
        payer = user,
        associated_token::mint = amm_pool.token_mint,
        associated_token::authority = user,
    )]
    pub user_token_account: Account<'info, TokenAccount>,

    #[account(mut)]
    pub user: Signer<'info>,

    pub token_program: Program<'info, Token>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub system_program: Program<'info, System>,
}
```

**Parameters**: `is_buy: bool`, `amount_in: u64`, `min_amount_out: u64`

**Logic — Buy (SOL → token)**:
```rust
require!(!global_config.paused, ThunderSwapError::ProgramPaused);
require!(amount_in > 0, ThunderSwapError::DustTrade);

// 1. Update TWAP before reserves change
update_twap(&mut amm_pool, Clock::get()?.slot)?;

// 2. Compute fees on sol_in
let fees = compute_fees(amount_in, &global_config)?;

// 3. AMM quote on net amount (includes LP fee baked in)
let tokens_out = quote_buy(amm_pool.sol_reserve, amm_pool.token_reserve, fees.net_amount)?;
require!(tokens_out >= min_amount_out, ThunderSwapError::SlippageExceeded);
require!(tokens_out > 0, ThunderSwapError::DustTrade);

// 4. Transfer protocol_fee from user to protocol_fee_vault
system_program::transfer(CpiContext::new(...), fees.protocol_fee)?;

// 5. Transfer creator_fee from user to creator_fee_vault
system_program::transfer(CpiContext::new(...), fees.creator_fee)?;

// 6. Transfer net_amount from user to sol_vault
system_program::transfer(CpiContext::new(...), fees.net_amount)?;

// 7. Transfer tokens_out from token_vault to user_token_account
let vault_auth_seeds = &[b"token-vault-authority", amm_pool.token_mint.as_ref(), &[amm_pool.token_vault_authority_bump]];
token::transfer(CpiContext::new_with_signer(..., &[vault_auth_seeds]), tokens_out)?;

// 8. Update pool state
amm_pool.sol_reserve = amm_pool.sol_reserve.checked_add(fees.net_amount).ok_or(ThunderSwapError::Overflow)?;
amm_pool.token_reserve = amm_pool.token_reserve.checked_sub(tokens_out).ok_or(ThunderSwapError::Overflow)?;
amm_pool.creator_fees_accumulated = amm_pool.creator_fees_accumulated.saturating_add(fees.creator_fee);

// 9. Emit event
emit!(SwapEvent { ... });
```

**Logic — Sell (token → SOL)**:
```rust
// 1. Update TWAP before reserves change
update_twap(&mut amm_pool, Clock::get()?.slot)?;

// 2. Transfer tokens_in from user to token_vault
token::transfer(..., amount_in)?;

// 3. AMM quote on gross tokens_in (LP fee implicit in k growth)
let sol_out_gross = quote_sell(amm_pool.sol_reserve, amm_pool.token_reserve, amount_in)?;

// 4. Compute fees on gross SOL output
let fees = compute_fees(sol_out_gross, &global_config)?;
require!(fees.net_amount >= min_amount_out, ThunderSwapError::SlippageExceeded);
require!(fees.net_amount > 0, ThunderSwapError::DustTrade);

// 5. Update reserves using AMM result (before fee extraction from output)
let k = (amm_pool.sol_reserve as u128) * (amm_pool.token_reserve as u128);
let new_token_reserve = (amm_pool.token_reserve as u128) + (amount_in as u128);
let new_sol_reserve = k / new_token_reserve;
amm_pool.token_reserve = new_token_reserve as u64;
amm_pool.sol_reserve = new_sol_reserve as u64;
// Note: sol_vault balance decreases by sol_out_gross (net_to_user + protocol + creator)
//       sol_reserve decreases by (sol_reserve - new_sol_reserve) which equals sol_out_gross
//       The LP fee is captured in k's growth (token reserve increased without sol leaving)

// 6. Transfer from sol_vault: net_amount to user, protocol_fee to protocol vault, creator_fee to creator vault
let sol_vault_seeds = &[b"sol-vault", amm_pool.token_mint.as_ref(), &[amm_pool.sol_vault_bump]];
system_program::transfer(CpiContext::new_with_signer(..., &[sol_vault_seeds]), fees.net_amount)?;  // → user
system_program::transfer(CpiContext::new_with_signer(..., &[sol_vault_seeds]), fees.protocol_fee)?; // → protocol
system_program::transfer(CpiContext::new_with_signer(..., &[sol_vault_seeds]), fees.creator_fee)?;  // → creator_fee_vault

amm_pool.creator_fees_accumulated = amm_pool.creator_fees_accumulated.saturating_add(fees.creator_fee);
```

---

### 5.5 `add_liquidity`

Add liquidity to a public (non-graduated) pool. Locked for graduated pools.

**Validation**:
```rust
require!(!global_config.paused, ThunderSwapError::ProgramPaused);
require!(!amm_pool.locked, ThunderSwapError::PoolLocked);
require!(!amm_pool.lp_burned, ThunderSwapError::PoolLiquidityLocked);  // graduated pools
```

**Parameters**: `sol_max: u64`, `token_max: u64`, `min_lp_out: u64`

**Logic** (pro-rata minting):
```rust
// Choose the deposit amount that fits within both maxes
let sol_required = (token_max as u128) * (amm_pool.sol_reserve as u128) / (amm_pool.token_reserve as u128);

let (sol_deposit, token_deposit) = if sol_required <= sol_max as u128 {
    (sol_required as u64, token_max)
} else {
    let token_required = (sol_max as u128) * (amm_pool.token_reserve as u128) / (amm_pool.sol_reserve as u128);
    require!(token_required <= token_max as u128, ThunderSwapError::SlippageExceeded);
    (sol_max, token_required as u64)
};

// LP to mint: take minimum of both sides to prevent rounding exploit
let lp_from_sol = (sol_deposit as u128) * (amm_pool.lp_supply as u128) / (amm_pool.sol_reserve as u128);
let lp_from_token = (token_deposit as u128) * (amm_pool.lp_supply as u128) / (amm_pool.token_reserve as u128);
let lp_to_mint = lp_from_sol.min(lp_from_token) as u64;

require!(lp_to_mint >= min_lp_out, ThunderSwapError::SlippageExceeded);
require!(lp_to_mint > 0, ThunderSwapError::InsufficientLiquidityMinted);

// Transfer sol_deposit from user to sol_vault (not sol_max — only what is needed)
// Transfer token_deposit from user to token_vault (not token_max)
// Mint lp_to_mint LP tokens to user

amm_pool.sol_reserve += sol_deposit;
amm_pool.token_reserve += token_deposit;
amm_pool.lp_supply += lp_to_mint;
```

**Emits**: `LiquidityAddedEvent`

---

### 5.6 `remove_liquidity`

Remove liquidity from a public pool by burning LP tokens. Locked for graduated pools.

**Validation**:
```rust
require!(!amm_pool.lp_burned, ThunderSwapError::PoolLiquidityLocked);
require!(lp_amount > 0, ThunderSwapError::DustTrade);
require!(lp_amount <= amm_pool.lp_supply, ThunderSwapError::InsufficientLpTokens);
```

**Parameters**: `lp_amount: u64`, `min_sol_out: u64`, `min_token_out: u64`

**Logic**:
```rust
let sol_out = ((lp_amount as u128) * (amm_pool.sol_reserve as u128) / (amm_pool.lp_supply as u128)) as u64;
let token_out = ((lp_amount as u128) * (amm_pool.token_reserve as u128) / (amm_pool.lp_supply as u128)) as u64;

require!(sol_out >= min_sol_out, ThunderSwapError::SlippageExceeded);
require!(token_out >= min_token_out, ThunderSwapError::SlippageExceeded);
require!(sol_out > 0 && token_out > 0, ThunderSwapError::DustTrade);

// Burn lp_amount LP tokens from user
// Transfer sol_out from sol_vault to user
// Transfer token_out from token_vault to user

amm_pool.sol_reserve -= sol_out;
amm_pool.token_reserve -= token_out;
amm_pool.lp_supply -= lp_amount;
```

**Emits**: `LiquidityRemovedEvent`

---

### 5.7 `claim_creator_fees`

Withdraw accumulated creator fees from the `creator_fee_vault` PDA.

**Accounts**:
```rust
#[derive(Accounts)]
pub struct ClaimCreatorFees<'info> {
    #[account(
        seeds = [b"amm-pool", amm_pool.token_mint.as_ref()],
        bump = amm_pool.bump,
        constraint = creator.key() == amm_pool.creator @ ThunderSwapError::UnauthorizedCreator,
    )]
    pub amm_pool: Account<'info, AmmPool>,

    /// CHECK: PDA validated
    #[account(
        mut,
        seeds = [b"creator-fee-vault", amm_pool.token_mint.as_ref()],
        bump = amm_pool.creator_fee_vault_bump,
    )]
    pub creator_fee_vault: UncheckedAccount<'info>,

    #[account(mut)]
    pub creator: Signer<'info>,

    pub system_program: Program<'info, System>,
}
```

**Logic**:
```rust
let rent_exempt_min = Rent::get()?.minimum_balance(0);
let vault_balance = creator_fee_vault.lamports();
require!(vault_balance > rent_exempt_min, ThunderSwapError::NothingToClaim);

let claimable = vault_balance - rent_exempt_min;

let creator_fee_vault_seeds = &[
    b"creator-fee-vault",
    amm_pool.token_mint.as_ref(),
    &[amm_pool.creator_fee_vault_bump]
];
system_program::transfer(
    CpiContext::new_with_signer(
        ctx.accounts.system_program.to_account_info(),
        system_program::Transfer {
            from: creator_fee_vault.to_account_info(),
            to: creator.to_account_info(),
        },
        &[creator_fee_vault_seeds],
    ),
    claimable,
)?;

emit!(CreatorFeeClaimEvent {
    token_mint: amm_pool.token_mint,
    pool: amm_pool.key(),
    creator: creator.key(),
    amount_claimed: claimable,
    slot: Clock::get()?.slot,
});
```

---

### 5.8 `update_global_config`

Update fee parameters, circuit breaker, or fee vault. Only callable by `GlobalConfig.authority` (Squads multisig).

**Accounts**:
```rust
#[account(
    mut,
    seeds = [b"global-config"],
    bump = global_config.bump,
    has_one = authority @ ThunderSwapError::UnauthorizedAuthority,
)]
pub global_config: Account<'info, GlobalConfig>,

pub authority: Signer<'info>,
```

**Parameters**: All `GlobalConfig` fields are optional updates via `Option<T>`. Validates that `lp_fee_bps + protocol_fee_bps + creator_fee_bps <= 500` (5% max total) as a sanity guard.

**Emits**: `GlobalConfigUpdatedEvent`

---

## 6. Events

All events are emitted with `emit!(...)` and parsed by the ThunderSwap indexer using Anchor's `EventParser`.

```rust
#[event]
pub struct PoolCreatedEvent {
    pub token_mint: Pubkey,
    pub pool: Pubkey,
    pub creator: Pubkey,
    pub sol_amount: u64,
    pub token_amount: u64,
    pub lp_minted: u64,
    pub slot: u64,
}

#[event]
pub struct GraduationPoolCreatedEvent {
    pub token_mint: Pubkey,
    pub pool: Pubkey,
    pub creator: Pubkey,
    pub sol_amount: u64,
    pub token_amount: u64,
    pub lp_burned: u64,
    pub lp_mint: Pubkey,
    pub lp_burn_record: Pubkey,
    pub slot: u64,
}

#[event]
pub struct SwapEvent {
    pub token_mint: Pubkey,
    pub pool: Pubkey,
    pub user: Pubkey,
    pub is_buy: bool,
    pub amount_in: u64,
    pub amount_out: u64,
    pub protocol_fee: u64,
    pub creator_fee: u64,
    pub sol_reserve_after: u64,
    pub token_reserve_after: u64,
    pub slot: u64,
}

#[event]
pub struct LiquidityAddedEvent {
    pub token_mint: Pubkey,
    pub pool: Pubkey,
    pub provider: Pubkey,
    pub sol_deposited: u64,
    pub token_deposited: u64,
    pub lp_minted: u64,
    pub slot: u64,
}

#[event]
pub struct LiquidityRemovedEvent {
    pub token_mint: Pubkey,
    pub pool: Pubkey,
    pub provider: Pubkey,
    pub sol_withdrawn: u64,
    pub token_withdrawn: u64,
    pub lp_burned: u64,
    pub slot: u64,
}

#[event]
pub struct CreatorFeeClaimEvent {
    pub token_mint: Pubkey,
    pub pool: Pubkey,
    pub creator: Pubkey,
    pub amount_claimed: u64,
    pub slot: u64,
}

#[event]
pub struct GlobalConfigInitializedEvent {
    pub authority: Pubkey,
    pub graduation_authority: Pubkey,
    pub protocol_fee_vault: Pubkey,
    pub slot: u64,
}

#[event]
pub struct GlobalConfigUpdatedEvent {
    pub updated_by: Pubkey,
    pub slot: u64,
}
```

---

## 7. Error Codes

```rust
#[error_code]
pub enum ThunderSwapError {
    // --- Math ---
    #[msg("Arithmetic overflow")]
    Overflow,

    #[msg("Division by zero")]
    DivisionByZero,

    // --- Pool state ---
    #[msg("Pool has empty reserves")]
    EmptyPool,

    #[msg("Insufficient initial liquidity — minimum 5 SOL per side for public pools")]
    InsufficientInitialLiquidity,

    #[msg("Insufficient liquidity minted — add more liquidity")]
    InsufficientLiquidityMinted,

    #[msg("Insufficient pool liquidity for this swap size")]
    InsufficientLiquidity,

    #[msg("Pool is locked by admin")]
    PoolLocked,

    #[msg("Pool liquidity is permanently locked — this is a graduated pool")]
    PoolLiquidityLocked,

    #[msg("Insufficient LP tokens")]
    InsufficientLpTokens,

    // --- Swap ---
    #[msg("Slippage tolerance exceeded — try again or increase slippage")]
    SlippageExceeded,

    #[msg("Trade amount too small — minimum trade required")]
    DustTrade,

    // --- Auth ---
    #[msg("Only the graduation authority can call this instruction")]
    UnauthorizedGraduationCaller,

    #[msg("Only the pool creator can claim creator fees")]
    UnauthorizedCreator,

    #[msg("Only GlobalConfig authority can call this instruction")]
    UnauthorizedAuthority,

    #[msg("Invalid protocol fee vault address")]
    InvalidProtocolFeeVault,

    // --- Config ---
    #[msg("Program is paused — try again later")]
    ProgramPaused,

    #[msg("Total fees cannot exceed 5% (500 bps)")]
    FeesTooHigh,

    // --- Claims ---
    #[msg("No creator fees available to claim")]
    NothingToClaim,
}
```

---

## 8. Security Invariants

These invariants must hold after every instruction. Verify in tests.

| Invariant | Checked After |
|---|---|
| `amm_pool.sol_reserve == sol_vault.lamports() - rent_exempt` | Every instruction that changes reserves |
| `k = sol_reserve * token_reserve` never decreases (except `remove_liquidity`) | `swap`, `add_liquidity` |
| `lp_supply` equals sum of all outstanding LP token balances (SPL mint supply) | `create_pool`, `add_liquidity`, `remove_liquidity` |
| If `lp_burned == true`: `add_liquidity` and `remove_liquidity` always fail | Any instruction on graduated pool |
| If `lp_burned == true`: `lp_supply == 0` | After `create_pool_from_graduation` |
| `creator_fee_vault.lamports() >= rent_exempt` at all times | After every `swap` |
| No instruction succeeds when `global_config.paused == true` (except `update_global_config`) | Global |
| LP mint authority is `None` after graduation | After `create_pool_from_graduation` |
| `LpBurnRecord` exists for every pool where `lp_burned == true` | After `create_pool_from_graduation` |
| `swap` with `amount_in = 0` always fails with `DustTrade` | `swap` |
| `claim_creator_fees` can only be called by `pool.creator` | `claim_creator_fees` |

---

## 9. CPI Integration with ThunderLaunch PLP

### How ThunderLaunch Calls ThunderSwap

For v1, `create_pool_from_graduation` uses the **Squads multisig as signer** (same wallet as `GlobalConfig.graduation_authority`). The Squads multisig signs the graduation transaction client-side.

For v2, the PLP program will call `create_pool_from_graduation` directly via Anchor CPI:

```rust
// In ThunderLaunch programs/plp/src/lib.rs — graduate instruction
// (v2 CPI implementation shown for reference)

use thunderswap::cpi::accounts::CreatePoolFromGraduation;
use thunderswap::program::ThunderSwap;

if graduation_dex == GraduationDex::ThunderSwap {
    let cpi_accounts = CreatePoolFromGraduation {
        global_config: ctx.accounts.thunderswap_global_config.to_account_info(),
        amm_pool: ctx.accounts.thunderswap_pool.to_account_info(),
        token_mint: ctx.accounts.mint.to_account_info(),
        sol_vault: ctx.accounts.ts_sol_vault.to_account_info(),
        creator_fee_vault: ctx.accounts.ts_creator_fee_vault.to_account_info(),
        token_vault_authority: ctx.accounts.ts_token_vault_authority.to_account_info(),
        token_vault: ctx.accounts.ts_token_vault.to_account_info(),
        lp_mint_authority: ctx.accounts.ts_lp_mint_authority.to_account_info(),
        lp_mint: ctx.accounts.ts_lp_mint.to_account_info(),
        lp_burn_record: ctx.accounts.ts_lp_burn_record.to_account_info(),
        graduation_authority: ctx.accounts.plp_graduation_authority.to_account_info(),
        token_program: ctx.accounts.token_program.to_account_info(),
        // ...
    };

    // The PLP graduation authority PDA signs the CPI
    let seeds = &[b"graduation-authority", &[ctx.bumps.plp_graduation_authority]];
    thunderswap::cpi::create_pool_from_graduation(
        CpiContext::new_with_signer(
            ctx.accounts.thunderswap_program.to_account_info(),
            cpi_accounts,
            &[seeds],
        ),
        sol_amount,
        token_amount,
        pool.creator,
    )?;
}
```

For this CPI to work, `GlobalConfig.graduation_authority` must be updated to the PLP graduation authority PDA address via `update_global_config` before v2 deployment.

### Graduation Sequence (v1 — Multisig Signs)

1. Bonding curve completes graduation threshold
2. ThunderLaunch cron/indexer detects graduation eligibility
3. Graduation authority (Squads multisig) signs a transaction that:
   a. Calls ThunderLaunch `release_vault_liquidity` → transfers SOL + tokens to graduation authority wallet
   b. Calls ThunderSwap `create_pool_from_graduation` → creates pool, burns LP, writes record
4. ThunderSwap indexer detects `GraduationPoolCreatedEvent`
5. Webhook sent to ThunderLaunch backend
6. Token status updated to `graduated` in DB

---

## 10. Testing Checklist

### Unit Tests (`tests/thunderswap.ts` — Anchor Bankrun)

**Math module (`math.rs`)**:
- [ ] `integer_sqrt(0)` = 0, `integer_sqrt(4)` = 2, `integer_sqrt(u128::MAX)` does not overflow
- [ ] `quote_buy` preserves k (within 1 due to integer division)
- [ ] `quote_sell` preserves k (within 1 due to integer division)
- [ ] `compute_fees` with 0 returns all-zero fees
- [ ] `compute_fees` fees sum to ≤ input amount
- [ ] TWAP updates correctly across slot boundaries

**`initialize_global_config`**:
- [ ] Creates GlobalConfig with correct initial values
- [ ] Fails if called twice (account already exists)

**`create_pool`**:
- [ ] Creates pool with correct reserves and LP supply
- [ ] Fails with `InsufficientInitialLiquidity` if `sol_amount < 5 SOL`
- [ ] Fails if pool for this mint already exists
- [ ] MINIMUM_LIQUIDITY LP tokens are burned (lp_supply = sqrt(sol*token) - 1000)
- [ ] Fails when program paused

**`create_pool_from_graduation`**:
- [ ] Creates pool, burns ALL LP tokens (`lp_supply == 0`)
- [ ] `LpBurnRecord` written with correct fields
- [ ] LP mint authority is revoked (None) after call
- [ ] `lp_burned == true`, `graduated_from_plp == true`
- [ ] Fails when called by non-`graduation_authority` signer with `UnauthorizedGraduationCaller`
- [ ] No `min_pool_sol_lamports` check (small liquidity amounts accepted)

**`swap`**:
- [ ] Buy: correct `tokens_out`, correct fees split to three destinations
- [ ] Sell: correct `sol_out`, correct fees split to three destinations
- [ ] Both: `amm_pool.sol_reserve == sol_vault.lamports() - rent_exempt` after swap
- [ ] Fails with `SlippageExceeded` if output below `min_amount_out`
- [ ] Fails with `DustTrade` if `amount_in == 0`
- [ ] Fails on locked pool
- [ ] Fails when program paused
- [ ] TWAP accumulators updated

**`add_liquidity`**:
- [ ] Pro-rata LP minting, correct pro-rata amounts used
- [ ] Excess amounts NOT transferred (only exact deposit amounts)
- [ ] Fails on graduated pool (`PoolLiquidityLocked`)
- [ ] Slippage check on `min_lp_out`

**`remove_liquidity`**:
- [ ] Correct pro-rata `sol_out` and `token_out`
- [ ] LP tokens burned from user account
- [ ] Fails on graduated pool
- [ ] Slippage checks on both outputs

**`claim_creator_fees`**:
- [ ] Transfers all claimable lamports to creator
- [ ] Leaves rent-exempt minimum in vault
- [ ] Fails when called by non-creator
- [ ] Fails with `NothingToClaim` when vault at rent-exempt minimum

**`update_global_config`**:
- [ ] Updates fields correctly when signed by authority
- [ ] Fails when signed by non-authority
- [ ] Fails if total fees > 500 bps

### Integration Tests (Devnet)

- [ ] Full graduation flow: bonding curve complete → multisig signs → pool created → LP burned on-chain
- [ ] Post-graduation swap through ThunderLaunch Trade Panel
- [ ] Creator fee accumulation over multiple swaps
- [ ] Creator fee claim from ThunderLaunch My Profile tab
- [ ] Indexer detects all events within 30 seconds
- [ ] Webhook reaches ThunderLaunch backend and updates token status
- [ ] Reconciliation cron catches graduation when webhook fails (test by disabling webhook)
- [ ] Program paused: all swaps rejected, update_global_config still works
