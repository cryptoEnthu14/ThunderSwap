# ThunderSwap — API Design

**Version**: 1.0
**Status**: Design Phase — ready for implementation
**Base URL**: `https://swap.thunderlaunch.io/api/thunderswap`
**Related**: [THUNDERSWAP_ARCHITECTURE.md](THUNDERSWAP_ARCHITECTURE.md), [THUNDERSWAP_DATABASE.md](THUNDERSWAP_DATABASE.md)

All endpoints are implemented as Next.js App Router API routes under `src/app/api/thunderswap/`. They follow the same conventions as ThunderLaunch's existing API routes.

---

## Table of Contents

1. [Conventions](#1-conventions)
2. [Authentication](#2-authentication)
3. [Rate Limiting](#3-rate-limiting)
4. [Pools](#4-pools)
5. [Trades](#5-trades)
6. [Quote](#6-quote)
7. [Creator Fees](#7-creator-fees)
8. [Liquidity Events](#8-liquidity-events)
9. [Cron Jobs](#9-cron-jobs)
10. [Webhooks](#10-webhooks)
11. [WebSocket / Real-Time](#11-websocket--real-time)
12. [Error Reference](#12-error-reference)

---

## 1. Conventions

### Request Format

- All requests and responses use `Content-Type: application/json`
- GET requests use query parameters; POST requests use JSON body
- All timestamps are ISO 8601 UTC strings (e.g., `"2026-03-27T14:00:00.000Z"`)
- All SOL amounts are in **lamports** (integer), never floating-point SOL
- All token amounts are in **raw token units** (integer), never decimal-adjusted
- Basis points are integers (e.g., `50` = 0.50%)

### Response Envelope

Every response uses this envelope:

```typescript
// Success
{
  "success": true,
  "data": { ... }          // or array
}

// Error
{
  "success": false,
  "error": {
    "code": "POOL_NOT_FOUND",
    "message": "No pool found for this token mint"
  }
}
```

### Pagination

All list endpoints use cursor-based pagination (consistent with ThunderLaunch patterns):

```
GET /pools?limit=20&offset=0
```

Response includes:
```json
{
  "success": true,
  "data": [...],
  "pagination": {
    "total": 1842,
    "limit": 20,
    "offset": 0,
    "has_more": true
  }
}
```

### SOL Price

All USD calculations use `getSolPrice()` from ThunderLaunch's existing `@/lib/services/solPrice` service (multi-source: Birdeye, CoinGecko, Pyth fallback). Never hardcode SOL price.

---

## 2. Authentication

### Public Endpoints (no auth required)

All `GET` endpoints are public. No API key needed for read operations.

### Internal / Cron Endpoints

Cron job endpoints (`/api/cron/*`) and webhook receivers (`/api/webhooks/*`) require:

```
Authorization: Bearer <CRON_SECRET>
```

Where `CRON_SECRET` is the same environment variable used by ThunderLaunch cron jobs. Set in `src/config/env.ts` under `env.cron.secret`.

### Service Role (Indexer)

The ThunderSwap indexer writes to Supabase using `SUPABASE_SERVICE_ROLE_KEY`. This never appears in API responses or client-side code.

---

## 3. Rate Limiting

Applied at the Vercel Edge layer. Same limits as ThunderLaunch API.

| Endpoint Group | Limit | Window |
|---|---|---|
| Public GET endpoints | 300 requests | 1 minute per IP |
| Quote endpoint | 60 requests | 1 minute per IP |
| Cron / webhook endpoints | N/A (server-to-server only) | — |

Rate limit response: HTTP `429` with `Retry-After` header.

---

## 4. Pools

### `GET /api/thunderswap/pools`

List all pools with optional filtering and sorting. Backed by `v_thunderswap_pools_enriched` view.

**Query Parameters**:

| Param | Type | Default | Description |
|---|---|---|---|
| `limit` | integer | `20` | Max results (max 100) |
| `offset` | integer | `0` | Pagination offset |
| `sort` | string | `tvl_usd` | Sort field: `tvl_usd`, `volume_24h_usd`, `price_change_24h_pct`, `created_at` |
| `order` | string | `desc` | `asc` or `desc` |
| `graduated` | boolean | — | Filter to graduated-only (`true`) or public-only (`false`) |
| `creator` | string | — | Filter by creator wallet address |
| `search` | string | — | Search by token name or symbol (case-insensitive, min 2 chars) |

**Response**:

```json
{
  "success": true,
  "data": [
    {
      "mint": "TokenMintAddress...",
      "pool_address": "AmmPoolPDA...",
      "creator": "CreatorWallet...",
      "token_name": "Thunder Dog",
      "token_symbol": "TDOG",
      "token_image": "https://...",
      "graduated_from_plp": true,
      "lp_burned": true,
      "locked": false,
      "sol_reserve": 125000000000,
      "token_reserve": 800000000000,
      "lp_supply": 0,
      "price_sol": "0.000000156250",
      "price_usd": "0.023437",
      "market_cap_usd": "23437.50",
      "tvl_usd": "37500.00",
      "volume_24h_usd": "12800.00",
      "price_change_24h_pct": "14.32",
      "trades_24h": 847,
      "created_at": "2026-03-20T10:00:00.000Z"
    }
  ],
  "pagination": {
    "total": 1842,
    "limit": 20,
    "offset": 0,
    "has_more": true
  }
}
```

---

### `GET /api/thunderswap/pools/[mint]`

Get a single pool by token mint address. Returns full detail including recent metrics.

**Path Parameter**: `mint` — SPL token mint address (base58)

**Response**:

```json
{
  "success": true,
  "data": {
    "mint": "TokenMintAddress...",
    "pool_address": "AmmPoolPDA...",
    "creator": "CreatorWallet...",
    "token_name": "Thunder Dog",
    "token_symbol": "TDOG",
    "token_image": "https://...",
    "token_description": "The fastest dog on Solana",
    "graduated_from_plp": true,
    "lp_burned": true,
    "lp_burn_tx": "BurnTxSignature...",
    "locked": false,
    "sol_reserve": 125000000000,
    "token_reserve": 800000000000,
    "lp_supply": 0,
    "creator_fees_accumulated": 45000000,
    "price_sol": "0.000000156250",
    "price_usd": "0.023437",
    "market_cap_usd": "23437.50",
    "tvl_sol": "250.00",
    "tvl_usd": "37500.00",
    "volume_1h_usd": "1200.00",
    "volume_24h_usd": "12800.00",
    "volume_7d_usd": "84000.00",
    "price_change_1h_pct": "2.14",
    "price_change_24h_pct": "14.32",
    "trades_24h": 847,
    "unique_traders_24h": 312,
    "created_at": "2026-03-20T10:00:00.000Z",
    "metrics_updated_at": "2026-03-27T13:45:00.000Z"
  }
}
```

**Errors**:
- `404` — `POOL_NOT_FOUND` if no pool exists for this mint

---

### `GET /api/thunderswap/pools/search`

Find a pool by base + quote mint pair. Supports both directions (SOL/TOKEN and TOKEN/SOL).

**Query Parameters**:

| Param | Type | Required | Description |
|---|---|---|---|
| `base_mint` | string | Yes | SOL mint (`So11111111111111111111111111111111111111112`) or token mint |
| `quote_mint` | string | Yes | The other side of the pair |

For v1, all pools are SOL/TOKEN pairs. Pass the token mint as either `base_mint` or `quote_mint`.

**Response**: Same shape as `GET /pools/[mint]` with `data` as single object or `null`.

---

## 5. Trades

### `GET /api/thunderswap/trades/[mint]`

Trade history for a pool. Paginated, most recent first.

**Path Parameter**: `mint` — token mint address

**Query Parameters**:

| Param | Type | Default | Description |
|---|---|---|---|
| `limit` | integer | `50` | Max results (max 200) |
| `offset` | integer | `0` | Pagination offset |
| `type` | string | — | Filter: `buy` or `sell` |
| `trader` | string | — | Filter by trader wallet |
| `from` | ISO datetime | — | Start of time range (UTC) |
| `to` | ISO datetime | — | End of time range (UTC) |

**Response**:

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid...",
      "trader": "TraderWallet...",
      "trade_type": "buy",
      "sol_amount": 5000000000,
      "token_amount": 32000000000,
      "price_usd": "0.023437",
      "price_impact_bps": 48,
      "protocol_fee": 7500000,
      "creator_fee": 7500000,
      "tx_signature": "TxSig...",
      "block_time": "2026-03-27T13:58:22.000Z"
    }
  ],
  "pagination": {
    "total": 847,
    "limit": 50,
    "offset": 0,
    "has_more": true
  }
}
```

---

### `GET /api/thunderswap/trades/wallet/[address]`

All trades by a specific wallet across all pools.

**Path Parameter**: `address` — wallet address

**Query Parameters**: `limit`, `offset`, `type`, `from`, `to` (same as above)

**Response**: Same shape as trades, with additional `mint`, `token_symbol` fields per row.

---

## 6. Quote

### `POST /api/thunderswap/quote`

Compute a swap quote. Does not execute anything — read-only calculation against current pool reserves. Returns the same math as the on-chain `swap` instruction.

**Request Body**:

```json
{
  "mint": "TokenMintAddress...",
  "is_buy": true,
  "amount_in": 5000000000,
  "slippage_bps": 100
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `mint` | string | Yes | Token mint address |
| `is_buy` | boolean | Yes | `true` = SOL→token, `false` = token→SOL |
| `amount_in` | integer | Yes | Lamports (buy) or raw token units (sell) |
| `slippage_bps` | integer | No | Default `100` (1%). Max `5000` (50%) |

**Response**:

```json
{
  "success": true,
  "data": {
    "mint": "TokenMintAddress...",
    "is_buy": true,
    "amount_in": 5000000000,
    "amount_out": 32000000000,
    "min_amount_out": 31680000000,
    "price_impact_bps": 48,
    "fees": {
      "lp_fee_lamports": 10000000,
      "protocol_fee_lamports": 7500000,
      "creator_fee_lamports": 7500000,
      "total_fee_lamports": 25000000,
      "total_fee_bps": 50
    },
    "price_before": "0.000000150000",
    "price_after": "0.000000156250",
    "sol_reserve": 125000000000,
    "token_reserve": 800000000000,
    "quoted_at": "2026-03-27T13:58:22.000Z"
  }
}
```

**Implementation**:

```typescript
// src/app/api/thunderswap/quote/route.ts
export async function POST(req: Request) {
  const body = await req.json();
  const { mint, is_buy, amount_in, slippage_bps = 100 } = body;

  // Input validation
  if (!mint || typeof is_buy !== 'boolean' || !amount_in || amount_in <= 0) {
    return Response.json({ success: false, error: { code: 'INVALID_PARAMS', message: 'Missing required fields' } }, { status: 400 });
  }
  if (slippage_bps < 0 || slippage_bps > 5000) {
    return Response.json({ success: false, error: { code: 'INVALID_SLIPPAGE', message: 'Slippage must be between 0 and 5000 bps' } }, { status: 400 });
  }

  // Fetch pool from DB (cached 30s in Redis/edge cache)
  const pool = await getPool(mint);
  if (!pool) {
    return Response.json({ success: false, error: { code: 'POOL_NOT_FOUND', message: 'No pool for this mint' } }, { status: 404 });
  }
  if (pool.locked) {
    return Response.json({ success: false, error: { code: 'POOL_LOCKED', message: 'Pool is currently paused' } }, { status: 503 });
  }

  // Replicate on-chain math (must be identical to src/math.rs)
  const fees = computeFees(amount_in, { lp: 20, protocol: 15, creator: 15 });
  const amount_out = is_buy
    ? quoteBuy(pool.sol_reserve, pool.token_reserve, fees.net_amount)
    : quoteSell(pool.sol_reserve, pool.token_reserve, amount_in) - fees.protocol_fee - fees.creator_fee;

  const min_amount_out = Math.floor(amount_out * (10_000 - slippage_bps) / 10_000);
  const price_impact_bps = computePriceImpact(pool, is_buy, amount_in);

  return Response.json({ success: true, data: { ... } });
}
```

**Errors**:
- `400` — `INVALID_PARAMS`, `INVALID_SLIPPAGE`
- `404` — `POOL_NOT_FOUND`
- `503` — `POOL_LOCKED`

> **Critical**: The TypeScript quote math must be exactly identical to the Rust `math.rs` implementation (same integer arithmetic, same truncation). Any divergence causes incorrect `min_amount_out` values on-chain. Maintain a shared test suite that runs both implementations against the same inputs.

---

## 7. Creator Fees

### `GET /api/thunderswap/creator-fees/[mint]`

Get claimable creator fees for a specific pool. Used by both ThunderLaunch My Profile tab and ThunderSwap creator UI.

**Path Parameter**: `mint` — token mint address

**Response**:

```json
{
  "success": true,
  "data": {
    "mint": "TokenMintAddress...",
    "pool_address": "AmmPoolPDA...",
    "creator": "CreatorWallet...",
    "token_name": "Thunder Dog",
    "token_symbol": "TDOG",
    "creator_fees_accumulated": 45000000,
    "total_claimed": 30000000,
    "estimated_claimable": 15000000,
    "estimated_claimable_usd": "2.25",
    "last_claimed_at": "2026-03-25T10:00:00.000Z",
    "claim_history_count": 3
  }
}
```

> **Note**: `estimated_claimable` is computed from DB records. The authoritative on-chain balance is in the `creator_fee_vault` PDA. Always show "estimated" in the UI and verify on-chain before signing the claim transaction.

---

### `GET /api/thunderswap/creator-fees/wallet/[address]`

All pools where this wallet is the creator, with claimable fee amounts. Drives the ThunderSwap creator dashboard.

**Path Parameter**: `address` — creator wallet address

**Query Parameters**: `limit` (default 20, max 100), `offset`

**Response**:

```json
{
  "success": true,
  "data": [
    {
      "mint": "TokenMintAddress...",
      "pool_address": "AmmPoolPDA...",
      "token_name": "Thunder Dog",
      "token_symbol": "TDOG",
      "token_image": "https://...",
      "graduated_from_plp": true,
      "creator_fees_accumulated": 45000000,
      "total_claimed": 30000000,
      "estimated_claimable": 15000000,
      "estimated_claimable_usd": "2.25",
      "last_claimed_at": "2026-03-25T10:00:00.000Z"
    }
  ],
  "summary": {
    "total_pools": 4,
    "total_accumulated": 180000000,
    "total_claimed": 120000000,
    "total_claimable": 60000000,
    "total_claimable_usd": "9.00"
  },
  "pagination": {
    "total": 4,
    "limit": 20,
    "offset": 0,
    "has_more": false
  }
}
```

---

### `GET /api/thunderswap/creator-fees/claims/[mint]`

Claim history for a specific pool.

**Query Parameters**: `limit` (default 20), `offset`

**Response**:

```json
{
  "success": true,
  "data": [
    {
      "amount_claimed": 15000000,
      "amount_claimed_usd": "2.25",
      "tx_signature": "ClaimTxSig...",
      "claimed_at": "2026-03-25T10:00:00.000Z"
    }
  ],
  "pagination": { "total": 3, "limit": 20, "offset": 0, "has_more": false }
}
```

---

## 8. Liquidity Events

### `GET /api/thunderswap/liquidity/[mint]`

Add/remove liquidity history for a public pool. Returns empty for graduated pools (no liquidity events possible).

**Query Parameters**: `limit` (default 50), `offset`, `type` (`add` | `remove`), `provider`

**Response**:

```json
{
  "success": true,
  "data": [
    {
      "provider": "ProviderWallet...",
      "event_type": "add",
      "sol_amount": 10000000000,
      "token_amount": 64000000000,
      "lp_amount": 25298221,
      "tx_signature": "TxSig...",
      "block_time": "2026-03-27T12:00:00.000Z"
    }
  ],
  "pagination": { "total": 12, "limit": 50, "offset": 0, "has_more": false }
}
```

---

## 9. Cron Jobs

All cron endpoints are in `src/app/api/cron/` and require `Authorization: Bearer <CRON_SECRET>`.

### `GET /api/cron/thunderswap-indexer`

Runs the ThunderSwap indexer: fetches new transactions from Solana, parses Anchor events, writes to DB.

**Schedule**: `*/2 * * * *` (every 2 minutes)

**Response**:

```json
{
  "success": true,
  "data": {
    "signatures_processed": 47,
    "events_parsed": 12,
    "pools_created": 1,
    "swaps_indexed": 10,
    "graduations_detected": 1,
    "errors": []
  }
}
```

---

### `GET /api/cron/thunderswap-metrics`

Recalculates price, TVL, and volume metrics for all active pools. Refreshes `mv_thunderswap_top_pools`.

**Schedule**: `*/15 * * * *` (every 15 minutes)

**Response**:

```json
{
  "success": true,
  "data": {
    "pools_updated": 1842,
    "sol_price_usd": "150.23",
    "mv_refreshed": true,
    "duration_ms": 4200
  }
}
```

---

### `GET /api/cron/thunderswap-reconciliation`

Finds ThunderLaunch tokens that graduated to ThunderSwap but have no `lp_burn_records` row — missed graduation events. Re-triggers sync.

**Schedule**: `*/15 * * * *` (every 15 minutes)

**Response**:

```json
{
  "success": true,
  "data": {
    "tokens_checked": 3,
    "tokens_synced": 1,
    "tokens_still_missing": 0,
    "alerts_created": 0
  }
}
```

---

### `GET /api/cron/thunderswap-partition-maintenance`

Pre-creates next month's `thunderswap_trades` partition before it is needed. Prevents insert failures at month boundary.

**Schedule**: `0 0 20 * *` (20th of each month at midnight UTC)

**Response**:

```json
{
  "success": true,
  "data": {
    "partition_created": "thunderswap_trades_2026_05",
    "already_existed": false
  }
}
```

---

## 10. Webhooks

### `POST /api/webhooks/thunderswap-graduation`

Receives graduation event from ThunderSwap indexer. Updates ThunderLaunch `pools` table to mark migration complete.

**Authentication**: `Authorization: Bearer <CRON_SECRET>`

**Request Body**:

```json
{
  "mint": "TokenMintAddress...",
  "pool_address": "AmmPoolPDA...",
  "lp_mint": "LpMintAddress...",
  "lp_amount_burned": 316227766,
  "lp_burn_tx": "BurnTxSignature...",
  "burn_slot": 312547890,
  "timestamp": "2026-03-27T14:00:00.000Z"
}
```

**Actions**:
1. Validates `mint` exists in ThunderLaunch `pools` table
2. Updates `pools` row: `migration_status = 'completed'`, `external_dex_pool_address`, `lp_burn_tx_sig`, `lp_burned_at`
3. Resolves any open `migration_alerts` for this mint
4. Returns `200` immediately (idempotent — safe to receive multiple times)

**Response**:

```json
{
  "success": true,
  "data": {
    "mint": "TokenMintAddress...",
    "action": "graduation_confirmed",
    "migration_status": "completed"
  }
}
```

**Errors**:
- `401` — missing or invalid auth
- `400` — missing required fields
- `404` — mint not found in ThunderLaunch pools table (logs warning, does not error — mint may be a public ThunderSwap pool)

> **Idempotency**: If called multiple times for the same mint, subsequent calls detect `migration_status = 'completed'` already set and return `200` with `action: "already_confirmed"`. Never errors on duplicate.

---

## 11. WebSocket / Real-Time

ThunderSwap uses Supabase Realtime (same as ThunderLaunch) for live updates. No custom WebSocket server needed.

### Pool Metrics (Live Price)

Subscribe to `thunderswap_pool_metrics` UPDATE events for a specific mint:

```typescript
// Frontend: src/hooks/useThunderSwapPool.ts
const channel = supabase
  .channel(`ts-pool-${mint}`)
  .on(
    'postgres_changes',
    {
      event: 'UPDATE',
      schema: 'public',
      table: 'thunderswap_pool_metrics',
      filter: `mint=eq.${mint}`,
    },
    (payload) => {
      setPoolMetrics(payload.new);
    }
  )
  .subscribe();
```

### Live Trade Feed

Subscribe to `thunderswap_trades` INSERT events for a specific pool:

```typescript
const channel = supabase
  .channel(`ts-trades-${mint}`)
  .on(
    'postgres_changes',
    {
      event: 'INSERT',
      schema: 'public',
      table: 'thunderswap_trades',
      filter: `mint=eq.${mint}`,
    },
    (payload) => {
      setTrades((prev) => [payload.new, ...prev].slice(0, 50));
    }
  )
  .subscribe();
```

### Cleanup

Always unsubscribe on component unmount:

```typescript
useEffect(() => {
  // setup channel
  return () => { supabase.removeChannel(channel); };
}, [mint]);
```

---

## 12. Error Reference

### HTTP Status Codes

| Code | Meaning | When Used |
|---|---|---|
| `200` | OK | Successful GET or POST |
| `400` | Bad Request | Missing/invalid parameters |
| `401` | Unauthorized | Missing or invalid auth token (cron/webhook endpoints) |
| `404` | Not Found | Pool, trade history, or resource not found |
| `429` | Too Many Requests | Rate limit exceeded |
| `500` | Internal Server Error | Unexpected server error (always logged) |
| `503` | Service Unavailable | Pool locked, program paused |

### Error Codes

| Code | HTTP | Description |
|---|---|---|
| `INVALID_PARAMS` | 400 | Missing or malformed required parameters |
| `INVALID_SLIPPAGE` | 400 | `slippage_bps` outside 0–5000 range |
| `INVALID_MINT` | 400 | Mint address is not valid base58 |
| `POOL_NOT_FOUND` | 404 | No pool exists for the given mint |
| `TRADE_HISTORY_EMPTY` | 404 | No trades found for given filters |
| `POOL_LOCKED` | 503 | Pool is paused by admin |
| `PROGRAM_PAUSED` | 503 | Global ThunderSwap program is paused |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests from this IP |
| `UNAUTHORIZED` | 401 | Invalid or missing `Authorization` header |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

### Error Response Shape

```json
{
  "success": false,
  "error": {
    "code": "POOL_NOT_FOUND",
    "message": "No pool found for mint: TokenMint...",
    "details": {}
  }
}
```

---

## Appendix: Route File Structure

```
src/app/api/thunderswap/
├── pools/
│   ├── route.ts                         GET /pools
│   ├── search/
│   │   └── route.ts                     GET /pools/search
│   └── [mint]/
│       └── route.ts                     GET /pools/[mint]
├── trades/
│   ├── [mint]/
│   │   └── route.ts                     GET /trades/[mint]
│   └── wallet/
│       └── [address]/
│           └── route.ts                 GET /trades/wallet/[address]
├── quote/
│   └── route.ts                         POST /quote
├── creator-fees/
│   ├── [mint]/
│   │   └── route.ts                     GET /creator-fees/[mint]
│   ├── wallet/
│   │   └── [address]/
│   │       └── route.ts                 GET /creator-fees/wallet/[address]
│   └── claims/
│       └── [mint]/
│           └── route.ts                 GET /creator-fees/claims/[mint]
└── liquidity/
    └── [mint]/
        └── route.ts                     GET /liquidity/[mint]

src/app/api/cron/
├── thunderswap-indexer/
│   └── route.ts
├── thunderswap-metrics/
│   └── route.ts
├── thunderswap-reconciliation/
│   └── route.ts
└── thunderswap-partition-maintenance/
    └── route.ts

src/app/api/webhooks/
└── thunderswap-graduation/
    └── route.ts
```
