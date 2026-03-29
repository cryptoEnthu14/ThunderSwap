# ThunderSwap — Database Design

**Version**: 1.0
**Status**: Design Phase — ready for implementation
**Database**: PostgreSQL 15 via Supabase (shared instance with ThunderLaunch)
**Related**: [THUNDERSWAP_ARCHITECTURE.md](THUNDERSWAP_ARCHITECTURE.md), [THUNDERSWAP_SMART_CONTRACT.md](THUNDERSWAP_SMART_CONTRACT.md)

This document is the authoritative schema specification for ThunderSwap. All tables follow ThunderLaunch's existing `sql/schema.sql` conventions (TIMESTAMPTZ, UUID primary keys, RLS, triggers). Run these as migrations against the shared Supabase instance.

---

## Table of Contents

1. [Schema Overview](#1-schema-overview)
2. [Tables](#2-tables)
   - [thunderswap_pools](#21-thunderswap_pools)
   - [thunderswap_trades](#22-thunderswap_trades)
   - [lp_burn_records](#23-lp_burn_records)
   - [thunderswap_pool_metrics](#24-thunderswap_pool_metrics)
   - [thunderswap_liquidity_events](#25-thunderswap_liquidity_events)
   - [thunderswap_creator_fee_claims](#26-thunderswap_creator_fee_claims)
3. [Views & Materialized Views](#3-views--materialized-views)
4. [Triggers & Functions](#4-triggers--functions)
5. [Row Level Security Policies](#5-row-level-security-policies)
6. [Indexes Reference](#6-indexes-reference)
7. [Cross-Table Relationships](#7-cross-table-relationships)
8. [Migration Files](#8-migration-files)
9. [Common Query Patterns](#9-common-query-patterns)

---

## 1. Schema Overview

### Tables

| Table | Rows (est.) | Purpose |
|---|---|---|
| `thunderswap_pools` | 10K–100K | One row per AMM pool (one per token) |
| `thunderswap_trades` | 10M+ | Every swap event, partitioned monthly |
| `lp_burn_records` | = graduated tokens | LP burn audit record for each graduated pool |
| `thunderswap_pool_metrics` | = pools | Latest computed price, TVL, volume per pool |
| `thunderswap_liquidity_events` | 100K+ | Add/remove liquidity events for public pools |
| `thunderswap_creator_fee_claims` | 100K+ | Creator fee withdrawal history |

### Relationships to ThunderLaunch Tables

| ThunderSwap Table | ThunderLaunch Table | Join Column | Purpose |
|---|---|---|---|
| `thunderswap_pools.mint` | `tokens.mint_address` | Soft join | Token name, symbol, image |
| `thunderswap_pools.creator` | `users.wallet_address` | Soft join | Creator profile in UI |
| `lp_burn_records.mint` | `pools.mint` | Soft join | Bonding curve audit linkage |

All joins are **soft** (no foreign key constraints across tables) — the indexer may write ThunderSwap records before ThunderLaunch records are updated, so hard FK constraints would cause insert failures.

### Who Writes Each Table

| Table | Writer |
|---|---|
| All tables | `service_role` (ThunderSwap indexer + cron jobs) |
| All tables | Read-only for `anon` and `authenticated` roles |

---

## 2. Tables

### 2.1 thunderswap_pools

Core pool state. Updated by the indexer on every `PoolCreatedEvent` and `GraduationPoolCreatedEvent`.

```sql
-- =========================================
-- THUNDERSWAP_POOLS TABLE
-- =========================================
CREATE TABLE thunderswap_pools (
  id                        UUID PRIMARY KEY DEFAULT uuid_generate_v4(),

  -- On-chain identifiers
  mint                      TEXT NOT NULL UNIQUE,     -- SPL token mint address
  pool_address              TEXT NOT NULL UNIQUE,     -- AmmPool PDA address
  lp_mint                   TEXT,                     -- LP token mint (NULL until indexed)

  -- Creator
  creator                   TEXT NOT NULL,            -- wallet address of token creator

  -- Reserves (mirrors on-chain AmmPool state)
  sol_reserve               BIGINT NOT NULL DEFAULT 0,     -- lamports
  token_reserve             BIGINT NOT NULL DEFAULT 0,     -- raw token units

  -- LP supply
  lp_supply                 BIGINT NOT NULL DEFAULT 0,     -- outstanding LP tokens (0 for graduated)

  -- Graduation state
  graduated_from_plp        BOOLEAN NOT NULL DEFAULT false,
  lp_burned                 BOOLEAN NOT NULL DEFAULT false,

  -- Pool status
  locked                    BOOLEAN NOT NULL DEFAULT false,  -- per-pool emergency pause

  -- Fee tracking (accumulated lifetime creator fees in lamports)
  creator_fees_accumulated  BIGINT NOT NULL DEFAULT 0,

  -- Metadata (populated from ThunderLaunch tokens table via JOIN — not stored here)
  -- name, symbol, image are fetched via JOIN on tokens.mint_address

  -- Timestamps
  created_at                TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at                TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  -- On-chain slot when pool was created
  creation_slot             BIGINT
);

-- Indexes
CREATE INDEX idx_ts_pools_mint          ON thunderswap_pools(mint);
CREATE INDEX idx_ts_pools_creator       ON thunderswap_pools(creator);
CREATE INDEX idx_ts_pools_graduated     ON thunderswap_pools(graduated_from_plp)
  WHERE graduated_from_plp = true;
CREATE INDEX idx_ts_pools_lp_burned     ON thunderswap_pools(lp_burned)
  WHERE lp_burned = true;
CREATE INDEX idx_ts_pools_created_at    ON thunderswap_pools(created_at DESC);

ALTER TABLE thunderswap_pools ENABLE ROW LEVEL SECURITY;

COMMENT ON TABLE thunderswap_pools IS
  'One row per ThunderSwap AMM pool. Mirrors on-chain AmmPool account state. '
  'Updated by the ThunderSwap indexer on PoolCreatedEvent and GraduationPoolCreatedEvent.';
COMMENT ON COLUMN thunderswap_pools.sol_reserve IS
  'Pool SOL reserves in lamports. Must equal sol_vault on-chain balance minus rent-exempt minimum.';
COMMENT ON COLUMN thunderswap_pools.creator_fees_accumulated IS
  'Lifetime creator fees earned in lamports. Includes both claimed and unclaimed fees.';
```

---

### 2.2 thunderswap_trades

Every swap on ThunderSwap. High-volume table — partitioned monthly.

```sql
-- =========================================
-- THUNDERSWAP_TRADES TABLE (PARTITIONED)
-- =========================================
CREATE TABLE thunderswap_trades (
  id                UUID        NOT NULL DEFAULT uuid_generate_v4(),
  pool_address      TEXT        NOT NULL,   -- AmmPool PDA address
  mint              TEXT        NOT NULL,   -- SPL token mint
  trader            TEXT        NOT NULL,   -- wallet address of swapper
  trade_type        trade_type  NOT NULL,   -- 'buy' | 'sell' (reuse ThunderLaunch enum)
  sol_amount        BIGINT      NOT NULL,   -- lamports (in for buy, out-to-user for sell)
  token_amount      BIGINT      NOT NULL,   -- raw token units (out for buy, in for sell)
  protocol_fee      BIGINT      NOT NULL DEFAULT 0,  -- lamports to protocol vault
  creator_fee       BIGINT      NOT NULL DEFAULT 0,  -- lamports to creator vault
  price_sol         NUMERIC(30, 18),        -- token price in SOL at time of trade
  price_usd         NUMERIC(30, 6),         -- token price in USD at time of trade
  price_impact_bps  INTEGER,                -- price impact in basis points
  sol_reserve_after BIGINT,                 -- pool sol_reserve after this trade
  token_reserve_after BIGINT,               -- pool token_reserve after this trade
  tx_signature      TEXT        NOT NULL,   -- Solana transaction signature
  block_time        TIMESTAMPTZ NOT NULL,   -- on-chain block timestamp (UTC)
  slot              BIGINT,                 -- on-chain slot number
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (block_time);

-- Create monthly partitions (add new ones as needed)
CREATE TABLE thunderswap_trades_2026_03
  PARTITION OF thunderswap_trades
  FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');

CREATE TABLE thunderswap_trades_2026_04
  PARTITION OF thunderswap_trades
  FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');

CREATE TABLE thunderswap_trades_2026_05
  PARTITION OF thunderswap_trades
  FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE TABLE thunderswap_trades_2026_06
  PARTITION OF thunderswap_trades
  FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE TABLE thunderswap_trades_2026_07
  PARTITION OF thunderswap_trades
  FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');

CREATE TABLE thunderswap_trades_2026_08
  PARTITION OF thunderswap_trades
  FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');

CREATE TABLE thunderswap_trades_2026_09
  PARTITION OF thunderswap_trades
  FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');

CREATE TABLE thunderswap_trades_2026_10
  PARTITION OF thunderswap_trades
  FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');

CREATE TABLE thunderswap_trades_2026_11
  PARTITION OF thunderswap_trades
  FOR VALUES FROM ('2026-11-01') TO ('2026-12-01');

CREATE TABLE thunderswap_trades_2026_12
  PARTITION OF thunderswap_trades
  FOR VALUES FROM ('2026-12-01') TO ('2027-01-01');

-- Default partition catches anything outside defined ranges
CREATE TABLE thunderswap_trades_default
  PARTITION OF thunderswap_trades DEFAULT;

-- Indexes (created on parent — propagate to all partitions)
CREATE INDEX idx_ts_trades_mint         ON thunderswap_trades(mint, block_time DESC);
CREATE INDEX idx_ts_trades_pool         ON thunderswap_trades(pool_address, block_time DESC);
CREATE INDEX idx_ts_trades_trader       ON thunderswap_trades(trader, block_time DESC);
CREATE INDEX idx_ts_trades_block_time   ON thunderswap_trades(block_time DESC);
CREATE INDEX idx_ts_trades_tx_sig       ON thunderswap_trades(tx_signature);

-- Unique constraint to prevent duplicate indexing of the same tx
CREATE UNIQUE INDEX idx_ts_trades_tx_unique ON thunderswap_trades(tx_signature, trade_type);

ALTER TABLE thunderswap_trades ENABLE ROW LEVEL SECURITY;

COMMENT ON TABLE thunderswap_trades IS
  'Every swap executed on ThunderSwap. Partitioned monthly for query performance. '
  'Populated by the ThunderSwap indexer from SwapEvent.';
COMMENT ON COLUMN thunderswap_trades.sol_amount IS
  'For buys: lamports spent by trader. For sells: net lamports received by trader (after fees).';
COMMENT ON COLUMN thunderswap_trades.trade_type IS
  'buy = SOL→token, sell = token→SOL. Reuses ThunderLaunch trade_type enum.';
```

---

### 2.3 lp_burn_records

Immutable audit record for every LP token burn at graduation. One row per graduated pool.

```sql
-- =========================================
-- LP_BURN_RECORDS TABLE
-- =========================================
CREATE TABLE lp_burn_records (
  id              UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),

  -- Pool identifiers
  mint            TEXT        NOT NULL UNIQUE,   -- SPL token mint address
  pool_address    TEXT        NOT NULL UNIQUE,   -- AmmPool PDA address
  lp_mint         TEXT        NOT NULL,          -- LP token mint address (now revoked)

  -- Burn details
  amount_burned   BIGINT      NOT NULL,          -- LP tokens burned (= integer_sqrt(sol * token))
  tx_sig          TEXT        NOT NULL,          -- Solana tx signature of the burn transaction
  burn_slot       BIGINT,                        -- on-chain slot of the burn
  burned_at       TIMESTAMPTZ NOT NULL,          -- UTC timestamp of burn

  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_lp_burn_records_mint         ON lp_burn_records(mint);
CREATE INDEX idx_lp_burn_records_pool         ON lp_burn_records(pool_address);
CREATE INDEX idx_lp_burn_records_burned_at    ON lp_burn_records(burned_at DESC);

ALTER TABLE lp_burn_records ENABLE ROW LEVEL SECURITY;

COMMENT ON TABLE lp_burn_records IS
  'Immutable audit record of LP token burns at graduation. One row per graduated pool. '
  'Written by indexer when GraduationPoolCreatedEvent is detected. Never updated.';
COMMENT ON COLUMN lp_burn_records.amount_burned IS
  'Total LP tokens burned = integer_sqrt(sol_amount × token_amount). '
  'Proves liquidity is permanently locked — no LP tokens remain in circulation.';
```

---

### 2.4 thunderswap_pool_metrics

Latest computed metrics per pool. Updated by the `thunderswap-metrics` cron (every 15 minutes), not on every trade write — avoids hot-row contention.

```sql
-- =========================================
-- THUNDERSWAP_POOL_METRICS TABLE
-- =========================================
CREATE TABLE thunderswap_pool_metrics (
  id                    UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  mint                  TEXT        NOT NULL UNIQUE REFERENCES thunderswap_pools(mint) ON DELETE CASCADE,

  -- Price
  price_sol             NUMERIC(30, 18) DEFAULT 0,   -- current price in SOL per token
  price_usd             NUMERIC(30, 6)  DEFAULT 0,   -- current price in USD per token

  -- Market cap
  market_cap_sol        NUMERIC(30, 6)  DEFAULT 0,
  market_cap_usd        NUMERIC(30, 6)  DEFAULT 0,

  -- TVL (total value locked in pool)
  tvl_sol               NUMERIC(30, 6)  DEFAULT 0,
  tvl_usd               NUMERIC(30, 6)  DEFAULT 0,

  -- Volume
  volume_1h_usd         NUMERIC(30, 6)  DEFAULT 0,
  volume_6h_usd         NUMERIC(30, 6)  DEFAULT 0,
  volume_24h_usd        NUMERIC(30, 6)  DEFAULT 0,
  volume_7d_usd         NUMERIC(30, 6)  DEFAULT 0,

  -- Trade counts
  trades_1h             INTEGER         DEFAULT 0,
  trades_24h            INTEGER         DEFAULT 0,

  -- Price changes
  price_change_1h_pct   NUMERIC(10, 4)  DEFAULT 0,
  price_change_6h_pct   NUMERIC(10, 4)  DEFAULT 0,
  price_change_24h_pct  NUMERIC(10, 4)  DEFAULT 0,

  -- All-time high
  ath_price_usd         NUMERIC(30, 6)  DEFAULT 0,
  ath_market_cap_usd    NUMERIC(30, 6)  DEFAULT 0,

  -- Holders (estimated from trade data)
  unique_traders_24h    INTEGER         DEFAULT 0,

  updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_ts_pool_metrics_mint       ON thunderswap_pool_metrics(mint);
CREATE INDEX idx_ts_pool_metrics_tvl        ON thunderswap_pool_metrics(tvl_usd DESC);
CREATE INDEX idx_ts_pool_metrics_volume     ON thunderswap_pool_metrics(volume_24h_usd DESC);
CREATE INDEX idx_ts_pool_metrics_updated    ON thunderswap_pool_metrics(updated_at DESC);

ALTER TABLE thunderswap_pool_metrics ENABLE ROW LEVEL SECURITY;

COMMENT ON TABLE thunderswap_pool_metrics IS
  'Latest computed metrics for each ThunderSwap pool. '
  'Updated by thunderswap-metrics cron every 15 minutes. Not updated on individual trades.';
```

---

### 2.5 thunderswap_liquidity_events

Records of liquidity add/remove events for public (non-graduated) pools. Graduated pools have no liquidity events (LP is burned at creation).

```sql
-- =========================================
-- THUNDERSWAP_LIQUIDITY_EVENTS TABLE
-- =========================================
CREATE TABLE thunderswap_liquidity_events (
  id              UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  pool_address    TEXT        NOT NULL,
  mint            TEXT        NOT NULL,
  provider        TEXT        NOT NULL,   -- wallet address of LP
  event_type      TEXT        NOT NULL    CHECK (event_type IN ('add', 'remove')),
  sol_amount      BIGINT      NOT NULL,   -- lamports added or removed
  token_amount    BIGINT      NOT NULL,   -- raw token units added or removed
  lp_amount       BIGINT      NOT NULL,   -- LP tokens minted (add) or burned (remove)
  tx_signature    TEXT        NOT NULL,
  block_time      TIMESTAMPTZ NOT NULL,
  slot            BIGINT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_ts_liq_events_pool     ON thunderswap_liquidity_events(pool_address, block_time DESC);
CREATE INDEX idx_ts_liq_events_mint     ON thunderswap_liquidity_events(mint);
CREATE INDEX idx_ts_liq_events_provider ON thunderswap_liquidity_events(provider);
CREATE UNIQUE INDEX idx_ts_liq_events_tx ON thunderswap_liquidity_events(tx_signature, event_type);

ALTER TABLE thunderswap_liquidity_events ENABLE ROW LEVEL SECURITY;

COMMENT ON TABLE thunderswap_liquidity_events IS
  'Add/remove liquidity events for public ThunderSwap pools. '
  'Not populated for graduated pools (LP burned at graduation, no subsequent liquidity changes).';
```

---

### 2.6 thunderswap_creator_fee_claims

History of every `claim_creator_fees` instruction execution. Append-only.

```sql
-- =========================================
-- THUNDERSWAP_CREATOR_FEE_CLAIMS TABLE
-- =========================================
CREATE TABLE thunderswap_creator_fee_claims (
  id              UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  mint            TEXT        NOT NULL,   -- SPL token mint
  pool_address    TEXT        NOT NULL,   -- AmmPool PDA address
  creator         TEXT        NOT NULL,   -- wallet that claimed fees
  amount_claimed  BIGINT      NOT NULL,   -- lamports claimed
  tx_signature    TEXT        NOT NULL,
  claimed_at      TIMESTAMPTZ NOT NULL,   -- on-chain block time (UTC)
  slot            BIGINT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_ts_fee_claims_mint       ON thunderswap_creator_fee_claims(mint);
CREATE INDEX idx_ts_fee_claims_creator    ON thunderswap_creator_fee_claims(creator, claimed_at DESC);
CREATE INDEX idx_ts_fee_claims_pool       ON thunderswap_creator_fee_claims(pool_address);
CREATE UNIQUE INDEX idx_ts_fee_claims_tx  ON thunderswap_creator_fee_claims(tx_signature);

ALTER TABLE thunderswap_creator_fee_claims ENABLE ROW LEVEL SECURITY;

COMMENT ON TABLE thunderswap_creator_fee_claims IS
  'History of every creator fee claim on ThunderSwap. Append-only — never updated. '
  'Populated by indexer on CreatorFeeClaimEvent.';
```

---

## 3. Views & Materialized Views

### 3.1 `v_thunderswap_pools_enriched` (View)

Enriches pool data with token metadata from ThunderLaunch `tokens` table and latest metrics. Used by the pool listing API.

```sql
CREATE OR REPLACE VIEW v_thunderswap_pools_enriched AS
SELECT
  p.id,
  p.mint,
  p.pool_address,
  p.creator,
  p.sol_reserve,
  p.token_reserve,
  p.lp_supply,
  p.graduated_from_plp,
  p.lp_burned,
  p.locked,
  p.creator_fees_accumulated,
  p.created_at,
  -- Token metadata from ThunderLaunch
  t.name         AS token_name,
  t.symbol       AS token_symbol,
  t.image_url    AS token_image,
  t.description  AS token_description,
  -- Latest metrics
  m.price_sol,
  m.price_usd,
  m.market_cap_usd,
  m.tvl_usd,
  m.volume_24h_usd,
  m.volume_7d_usd,
  m.trades_24h,
  m.price_change_24h_pct,
  m.updated_at   AS metrics_updated_at
FROM thunderswap_pools p
LEFT JOIN tokens t
  ON t.mint_address = p.mint            -- ThunderLaunch tokens table
LEFT JOIN thunderswap_pool_metrics m
  ON m.mint = p.mint;

COMMENT ON VIEW v_thunderswap_pools_enriched IS
  'Pool data joined with ThunderLaunch token metadata and latest metrics. '
  'Use for pool listing API — avoids repeated JOINs in application code.';
```

### 3.2 `mv_thunderswap_top_pools` (Materialized View)

Top pools by TVL and 24h volume. Refreshed by the metrics cron every 15 minutes. Used for the ThunderSwap home page pool discovery feed.

```sql
CREATE MATERIALIZED VIEW mv_thunderswap_top_pools AS
SELECT
  p.mint,
  p.pool_address,
  p.creator,
  p.graduated_from_plp,
  t.name         AS token_name,
  t.symbol       AS token_symbol,
  t.image_url    AS token_image,
  m.price_usd,
  m.market_cap_usd,
  m.tvl_usd,
  m.volume_24h_usd,
  m.price_change_24h_pct,
  m.trades_24h,
  p.created_at
FROM thunderswap_pools p
JOIN thunderswap_pool_metrics m ON m.mint = p.mint
LEFT JOIN tokens t ON t.mint_address = p.mint
WHERE p.locked = false
ORDER BY m.tvl_usd DESC NULLS LAST
LIMIT 200;

CREATE UNIQUE INDEX idx_mv_top_pools_mint ON mv_thunderswap_top_pools(mint);
CREATE INDEX idx_mv_top_pools_tvl         ON mv_thunderswap_top_pools(tvl_usd DESC);
CREATE INDEX idx_mv_top_pools_volume      ON mv_thunderswap_top_pools(volume_24h_usd DESC);

COMMENT ON MATERIALIZED VIEW mv_thunderswap_top_pools IS
  'Top 200 pools by TVL. Refreshed every 15 min by thunderswap-metrics cron. '
  'Call: REFRESH MATERIALIZED VIEW CONCURRENTLY mv_thunderswap_top_pools';
```

### 3.3 `v_creator_fee_summary` (View)

Used by the creator fee claim UI to show claimable amounts per token.

```sql
CREATE OR REPLACE VIEW v_creator_fee_summary AS
SELECT
  p.mint,
  p.pool_address,
  p.creator,
  t.name           AS token_name,
  t.symbol         AS token_symbol,
  t.image_url      AS token_image,
  p.creator_fees_accumulated,
  COALESCE(SUM(c.amount_claimed), 0)                             AS total_claimed,
  p.creator_fees_accumulated - COALESCE(SUM(c.amount_claimed), 0) AS estimated_claimable,
  MAX(c.claimed_at)                                              AS last_claimed_at
FROM thunderswap_pools p
LEFT JOIN tokens t
  ON t.mint_address = p.mint
LEFT JOIN thunderswap_creator_fee_claims c
  ON c.mint = p.mint
GROUP BY p.mint, p.pool_address, p.creator, t.name, t.symbol, t.image_url, p.creator_fees_accumulated;

COMMENT ON VIEW v_creator_fee_summary IS
  'Per-token fee summary for creator UI. estimated_claimable is approximate — '
  'actual claimable lamports are authoritative from the creator_fee_vault on-chain balance.';
```

---

## 4. Triggers & Functions

### 4.1 `update_updated_at_column` (Reuse from ThunderLaunch)

ThunderLaunch already defines this function in `sql/schema.sql`. Apply it to tables that have `updated_at`.

```sql
-- Apply to thunderswap_pools (thunderswap_pool_metrics updated_at set manually by cron)
CREATE TRIGGER update_thunderswap_pools_updated_at
  BEFORE UPDATE ON thunderswap_pools
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### 4.2 `create_pool_metrics_on_insert`

Auto-create a `thunderswap_pool_metrics` row with default zeros whenever a pool is inserted. Prevents null joins in the enriched view.

```sql
CREATE OR REPLACE FUNCTION create_thunderswap_pool_metrics_on_insert()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO thunderswap_pool_metrics (mint)
  VALUES (NEW.mint)
  ON CONFLICT (mint) DO NOTHING;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_create_ts_pool_metrics
  AFTER INSERT ON thunderswap_pools
  FOR EACH ROW EXECUTE FUNCTION create_thunderswap_pool_metrics_on_insert();
```

### 4.3 `create_monthly_trade_partition`

Called by the metrics cron to pre-create next month's partition before it is needed.

```sql
CREATE OR REPLACE FUNCTION create_thunderswap_trade_partition_if_not_exists(
  partition_date DATE
) RETURNS VOID AS $$
DECLARE
  partition_name TEXT;
  start_date     DATE;
  end_date       DATE;
BEGIN
  partition_name := 'thunderswap_trades_' || TO_CHAR(partition_date, 'YYYY_MM');
  start_date     := DATE_TRUNC('month', partition_date);
  end_date       := start_date + INTERVAL '1 month';

  IF NOT EXISTS (
    SELECT 1 FROM pg_class WHERE relname = partition_name
  ) THEN
    EXECUTE FORMAT(
      'CREATE TABLE %I PARTITION OF thunderswap_trades FOR VALUES FROM (%L) TO (%L)',
      partition_name, start_date, end_date
    );
    RAISE NOTICE 'Created partition: %', partition_name;
  END IF;
END;
$$ LANGUAGE plpgsql;
```

### 4.4 `refresh_top_pools_mv`

Refreshes the materialized view concurrently (no locking). Called by the metrics cron.

```sql
CREATE OR REPLACE FUNCTION refresh_thunderswap_top_pools()
RETURNS VOID AS $$
BEGIN
  REFRESH MATERIALIZED VIEW CONCURRENTLY mv_thunderswap_top_pools;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

## 5. Row Level Security Policies

All ThunderSwap tables use the same RLS pattern as ThunderLaunch:
- Public read (SELECT) for all authenticated and anonymous users
- All writes (INSERT, UPDATE, DELETE) restricted to `service_role` only (the indexer uses the service role key)

```sql
-- =========================================
-- RLS: thunderswap_pools
-- =========================================
CREATE POLICY "Public read thunderswap_pools"
  ON thunderswap_pools FOR SELECT
  USING (true);

CREATE POLICY "Service role manages thunderswap_pools"
  ON thunderswap_pools FOR ALL
  USING (auth.role() = 'service_role');


-- =========================================
-- RLS: thunderswap_trades
-- =========================================
CREATE POLICY "Public read thunderswap_trades"
  ON thunderswap_trades FOR SELECT
  USING (true);

CREATE POLICY "Service role manages thunderswap_trades"
  ON thunderswap_trades FOR ALL
  USING (auth.role() = 'service_role');


-- =========================================
-- RLS: lp_burn_records
-- =========================================
CREATE POLICY "Public read lp_burn_records"
  ON lp_burn_records FOR SELECT
  USING (true);

CREATE POLICY "Service role manages lp_burn_records"
  ON lp_burn_records FOR ALL
  USING (auth.role() = 'service_role');


-- =========================================
-- RLS: thunderswap_pool_metrics
-- =========================================
CREATE POLICY "Public read thunderswap_pool_metrics"
  ON thunderswap_pool_metrics FOR SELECT
  USING (true);

CREATE POLICY "Service role manages thunderswap_pool_metrics"
  ON thunderswap_pool_metrics FOR ALL
  USING (auth.role() = 'service_role');


-- =========================================
-- RLS: thunderswap_liquidity_events
-- =========================================
CREATE POLICY "Public read thunderswap_liquidity_events"
  ON thunderswap_liquidity_events FOR SELECT
  USING (true);

CREATE POLICY "Service role manages thunderswap_liquidity_events"
  ON thunderswap_liquidity_events FOR ALL
  USING (auth.role() = 'service_role');


-- =========================================
-- RLS: thunderswap_creator_fee_claims
-- =========================================
CREATE POLICY "Public read thunderswap_creator_fee_claims"
  ON thunderswap_creator_fee_claims FOR SELECT
  USING (true);

CREATE POLICY "Service role manages thunderswap_creator_fee_claims"
  ON thunderswap_creator_fee_claims FOR ALL
  USING (auth.role() = 'service_role');
```

---

## 6. Indexes Reference

Complete index list for performance verification.

| Table | Index | Columns | Type | Purpose |
|---|---|---|---|---|
| `thunderswap_pools` | `idx_ts_pools_mint` | `mint` | UNIQUE | Pool lookup by mint |
| `thunderswap_pools` | `idx_ts_pools_creator` | `creator` | B-tree | Creator fee UI — "my pools" |
| `thunderswap_pools` | `idx_ts_pools_graduated` | `graduated_from_plp` | Partial (true) | Graduated pool filter |
| `thunderswap_pools` | `idx_ts_pools_lp_burned` | `lp_burned` | Partial (true) | Locked pool filter |
| `thunderswap_pools` | `idx_ts_pools_created_at` | `created_at DESC` | B-tree | Sort by newest |
| `thunderswap_trades` | `idx_ts_trades_mint` | `(mint, block_time DESC)` | B-tree | Trade history by token |
| `thunderswap_trades` | `idx_ts_trades_pool` | `(pool_address, block_time DESC)` | B-tree | Trade history by pool |
| `thunderswap_trades` | `idx_ts_trades_trader` | `(trader, block_time DESC)` | B-tree | "My trades" UI |
| `thunderswap_trades` | `idx_ts_trades_block_time` | `block_time DESC` | B-tree | Recent trades feed |
| `thunderswap_trades` | `idx_ts_trades_tx_unique` | `(tx_signature, trade_type)` | UNIQUE | Dedup idempotency |
| `lp_burn_records` | `idx_lp_burn_records_mint` | `mint` | UNIQUE | Lookup by mint |
| `lp_burn_records` | `idx_lp_burn_records_burned_at` | `burned_at DESC` | B-tree | Recent graduations |
| `thunderswap_pool_metrics` | `idx_ts_pool_metrics_tvl` | `tvl_usd DESC` | B-tree | Sort pools by TVL |
| `thunderswap_pool_metrics` | `idx_ts_pool_metrics_volume` | `volume_24h_usd DESC` | B-tree | Sort pools by volume |
| `thunderswap_creator_fee_claims` | `idx_ts_fee_claims_creator` | `(creator, claimed_at DESC)` | B-tree | Creator history UI |
| `thunderswap_creator_fee_claims` | `idx_ts_fee_claims_tx` | `tx_signature` | UNIQUE | Dedup idempotency |
| `mv_thunderswap_top_pools` | `idx_mv_top_pools_mint` | `mint` | UNIQUE | Concurrent refresh |
| `mv_thunderswap_top_pools` | `idx_mv_top_pools_tvl` | `tvl_usd DESC` | B-tree | TVL sort |

---

## 7. Cross-Table Relationships

### Within ThunderSwap

```
thunderswap_pools (1) ──── (1) thunderswap_pool_metrics    [mint → mint]
thunderswap_pools (1) ──── (1) lp_burn_records             [mint → mint, only if graduated]
thunderswap_pools (1) ──── (*) thunderswap_trades          [pool_address → pool_address]
thunderswap_pools (1) ──── (*) thunderswap_liquidity_events [pool_address → pool_address]
thunderswap_pools (1) ──── (*) thunderswap_creator_fee_claims [mint → mint]
```

### With ThunderLaunch (Soft Joins)

```
thunderswap_pools.mint       →  tokens.mint_address        [token name, symbol, image]
thunderswap_pools.creator    →  users.wallet_address        [creator profile]
lp_burn_records.mint         →  pools.mint                  [bonding curve history]
thunderswap_pools.mint       →  pools.mint                  [graduation status sync]
```

All cross-table joins are `LEFT JOIN` — ThunderSwap records may exist before ThunderLaunch records are updated.

### ThunderLaunch `pools` Table Updates at Graduation

When a pool graduates, the ThunderLaunch `pools` table must be updated. The indexer webhook handler updates these columns:

```sql
UPDATE pools
SET
  migration_status          = 'completed',
  external_dex_pool_address = <thunderswap pool_address>,
  graduated_at              = <burn timestamp>
WHERE mint = <token mint>;
```

---

## 8. Migration Files

Each migration is a standalone SQL file applied in order. Place in `sql/migrations/` alongside ThunderLaunch migrations.

### File: `sql/migrations/001_thunderswap_core_tables.sql`

```sql
-- ThunderSwap core tables: pools, trades, lp_burn_records
-- Run once on target Supabase instance

-- thunderswap_pools
[full CREATE TABLE for thunderswap_pools as above]

-- lp_burn_records
[full CREATE TABLE for lp_burn_records as above]

-- thunderswap_pool_metrics
[full CREATE TABLE for thunderswap_pool_metrics as above]

-- thunderswap_liquidity_events
[full CREATE TABLE for thunderswap_liquidity_events as above]

-- thunderswap_creator_fee_claims
[full CREATE TABLE for thunderswap_creator_fee_claims as above]

-- thunderswap_trades (partitioned)
[full CREATE TABLE and first 6 months of partitions as above]

-- Triggers
[create_pool_metrics_on_insert trigger as above]
[update_thunderswap_pools_updated_at trigger as above]

-- RLS policies
[all RLS policies as above]
```

### File: `sql/migrations/002_thunderswap_views.sql`

```sql
-- ThunderSwap views and materialized views
[v_thunderswap_pools_enriched]
[mv_thunderswap_top_pools]
[v_creator_fee_summary]
[refresh_thunderswap_top_pools function]
[create_thunderswap_trade_partition_if_not_exists function]
```

### File: `sql/migrations/003_thunderswap_add_graduation_columns_to_pools.sql`

Adds ThunderSwap graduation tracking columns to ThunderLaunch's existing `pools` table:

```sql
-- Add ThunderSwap graduation columns to ThunderLaunch pools table
ALTER TABLE pools
  ADD COLUMN IF NOT EXISTS external_dex_pool_address TEXT,
  ADD COLUMN IF NOT EXISTS lp_burn_tx_sig            TEXT,
  ADD COLUMN IF NOT EXISTS lp_burned_at              TIMESTAMPTZ;

CREATE INDEX IF NOT EXISTS idx_pools_external_dex_pool
  ON pools(external_dex_pool_address)
  WHERE external_dex_pool_address IS NOT NULL;

COMMENT ON COLUMN pools.external_dex_pool_address IS
  'ThunderSwap AmmPool PDA address for graduated tokens. NULL while on bonding curve.';
COMMENT ON COLUMN pools.lp_burn_tx_sig IS
  'Transaction signature of the LP burn at graduation. Mirrors lp_burn_records.tx_sig.';
COMMENT ON COLUMN pools.lp_burned_at IS
  'UTC timestamp of LP burn. Set by graduation webhook handler.';
```

---

## 9. Common Query Patterns

These are the most frequent queries. Verify each uses an index via `EXPLAIN ANALYZE`.

### Get pool by mint (pool detail page)

```sql
SELECT * FROM v_thunderswap_pools_enriched WHERE mint = $1;
```

### Recent trades for a pool (trade history panel)

```sql
SELECT
  trader, trade_type, sol_amount, token_amount,
  price_usd, price_impact_bps, tx_signature, block_time
FROM thunderswap_trades
WHERE mint = $1
ORDER BY block_time DESC
LIMIT 50;
-- Uses: idx_ts_trades_mint (covers mint + block_time DESC)
```

### Top pools by TVL (home page)

```sql
SELECT * FROM mv_thunderswap_top_pools
ORDER BY tvl_usd DESC
LIMIT 20;
-- Uses: materialized view index
```

### Creator fee summary for a wallet (My Profile → Created tab)

```sql
SELECT * FROM v_creator_fee_summary
WHERE creator = $1
ORDER BY creator_fees_accumulated DESC;
```

### Volume calculation for metrics cron

```sql
SELECT
  mint,
  SUM(sol_amount) FILTER (WHERE block_time >= NOW() - INTERVAL '24 hours') AS volume_24h_sol,
  COUNT(*)        FILTER (WHERE block_time >= NOW() - INTERVAL '24 hours') AS trades_24h,
  COUNT(DISTINCT trader) FILTER (WHERE block_time >= NOW() - INTERVAL '24 hours') AS unique_traders_24h
FROM thunderswap_trades
WHERE mint = $1
  AND block_time >= NOW() - INTERVAL '7 days'   -- narrow partition scan
GROUP BY mint;
-- Uses: idx_ts_trades_mint (mint + block_time DESC)
```

### Check if LP burn record exists (reconciliation cron)

```sql
SELECT tx_sig, amount_burned, burned_at
FROM lp_burn_records
WHERE mint = $1;
-- Uses: idx_lp_burn_records_mint (UNIQUE)
```

### Tokens graduated but not yet synced (reconciliation cron)

```sql
-- Find ThunderLaunch tokens that graduated to ThunderSwap but don't have
-- a lp_burn_records row yet (missed graduation event)
SELECT p.mint, p.token_address, p.graduation_dex
FROM pools p
WHERE p.graduation_dex = 'thunderswap'
  AND p.graduation_status = 'completed'
  AND NOT EXISTS (
    SELECT 1 FROM lp_burn_records lb WHERE lb.mint = p.mint
  )
ORDER BY p.graduated_at ASC
LIMIT 100;
```

### Price change calculation (metrics cron)

```sql
-- Get price 24h ago and now for price_change_24h_pct
WITH price_now AS (
  SELECT price_sol
  FROM thunderswap_trades
  WHERE mint = $1
  ORDER BY block_time DESC
  LIMIT 1
),
price_24h_ago AS (
  SELECT price_sol
  FROM thunderswap_trades
  WHERE mint = $1
    AND block_time <= NOW() - INTERVAL '24 hours'
  ORDER BY block_time DESC
  LIMIT 1
)
SELECT
  n.price_sol AS current_price,
  h.price_sol AS price_24h_ago,
  CASE WHEN h.price_sol > 0
    THEN ((n.price_sol - h.price_sol) / h.price_sol * 100)::NUMERIC(10, 4)
    ELSE 0
  END AS price_change_24h_pct
FROM price_now n, price_24h_ago h;
```
