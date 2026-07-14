# ψ-Perp: Design v2 — Wrapper Around Percolator

## A Perpetual Futures DEX on Solana — Percolator Risk Engine (as-is) + Intelligent Wrapper

**Status:** Draft v2
**Date:** 2026-07-14
**Philosophy:** Build WITH Percolator, not instead of it.

---

## 1. The Pivot from v1

**v1 design:** ψ(t) replaces Percolator's A/K/F indices. One SDE-driven variable instead of three discrete accumulators.

**Why that was wrong:** Percolator is a complete, formally verified risk engine. The A/K/F indices exist for a reason — they handle edge cases (cross-slab contagion, partial fills, fee accounting) that the ψ(t) SDE doesn't cover. Replacing them means re-proving correctness. That's not innovation; that's recklessness.

**v2 design:** Percolator stays untouched. We build a WRAPPER program that:
1. Computes a better price feed (DPMM: oracle + inventory adjustment)
2. Runs ψ(t) as a read-only public monitor (alongside A/K/F, not replacing them)
3. Manages LP liquidity pools (PDLP/TWM — something Percolator doesn't do)
4. Adds a pre-liquidation compute-saving filter (CI option boundary)
5. Orchestrates the trade flow: user → wrapper → Percolator CPI → result

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                   WRAPPER PROGRAM (ours)                      │
│                                                              │
│  User-facing instructions:                                   │
│    deposit_liquidity, withdraw_liquidity                      │
│    open_position, close_position                             │
│    crank_wrapper                                             │
│                                                              │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────────┐  │
│  │  DPMM Mark   │  │  ψ(t) Monitor │  │  PDLP LP Pool    │  │
│  │  Price        │  │               │  │                  │  │
│  │              │  │  Public        │  │  Deposit/withdraw│  │
│  │  P_mark =    │  │  read-only.    │  │  LP token mint/  │  │
│  │  oracle ·    │  │  Updated per   │  │  burn. Pool acts │  │
│  │  (1+kξ²sgnξ) │  │  oracle tick.  │  │  as counterparty │  │
│  └──────┬───────┘  └──────┬────────┘  └────────┬─────────┘  │
│         │                 │                     │             │
│         │          ┌──────┴──────────┐          │             │
│         │          │  CI Option      │          │             │
│         │          │  Safety Filter  │          │             │
│         │          │                 │          │             │
│         │          │  Pre-checks     │          │             │
│         │          │  positions      │          │             │
│         │          │  before calling │          │             │
│         │          │  Percolator liq │          │             │
│         │          └──────┬──────────┘          │             │
│         │                 │                     │             │
│         └─────────────────┼─────────────────────┘             │
│                           │                                   │
│                    PRICE FEED (P_mark)                         │
│                           │                                   │
└───────────────────────────┼───────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                  PERCOLATOR (unchanged)                        │
│                                                              │
│  Receives from wrapper:  price, position_params, collateral   │
│  Returns to wrapper:     margin_ok, funding_due, liq_flag    │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐   │
│  │  A index │  │  K index │  │  F index │  │  Crank    │   │
│  │  (pos    │  │  (mark/  │  │  (funding │  │  cursor   │   │
│  │  scaling)│  │  ADL)    │  │  effects) │  │           │   │
│  └──────────┘  └──────────┘  └──────────┘  └───────────┘   │
│                                                              │
│  All three indices: O(1) per account, order-independent.     │
│  Wrapper never reads or writes A/K/F directly.               │
└──────────────────────────────────────────────────────────────┘
```

### Data Flow Per Trade

```
1. Pyth oracle posts P_oracle (every ~400ms on Solana)
2. Wrapper computes P_mark = P_oracle · (1 + k·ξ²·sign(ξ))
3. Wrapper updates ψ monitor: ψ_new = ψ_old · (1 + ξ · ΔP/P)
4. User calls open_position(side, collateral, leverage)
5. Wrapper:
   a. Transfers USDC from user to vault PDA
   b. Computes P_mark
   c. Calls Percolator CPI: open_position(P_mark, collateral, leverage, side)
   d. Percolator checks: margin_required < collateral? funding accrued?
   e. Percolator returns: Ok(PositionCreated) or Err(InsufficientMargin)
   f. Wrapper updates ξ from new total open interest
6. Keeper calls crank_wrapper(max_iterations):
   a. Read oracle → update DPMM → update ψ
   b. For each open position:
      - CI safety filter: is position near liquidation boundary?
      - If YES: call Percolator CPI → check_liquidation → liquidate if needed
      - If NO: skip (saves CU)
   c. Percolator handles all liquidations, ADL, funding accrual internally
   d. Wrapper reads post-crank state, updates ξ
7. User calls close_position():
   a. Wrapper calls Percolator CPI: close_position()
   b. Percolator computes final PnL (using P_mark, funding accrued, A/K/F)
   c. Wrapper transfers USDC from vault back to user
   d. Wrapper updates ξ
```

---

## 3. What We Build (The Wrapper)

### 3.1 DPMM Mark Price

**What it is:** The price we feed into Percolator instead of raw oracle.

```
ξ = (total_long_notional - total_short_notional) / (total_long_notional + total_short_notional)
P_mark = P_oracle · (1 + k · ξ² · sign(ξ))
```

**Why Percolator benefits:** Percolator's bounded envelope check runs on P_mark. If P_mark deviates too far from the last accepted price, the crank fails closed. The DPMM adjustment is small (k ≈ 0.01 → max ±1% at extreme ξ) — well within typical envelope bounds (±2-5%).

**What changes vs default Percolator usage:** Nothing internal. We just pass a different price. Percolator doesn't know or care where the price comes from.

| Aspect | Default Percolator | With DPMM Wrapper |
|--------|-------------------|-------------------|
| Price source | Raw oracle (Pyth) | P_mark = oracle · (1 + kξ²sgn(ξ)) |
| Envelope check | On raw oracle change | On P_mark change (same bounds) |
| Funding direction | oracle vs mark | oracle vs DPMM mark — more accurate |
| LP adverse selection | Full exposure | Reduced — spread absorbs some |

### 3.2 ψ(t) Solvency Monitor

**What it is:** A public, continuously-updated health metric for the pool.

```
ψ(0) = 1.0
ψ(t+Δt) = ψ(t) · (1 + ξ(t) · (P_{t+Δt} - P_t) / P_t)
```

**Where it lives:** A field in the wrapper's state account. Anyone can read it. No one can modify it except the wrapper's crank instruction.

**What it does NOT do:**
- Does NOT feed into Percolator's margin calculations
- Does NOT trigger liquidations (Percolator's A/K/F do that)
- Does NOT replace any Percolator logic

**What it DOES do:**
- **Keeper signal:** ψ dropping fast → crank more aggressively. ψ stable → crank less often.
- **LP signal:** ψ > 1 → pool healthy, good deposit timing. ψ < 0.95 → pool stressed, LP withdrawals may face delays.
- **Trader signal:** ψ < 1 and falling → ADL risk increasing. Close or reduce position.
- **Frontend display:** Simple number. 1.00 = balanced. 0.95 = caution. 0.90 = danger.
- **Funding rate tuning:** ψ persistently below 1 → funding rate is too low (longs aren't paying enough). ψ persistently above 1 → funding rate too high. The wrapper can suggest ℓ adjustments.

**Relationship to A/K/F:**

| | A/K/F (Percolator internal) | ψ(t) (Wrapper public) |
|---|---|---|
| Update | Per crank (minutes) | Per oracle tick (~400ms) |
| Visibility | Program-internal only | Anyone can read |
| Purpose | Correct risk accounting | Transparency + early warning |
| Math | Accumulator indices | SDE continuous process |
| Authority | Authoritative (triggers liq/ADL) | Advisory (signal only) |

They measure the same thing differently. A/K/F are the court. ψ(t) is the weather report.

### 3.3 CI Option Safety Filter

**What it is:** A cheap pre-check before invoking Percolator's (expensive) liquidation logic.

The CI option exercise boundary:

```
S_ℓ(q) = q/(r + σ²/2) · [g - g^{1/γ_p}]
where g = 1 + rK/q, γ_p = -2r/σ²
```

**On-chain implementation (simplified):**

```rust
fn is_near_liquidation(position: &Position, mark_price: u64, volatility: u64) -> bool {
    let entry = position.entry_price;
    let leverage = position.leverage;
    // Rough liq price for a long: entry * (leverage - 1) / leverage
    let rough_liq = (entry as u128 * (leverage - 1) as u128 / leverage as u128) as u64;
    // CI buffer: tighten when volatility is high
    let buffer = volatility * CI_BUFFER_BPS / 10_000;
    let threshold = rough_liq * (10_000 + buffer) / 10_000;
    mark_price <= threshold
}
```

If this returns false → position is clearly safe → skip Percolator CPI → save ~25,000 CU.
If this returns true → position might need liquidation → call Percolator → let it decide.

**Why additive:** Percolator still has final say. This is a compute optimization, not a logic change.

### 3.4 LP Pool (PDLP)

**What it is:** The counterparty liquidity pool. When traders go long, the pool is effectively short. LPs deposit USDC, receive LP tokens, earn fees.

**Why Percolator doesn't have this:** Percolator is a risk engine for TRADER positions. It doesn't manage LP capital. The wrapper fills this gap.

**Basic mechanics (MVP):**
- LPs deposit USDC → mint LP tokens proportional to pool share
- Pool USDC = counterparty to all trader positions
- Pool PnL = -(sum of all trader PnL) — when traders win, pool loses; when traders lose, pool wins
- Trading fees (spread) accrue to pool
- LP withdraws → burn LP tokens → receive share of pool USDC

**Post-MVP (TWM):** Full PDLP with target weight mechanism. Pool maintains optimal collateral weights. LP share pricing includes discount/premium for directional imbalance.

---

## 4. Solana Program Spec

### 4.1 Accounts

```rust
#[account]
pub struct WrapperState {
    // Version
    pub version: u8,

    // DPMM
    pub dpmm_k: u64,                // Shape param (fixed-point, e.g. 0.01 = 100_000_000)
    
    // ψ(t) Monitor
    pub psi: u64,                   // Current ψ (fixed-point, 1.0 = PRECISION)
    pub psi_last_price: u64,        // Last oracle price for ψ update
    pub xi: i64,                    // Current ξ = (L-S)/(L+S) (fixed-point)
    pub psi_update_slot: u64,       // Last slot when ψ was updated
    
    // LP Pool
    pub lp_total_supply: u64,       // Total LP tokens outstanding
    pub lp_pool_usdc: u64,          // USDC in pool (deposits - withdrawals + PnL)
    
    // Percolator reference
    pub perp_market: Pubkey,        // Percolator's market PDA
    
    // Oracle
    pub oracle_feed: Pubkey,        // Pyth price account
    
    // Auth
    pub vault_usdc: Pubkey,         // USDC vault PDA (owned by wrapper)
    pub authority: Pubkey,          // Admin (can update params)
    pub bump: u8,
}

// Position account lives in Percolator. Wrapper reads it via CPI.
// Wrapper does NOT maintain its own position registry.
```

### 4.2 Instructions

```rust
#[program]
pub mod psi_perp_wrapper {
    // ── Admin ──
    pub fn initialize(ctx: Context<Initialize>, dpmm_k: u64, oracle: Pubkey) -> Result<()>;
    pub fn update_params(ctx: Context<AdminOnly>, dpmm_k: u64) -> Result<()>;
    
    // ── LP ──
    pub fn deposit_liquidity(ctx: Context<LPDeposit>, amount: u64) -> Result<()>;
    pub fn withdraw_liquidity(ctx: Context<LPWithdraw>, lp_tokens: u64) -> Result<()>;
    
    // ── Trader ──
    pub fn open_position(ctx: Context<OpenPosition>, side: Side, collateral: u64, leverage: u64) -> Result<()>;
    pub fn close_position(ctx: Context<ClosePosition>) -> Result<()>;
    
    // ── Keeper (permissionless) ──
    pub fn crank(ctx: Context<Crank>, max_iterations: u16) -> Result<()>;
}
```

### 4.3 Key Implementation: open_position

```rust
fn open_position(ctx: Context<OpenPosition>, side: Side, collateral: u64, leverage: u64) -> Result<()> {
    let wrapper = &mut ctx.accounts.wrapper_state;
    let oracle_price = pyth::get_price(&ctx.accounts.oracle_feed)?;
    
    // 1. Update ψ(t) monitor
    let price_ratio = (oracle_price as i128 - wrapper.psi_last_price as i128)
        * PRECISION_I128 / wrapper.psi_last_price as i128;
    let dpsi_ratio = wrapper.xi as i128 * price_ratio / PRECISION_I128;
    wrapper.psi = (wrapper.psi as i128 * (PRECISION_I128 + dpsi_ratio) / PRECISION_I128) as u64;
    wrapper.psi_last_price = oracle_price;
    
    // 2. Compute DPMM mark price
    let mark_price = dpmm_mark(oracle_price, wrapper.xi, wrapper.dpmm_k);
    
    // 3. Transfer USDC from user to vault
    spl_token::transfer(
        &ctx.accounts.user_usdc,
        &ctx.accounts.vault_usdc,
        &ctx.accounts.user_authority,
        collateral,
    )?;
    
    // 4. Call Percolator to open position
    //    Percolator receives mark_price, checks margin, creates position
    percolator::cpi_open_position(
        &ctx.accounts.percolator_program,
        &ctx.accounts.perp_market,
        ctx.accounts.user.key(),
        mark_price,
        collateral,
        leverage,
        side,
    )?;
    
    // 5. Recompute ξ from Percolator's open interest
    let (total_long, total_short) = percolator::cpi_get_open_interest(
        &ctx.accounts.perp_market,
    )?;
    let total = total_long as i128 + total_short as i128;
    wrapper.xi = if total > 0 {
        ((total_long as i128 - total_short as i128) * PRECISION_I128 / total) as i64
    } else {
        0
    };
    
    Ok(())
}
```

### 4.4 Key Implementation: crank

```rust
fn crank(ctx: Context<Crank>, max_iterations: u16) -> Result<()> {
    let wrapper = &mut ctx.accounts.wrapper_state;
    let oracle_price = pyth::get_price(&ctx.accounts.oracle_feed)?;
    
    // 1. Update DPMM & ψ
    update_psi(wrapper, oracle_price);
    let mark_price = dpmm_mark(oracle_price, wrapper.xi, wrapper.dpmm_k);
    
    // 2. Check bounded envelope (Percolator-style safety)
    let price_delta = (mark_price as i128 - wrapper.psi_last_price as i128).abs();
    let max_delta = wrapper.psi_last_price as i128 * ENVELOPE_MAX_BPS as i128 / 10_000;
    require!(price_delta <= max_delta, ErrorCode::EnvelopeExceeded);
    
    // 3. Iterate open positions via Percolator's cursor
    let cursor = percolator::cpi_get_crank_cursor(&ctx.accounts.perp_market)?;
    let positions = percolator::cpi_get_positions_at_cursor(
        &ctx.accounts.perp_market, cursor, max_iterations,
    )?;
    
    let mut liq_count = 0u16;
    for pos in &positions {
        // CI safety filter
        if ci_filter::is_near_liquidation(pos, mark_price, wrapper.volatility_estimate()) {
            // Near liquidation boundary → call Percolator
            percolator::cpi_check_and_liquidate(
                &ctx.accounts.percolator_program,
                &ctx.accounts.perp_market,
                pos,
                mark_price,
            )?;
            liq_count += 1;
        }
        // else: clearly safe, skip Percolator call
    }
    
    // 4. Update ξ post-crank
    let (total_long, total_short) = percolator::cpi_get_open_interest(&ctx.accounts.perp_market)?;
    wrapper.xi = compute_xi(total_long, total_short);
    
    // 5. Accrue funding via Percolator
    percolator::cpi_accrue_funding(
        &ctx.accounts.percolator_program,
        &ctx.accounts.perp_market,
        mark_price,
    )?;
    
    emit!(CrankEvent {
        slot: Clock::get()?.slot,
        oracle_price,
        mark_price,
        psi: wrapper.psi,
        xi: wrapper.xi,
        positions_checked: positions.len() as u16,
        liquidations: liq_count,
    });
    
    Ok(())
}
```

### 4.5 Compute Budget

| Instruction | CU | Notes |
|------------|-----|-------|
| deposit_liquidity | ~8,000 | Token transfer + LP mint |
| open_position | ~25,000 | ψ update + DPMM + USDC xfer + Percolator CPI |
| close_position | ~30,000 | ψ update + Percolator CPI (PnL calc) + USDC xfer back |
| crank (no liquidations) | ~50,000 | ψ + DPMM + iterate 30 pos with CI filter all passing |
| crank (with liquidations) | ~50,000 + 25,000×N | Each liq adds Percolator CPI cost |
| crank (batch 20, 4 liq) | ~150,000 | Well within 1.4M limit |

---

## 5. MVP Scope

### In Scope
- Wrapper program with DPMM + ψ monitor + CI filter + basic LP pool
- Percolator compiled as library (same program binary)
- Pyth SOL/USD oracle
- Single market: SOL-PERP, USDC collateral, fixed 10x leverage
- open_position / close_position / crank / deposit_liquidity / withdraw_liquidity
- CLI dev tooling (TypeScript + Anchor)

### Out of Scope (Post-MVP)
- Multiple leverage levels, multiple markets, multiple collateral types
- PDLP Target Weight Mechanism (static pool for MVP)
- DPMM k auto-calibration
- ψ(t) backtesting/simulation
- Insurance fund staking
- Governance
- Frontend

### Line Count

| Component | Lines |
|-----------|-------|
| state.rs (accounts, constants, errors) | ~200 |
| dpmm.rs (mark price) | ~50 |
| psi.rs (monitor) | ~100 |
| ci_filter.rs (safety pre-check) | ~80 |
| lp.rs (deposit/withdraw) | ~120 |
| crank.rs (orchestration) | ~150 |
| lib.rs (instruction dispatch) | ~200 |
| percolator_glue.rs (CPI wrappers) | ~150 |
| Tests | ~500 |
| Percolator crate | 0 (Cargo dep) |
| **Total** | **~1,550** |

---

## 6. Innovation Summary

| Layer | What | Novel? | Risk Level |
|-------|------|--------|------------|
| DPMM mark price | oracle · (1+kξ²sgnξ) | Adaptation of Mohanty et al. to perps | Low — small multiplier on oracle |
| ψ(t) monitor | Public read-only SDE signal | Yes — no perp DEX exposes continuous solvency metric | Low — no authority, advisory only |
| CI safety filter | Pre-liquidation CU optimization | Yes — optimal stopping theory on-chain | Low — Percolator still does final liq check |
| PDLP LP pool | Counterparty liquidity management | Using Chitra et al. formalism | Medium — LP pool is new code |

**None of these replace anything in Percolator.** They add observability (ψ), efficiency (CI filter), better pricing (DPMM), and LP infrastructure (PDLP). Percolator's A/K/F indices, margin engine, bounded envelope, pro-rata ADL, and keeper crank remain untouched.

---

## 7. Relationship to Existing Literature

See `perp-futures-literature-review.md` for the full survey.

Directly applied:
- **DPMM** (Mohanty et al. 2025): mark price formula adapted for perps
- **ψ(t) SDE** (Shang et al. 2025): monitor implementation, discretized Euler step
- **CI options** (Singh et al. 2025, Feinstein 2025): safety filter boundary
- **PDLP** (Chitra et al. 2025): LP pool mechanics
- **Funding uniqueness** (Kim & Park 2025): informs Percolator's ℓ parameter tuning

Not applied (kept for future):
- **ψ-HJB** (Le 2026): optimal market making — post-MVP
- **Risk-weighted ADL** (ADL Impossibilities 2025): post-MVP enhancement to Percolator's pro-rata
