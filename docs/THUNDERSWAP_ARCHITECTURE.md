# ThunderSwap — Architecture Document

**Version**: 1.0
**Status**: Design Phase
**Date**: 2026-03-27
**Related**: [THUNDERSWAP_PLANNING.md](THUNDERSWAP_PLANNING.md)

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [High-Level Component Diagram](#2-high-level-component-diagram)
3. [AMM Design](#3-amm-design)
4. [Solana Program Architecture](#4-solana-program-architecture)
5. [Indexer Architecture](#5-indexer-architecture)
6. [Backend & API Layer](#6-backend--api-layer)
7. [Frontend Architecture](#7-frontend-architecture)
8. [ThunderLaunch Integration Points](#8-thunderlaunch-integration-points)
9. [Security Architecture](#9-security-architecture)
10. [Scalability](#10-scalability)
11. [Multi-Chain Strategy](#11-multi-chain-strategy)
12. [Tech Stack](#12-tech-stack)
13. [Appendices](#appendices)

---

## 1. System Overview

### What ThunderSwap Is

ThunderSwap is the native decentralized exchange (DEX) built by the ThunderLaunch team. It serves two roles:

1. **Graduation destination** — when a token's bonding curve completes on ThunderLaunch, all liquidity permanently migrates to ThunderSwap via a direct on-chain CPI call. All subsequent trading for that token happens on ThunderSwap. Users never leave ThunderLaunch — the existing Trade Panel routes transactions to ThunderSwap transparently.

2. **General-purpose permissionless DEX** — any project or wallet can create a liquidity pool on ThunderSwap. It is not limited to ThunderLaunch-graduated tokens.

ThunderSwap is a separate Solana program with its own Program ID, its own indexer, and its own UI. It shares infrastructure (Supabase, Vercel, RPC) with ThunderLaunch but has no compile-time dependency on the ThunderLaunch codebase at the program level.

### Design Principles

| Principle | Description |
|---|---|
| **Graduation-native** | The CPI path from ThunderLaunch is a first-class concern, not an afterthought. It has its own dedicated instruction with separate validation logic. |
| **Blockchain-first** | Swaps execute at Solana speed with no backend intermediary on the critical path. The backend indexes and reads; it does not participate in trade execution. |
| **Permissionless pools** | Any wallet can call `create_pool` with sufficient liquidity. The only privileged path is the graduation CPI, enforced by requiring the ThunderLaunch graduation authority PDA as a signer. |
| **Locked liquidity by design** | LP tokens burned at graduation are not a feature toggle — they are a protocol guarantee enforced in the same on-chain transaction that creates the pool. |
| **Pull-model creator fees** | Creator fees accumulate passively in a PDA vault. Creators claim at their discretion by calling `claim_creator_fees`. No push/auto-transfer overhead on every swap. |
| **Multi-chain by roadmap, not complexity** | v1 is Solana-only. Chain-specific code is isolated cleanly so v2 (Base) and v3 (Ethereum) do not require refactoring the core. |
| **Flat fees for v1** | 50 bps total (20 LP + 15 protocol + 15 creator), no dynamic tiers. Simplicity supports auditability and a clean user experience. Tiers can be added in v2. |

### Key Performance Targets

| Operation | Target |
|---|---|
| Swap execution | < 3 seconds (Solana finality) |
| Pool creation | < 30 seconds |
| Graduation CPI | Idempotent, retryable, finalized within one block |
| Price oracle (TWAP) | Updated on every swap |
| Indexer event lag | < 15 seconds |
| API response time | < 200ms (cached pool reads) |

---

## 2. High-Level Component Diagram

### System Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                            User Layer                                │
│  ThunderLaunch Trade Panel  │  ThunderSwap UI  │  3rd-party / Bots  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                         Frontend Layer                               │
│            Next.js — swap.thunderlaunch.io                          │
│         React Query · Tailwind · @solana/wallet-adapter             │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ REST / WebSocket
┌──────────────────────────────▼──────────────────────────────────────┐
│                      Backend / API Layer                             │
│   Next.js API Routes  │  ThunderSwap Indexer  │  Cron Jobs          │
│                   Supabase (PostgreSQL + Realtime)                   │
└────────────┬──────────────────────────────────────┬─────────────────┘
             │ CPI at graduation                    │ RPC reads/writes
┌────────────▼──────────────────────────────────────▼─────────────────┐
│                         Solana Blockchain                            │
│   ThunderLaunch PLP Program  ──── CPI ────►  ThunderSwap Program    │
│   (bonding curve + graduation)               (AMM + LP burn)        │
└─────────────────────────────────────────────────────────────────────┘
```

### Swap Data Flow

```
User → ThunderSwap / ThunderLaunch UI
  → Wallet signs swap transaction
  → ThunderSwap Program executes swap
  → Updates pool reserves + TWAP accumulator
  → Distributes fees (LP stays in pool, protocol + creator go to vaults)
  → Emits SwapEvent
  → ThunderSwap Indexer picks up event
  → Writes to thunderswap_trades + updates pool metrics in Supabase
  → Supabase Realtime pushes update
  → UI re-renders with new price and reserves
```

### Graduation Data Flow

```
ThunderLaunch PLP: graduate(dex = ThunderSwap)
  ├── Deducts 1.5 SOL graduation fee
  ├── Revokes mint + freeze authority
  ├── Emits GraduationEvent
  └── CPI ──► ThunderSwap: create_pool_from_graduation(sol_amount, token_amount, creator, mint)
        ├── Creates PoolState account
        ├── Creates sol_vault, token_vault, lp_mint, creator_fee_vault PDAs
        ├── Transfers bonding curve reserves into vaults
        ├── Mints LP tokens to lp_burn_vault
        ├── Burns LP tokens from lp_burn_vault (same transaction)
        ├── Writes LpBurnRecord on-chain
        └── Emits GraduationPoolCreatedEvent

ThunderSwap Indexer detects GraduationPoolCreatedEvent
  ├── Upserts thunderswap_pools record
  ├── Inserts lp_burn_records record
  └── POSTs webhook → ThunderLaunch /api/webhooks/thunderswap-graduation
        └── ThunderLaunch updates token status: graduated = true, migration_status = completed
```

---

## 3. AMM Design

### 3.1 Model — Constant Product (x · y = k)

ThunderSwap v1 uses the constant product AMM model (Uniswap V2 style). This is the right choice for v1 because:

- It is the most battle-tested AMM design with the best-understood security properties
- The math is identical to ThunderLaunch's existing bonding curve (`quote_buy_constant_product` and `quote_sell_constant_product` in `lib.rs`) — the team already knows it
- It is the simplest model to audit cleanly
- Unlike the bonding curve, ThunderSwap pools use **real reserves only** (no virtual reserve inflation)

Concentrated liquidity (Uniswap V3 style) is a v2 consideration after v1 proves stability.

### 3.2 Core Math

```
Invariant:      x · y = k
                where x = SOL reserves, y = token reserves

Buy (SOL → token):
  tokens_out = y - k / (x + sol_in_after_fee)
  Validate:   tokens_out >= min_tokens_out

Sell (token → SOL):
  sol_out = x - k / (y + tokens_in_after_fee)
  Validate:   sol_out >= min_sol_out

Spot price:     P = x / y    (SOL per token)

Price impact:   impact_bps = |( P_after - P_before ) / P_before| × 10,000

Initial LP supply (pool creation):
  lp_supply = sqrt(sol_amount × token_amount)
```

All arithmetic uses Rust's `checked_*` methods to prevent overflow. No `unwrap()` on arithmetic — same convention as the existing PLP program.

### 3.3 Fee Distribution

Fees are applied on every swap before the constant product calculation. Total: **50 bps flat**.

```
Input amount:  sol_in (or token_in)

Fee splits:
  lp_fee         = sol_in × 20 / 10,000   → stays in pool (auto-compounds)
  protocol_fee   = sol_in × 15 / 10,000   → protocol_fee_vault PDA
  creator_fee    = sol_in × 15 / 10,000   → creator_fee_vault PDA

net_in         = sol_in - lp_fee - protocol_fee - creator_fee
amount_out     = constant_product_quote(net_in)
```

The LP fee stays in the pool reserves, meaning LP providers earn passively with every swap without any claim action. The protocol and creator fees are extracted to separate PDA vaults and claimed separately.

### 3.4 Creator Fee — Pull Model

Creator fees accumulate in a `creator_fee_vault` PDA keyed by pool mint. The creator calls `claim_creator_fees` at any time to withdraw the accumulated balance.

```
PDA seeds: ["creator-fee-vault", mint]
```

This is identical to ThunderLaunch's existing on-chain creator fee pattern (see `claim_creator_fees` in `lib.rs`). ThunderSwap mirrors this implementation exactly.

Both ThunderLaunch (My Profile → Created tab) and ThunderSwap's own UI surface claimable amounts and trigger the claim instruction.

### 3.5 TWAP Oracle

ThunderSwap v1 implements a Uniswap V2-style on-chain TWAP. Updated on every swap instruction.

```rust
// Added to PoolState
pub last_price_timestamp: i64,
pub cumulative_price_sol_per_token: u128,  // u128 prevents overflow for ~136 years

// Updated in swap instruction
let time_elapsed = current_timestamp - pool.last_price_timestamp;
pool.cumulative_price_sol_per_token += (pool.sol_reserves / pool.token_reserves) * time_elapsed as u128;
pool.last_price_timestamp = current_timestamp;

// TWAP over window W:
// twap = (cumulative[t1] - cumulative[t0]) / (t1 - t0)
```

Default TWAP window: 5 minutes. Integrators reading the oracle should average over at least this window. A dedicated Pyth/Switchboard integration is a v2 item.

### 3.6 Slippage Protection

The `swap` instruction takes `min_amount_out: u64`. If `amount_out < min_amount_out`, the instruction returns `ThunderSwapError::SlippageExceeded` and the transaction fails with no state change.

Frontend defaults:
- 0.5% for stable-ish pairs
- 1.0% general default
- User-configurable up to 50%

### 3.7 Pool Minimum Liquidity

| Pool Type | Minimum | Enforced In |
|---|---|---|
| Public `create_pool` | 5 SOL per side | `create_pool` instruction |
| `create_pool_from_graduation` | None | Bypassed — bonding curve guarantees sufficiency |

The 5 SOL minimum prevents dust pools and spam. It is stored in `GlobalConfig.min_pool_liquidity_lamports` and can be updated by the Squads multisig without a program upgrade.

---

## 4. Solana Program Architecture

**Language**: Rust
**Framework**: Anchor 0.30+
**Same framework as ThunderLaunch PLP program — conventions and patterns are consistent**

### 4.1 Program Structure

```
programs/thunderswap/
├── Cargo.toml
├── Xargo.toml
└── src/
    ├── lib.rs                            # Entry point, instruction dispatch
    ├── state.rs                          # Account structs
    ├── errors.rs                         # ThunderSwapError enum
    ├── math.rs                           # Constant product, TWAP, fee math
    └── instructions/
        ├── initialize_global_config.rs
        ├── create_pool.rs                # Public permissionless path
        ├── create_pool_from_graduation.rs # Graduation CPI path (exclusive to PLP)
        ├── swap.rs
        ├── add_liquidity.rs
        ├── remove_liquidity.rs
        ├── claim_creator_fees.rs
        └── update_global_config.rs
```

### 4.2 Account Structures

#### GlobalConfig — one per program

Governed by Squads 3-of-5 multisig. PDA seeds: `["global-config"]`.

```rust
#[account]
pub struct GlobalConfig {
    pub authority: Pubkey,                // Squads multisig PDA
    pub protocol_fee_vault: Pubkey,       // Receives 15 bps protocol fee
    pub protocol_fee_bps: u16,            // 15
    pub lp_fee_bps: u16,                  // 20
    pub creator_fee_bps: u16,             // 15
    pub min_pool_liquidity_lamports: u64, // 5_000_000_000 (5 SOL)
    pub plp_graduation_authority: Pubkey, // ThunderLaunch PLP graduation PDA — only this can call create_pool_from_graduation
    pub paused: bool,                     // Global circuit breaker
    pub bump: u8,
}
```

#### PoolState — one per token pair (SOL/TOKEN for v1)

PDA seeds: `["pool", mint]`.

```rust
#[account]
pub struct PoolState {
    pub mint: Pubkey,                     // Token mint address
    pub creator: Pubkey,                  // Original token creator (receives creator fees)
    pub sol_vault: Pubkey,                // PDA holding SOL reserves
    pub token_vault: Pubkey,              // ATA holding token reserves
    pub token_vault_authority: Pubkey,    // PDA owning token ATA
    pub lp_mint: Pubkey,                  // LP token mint
    pub creator_fee_vault: Pubkey,        // Accumulated creator fees (pull model)
    // Reserves — real only, no virtual inflation
    pub sol_reserves: u64,
    pub token_reserves: u64,
    // TWAP
    pub cumulative_price_sol_per_token: u128,
    pub last_price_timestamp: i64,
    // Graduation state
    pub graduated_from_plp: bool,         // True if seeded via graduation CPI
    pub lp_burned: bool,                  // True if LP was burned at graduation
    pub lp_burn_tx: [u8; 64],            // Burn tx signature (bytes) for DB reconciliation
    // Fee config (copied from GlobalConfig at pool creation — allows future per-pool overrides)
    pub protocol_fee_bps: u16,
    pub lp_fee_bps: u16,
    pub creator_fee_bps: u16,
    pub creator_fees_accumulated: u64,    // Lifetime creator fees in lamports
    pub locked: bool,                     // Pool-level circuit breaker
    // Liquidity tracking
    pub lp_supply: u64,
    // Bumps
    pub bump: u8,
    pub sol_vault_bump: u8,
    pub token_vault_authority_bump: u8,
    pub lp_mint_bump: u8,
    pub creator_fee_vault_bump: u8,
}
```

#### LpBurnRecord — one per graduated pool

Written in the same transaction that burns LP tokens. PDA seeds: `["lp-burn-record", pool]`.

```rust
#[account]
pub struct LpBurnRecord {
    pub pool: Pubkey,
    pub lp_mint: Pubkey,
    pub amount_burned: u64,
    pub burn_tx: [u8; 64],  // transaction signature
    pub timestamp: i64,
    pub bump: u8,
}
```

This record provides the trustless on-chain proof that LP tokens were burned. The DB (`lp_burn_records` table) mirrors it for fast querying.

### 4.3 PDA Seed Reference

| Account | Seeds | Program |
|---|---|---|
| GlobalConfig | `["global-config"]` | ThunderSwap |
| PoolState | `["pool", mint]` | ThunderSwap |
| SOL vault | `["vault-sol", mint]` | ThunderSwap |
| Token vault authority | `["vault-token-authority", mint]` | ThunderSwap |
| LP mint | `["lp-mint", mint]` | ThunderSwap |
| Creator fee vault | `["creator-fee-vault", mint]` | ThunderSwap |
| LP burn record | `["lp-burn-record", pool]` | ThunderSwap |
| PLP graduation authority | `["graduation-authority"]` | ThunderLaunch PLP |

### 4.4 Instructions

#### `initialize_global_config`

- **Signer**: Deployment wallet (upgrade authority, transferred to Squads at init)
- **Creates**: `GlobalConfig` PDA
- **Sets**: Fee rates, min liquidity, protocol fee vault, `plp_graduation_authority`
- **One-time**: Only callable if `GlobalConfig` does not yet exist

---

#### `create_pool` — Public permissionless path

- **Signer**: Any wallet (becomes the pool creator)
- **Validates**:
  - `sol_amount >= GlobalConfig.min_pool_liquidity_lamports`
  - `token_amount >= GlobalConfig.min_pool_liquidity_lamports` (approximate — validated against SOL value)
  - Pool for this mint does not already exist
  - `GlobalConfig.paused == false`
- **Creates**: `PoolState`, `sol_vault`, token ATA + vault authority, `lp_mint`, `creator_fee_vault`
- **Transfers**: SOL from creator to `sol_vault`, tokens from creator to `token_vault`
- **Mints**: `sqrt(sol_amount × token_amount)` LP tokens to creator
- **Emits**: `PoolCreatedEvent { mint, pool, creator, sol_amount, token_amount, timestamp }`

---

#### `create_pool_from_graduation` — Exclusive graduation path

- **Signer**: `GlobalConfig.plp_graduation_authority` (the ThunderLaunch PLP program's graduation PDA — no other wallet can satisfy this constraint)
- **Validates**:
  - Signer matches `GlobalConfig.plp_graduation_authority` exactly
  - Pool for this mint does not already exist
  - `GlobalConfig.paused == false`
- **Bypasses**: Min liquidity check (bonding curve guarantees sufficient liquidity)
- **Creates**: Same accounts as `create_pool`
- **Sets**: `pool.graduated_from_plp = true`
- **Immediately within the same instruction**:
  1. Mints LP tokens to `lp_burn_vault` (temporary PDA)
  2. Burns LP tokens via SPL token burn CPI
  3. Sets `pool.lp_burned = true`, `pool.lp_burn_tx = current_tx_sig`
  4. Writes `LpBurnRecord` on-chain
  5. Revokes LP mint authority
- **Emits**: `GraduationPoolCreatedEvent { mint, pool, creator, sol_amount, token_amount, lp_burned_amount, lp_mint, timestamp }`

After this instruction:
- `add_liquidity` and `remove_liquidity` are permanently locked for this pool (`pool.lp_burned == true`)
- The pool is fully operational for swaps immediately

---

#### `swap`

- **Signer**: User wallet
- **Validates**:
  - `GlobalConfig.paused == false`
  - `pool.locked == false`
  - `amount_out >= min_amount_out` (slippage)
  - Sufficient reserves for the requested swap
- **Fee logic**:
  1. Deduct `lp_fee` (20 bps) — stays in pool reserves (auto-compounding)
  2. Transfer `protocol_fee` (15 bps) → `GlobalConfig.protocol_fee_vault`
  3. Transfer `creator_fee` (15 bps) → `pool.creator_fee_vault`
  4. Apply constant product to `net_in` to compute `amount_out`
- **Updates**: `pool.sol_reserves`, `pool.token_reserves`, `pool.creator_fees_accumulated`, TWAP
- **Emits**: `SwapEvent { mint, pool, user, is_buy, sol_amount, token_amount, price_sol, price_impact_bps, timestamp }`

---

#### `add_liquidity`

- **Signer**: Any wallet
- **Validates**: `pool.lp_burned == false` (locked for graduated pools)
- **Pro-rata LP minting**: `lp_minted = min(sol_in/sol_reserves, token_in/token_reserves) × lp_supply`
- **Emits**: `LiquidityAddedEvent`

---

#### `remove_liquidity`

- **Signer**: LP token holder
- **Validates**: `pool.lp_burned == false` (locked for graduated pools)
- **Pro-rata redemption**: burn LP tokens, return proportional reserves to user
- **Emits**: `LiquidityRemovedEvent`

---

#### `claim_creator_fees`

- **Signer**: `pool.creator` (the wallet that created the token on ThunderLaunch)
- **Transfers**: All lamports from `creator_fee_vault` to `pool.creator` (minus rent-exempt minimum)
- **Emits**: `CreatorFeeClaimEvent { mint, pool, creator, amount_claimed, timestamp }`

This mirrors the existing ThunderLaunch `claim_creator_fees` implementation exactly. Uses `invoke_signed` with the vault's PDA seeds.

---

#### `update_global_config`

- **Signer**: `GlobalConfig.authority` (Squads 3-of-5 multisig)
- **Updates**: Any `GlobalConfig` field — fee rates, min liquidity, protocol vault, paused state
- **Emits**: `GlobalConfigUpdatedEvent`

---

### 4.5 CPI Flow — ThunderLaunch to ThunderSwap at Graduation

The ThunderLaunch PLP `graduate` instruction (in `programs/plp/src/lib.rs`) gains a conditional CPI branch for the ThunderSwap path:

```rust
match dex {
    GraduationDex::ThunderSwap => {
        // Build CPI accounts for ThunderSwap::create_pool_from_graduation
        let cpi_program = ctx.accounts.thunderswap_program.to_account_info();
        let cpi_accounts = CreatePoolFromGraduation {
            global_config: ctx.accounts.thunderswap_global_config.to_account_info(),
            pool: ctx.accounts.thunderswap_pool.to_account_info(),
            mint: ctx.accounts.mint.to_account_info(),
            // ... all required accounts
        };
        // graduation_authority PDA is the signer — satisfies ThunderSwap's constraint
        let seeds = &[b"graduation-authority", &[ctx.bumps.graduation_authority]];
        let signer = &[&seeds[..]];
        thunderswap::cpi::create_pool_from_graduation(
            CpiContext::new_with_signer(cpi_program, cpi_accounts, signer),
            sol_amount,
            token_amount,
            ctx.accounts.global_config.creator, // from PLP pool account
        )?;
    }
    GraduationDex::Raydium | GraduationDex::Orca | GraduationDex::Jupiter => {
        // Existing off-chain seeding flow via release_liquidity
        release_liquidity_to_authority(&ctx, sol_amount, token_amount)?;
    }
}
```

The `graduation_authority` PDA (seeds: `["graduation-authority"]` on the PLP program) is the only signer that satisfies ThunderSwap's `create_pool_from_graduation` constraint. This cannot be spoofed by any external wallet.

---

## 5. Indexer Architecture

### 5.1 ThunderSwap Indexer

Mirrors the existing `PLPIndexer` class pattern from `src/lib/indexer/plpIndexer.ts`.

```
thunderswap-repo/src/lib/indexer/thunderswapIndexer.ts
```

```typescript
export class ThunderSwapIndexer {
  private connection: Connection;
  private supabase: SupabaseClient<Database>;
  private programId: PublicKey;
  private eventParser: EventParser;      // Anchor EventParser with ThunderSwap IDL
  private checkInterval: number;         // 15s default
  private batchSize: number;             // 100 signatures per run

  async start(): Promise<void> { /* polling loop — same as PLPIndexer */ }

  private async processBatch(signatures: string[]): Promise<void> {
    // Parse each tx, extract anchor events, dispatch to handlers
  }

  private async handleSwapEvent(event: SwapEvent): Promise<void> {
    // INSERT thunderswap_trades, UPDATE thunderswap_pool_metrics
  }

  private async handleGraduationPoolCreatedEvent(event: GraduationPoolCreatedEvent): Promise<void> {
    // UPSERT thunderswap_pools
    // INSERT lp_burn_records
    // POST webhook → ThunderLaunch /api/webhooks/thunderswap-graduation
  }

  private async handlePoolCreatedEvent(event: PoolCreatedEvent): Promise<void> {
    // UPSERT thunderswap_pools
  }

  private async handleCreatorFeeClaimEvent(event: CreatorFeeClaimEvent): Promise<void> {
    // INSERT thunderswap_creator_fee_claims
  }
}
```

Events handled:

| Event | DB Action |
|---|---|
| `PoolCreatedEvent` | UPSERT `thunderswap_pools` |
| `GraduationPoolCreatedEvent` | UPSERT `thunderswap_pools`, INSERT `lp_burn_records`, POST webhook to ThunderLaunch |
| `SwapEvent` | INSERT `thunderswap_trades`, UPDATE `thunderswap_pool_metrics` |
| `LiquidityAddedEvent` | UPDATE `thunderswap_pools` reserves |
| `LiquidityRemovedEvent` | UPDATE `thunderswap_pools` reserves |
| `CreatorFeeClaimEvent` | INSERT `thunderswap_creator_fee_claims` |
| `GlobalConfigUpdatedEvent` | Update config cache |

### 5.2 Hybrid Graduation Event Discovery

Three redundant layers — all idempotent:

```
Layer 1 — Webhook (primary, < 1 second latency):
  ThunderSwap indexer detects GraduationPoolCreatedEvent
  → POST /api/webhooks/thunderswap-graduation on ThunderLaunch
  → ThunderLaunch updates: pools.migration_status = 'completed',
    pools.external_dex_pool_address, resolves migration_alerts

Layer 2 — On-chain log listener (fallback):
  ThunderLaunch PLPIndexer already watches GraduationEvent from PLP program
  → When graduation_dex == ThunderSwap, calls GET /api/thunderswap/pools/{mint}
  → If pool exists, marks migration complete
  → Catches webhook failures and ThunderLaunch downtime scenarios

Layer 3 — Reconciliation cron (safety net, every 15 minutes):
  Queries Supabase for tokens WHERE graduated = true
    AND migration_status != 'completed'
    AND graduation_dex = 'thunderswap'
  → For each: calls ThunderSwap API to verify pool
  → Updates DB if found, creates migration_alert if not
```

### 5.3 Database Schema (New Tables)

All tables follow the conventions in ThunderLaunch's `sql/schema.sql` (TIMESTAMPTZ, uuid_generate_v4, RLS, triggers).

#### `thunderswap_pools`

```sql
CREATE TABLE thunderswap_pools (
  id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  mint                TEXT NOT NULL UNIQUE,
  pool_address        TEXT NOT NULL UNIQUE,
  creator             TEXT NOT NULL,
  sol_reserves        BIGINT NOT NULL DEFAULT 0,
  token_reserves      BIGINT NOT NULL DEFAULT 0,
  lp_mint             TEXT,
  lp_supply           BIGINT NOT NULL DEFAULT 0,
  graduated_from_plp  BOOLEAN NOT NULL DEFAULT false,
  lp_burned           BOOLEAN NOT NULL DEFAULT false,
  locked              BOOLEAN NOT NULL DEFAULT false,
  creator_fees_accumulated BIGINT NOT NULL DEFAULT 0,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_thunderswap_pools_mint ON thunderswap_pools(mint);
CREATE INDEX idx_thunderswap_pools_creator ON thunderswap_pools(creator);
CREATE INDEX idx_thunderswap_pools_graduated ON thunderswap_pools(graduated_from_plp);
```

#### `thunderswap_trades`

Partitioned monthly (same pattern as ThunderLaunch trade tables).

```sql
CREATE TABLE thunderswap_trades (
  id              UUID NOT NULL DEFAULT uuid_generate_v4(),
  pool_address    TEXT NOT NULL,
  mint            TEXT NOT NULL,
  trader          TEXT NOT NULL,
  trade_type      trade_type NOT NULL,  -- existing enum: 'buy' | 'sell'
  sol_amount      BIGINT NOT NULL,
  token_amount    BIGINT NOT NULL,
  price_sol       NUMERIC(30, 18),
  price_usd       NUMERIC(30, 6),
  price_impact_bps INT,
  tx_signature    TEXT NOT NULL,
  block_time      TIMESTAMPTZ NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (block_time);

CREATE INDEX idx_thunderswap_trades_mint ON thunderswap_trades(mint);
CREATE INDEX idx_thunderswap_trades_pool ON thunderswap_trades(pool_address);
CREATE INDEX idx_thunderswap_trades_block_time ON thunderswap_trades(block_time DESC);
CREATE INDEX idx_thunderswap_trades_trader ON thunderswap_trades(trader);
```

#### `lp_burn_records`

```sql
CREATE TABLE lp_burn_records (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  mint            TEXT NOT NULL UNIQUE,
  pool_address    TEXT NOT NULL,
  lp_mint         TEXT NOT NULL,
  amount_burned   BIGINT NOT NULL,
  tx_sig          TEXT NOT NULL,
  burned_at       TIMESTAMPTZ NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_lp_burn_records_mint ON lp_burn_records(mint);
```

#### `thunderswap_pool_metrics`

```sql
CREATE TABLE thunderswap_pool_metrics (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  mint            TEXT NOT NULL UNIQUE REFERENCES thunderswap_pools(mint),
  price_sol       NUMERIC(30, 18),
  price_usd       NUMERIC(30, 6),
  tvl_sol         NUMERIC(30, 6),
  tvl_usd         NUMERIC(30, 6),
  volume_1h_usd   NUMERIC(30, 6) DEFAULT 0,
  volume_24h_usd  NUMERIC(30, 6) DEFAULT 0,
  trades_24h      INT DEFAULT 0,
  price_change_24h_pct NUMERIC(10, 4) DEFAULT 0,
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### `thunderswap_creator_fee_claims`

```sql
CREATE TABLE thunderswap_creator_fee_claims (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  mint            TEXT NOT NULL,
  pool_address    TEXT NOT NULL,
  creator         TEXT NOT NULL,
  amount_claimed  BIGINT NOT NULL,
  tx_sig          TEXT NOT NULL,
  claimed_at      TIMESTAMPTZ NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ts_fee_claims_creator ON thunderswap_creator_fee_claims(creator);
CREATE INDEX idx_ts_fee_claims_mint ON thunderswap_creator_fee_claims(mint);
```

---

## 6. Backend & API Layer

### 6.1 Hosting — swap.thunderlaunch.io (Recommended)

ThunderSwap frontend and API are hosted on the **`swap.thunderlaunch.io` subdomain** rather than a separate domain for v1.

Benefits:
- Shares the Next.js deployment on Vercel — no separate pipeline
- Shares the Supabase instance — no cross-DB joins for creator fee queries
- Shares auth (same Supabase users table)
- Shares Vercel cron infrastructure (`vercel.json`)
- Dramatically reduces operational overhead for v1

Migration to `thunderswap.io` is straightforward at v2 if brand separation is desired.

### 6.2 API Routes

Following the Next.js App Router pattern at `src/app/api/thunderswap/`:

| Method | Path | Description |
|---|---|---|
| GET | `/api/thunderswap/pools` | List pools with sorting (TVL, volume, recency) |
| GET | `/api/thunderswap/pools/{mint}` | Get pool by token mint |
| GET | `/api/thunderswap/pools/search?baseMint=&quoteMint=` | Find pool by pair |
| POST | `/api/thunderswap/quote` | Get swap quote `{ poolAddress, inputMint, outputMint, inputAmount, slippageBps }` |
| GET | `/api/thunderswap/creator-fees/{mint}` | Get claimable creator fees for a pool |
| GET | `/api/thunderswap/creator-fees/wallet/{address}` | All claimable fees for a creator (all pools) |
| GET | `/api/thunderswap/trades/{mint}` | Trade history for a pool (paginated) |
| POST | `/api/webhooks/thunderswap-graduation` | Graduation event receiver from ThunderSwap indexer |

The existing `src/lib/graduation/thunderswap.ts` stubs (`createPoolOnThunderSwap`, `verifyTokenLiveOnThunderSwap`) target these endpoints. Once ThunderSwap is live, updating `THUNDERSWAP_API_BASE` is the only change needed in ThunderLaunch.

### 6.3 Cron Jobs

Added to `vercel.json`:

| Path | Schedule | Purpose |
|---|---|---|
| `/api/cron/thunderswap-indexer` | `*/2 * * * *` | Primary event indexer (every 2 min) |
| `/api/cron/thunderswap-reconciliation` | `*/15 * * * *` | Graduation reconciliation |
| `/api/cron/thunderswap-metrics` | `*/15 * * * *` | Update pool metrics (TVL, volume, price change) |
| `/api/cron/thunderswap-lp-burn-verify` | `0 * * * *` | Hourly on-chain LP burn verification |

### 6.4 Real-Time (WebSocket)

Reuses Supabase Realtime (already used on ThunderLaunch home page and token detail page). ThunderSwap UI subscribes to `thunderswap_trades` INSERT events and `thunderswap_pool_metrics` UPDATE events using the same `supabase.channel()` pattern.

---

## 7. Frontend Architecture

### 7.1 Route Structure

Implemented as a Next.js App Router route group `(thunderswap)` within the same app:

```
src/app/(thunderswap)/
├── swap/
│   └── page.tsx                   # Main swap UI
├── pools/
│   ├── page.tsx                   # Pool discovery / token list
│   └── [mint]/
│       └── page.tsx               # Pool detail page (chart, reserves, trades)
└── creator-fees/
    └── page.tsx                   # Creator fee claim dashboard
```

### 7.2 Swap UI Component Structure

Mirrors `TradingPanel.tsx` patterns from ThunderLaunch:

```
SwapPanel.tsx
├── TokenSelector.tsx              # Input/output token selection (SOL base for v1)
├── SwapInput.tsx                  # Amount input with USD conversion
├── SlippageSettings.tsx           # 0.5% / 1.0% / custom
├── SwapQuoteDisplay.tsx           # Rate, price impact, fee breakdown (20/15/15 bps)
├── SwapButton.tsx                 # Builds and sends transaction via wallet-adapter
└── RecentSwaps.tsx                # Live trade feed from thunderswap_trades
```

UI color scheme follows ThunderLaunch's Dark Professional theme (defined in `CLAUDE.md`) — `bg-gray-900` primary, `bg-gray-800` cards, `border-gray-700`, `text-white` headings.

### 7.3 Creator Fee Claim UI

Appears in two locations (per spec):

**ThunderLaunch — My Profile → Created tab**:
- Existing tab gains a "ThunderSwap Fees" column per token row
- Shows claimable SOL amount from `GET /api/thunderswap/creator-fees/{mint}`
- "Claim" button builds `claim_creator_fees` transaction and sends via wallet-adapter

**ThunderSwap — `/creator-fees` page**:
- Lists all pools where `creator == connected wallet`
- Shows lifetime earned, claimable amount, last claimed
- Claim All button batches claims across pools

### 7.4 Token List / Discovery

- **Graduated tokens** (from ThunderLaunch): auto-listed when `lp_burn_records` row is created. Marked with "Graduated" badge.
- **Public pools**: listed after `PoolCreatedEvent` is indexed. Marked with "Unverified" badge.
- **Sorting**: TVL (default), 24h volume, recency, price change
- No on-chain token registry for v1 — `thunderswap_pools` table query is sufficient. A registry or Jupiter token list integration is a v2 item.

---

## 8. ThunderLaunch Integration Points

### 8.1 On-Chain — Graduation CPI

The only on-chain integration point. ThunderLaunch's PLP `graduate` instruction calls ThunderSwap's `create_pool_from_graduation` via CPI when `graduation_dex == ThunderSwap`. The `plp_graduation_authority` PDA is the signer — no external wallet can call this path.

Files to update in ThunderLaunch:
- `programs/plp/src/lib.rs` — add CPI call in `graduate` instruction
- `target/idl/plp.json` — regenerated after program change
- `src/lib/solana/plp.idl.json` — copy of IDL for frontend

### 8.2 Off-Chain — API Integration

The existing `src/lib/graduation/thunderswap.ts` defines the full API contract. Once ThunderSwap is live, update `THUNDERSWAP_API_BASE` to point to the live endpoint. No other changes needed in ThunderLaunch.

Key functions:
- `createPoolOnThunderSwap()` → `POST /api/thunderswap/pools/create` (off-chain fallback path only)
- `verifyTokenLiveOnThunderSwap()` → `GET /api/thunderswap/pools/{mint}`

### 8.3 Webhook — Graduation Status Sync

**ThunderSwap → ThunderLaunch**:
When ThunderSwap indexer detects `GraduationPoolCreatedEvent`, it POSTs to ThunderLaunch:
```
POST /api/webhooks/thunderswap-graduation
Authorization: Bearer <CRON_SECRET>
{
  "mint": "...",
  "pool_address": "...",
  "lp_mint": "...",
  "lp_amount_burned": 12345,
  "lp_burn_tx": "...",
  "timestamp": "2026-03-27T..."
}
```

ThunderLaunch handler (new endpoint to create):
- Updates `pools.migration_status = 'completed'`
- Updates `pools.external_dex_pool_address`
- Resolves open `migration_alerts` for this mint

### 8.4 Shared Database (Same Supabase Instance)

Key cross-references between ThunderLaunch and ThunderSwap tables:

| ThunderSwap Table | References ThunderLaunch Table | Purpose |
|---|---|---|
| `thunderswap_pools.mint` | `tokens.mint_address` | Token metadata (name, symbol, image) |
| `thunderswap_pools.creator` | `users.wallet_address` | Creator profile for fee claim UI |
| `lp_burn_records.mint` | `pools.mint` | Graduation record linkage |

ThunderLaunch's "My Profile → Created tab" JOINs `thunderswap_pools` on `creator = wallet_address` to show ThunderSwap fee data alongside bonding curve fee data.

### 8.5 Trade Panel Routing (Post-Graduation)

After a token graduates, ThunderLaunch's `TradingPanel.tsx` detects `pool.graduated_from_plp = true` and routes swap transactions to the ThunderSwap program instruction instead of the PLP bonding curve. The UX is identical — the routing is transparent to the user.

---

## 9. Security Architecture

### 9.1 Governance — Squads 3-of-5 Multisig

`GlobalConfig.authority` is set to the Squads multisig PDA at initialization. All privileged operations require M-of-N signatures via the Squads protocol.

| Signer | Holder |
|---|---|
| Key 1 | CEO / Founder |
| Key 2 | CTO / Lead Dev |
| Key 3 | COO / Second Founder |
| Key 4 | Cold hardware wallet (offline, stored in safe) |
| Key 5 | Legal entity wallet |

3-of-5 means: 2 key losses/compromises before loss of control, while allowing operations when one signer is unavailable.

Squads governs: fee rate changes, min liquidity threshold, protocol fee vault, emergency pool lock (`paused`), program upgrade authority.

### 9.2 On-Chain Security Model

| Risk | Mitigation |
|---|---|
| Unauthorized graduation pool creation | `create_pool_from_graduation` requires `plp_graduation_authority` PDA as signer — enforced by Anchor `has_one` constraint |
| LP token re-minting after burn | LP mint authority revoked in the same tx as the burn — impossible to re-mint |
| Re-entrancy | Solana's single-threaded execution prevents re-entrancy; CPI loops are blocked by the runtime |
| Sandwich attacks | `min_amount_out` slippage parameter enforced on every swap; frontend sets sensible defaults |
| Price manipulation via TWAP | TWAP requires sustained manipulation over the full window (5 min default); cost scales with pool size |
| Integer overflow | All arithmetic uses `checked_*` — same convention as ThunderLaunch PLP |
| Pool draining | Two independent circuit breakers: `pool.locked` (pool-level) and `GlobalConfig.paused` (global) |
| Fake ThunderLaunch authority | `plp_graduation_authority` is derived from PDA seeds on the PLP program's Program ID — cannot be spoofed by any external wallet |
| Creator fee theft | `claim_creator_fees` validates `signer == pool.creator` — only the registered creator can claim |

### 9.3 Program Upgrade Policy

- v1 launch: Program is upgradeable; upgrade authority held by Squads multisig
- Post-audit: After external audit passes and 90 days of stable mainnet operation, evaluate freezing the program (upgrade authority → `None`)
- The freeze timeline and decision criteria are documented in the ops runbook

### 9.4 Audit Requirements

**Mandatory before mainnet.** Scope:
- ThunderSwap Anchor program (all instructions and state transitions)
- CPI path from ThunderLaunch PLP to ThunderSwap
- LP burn mechanism and LpBurnRecord integrity
- TWAP oracle resistance to manipulation
- Fee math (no rounding exploits)
- Slippage and min-amount-out enforcement

Recommended auditing firms: OtterSec, Neodyme, Trail of Bits (Solana team).

---

## 10. Scalability

### 10.1 On-Chain

- `PoolState` account size is fixed at ~370 bytes — predictable rent, no dynamic resizing
- Swaps are single-instruction, single-signer — no address lookup tables required for v1
- TWAP `u128` accumulator is overflow-safe for ~136 years at 1 swap/second
- Each pool is a fully independent account — no cross-pool state contention
- Program can handle thousands of pools with no architectural changes

### 10.2 Indexer

Follows proven PLPIndexer patterns:
- Batch signature fetching (100 per run, configurable)
- 15-second poll interval (matching ThunderLaunch indexer)
- Anchor `EventParser` for efficient log extraction
- Idempotent upserts with duplicate error (`23505`) handling
- For higher throughput: switch from polling to Helius enhanced websocket subscriptions (`logsSubscribe` on ThunderSwap program ID)

### 10.3 Database

- `thunderswap_trades` partitioned by time (monthly) — same pattern as ThunderLaunch trade tables
- Standard indexes on `mint`, `pool_address`, `block_time`, `trader`
- `thunderswap_pool_metrics` updated by cron (not on every trade write) — avoids hot-row contention at high volume
- Supabase PgBouncer connection pooling (already configured in ThunderLaunch)

### 10.4 API Layer

- Vercel Edge Functions for the quote endpoint — lowest possible latency
- Redis cache (30-second TTL) for pool state reads — optional for v1, add when RPC costs become significant
- Supabase connection pooling via PgBouncer handles concurrent API requests

---

## 11. Multi-Chain Strategy

### 11.1 Rollout Timeline

| Version | Chain | Timeline | Notes |
|---|---|---|---|
| v1 | Solana mainnet | Initial launch | Anchor / Rust program |
| v2 | Base (L2) | 6 months post Solana v1 | Solidity / Foundry; EVM port |
| v3 | Ethereum mainnet | 10 months post Solana v1 | After Base proves stability; full audit required |

**Why Base before Ethereum mainnet**: Gas fees on Ethereum mainnet make bonding curve micro-trades economically unviable (> $10/trade at typical gas prices). Base has ~100× lower gas, $2B+ TVL, and a growing memecoin culture that matches ThunderLaunch's target market.

### 11.2 Chain Isolation Principles

- Each chain version lives in a separate program/contract directory (`programs/thunderswap-solana/`, `contracts/thunderswap-evm/`)
- Chain-agnostic business logic (fee math, TWAP formulas) is documented in this architecture document and re-implemented per chain — no shared code between Rust and Solidity
- The frontend uses a chain abstraction layer (`src/lib/thunderswap/chain.ts`) that routes to the correct program SDK based on the connected wallet's chain
- The database uses the existing `chain` enum from ThunderLaunch's schema (`solana`, `base`, `ethereum`) to tag all ThunderSwap pool and trade records

### 11.3 Base (v2) Design

- Uniswap V2 factory pattern (`CREATE2` for deterministic pool addresses)
- Same 50 bps fee structure (20/15/15) — maintains brand consistency
- Graduation from ThunderLaunch EVM: `IThunderSwapFactory.createPoolFromGraduation()` call in Solidity
- LP burn: `ERC20.burn()` in the same transaction as pool creation
- Creator fee vault: ERC-20 token accumulates creator fees; creator calls `claimCreatorFees()` to withdraw

### 11.4 Ethereum Mainnet (v3) Design

- Evaluate concentrated liquidity (V3 style) for higher capital efficiency at Ethereum TVL levels
- Revisit fee tiers given higher gas costs
- TWAP oracle: integrate Chainlink or Uniswap V3 oracle (more robust than in-pool TWAP at this scale)
- Full independent audit required before deployment

---

## 12. Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| Solana Program | Rust, Anchor 0.30+ | Same framework as ThunderLaunch PLP |
| Program Testing | Anchor test framework, Bankrun | Matches ThunderLaunch test setup |
| Frontend | Next.js 14+ (App Router) | Shared with ThunderLaunch |
| Styling | Tailwind CSS | Shared Dark Professional theme |
| Wallet | `@solana/wallet-adapter` | Shared with ThunderLaunch |
| Solana SDK | `@solana/web3.js`, `@coral-xyz/anchor` | Shared with ThunderLaunch |
| Database | Supabase (PostgreSQL 15) | Shared Supabase instance |
| Real-time | Supabase Realtime | Same WebSocket pattern as ThunderLaunch |
| Server state | React Query | Shared with ThunderLaunch |
| Indexer | TypeScript, Anchor `EventParser` | Mirrors `PLPIndexer` class |
| Cron | Vercel Cron Jobs | Same `vercel.json` infrastructure |
| RPC | Helius (primary), QuickNode (fallback) | Same providers as ThunderLaunch |
| Governance | Squads Protocol (3-of-5 multisig) | Controls `GlobalConfig` |
| Hosting | Vercel (`swap.thunderlaunch.io`) | Same Vercel project as ThunderLaunch |
| Monitoring | ThunderLaunch observability infrastructure | Extend existing alert system |
| EVM (v2) | Solidity, Foundry | Base deployment |

---

## Appendices

### Appendix A — Open Items Resolution

All open items from the planning document are resolved in this architecture:

| Item | Decision | Rationale |
|---|---|---|
| AMM model | Constant product x·y=k (Uniswap V2) | Same math as ThunderLaunch PLP, battle-tested, simplest to audit |
| Creator fee claim | Pull model (`claim_creator_fees` instruction) | Mirrors existing ThunderLaunch on-chain pattern exactly |
| UI hosting | `swap.thunderlaunch.io` subdomain | Shared infra cuts operational overhead; separable at v2 |
| Token discovery | Graduated-first, DB-backed, Unverified badge for public | No registry overhead for v1 |
| Price oracle | TWAP from pool reserves | Sufficient for v1; Pyth/Switchboard at v2 |

### Appendix B — Key ThunderLaunch Files for Reference

These existing ThunderLaunch files are the primary reference for ThunderSwap implementation:

| File | Relevance |
|---|---|
| `programs/plp/src/lib.rs` | Complete Anchor program: account structs, constant product math, PDA patterns, fee escrow, graduation instruction, LP token minting |
| `src/lib/graduation/thunderswap.ts` | REST API client stub — defines the exact API contract ThunderSwap must fulfill |
| `src/lib/graduation/executeGraduationFlow.ts` | Full graduation orchestration — defines how ThunderSwap integrates as graduation destination |
| `src/lib/indexer/plpIndexer.ts` | `PLPIndexer` class — ThunderSwap indexer mirrors this structure |
| `sql/schema.sql` | Full Supabase schema including enums, partitioning patterns, and RLS conventions |
| `src/lib/graduation/releaseVaultLiquidity.ts` | Current vault release logic — replaced by CPI for ThunderSwap path |

### Appendix C — Deployment Checklist (Pre-Mainnet)

- [ ] Anchor program builds with zero warnings
- [ ] All instructions covered by unit tests (Bankrun)
- [ ] Integration test: full graduation CPI flow on devnet
- [ ] Integration test: LP burn verified on-chain
- [ ] Integration test: creator fee accumulation and claim
- [ ] Integration test: slippage protection rejects out-of-range swaps
- [ ] Integration test: `create_pool_from_graduation` fails when called without PLP authority
- [ ] External security audit completed
- [ ] Bug bounty program open for minimum 2 weeks
- [ ] Squads multisig configured and tested on devnet
- [ ] Upgrade authority transferred to Squads multisig
- [ ] Indexer running on devnet without errors for 48 hours
- [ ] Webhook integration tested end-to-end with ThunderLaunch
- [ ] Reconciliation cron verified catching missed events
- [ ] Frontend tested on mobile (375px), tablet (768px), desktop (1024px+)
