# ThunderSwap — Product Planning Document

**Status**: Requirements confirmed, pre-development
**Last Updated**: 2026-03-27
**Companion Docs**: `THUNDERSWAP_ARCHITECTURE.md` (pending), `THUNDERSWAP_DATABASE.md` (pending), `THUNDERSWAP_API.md` (pending)

---

## 1. What is ThunderSwap?

ThunderSwap is ThunderLaunch's native decentralized exchange (DEX). It serves two roles:

1. **Graduation destination** — when a token completes its bonding curve on ThunderLaunch, liquidity permanently migrates to ThunderSwap. All future trading for that token happens on ThunderSwap.
2. **General-purpose DEX** — any project or user can create a liquidity pool on ThunderSwap (permissionless), not just ThunderLaunch-graduated tokens.

ThunderSwap is a standalone product with its own smart contracts, indexer, and UI — but deeply integrated with ThunderLaunch at the graduation layer.

---

## 2. User Experience

### 2.1 Trading Graduated Tokens (ThunderLaunch Users)

- After a token graduates, ThunderLaunch's existing **Trade Panel UI routes transactions to ThunderSwap on-chain**
- Users never leave ThunderLaunch to trade a graduated token — experience is identical to pre-graduation
- Graduated tokens are also tradeable directly via the **ThunderSwap API** (accessible to third-party integrators, bots, and aggregators like Jupiter)

### 2.2 Creator Rewards

- Token creators accumulate **15 bps (0.15%)** on every swap of their graduated token
- Creators can **view and claim rewards** from two places:
  - **ThunderLaunch**: My Profile → Created tab (same place they manage their token)
  - **ThunderSwap**: Dedicated creator rewards UI on ThunderSwap's own platform
- Both UIs show real-time pending rewards and allow on-demand claiming

---

## 3. Confirmed Architecture Decisions

### 3.1 Pool Creation Access

| Access Type | Decision |
|---|---|
| `create_pool` instruction | **Open / permissionless** — anyone with minimum liquidity can call it |
| Graduation CPI path | **Exclusive to ThunderLaunch PLP program** — only the graduation authority PDA can trigger pool creation during graduation |
| ThunderLaunch-originated pool fee discount | **None** — same fees for all pools regardless of origin |

Rationale: Making `create_pool` permissionless makes ThunderSwap a general-purpose DEX with broader liquidity and volume. The graduation exclusivity is enforced by design (only the PLP graduation authority PDA signs the CPI call), not by restricting the instruction.

### 3.2 LP Token Handling

- LP tokens are **burned at graduation** — liquidity is permanently locked
- Before burning, the following are recorded in the `pools` table:

| Column | Type | Description |
|---|---|---|
| `lp_token_mint` | TEXT | LP token mint address |
| `lp_tokens_burned` | BIGINT | Total LP tokens burned |
| `lp_burn_tx_signature` | TEXT | On-chain burn transaction signature |
| `lp_burned_at` | TIMESTAMPTZ | Timestamp of LP burn |

### 3.3 Admin & Governance

ThunderSwap `GlobalConfig` (fees, emergency pause, upgrade authority) is controlled by a **Squads 3-of-5 multisig**.

Squads is Solana's leading multisig protocol — requires M-of-N wallet signatures to execute privileged instructions. Used by Jito, Jupiter, and most major Solana protocols.

| Signer | Holder |
|---|---|
| Key 1 | CEO / Founder |
| Key 2 | CTO / Lead Dev |
| Key 3 | COO / Second Founder |
| Key 4 | Cold hardware wallet (offline, stored in safe) |
| Key 5 | Legal entity wallet (company-controlled) |

3-of-5 means 2 key losses/compromises before loss of control. Allows operations when one signer is unavailable.

Governed operations: fee parameter changes, emergency pool pause, program upgrade authority, GlobalConfig updates.

### 3.4 Graduation Event Discovery (ThunderSwap Indexer — Hybrid)

ThunderSwap's indexer uses three redundant paths to discover graduation events:

| Path | Mechanism | Latency |
|---|---|---|
| **Primary** | ThunderLaunch backend sends webhook to ThunderSwap indexer | < 1 second |
| **Fallback** | ThunderSwap on-chain log listener watches for `GraduationCompleted` events in PLP program logs | Seconds |
| **Reconciliation** | Cron job every 15 minutes — scans DB for tokens with `graduation_status='completed'` but no ThunderSwap pool record | Minutes |

All three paths are idempotent. Processing the same graduation event multiple times is safe.

### 3.5 Minimum Liquidity Requirements

| Pool Type | Minimum | Rationale |
|---|---|---|
| Public pool creation | **5 SOL per side** | Prevents dust/spam pools; low enough for real projects |
| Graduation-originated pool | **None** | Bonding curve completion guarantees sufficient liquidity |

---

## 4. Fee Structure

### 4.1 ThunderLaunch Bonding Curve Fees (Unchanged)

These are existing ThunderLaunch fees — no changes for ThunderSwap.

| Fee | Amount | Collected By |
|---|---|---|
| Buy/sell on bonding curve | 1% (100 bps) | ThunderLaunch protocol |
| Graduation flat fee | 1.5 SOL | ThunderLaunch protocol |

No creator bonus at graduation — creators earn ongoing swap fees instead (see below).

### 4.2 ThunderSwap Post-Graduation Swap Fees

Charged on **every swap** — not one-time fees.

| Component | Basis Points | Percentage | Recipient |
|---|---|---|---|
| LP fee | 20 bps | 0.20% | Liquidity providers (stays in pool) |
| Protocol fee | 15 bps | 0.15% | ThunderSwap treasury |
| Creator fee | 15 bps | 0.15% | Token creator wallet |
| **Total** | **50 bps** | **0.50%** | — |

