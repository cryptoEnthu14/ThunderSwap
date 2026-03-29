# ThunderSwap — Security Checklist

**Version**: 1.0
**Status**: Design Phase — must be reviewed before audit engagement
**Related**: [THUNDERSWAP_SMART_CONTRACT.md](THUNDERSWAP_SMART_CONTRACT.md), [THUNDERSWAP_ARCHITECTURE.md](THUNDERSWAP_ARCHITECTURE.md)

This document covers known DEX attack vectors, on-chain security invariants, off-chain security considerations, and the pre-audit preparation checklist. Review this document before writing any Rust code and again before engaging an external auditor.

---

## Table of Contents

1. [On-Chain Attack Vectors](#1-on-chain-attack-vectors)
2. [Program Security Invariants](#2-program-security-invariants)
3. [Anchor-Specific Security](#3-anchor-specific-security)
4. [Off-Chain Security](#4-off-chain-security)
5. [API Security](#5-api-security)
6. [Key Management](#6-key-management)
7. [Pre-Audit Checklist](#7-pre-audit-checklist)
8. [Audit Scope & Engagement Guide](#8-audit-scope--engagement-guide)
9. [Incident Response](#9-incident-response)
10. [Ongoing Security Practices](#10-ongoing-security-practices)

---

## 1. On-Chain Attack Vectors

### 1.1 Sandwich Attacks

**What it is**: A bot observes a pending swap in the mempool, front-runs it with a buy (pushing price up), lets the victim's swap execute at a worse price, then back-runs with a sell.

**Solana context**: Solana does not have a traditional mempool — transactions are processed in order within a slot. Sandwich attacks are harder on Solana than on Ethereum, but are possible via:
- Jito bundles (MEV bundles that guarantee ordering)
- Validators who accept bribes for transaction ordering

**ThunderSwap mitigations**:
- `min_amount_out` parameter enforced on-chain. If price moves more than the user's slippage tolerance, the transaction fails. This is the primary defense.
- Frontend defaults: 0.5% for small trades, 1.0% general. Warn users with > 5% price impact.
- Display price impact prominently in the swap UI before signing.

**Implementation check**: Verify `require!(amount_out >= min_amount_out, ThunderSwapError::SlippageExceeded)` is checked AFTER all state mutations. An attacker cannot bypass it by manipulating the pool mid-instruction (Solana's single-threaded execution prevents this).

**Test**: Write a test that simulates a sandwich by calling `swap` twice in the same slot and verify the second swap's `min_amount_out` check catches the price change.

---

### 1.2 Price Oracle Manipulation (TWAP)

**What it is**: An attacker moves the pool price dramatically in a single slot to manipulate the TWAP reading, causing downstream protocols using the oracle to misprice assets.

**ThunderSwap context**: The TWAP is slot-weighted. A single-slot price spike has minimal effect on the cumulative average because:
- The attacker can only influence one slot worth of weight
- Moving the price requires real capital (the AMM enforces x*y=k)
- Reverting the price (to avoid their own loss) brings the TWAP back

**Mitigations**:
- TWAP is updated BEFORE reserves change in `swap`. This means the TWAP reflects the price at the START of each slot, not the end, reducing the attacker's influence.
- For v1, ThunderSwap's TWAP is informational only (displayed in the UI). No other protocol depends on it for liquidations or collateral pricing in v1.
- Document clearly: ThunderSwap TWAP is NOT suitable for use as a liquidation oracle without a minimum observation window of 30+ minutes and minimum liquidity requirements.

**Implementation check**: TWAP update must be the FIRST operation in `swap`, before any reserve changes. See `THUNDERSWAP_SMART_CONTRACT.md` Section 4.6.

---

### 1.3 Flash Loan Attacks

**What it is**: An attacker borrows a large amount, manipulates a pool price within one transaction, exploits a dependent protocol at the manipulated price, then repays the loan — all within one atomic transaction.

**Solana context**: Solana does not natively support flash loans (no atomic multi-instruction loans the way Aave works on Ethereum). However, protocols like Solend and Marginfi support flash loans via CPI.

**ThunderSwap mitigations**:
- ThunderSwap v1 has no liquidation mechanism or under-collateralized lending — there is nothing to exploit via flash loan price manipulation against ThunderSwap itself.
- The AMM enforces real reserve ratios — a flash loan cannot bypass x*y=k.
- Third-party protocols using ThunderSwap as a price oracle should implement their own TWAP windows and minimum liquidity checks.

**Implementation check**: Verify no instruction allows the pool to end in a state where `sol_reserve * token_reserve < initial_k` (k should be non-decreasing except during `remove_liquidity`).

---

### 1.4 LP Token Inflation Attack

**What it is**: On initial pool creation with a very small amount, an attacker mints 1 LP token, then donates tokens directly to the vault (bypassing the AMM), inflating the value of their 1 LP token and extracting value from subsequent LPs.

**Mitigation**: `MINIMUM_LIQUIDITY = 1_000` LP tokens are permanently burned at pool creation. This means:
- The attacker cannot be the sole LP holder of 1 token
- The minimum burn makes inflation attacks uneconomical (attacker would need to donate enormous amounts to move the 1000-token locked floor)

**Implementation check**:
```rust
require!(lp_raw > MINIMUM_LIQUIDITY, ThunderSwapError::InsufficientLiquidityMinted);
let lp_to_creator = lp_raw - MINIMUM_LIQUIDITY;
// MINIMUM_LIQUIDITY tokens are burned to zero address
```

**Test**: Attempt to create a pool with `sol_amount = 1` and `token_amount = 1`. Verify it fails with `InsufficientLiquidityMinted`. Attempt with `sol_amount = MINIMUM_LIQUIDITY^2 - 1` (produces `sqrt < MINIMUM_LIQUIDITY`). Verify failure.

---

### 1.5 Rounding / Precision Exploits

**What it is**: Integer division truncation creates rounding errors that, when accumulated over thousands of trades, leak value out of the pool or can be exploited for profit.

**ThunderSwap approach**:
- All division truncates toward zero (Rust default for integer division). This always favors the pool (less goes out, more stays in).
- k never decreases after a swap (the truncated `new_token_reserve` is always rounded down, leaving more tokens in the pool than the pure formula would suggest).
- Fee calculation: fees are floor-divided, meaning users pay slightly more than the exact fee in their favor. Protocol and creator receive slightly less. This is acceptable and prevents LP drain.

**Implementation check**:
```rust
// After every swap, verify:
let k_after = (pool.sol_reserve as u128) * (pool.token_reserve as u128);
let k_before = /* stored before swap */;
assert!(k_after >= k_before);  // k must not decrease
```

**Test**: Run 10,000 swaps of alternating buy/sell with random amounts. Verify `k` is monotonically non-decreasing throughout.

---

### 1.6 Unauthorized Account Substitution

**What it is**: An attacker passes a malicious account in place of an expected PDA (e.g., substitutes a fake `creator_fee_vault` to steal fees, or passes a fake `global_config` with different fee rates).

**Mitigation**: All PDAs are verified via Anchor's constraint system:
- `seeds = [...]` — Anchor derives the expected address and checks it matches
- `has_one = authority` — verifies a stored pubkey matches the signer
- `constraint = x.key() == y` — explicit equality checks for non-PDA accounts

**Critical constraints to verify in code review**:

| Account | Constraint Required |
|---|---|
| `global_config` | `seeds = [b"global-config"], bump` |
| `amm_pool` | `seeds = [b"amm-pool", token_mint], bump` |
| `sol_vault` | `seeds = [b"sol-vault", token_mint], bump` |
| `creator_fee_vault` | `seeds = [b"creator-fee-vault", token_mint], bump` |
| `protocol_fee_vault` | `constraint = key == global_config.protocol_fee_vault` |
| `graduation_authority` | `constraint = key == global_config.graduation_authority` |
| `creator` (claim) | `constraint = key == amm_pool.creator` |
| `token_vault_authority` | `seeds = [b"token-vault-authority", token_mint], bump` |
| `lp_mint_authority` | `seeds = [b"lp-mint-authority", token_mint], bump` |

**Test**: For each instruction, try passing an account with incorrect seeds. Verify Anchor rejects it with a constraint violation (not a panic or undefined behavior).

---

### 1.7 CPI Reentrancy

**What it is**: A malicious program called via CPI modifies shared state mid-execution to cause unexpected behavior.

**Solana context**: Solana's runtime prevents CPI loops (a program cannot CPI back into itself). Reentrancy of the traditional kind is not possible on Solana.

**Residual risk**: A malicious token program could behave unexpectedly during `token::transfer` CPI. ThunderSwap mitigates this by:
- Only interacting with the standard SPL Token program (not Token-2022 extensions in v1)
- Using Anchor's `token::transfer`, `token::mint_to`, `token::burn` which wrap the canonical SPL program
- Not calling any unknown/unverified programs

**Implementation check**: Verify `token_program` is constrained to `Token` (not `UncheckedAccount`) in all instruction contexts:
```rust
pub token_program: Program<'info, Token>,
```

---

### 1.8 Graduation Authority Spoofing

**What it is**: An attacker tries to call `create_pool_from_graduation` while pretending to be the graduation authority, bypassing the 5 SOL minimum to create a dust pool or perform an unexpected action.

**Mitigation**: `create_pool_from_graduation` has a hard constraint:
```rust
constraint = graduation_authority.key() == global_config.graduation_authority
    @ ThunderSwapError::UnauthorizedGraduationCaller
```

The `global_config` is itself a PDA. An attacker cannot substitute a fake `GlobalConfig` because the seeds `[b"global-config"]` deterministically produce the program's config address.

**Test**: Call `create_pool_from_graduation` with a random signer. Verify it fails with `UnauthorizedGraduationCaller`. Call it with the `authority` (Squads multisig) instead of `graduation_authority`. Verify it fails.

---

### 1.9 LP Burn Bypass

**What it is**: An attacker calls `add_liquidity` or `remove_liquidity` on a graduated pool to add/remove liquidity that should be permanently locked.

**Mitigation**: Both instructions have:
```rust
require!(!amm_pool.lp_burned, ThunderSwapError::PoolLiquidityLocked);
```

Additionally, the LP mint authority is revoked in `create_pool_from_graduation` — even if the lock check was bypassed, no new LP tokens could be minted.

**Test**: After calling `create_pool_from_graduation`, attempt `add_liquidity`. Verify `PoolLiquidityLocked`. Attempt to mint LP tokens directly via SPL token program. Verify it fails because mint authority is `None`.

---

### 1.10 Integer Overflow

**What it is**: Large reserve values multiplied together overflow the integer type, wrapping to incorrect values.

**ThunderSwap approach**:
- k = `sol_reserve * token_reserve` is computed as `u128` (max ~3.4 × 10^38)
- Maximum pool size before u128 overflow: `sol_reserve = token_reserve = sqrt(u128::MAX) ≈ 1.84 × 10^19`
- In lamports, `1.84 × 10^19 lamports = 18.4 billion SOL` — far beyond any realistic pool
- All arithmetic uses `checked_*` with explicit error returns — no panics, no wrapping

**Implementation check**: Every multiplication and addition in `math.rs` uses `.checked_mul().ok_or(ThunderSwapError::Overflow)?`. Run `cargo clippy` with `clippy::integer_arithmetic` lint enabled.

---

## 2. Program Security Invariants

These invariants must hold after every instruction executes. Verify in tests and in the audit.

### Invariant 1: k Non-Decreasing

```
After swap or add_liquidity:
  sol_reserve_after * token_reserve_after >= sol_reserve_before * token_reserve_before
```

Exception: `remove_liquidity` decreases k proportionally to LP burned.

### Invariant 2: sol_vault Balance

```
sol_vault.lamports() == amm_pool.sol_reserve + rent_exempt_minimum
```

This holds because protocol and creator fees are extracted directly to their vaults in the same instruction — they never sit in `sol_vault`. Verify after every instruction that modifies reserves.

### Invariant 3: LP Supply Matches SPL Mint

```
amm_pool.lp_supply == lp_mint.supply (from SPL token program)
```

For graduated pools: `amm_pool.lp_supply == 0` AND `lp_mint.supply == 0` AND `lp_mint.mint_authority == None`.

### Invariant 4: Graduated Pool Immutability

```
If amm_pool.lp_burned == true:
  - amm_pool.lp_supply == 0
  - lp_mint.mint_authority == None (revoked)
  - LpBurnRecord exists at PDA [b"lp-burn-record", amm_pool]
  - add_liquidity always fails
  - remove_liquidity always fails
```

### Invariant 5: No Dust Swaps

```
Every successful swap produces amount_out > 0
```

Enforced by `require!(amount_out > 0, ThunderSwapError::DustTrade)` before state mutation.

### Invariant 6: Slippage Always Applied

```
Every successful swap satisfies: amount_out >= min_amount_out
```

The check happens AFTER computing `amount_out` and BEFORE any state mutation or CPI call.

### Invariant 7: Fee Vault Rent Minimum

```
creator_fee_vault.lamports() >= rent_exempt_minimum at all times
```

`claim_creator_fees` transfers `vault_balance - rent_exempt_minimum` only.

### Invariant 8: Creator-Only Fee Claims

```
claim_creator_fees can only be called when: signer.key() == amm_pool.creator
```

### Invariant 9: Authority-Only Config Changes

```
update_global_config can only be called when: signer.key() == global_config.authority
```

---

## 3. Anchor-Specific Security

### 3.1 Account Discriminator Checks

Anchor automatically checks the 8-byte discriminator on every account access. This prevents type confusion attacks (passing an `AmmPool` account where `GlobalConfig` is expected). No additional code needed — this is Anchor's default behavior with typed accounts.

**Verify**: Never use `UncheckedAccount` for typed accounts (only use it for system lamport PDAs like `sol_vault`). Mark `UncheckedAccount` usages with `/// CHECK: <reason>` to make them explicit.

### 3.2 Signer Checks

Every instruction that mutates state must have at least one `Signer`. Verify:
- `swap`: `user: Signer` ✓
- `add_liquidity`: `provider: Signer` ✓
- `remove_liquidity`: `provider: Signer` ✓
- `claim_creator_fees`: `creator: Signer` ✓
- `create_pool`: `creator: Signer` ✓
- `create_pool_from_graduation`: `graduation_authority: Signer` ✓
- `update_global_config`: `authority: Signer` ✓
- `initialize_global_config`: `authority: Signer` ✓

### 3.3 Closing Accounts

ThunderSwap does not close any accounts in v1 (pools persist indefinitely). If account closing is added later, always use Anchor's `close = recipient` constraint which zeroes the data and returns lamports atomically — never implement manual account closing.

### 3.4 Program Upgrade Authority

After deployment, the upgrade authority must be transferred to the Squads multisig:

```bash
anchor upgrade --program-id <PROGRAM_ID> --provider.cluster mainnet \
  --upgrade-authority <SQUADS_MULTISIG_PDA>
```

Verify with: `solana program show <PROGRAM_ID>` — `Upgrade Authority` must show the Squads PDA, not the deployer wallet.

### 3.5 IDL Authority

The Anchor IDL (used by frontends and indexers to decode events) should also be controlled by the multisig after mainnet deployment. Set IDL authority to the Squads PDA:

```bash
anchor idl set-authority --program-id <PROGRAM_ID> --new-authority <SQUADS_PDA>
```

---

## 4. Off-Chain Security

### 4.1 Indexer Security

The ThunderSwap indexer runs with `SUPABASE_SERVICE_ROLE_KEY` — it has full read/write access to all tables. Protect this key:
- Never log the service role key
- Never include it in error responses
- Rotate it immediately if accidentally committed to git
- The indexer process should be the only consumer of this key

### 4.2 Graduation Webhook Authenticity

The `POST /api/webhooks/thunderswap-graduation` endpoint must verify the `Authorization: Bearer <CRON_SECRET>` header on every request. Without this:
- An attacker could POST fake graduation events, marking tokens as "graduated" without actual on-chain graduation
- This would cause the UI to show incorrect token status

**Additional verification**: After receiving the webhook, the handler should verify the `lp_burn_records` row exists in the DB (written by the indexer from on-chain data) before updating `pools.migration_status`. This double-check ensures the webhook cannot set graduation status for a token that hasn't actually graduated on-chain.

### 4.3 Quote API Integrity

The `POST /api/thunderswap/quote` endpoint computes swap quotes. If the TypeScript math diverges from the Rust program math, users will sign transactions with incorrect `min_amount_out` values — either overpaying slippage or having transactions fail unexpectedly.

**Mitigation**: Maintain a shared test suite (`tests/quote-parity.test.ts`) that runs 1,000+ randomized quote scenarios against both the TypeScript implementation and the Rust implementation (via Bankrun or local validator). Run this test in CI on every PR.

### 4.4 RPC Reliability

Pool reserve reads for quotes use the DB (updated by the indexer every ~15 seconds). This means quotes can be based on data up to 15 seconds stale.

- Always display "quote may be outdated — confirm before signing" in the UI
- The on-chain `min_amount_out` is the true slippage protection — a stale quote only affects UX, not security
- For production: consider fetching live reserves directly from RPC for the quote endpoint (with DB as fallback) to reduce staleness

---

## 5. API Security

### 5.1 Input Validation

Every API endpoint must validate inputs at the boundary. Never trust client-supplied values:

| Input | Validation Required |
|---|---|
| `mint` | Valid base58 string, length 32–44 chars |
| `amount_in` | Positive integer, max `u64::MAX` (9.2 × 10^18) |
| `slippage_bps` | Integer 0–5000 |
| `limit` | Integer 1–200 |
| `offset` | Non-negative integer |
| `is_buy` | Strict boolean |
| Wallet addresses | Valid base58, length 32–44 chars |

### 5.2 SQL Injection Prevention

All DB queries use Supabase's parameterized query builder (`.eq()`, `.filter()`, `.select()`). Never construct raw SQL strings with user input. The Supabase client handles parameterization automatically.

### 5.3 Rate Limiting

The quote endpoint is the most computationally sensitive. Apply stricter rate limiting (60 requests/minute per IP) to prevent abuse. Use Vercel's built-in edge rate limiting or a Redis-backed rate limiter (same pattern as ThunderLaunch).

### 5.4 CORS

Configure CORS to allow requests only from:
- `https://thunderlaunch.io`
- `https://swap.thunderlaunch.io`
- `http://localhost:3000` (development)

Never use `Access-Control-Allow-Origin: *` for production endpoints.

### 5.5 Sensitive Data in Logs

Never log:
- `SUPABASE_SERVICE_ROLE_KEY`
- `CRON_SECRET`
- Full wallet private keys
- Full transaction signatures in error messages (use first 8 chars)

---

## 6. Key Management

### 6.1 Squads Multisig Setup

The Squads 3-of-5 multisig controls `GlobalConfig`. Setup procedure:

1. Create Squads multisig at [app.squads.so](https://app.squads.so) on devnet first to verify
2. Add all 5 signers (verify each pubkey independently before adding)
3. Set threshold to 3
4. Record the Squads vault PDA address
5. Deploy ThunderSwap program with initial deployer wallet as authority
6. Call `update_global_config` to set `authority = squads_vault_pda`
7. Transfer program upgrade authority to Squads vault PDA
8. Verify: `solana program show <PROGRAM_ID>` shows Squads PDA as upgrade authority
9. Test: execute one `update_global_config` via Squads to verify end-to-end

### 6.2 Key Separation

| Key | Who Holds It | Used For |
|---|---|---|
| Squads Key 1–5 | 5 different team members | Governance (3-of-5) |
| Deployment wallet | CTO only | Initial deployment only — emptied after |
| `SUPABASE_SERVICE_ROLE_KEY` | Server env only | Indexer DB writes |
| `CRON_SECRET` | Server env only | Cron + webhook auth |
| `HELIUS_API_KEY` | Server env only | RPC access |

### 6.3 Key Rotation Policy

- Rotate `SUPABASE_SERVICE_ROLE_KEY` and `CRON_SECRET` every 90 days
- Immediately rotate any key that may have been exposed (committed to git, logged, etc.)
- The deployment wallet should be considered compromised after use and replaced

### 6.4 Hardware Wallets

Squads Keys 4 and 5 (cold storage wallets) must be hardware wallets (Ledger or Trezor). Store Key 4 offline in a physical safe. Store Key 5 at a different physical location from Key 4.

---

## 7. Pre-Audit Checklist

Complete all items before engaging an auditor. Auditors bill by time — arriving with clean code reduces cost and audit duration.

### Code Quality

- [ ] All `checked_*` arithmetic in `math.rs` — zero unwraps on arithmetic
- [ ] All `UncheckedAccount` usages documented with `/// CHECK:` comments explaining why they are safe
- [ ] No `#[allow(unused)]` or `#[allow(dead_code)]` suppressions
- [ ] `cargo clippy -- -D warnings` passes with zero warnings
- [ ] `cargo fmt` applied — consistent formatting throughout
- [ ] No `TODO` or `FIXME` comments in program code
- [ ] No `println!` or `msg!` debug logging left in hot paths

### Test Coverage

- [ ] All 8 instructions have positive (success) tests
- [ ] All 8 instructions have negative (failure) tests for each validation
- [ ] All 17 error codes are triggered by at least one test
- [ ] All 9 security invariants have dedicated invariant tests
- [ ] Fuzz test: 10,000+ random swap sequences verify k non-decreasing
- [ ] Fuzz test: 1,000+ random add/remove liquidity sequences verify LP math
- [ ] Test: sandwich attack scenario (min_amount_out enforcement)
- [ ] Test: LP inflation attack (MINIMUM_LIQUIDITY protection)
- [ ] Test: graduation authority spoofing (UnauthorizedGraduationCaller)
- [ ] Test: LP burn bypass attempt (PoolLiquidityLocked)
- [ ] Test: overflow scenarios with near-max u64 reserve values
- [ ] Test: claim_creator_fees with non-creator signer
- [ ] Integration test: full graduation CPI flow on local validator

### Documentation

- [ ] All account structs have field-level comments
- [ ] All instructions have doc comments explaining preconditions and postconditions
- [ ] `THUNDERSWAP_SMART_CONTRACT.md` matches the actual implementation
- [ ] `THUNDERSWAP_SECURITY_CHECKLIST.md` (this document) reviewed against final code

### Dependency Review

- [ ] `Cargo.toml` pinned to exact Anchor version (same as ThunderLaunch)
- [ ] No unverified crates — only `anchor-lang`, `anchor-spl`, `spl-token`, `solana-program`
- [ ] `cargo audit` passes with zero high-severity advisories
- [ ] Anchor version is current stable (check [anchor-lang releases](https://github.com/coral-xyz/anchor/releases))

### Deployment Configuration

- [ ] Program deployed to devnet and all integration tests pass
- [ ] `GlobalConfig` initialized with correct fee rates on devnet
- [ ] Squads multisig configured and tested on devnet
- [ ] Program upgrade authority is Squads multisig (not deployer) on devnet
- [ ] IDL authority is Squads multisig on devnet

---

## 8. Audit Scope & Engagement Guide

### Recommended Auditing Firms

In order of preference for Solana/Anchor programs:

1. **OtterSec** — strongest Solana/Anchor specialization, fastest turnaround
2. **Neodyme** — deep Solana expertise, thorough audit reports
3. **Trail of Bits** — comprehensive but slower; use if budget allows dual audit

### Audit Scope

Provide the following to the auditor:

```
In scope:
  programs/thunderswap/src/**          (the full Anchor program)
  src/math.rs                          (math module — critical)

Specific focus areas:
  1. Fee math correctness (rounding, overflow, fee extraction order)
  2. LP burn mechanism in create_pool_from_graduation
  3. TWAP manipulation resistance
  4. All PDA seed validation and account constraints
  5. Integer overflow in all arithmetic paths
  6. Slippage enforcement (min_amount_out checked AFTER computation)
  7. Graduation authority validation
  8. Creator fee claim access control

Out of scope (v1):
  Frontend code
  Indexer/backend code
  ThunderLaunch PLP program (already audited separately)
```

### Audit Deliverables to Request

- Full written report with severity ratings (Critical, High, Medium, Low, Informational)
- Remediation guidance for each finding
- Re-audit of fixes (one round included in engagement)
- Public disclosure timeline agreement (90 days after fix deployment)

### Bug Bounty

After mainnet deployment, establish a bug bounty program:
- **Critical** (fund loss): $50,000
- **High** (significant impact, no fund loss): $10,000
- **Medium**: $2,500
- **Low / Informational**: $500

Recommended platform: Immunefi or Code4rena.

---

## 9. Incident Response

### Severity Levels

| Level | Description | Example |
|---|---|---|
| **P0 — Critical** | Active fund loss or imminent exploit | Attacker draining pool, unauthorized graduation |
| **P1 — High** | Vulnerability confirmed, not yet exploited | Critical audit finding before mainnet |
| **P2 — Medium** | Degraded service, no fund loss | Indexer down, metrics stale |
| **P3 — Low** | Minor issue, no immediate risk | UI bug, incorrect display |

### P0 Response Procedure

1. **Pause the program immediately** via Squads: call `update_global_config` with `paused = true` (requires 3-of-5 signatures — all 5 signers must be reachable within 30 minutes)
2. **Pause the frontend**: deploy emergency "maintenance mode" page to `swap.thunderlaunch.io`
3. **Alert team** via out-of-band channel (Signal group, not Slack/Discord which may be compromised)
4. **Assess**: determine if funds are actively draining or just at risk
5. **If draining**: coordinate with Squads to drain protocol_fee_vault to cold storage
6. **Investigate**: reproduce the attack in a local validator fork
7. **Fix and re-audit**: do not reopen without a code fix and at minimum an internal review
8. **Post-mortem**: publish full post-mortem within 72 hours of resolution

### Emergency Contacts

Document and share with all Squads signers:
- All 5 Squads signer phone numbers (for out-of-band coordination)
- Helius emergency support contact
- Supabase support contact
- Legal counsel contact

### Circuit Breakers

Two independent kill switches:

| Switch | How to Activate | Effect |
|---|---|---|
| `GlobalConfig.paused = true` | `update_global_config` via Squads (3-of-5) | All swaps, liquidity add/remove blocked. Claims still work. |
| `AmmPool.locked = true` | `update_global_config` or add per-pool lock instruction (v2) | Single pool locked, others continue |

**Important**: `claim_creator_fees` is NOT blocked by `paused`. Creators can always claim their accumulated fees even when the program is paused. This is intentional — creators should not be unable to access their funds during an incident.

---

## 10. Ongoing Security Practices

### CI/CD Security Gates

Every PR must pass:
- [ ] `cargo test` — all tests pass
- [ ] `cargo clippy -- -D warnings` — zero warnings
- [ ] `cargo audit` — no high/critical advisories
- [ ] Quote parity test — TypeScript math matches Rust math

### Dependency Updates

- Review `cargo update` output monthly
- Never auto-merge dependency updates without manual review
- Pin Anchor version — do not update Anchor without a full re-test cycle

### Monitoring

After mainnet, monitor:
- Pool TVL changes > 20% in 5 minutes (potential exploit or large whale)
- Unusual fee vault drain patterns (claim_creator_fees on pools with 0 accumulated)
- Indexer lag > 60 seconds (gap in event processing)
- Failed graduation webhooks (reconciliation cron will catch these)

Integrate with ThunderLaunch's existing monitoring infrastructure (`evaluateEventAlert` pattern in ThunderLaunch codebase).

### Periodic Security Reviews

- **Monthly**: review access logs for unusual API patterns
- **Quarterly**: rotate `CRON_SECRET` and `SUPABASE_SERVICE_ROLE_KEY`
- **Before each major feature release**: internal security review against this checklist
- **Annually**: full re-audit if TVL > $1M or significant code changes have been made