**Fee design decisions:**
- **Flat fees for v1** — no dynamic tiers (simplicity; dynamic market-cap-based tiers can be added in v2)
- **Same fees for all pools** — no discount for ThunderLaunch-originated pools
- **Compared to PumpSwap (0.30% total)**: 0.50% is competitive while giving creators 3× more per swap than pump.fun's 5 bps
- Creator fee goes to the wallet that created the token on ThunderLaunch

---

## 5. Multi-Chain Rollout

| Phase | Chain | Target Timeline | Notes |
|---|---|---|---|
| v1 | **Solana mainnet** | Initial launch | Anchor / native Solana program |
| v2 | **Base (L2)** | 6 months after Solana v1 | EVM port in Solidity; lower gas makes bonding curve micro-trades viable |
| v3 | **Ethereum mainnet** | 10 months after Solana v1 | After Base proves stability; requires external audit |

Base is prioritized over Ethereum mainnet: gas fees on Ethereum mainnet make bonding curve trading economically unviable below ~$10 per trade. Base has $2B+ TVL, growing memecoin culture, and ~100× cheaper gas.

EVM deployment timeline includes: Solidity rewrite (2 months) → internal testing (2 months) → external audit (2 months) → bug bounty period → mainnet.

---

## 6. Competitive Positioning

| Feature | ThunderSwap | PumpSwap | Raydium |
|---|---|---|---|
| Total swap fee | 0.50% | 0.30% | 0.25–0.30% |
| Creator fee (post-grad) | 0.15% (flat) | 0.05% (or dynamic up to 0.95%) | None |
| LP fee | 0.20% | 0.20% | 0.20–0.25% |
| Permissionless pools | Yes | No (graduated tokens only) | Yes |
| Native ThunderLaunch integration | Yes (seamless) | Yes (pump.fun only) | No |
| Graduated token liquidity lock | Yes (LP burned) | Yes (LP burned) | No |

ThunderSwap's differentiation: seamless UX for ThunderLaunch users + meaningful ongoing creator income.

---

## 7. Open Items Before Development Starts

The following require resolution before or during architecture/design phase:

| Item | Status | Notes |
|---|---|---|
| AMM model selection | **Open** | Constant product (x·y=k like Uniswap V2) vs concentrated liquidity (like Uniswap V3) — to be decided in Architecture doc |
| ThunderSwap program ID | **Open** | Determined at deployment |
| Creator fee claim mechanism | **Open** | Push (automatic transfer per swap) vs pull (creator claims accumulated balance) — to be decided |
| ThunderSwap UI hosting | **Open** | Subdomain of thunderlaunch.io or separate domain? |
| Token list / discovery | **Open** | How does ThunderSwap's UI surface tokens — all pools or curated? |
| Price oracle | **Open** | TWAP from ThunderSwap pools, or integrate Pyth/Switchboard? |
| v2 dynamic fee tiers | **Deferred** | Design in v2 planning phase |

---

## 8. Documents to Create Before Development

The following documents should be created and reviewed before writing any code:

| Document | Purpose | Priority |
|---|---|---|
| `THUNDERSWAP_ARCHITECTURE.md` | System design, component diagram, data flow, tech stack, scalability approach | **P0 — required first** |
| `THUNDERSWAP_SMART_CONTRACT.md` | Anchor program structure, account layouts, all instructions, CPI flows, security invariants | **P0 — required before writing Rust** |
| `THUNDERSWAP_DATABASE.md` | ThunderSwap-specific tables, indexes, RLS policies, schema for pools/swaps/creator fees | **P0 — required before backend** |
| `THUNDERSWAP_API.md` | REST endpoints, WebSocket feeds, authentication, rate limiting | **P1 — required before frontend** |
| `THUNDERSWAP_SECURITY_CHECKLIST.md` | Known DEX attack vectors (sandwich, flash loan, price manipulation), invariants to enforce, audit prep | **P1 — required before audit** |

### Recommended creation order:
1. **Architecture** — establishes the system shape everything else follows
2. **Smart Contract Design** — most critical; errors here are the hardest and most expensive to fix
3. **Database Schema** — follows from architecture
4. **API Design** — follows from smart contract + database
5. **Security Checklist** — done alongside smart contract design, before audit

---

## 9. Development Phases

### Phase 1 — Smart Contract (Solana)
- Design and implement ThunderSwap Anchor program
- AMM pool creation, swap, LP token mechanics
- Fee distribution (LP, protocol, creator)
- Graduation CPI integration with PLP program
- Devnet deployment and testing

### Phase 2 — Indexer & Backend
- ThunderSwap indexer (graduation event discovery — hybrid approach)
- Creator fee accumulation and claim tracking
- Pool and swap event recording
- API endpoints

### Phase 3 — Frontend Integration
- ThunderLaunch Trade Panel routing to ThunderSwap
- ThunderSwap standalone UI (swap, pools, creator rewards)
- Creator rewards claim UI on ThunderLaunch (My Profile → Created tab)

### Phase 4 — Audit & Mainnet
- External smart contract audit
- Bug bounty program
- Staged mainnet deployment

### Phase 5 — EVM (Base)
- Solidity port of bonding curve + AMM
- Base testnet/mainnet deployment
- Cross-chain creator reward tracking

---

## 10. References

- `docs/THUNDERSWAP_GRADUATION_IMPLEMENTATION.md` — original graduation integration checklist (superseded by this document)
- `src/config/graduation.ts` — graduation configuration (defaults to 'thunderswap')
- `src/lib/graduation/graduateToken.ts` — graduation flow (ThunderSwap pool creation to be added here)
- `src/lib/solana/plp.ts` — PLP integration (already has `encodeGraduationDex` with thunderswap case)
- `programs/plp/src/lib.rs` — Smart contract graduation logic
