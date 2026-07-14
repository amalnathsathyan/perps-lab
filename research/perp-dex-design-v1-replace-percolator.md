# ψ-Perp: Formal Design Document

## A Perpetual Futures DEX on Solana with ψ(t) Solvency Tracking + Percolator Risk Engine

**Status:** Draft v1
**Date:** 2026-07-14
**Based on:** Literature review of 14 papers (see `perp-futures-literature-review.md`)

---

## 1. Executive Summary

We build an on-chain perpetual futures DEX on Solana. Two innovations differentiate us:

1. **ψ(t) continuous solvency tracking** (from PvpAMM, AFT 2025) — replaces static margin checks with a stochastic process that tracks global pool health in real time.

2. **Percolator risk engine** (Toly Yakovenko's formally verified Rust crate) — handles margin, liquidation, ADL with pro-rata fairness and bounded price envelopes.

The architecture is **Oracle + DPMM mark price → ψ(t) solvency tracker → Percolator risk engine**. No CLOB, no vAMM curve. The simplest thing that demonstrates the innovation.

---

## 2. Architecture Decision: Why Oracle + DPMM + ψ(t)

### 2.1 The Options

| Mechanism | How it works | On-chain complexity | Cold start | MEV risk |
|-----------|-------------|---------------------|------------|----------|
| **CLOB** | Order book matching | High (matching engine on SVM) | Needs market makers | Front-running |
| **vAMM** | x·y=k virtual pool | Medium (AMM math per trade) | Self-starts | Sandwich attacks |
| **Oracle** | Trade at oracle ± spread | Low (one SLOAD + arithmetic) | Instant | Oracle manipulation |
| **Oracle + DPMM + ψ** | Oracle base, inventory-adjusted mark, continuous solvency | Low-Medium (ψ cumulative product + DPMM quadratic) | Instant | Spread + ψ bounds mitigate |

### 2.2 Decision: Oracle + DPMM + ψ(t)

**Reasoning:**

1. **Solana SVM can't do a proper CLOB on-chain.** Order matching is O(n log n) per block. Hyperliquid, dYdX run order books off-chain with on-chain settlement. We don't want to build an off-chain matching engine for an MVP.

2. **vAMM has known failure modes.** PvpAMM paper explicitly critiques GMX's LPT mechanism for exponential position growth under frequent price updates. Static AMM curves (x·y=k) have no concept of solvency — they'll quote prices that bankrupt the pool.

3. **Oracle pricing is the standard for on-chain perps.** Jupiter Perps, GMX, GNS all use oracle + spread. It works at scale. The spread protects LPs. The innovation is in HOW we set the spread and track risk, not in price discovery.

4. **DPMM mark price is the differentiator.** Instead of a fixed spread, the mark price adjusts quadratically with position imbalance ξ(t). When the pool is net long, mark ticks up (shorts get better entry). When net short, mark ticks down. This is the DPMM mechanism that achieved 90.8% win rate vs 48.6% for static AMMs (Mohanty et al., 2025).

5. **ψ(t) makes solvency continuous.** Every other perp DEX checks solvency at discrete intervals (crank turns). ψ(t) tracks it continuously via the SDE dψ/ψ = ξ·dP/P. When ψ < 1, haircuts are mathematically necessary — not a heuristic threshold.

### 2.3 What ψ(t) Replaces in Percolator

Percolator uses three lazy side indices updated at each crank:

| Percolator Index | What it tracks | ψ(t) Equivalent |
|-----------------|----------------|-----------------|
| **A** | Position scaling (effective exposure) | ∂ψ/∂ξ captures exposure sensitivity |
| **K** | Mark/ADL overhang (unrealized PnL vs collateral) | ψ < 1 means pool underwater → haircut = 1-ψ |
| **F** | Funding effects (accumulated funding payments) | sign(dψ) = funding direction, magnitude = funding intensity |

**ψ collapses three indices into one SDE-governed variable.** This is simpler AND more powerful — the SDE gives us provable properties (martingale under μ=0, convergence a.s., explicit bounds).

---

## 3. System Architecture

### 3.1 Component Diagram

```
┌──────────────────────────────────────────────────────────┐
│                       USER                                │
│  deposit_collateral | open_position | close_position     │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                   ψ-PERP PROGRAM                          │
│                                                          │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │  Collateral │  │   Position   │  │   ψ(t) Engine  │  │
│  │   Manager   │  │   Manager    │  │                │  │
│  │             │  │              │  │  ψ = Σ wⱼψⱼ/m  │  │
│  │ deposit/    │  │ open/close/  │  │  dψ/ψ = ξ·dP/P │  │
│  │ withdraw    │  │ PnL calc     │  │  ξ = imbalance │  │
│  └──────┬──────┘  └──────┬───────┘  └───────┬────────┘  │
│         │                │                   │           │
│         └────────────────┼───────────────────┘           │
│                          │                               │
│                          ▼                               │
│  ┌──────────────────────────────────────────────────┐    │
│  │              PERCOLATOR RISK ENGINE               │    │
│  │                                                   │    │
│  │  • Margin checks (initial + maintenance)          │    │
│  │  • Funding rate accrual (from ψ-driven formula)   │    │
│  │  • Liquidation engine (CI option boundaries)      │    │
│  │  • Pro-rata ADL (ψ-weighted)                      │    │
│  │  • Insurance fund                                 │    │
│  │  • Keeper crank (permissionless)                  │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────┐  ┌──────────────────────────────────┐  │
│  │  DPMM Mark   │  │        Oracle Gateway            │  │
│  │   Price      │  │                                  │  │
│  │              │  │  Pyth / Switchboard price feed   │  │
│  │  P_mark =    │  │  + confidence interval check     │  │
│  │  oracle ·    │  │  + staleness check               │  │
│  │  (1+kξ²sgnξ) │  │  + bounded envelope (Percolator) │  │
│  └──────────────┘  └──────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                   KEEPER NETWORK                          │
│  crank_liquidations | crank_settlements | crank_funding  │
└──────────────────────────────────────────────────────────┘
```

### 3.2 Data Flow

```
1. Oracle posts price P_oracle
2. DPMM computes P_mark = P_oracle · (1 + k·ξ²·sign(ξ))
3. User requests trade at P_mark
4. Percolator checks margin: c ≥ m_I · |q| · P_mark
5. Position created → ψ(t) updated: ψ_new = (Σ wⱼ·ψ(tⱼ) + w_new·ψ(t)) / (Σ mⱼ + m_new)
6. Every crank:
   a. Oracle price checked against bounded envelope
   b. ψ(t) SDE advanced: Δψ = ψ·ξ·(ΔP/P)  (drift zero under martingale measure)
   c. If ψ < ψ_liquidation_threshold → trigger liquidations
   d. Funding accrued: F = ℓ(φ-Y) - 𝓛φ + rY (Kim & Park design)
   e. Pro-rata ADL if ψ < ψ_adl_threshold
7. User closes: w̃ = w_final · ψ(t_entry)/ψ(t_close), receive max(0, w̃)
```

### 3.3 Key Invariants

```
I1: Σ mⱼ = Σ w̃ⱼ(t)  ∀t           (Pool solvency — PvpAMM Theorem 4)
I2: ψ(t) > 0  ∀t                    (ψ never goes negative — PvpAMM Prop 4)
I3: lim_{t→tⱼ⁺} w̃ⱼ(t) = mⱼ         (No flash-loan profit — PvpAMM Theorem 8)
I4: E[ψ(t)] = 1 under μ=0           (Martingale property — PvpAMM Theorem 5)
I5: P_mark ∈ [P_oracle·(1-ε), P_oracle·(1+ε)]  (Bounded envelope — Percolator)
```

---

## 4. Core Mathematics

### 4.1 ψ(t) SDE

**Position value (conventional, pre-liquidation):**

```
wⱼ(t) = mⱼ · (1 - bⱼ + bⱼ · P_t/P_{tⱼ}) · 𝟙(τⱼ ≥ t)
```

where mⱼ = collateral, bⱼ = leverage multiplier, P_t = oracle price, τⱼ = liquidation time.

**Imbalance parameter:**

```
ξ(t) = Σ(mⱼ·bⱼ · P_t/P_{tⱼ} · ψ(tⱼ) · 𝟙(τⱼ ≥ t)) / Σ(wⱼ(t) · ψ(tⱼ))
```

sign(ξ) > 0 means net long, sign(ξ) < 0 means net short. |ξ| is leverage-weighted imbalance magnitude.

**The SDE:**

```
dψ(t)/ψ(t) = ξ(t) · dP_t/P_t
```

Under GBM (dP/P = μ dt + σ dB):

```
ψ(t) = ψ(0) · exp{∫₀ᵗ ξ(s)σ(s)dB_s + ∫₀ᵗ [ξ(s)μ(s) - ½ξ²(s)σ²(s)]ds}
```

**Discrete approximation (on-chain):**

```
ψ_{n+1} = ψ_n · (1 + ξ_n · (P_{n+1} - P_n)/P_n)
```

This is a cumulative product — O(1) per update, no loops. Store ψ_n and update on each oracle price change or position change.

### 4.2 DPMM Mark Price

```
P_mark = P_oracle · (1 + k · ξ² · sign(ξ))
```

**Parameter k calibration:**
- k = 0: pure oracle pricing (static spread)
- k small (0.001-0.01): gentle inventory adjustment
- k large (0.1+): aggressive inventory rebalancing

From the PvpAMM arbitrage condition (Theorem 6), the optimal k ensures:

```
Profit_arb = ξ(ξ - b)(ΔP)² > 0  when arbitrage condition holds
```

This means k should be set such that the DPMM spread makes arbitrage unprofitable for b values below the pool's average leverage. Empirically: start with k = 0.01 and tune based on simulation.

### 4.3 Funding Rate (Kim & Park Design)

```
Φ_t = ℓ(φ_t - Y_t) - 𝓛φ_t + rY_t
```

where:
- φ_t = target value (oracle price)
- Y_t = current mark price
- ℓ = mean-reversion strength (must exceed critical threshold)
- 𝓛 = infinitesimal generator of the price process

**Simplified for implementation:**

```
funding_rate_t = ℓ · (P_oracle - P_mark)/P_oracle
```

With Percolator's bounded envelope providing the ℓ > ℓ_crit guarantee. When P_mark > P_oracle (pool net long), longs pay shorts. When P_mark < P_oracle (pool net short), shorts pay longs.

### 4.4 Liquidation Boundary (CI Option Theory)

```
S_ℓ = q/(r + σ²/2) · [g - g^{1/γ_p}]
```

where g = 1 + rK/q, γ_p = -2r/σ².

**Practical implementation:** Instead of solving this closed form on-chain, pre-compute a lookup table:

```
S_ℓ(maintenance_margin, σ, r) → liquidation_price_ratio
```

For MVP: use the standard maintenance margin approach (Percolator's existing logic) with the CI option boundary as a configurable override.

### 4.5 Pro-Rata ADL with ψ-Weighting

When ψ < ψ_adl (e.g., 0.95), trigger ADL on the winning side:

```
haircut_i = (1 - ψ) · (leverage_i · equity_i) / Σ(leverage_j · equity_j)
```

This is the risk-weighted variant from the ADL optimality paper. Higher-leverage winners get larger haircuts, which is both fairer (they contributed more risk) and more efficient (less total haircut needed).

---

## 5. Solana Program Design

### 5.1 Accounts

```rust
// Core state accounts
#[account]
pub struct PerpMarket {
    pub market_authority: Pubkey,       // PDA authority
    pub oracle: Pubkey,                 // Pyth/Switchboard feed
    pub collateral_mint: Pubkey,        // USDC mint
    pub vault: Pubkey,                  // USDC vault (PDA)
    pub insurance_fund: Pubkey,         // Insurance fund vault
    pub psi: u64,                       // ψ(t) current value (fixed-point)
    pub xi: i64,                        // ξ(t) current imbalance
    pub total_collateral: u64,          // Σ mⱼ
    pub total_weighted_positions: u64,  // Σ wⱼ·ψ(tⱼ)
    pub dpmm_k: u64,                    // DPMM shape parameter
    pub funding_l: u64,                 // Funding mean-reversion ℓ
    pub psi_adl_threshold: u64,         // ψ threshold for ADL trigger
    pub envelope_max_move_bps: u64,     // Max oracle move per crank (bps)
    pub bump: u8,
}

#[account]
pub struct Position {
    pub owner: Pubkey,
    pub market: Pubkey,
    pub collateral: u64,                // mⱼ (initial USDC deposit)
    pub leverage: u64,                  // bⱼ (multiplier, stored as fixed-point)
    pub entry_price: u64,               // P_{tⱼ}
    pub entry_psi: u64,                 // ψ(tⱼ) at entry
    pub side: Side,                     // Long or Short
    pub created_at: i64,                // Timestamp
    pub funding_checkpoint: u64,        // Last funding accrual timestamp
}

#[account]
pub struct KeeperState {
    pub crank_cursor: u64,              // Current position in processing queue
    pub last_crank_time: i64,
    pub total_liquidations: u64,
    pub total_adl_events: u64,
}
```

### 5.2 Instructions

```rust
pub enum PerpInstruction {
    // Market admin
    InitializeMarket {
        dpmm_k: u64,
        funding_l: u64,
        psi_adl_threshold: u64,
        envelope_max_move_bps: u64,
    },
    
    // User actions
    DepositCollateral { amount: u64 },
    WithdrawCollateral { amount: u64 },
    OpenPosition {
        side: Side,
        collateral: u64,
        leverage: u64,
    },
    ClosePosition {},
    
    // Keeper actions (permissionless)
    CrankLiquidations { max_iterations: u16 },
    CrankFunding { max_iterations: u16 },
    CrankSettlements { max_iterations: u16 },
    
    // Admin
    UpdateDPMMParams { k: u64 },
    SetMarketPaused { paused: bool },
}
```

### 5.3 Key Implementation Details

**ψ(t) update (called on every oracle change and position change):**

```rust
fn update_psi(market: &mut PerpMarket, new_price: u64) {
    let price_ratio = (new_price as i128 - market.last_price as i128) 
                      * PRECISION / market.last_price as i128;
    let dpsi = (market.psi as i128 * market.xi as i128 * price_ratio) / PRECISION;
    market.psi = (market.psi as i128 + dpsi) as u64;
    market.last_price = new_price;
}
```

**DPMM mark price:**

```rust
fn compute_mark_price(market: &PerpMarket, oracle_price: u64) -> u64 {
    let xi_squared = (market.xi as i128 * market.xi as i128) / PRECISION;
    let adjustment = market.dpmm_k as i128 * xi_squared * market.xi.signum() / PRECISION;
    (oracle_price as i128 * (PRECISION + adjustment) / PRECISION) as u64
}
```

**Position value (PvpAMM style):**

```rust
fn position_value(pos: &Position, market: &PerpMarket, current_price: u64) -> u64 {
    // Conventional value
    let raw_value = pos.collateral as i128 
        * (PRECISION - pos.leverage as i128 
           + pos.leverage as i128 * current_price as i128 / pos.entry_price as i128)
        / PRECISION;
    
    // PvpAMM-adjusted value
    let adjusted = raw_value * pos.entry_psi as i128 / market.psi as i128;
    
    max(0, adjusted) as u64
}
```

### 5.4 Compute Budget Estimation

| Instruction | CU Estimate | Notes |
|------------|-------------|-------|
| DepositCollateral | ~5,000 | Simple token transfer + account update |
| OpenPosition | ~15,000 | Margin check + ψ update + position PDA init |
| ClosePosition | ~20,000 | PnL calc + ψ update + token transfer + account close |
| CrankLiquidations (per position) | ~25,000 | Price check + margin check + ADL calc + transfer |
| CrankFunding (per position) | ~8,000 | Funding accrual arithmetic |
| Update ψ (standalone) | ~3,000 | One multiply-add |

**Solana limit: 1.4M CU per tx.** A crank processing 50 liquidations = 50 × 25,000 = 1.25M CU. Fit in one tx.

---

## 6. MVP Scope

### 6.1 In Scope

- Single market: SOL-PERP with USDC collateral
- Oracle pricing via Pyth SOL/USD feed
- DPMM mark price with configurable k
- ψ(t) solvency tracker (cumulative product)
- Fixed leverage (10x) — no user selection yet
- Open position (long or short)
- Close position (return adjusted collateral)
- Basic liquidation at maintenance margin threshold
- Pro-rata ADL when ψ < ψ_adl
- Permissionless keeper crank (liquidation + funding)
- Funding rate: 8-hour TWAP with ℓ mean-reversion
- Percolator-style slab structure (single slab for MVP)
- Bounded oracle price envelope

### 6.2 Out of Scope (Post-MVP)

- Multiple leverage levels (hardcoded 10x for MVP)
- Multiple collateral types (USDC only)
- LP pool / PDLP mechanism (counterparty is the pool itself)
- Multiple markets / cross-margin
- DPMM k auto-calibration (manual admin set)
- CI option liquidation boundaries (standard maintenance margin for MVP)
- Insurance fund staking / yield
- Governance / fee parameters
- Limit orders / TP/SL
- Mobile SDK / frontend

### 6.3 Line Count Estimate

| Component | Lines | Reuse |
|-----------|-------|-------|
| PerpMarket account + instructions | ~300 | New |
| Position account + instructions | ~250 | New |
| ψ(t) engine | ~150 | New |
| DPMM mark price | ~50 | New |
| Oracle gateway (Pyth CPI) | ~100 | New |
| Percolator integration (glue code) | ~200 | New |
| Keeper crank logic | ~200 | Adapted from Percolator |
| Percolator risk engine | ~0 | Import crate as-is |
| Tests (LiteSVM + Mollusk) | ~500 | New |
| CLI / dev scripts | ~200 | New |
| **Total** | **~1,950** | |

### 6.4 What We Use From Percolator As-Is

- i128/u128 fixed-point math utilities
- Margin calculator (initial + maintenance)
- Funding rate accumulator
- Liquidation price calculator
- Slab/shard data structure pattern
- Keeper crank cursor pattern
- Bounded envelope checker

### 6.5 What We Modify/Add

- **Replace A/K/F indices with ψ(t):** The main change. Instead of updating three separate accumulators, update one ψ value.
- **Add DPMM mark price:** New. Percolator uses raw oracle price. We adjust it.
- **Add CI option liquidation (post-MVP):** Replace fixed maintenance margin with endogenous boundary.

---

## 7. Innovation Roadmap

### Phase 1: MVP (2-3 weeks)
Deliver: SOL-PERP on devnet with ψ(t) + DPMM + Percolator
Prove: ψ(t) tracks solvency correctly in live market conditions

### Phase 2: ψ(t) Validation (1-2 weeks)
Deliver: Backtest ψ(t) against historical SOL price data
Prove: ψ(t) SDE properties hold empirically (martingale under μ=0, convergence)

### Phase 3: CI Option Liquidation (1 week)
Deliver: Replace static maintenance margin with CI option boundary
Prove: Fewer false-positive liquidations vs fixed threshold

### Phase 4: ψ-HJB Market Making (Research)
Deliver: Optimal quote strategy via HJB with ψ state variable
Prove: Improved LP profitability vs static DPMM

### Phase 5: Multi-Market + Risk-Weighted ADL (Production)
Deliver: Multiple perp markets, ψ-weighted ADL, LP pools
Prove: System handles cross-market contagion

---

## 8. Testing Strategy

### Unit Tests (LiteSVM/Mollusk)

```
test_psi_update_single_position      // ψ tracks single long correctly
test_psi_update_balanced_positions   // ψ stays at 1 when ξ=0
test_psi_update_imbalanced           // ψ moves with price when ξ≠0
test_psi_flash_loan_resistance       // Entry/exit same price = no profit
test_psi_martingale_property         // E[ψ]=1 under zero-drift simulation
test_dpmm_mark_price_zero_imbalance  // P_mark = P_oracle when ξ=0
test_dpmm_mark_price_net_long        // P_mark > P_oracle when ξ>0
test_dpmm_mark_price_net_short       // P_mark < P_oracle when ξ<0
test_liquidation_undercollateralized // Position liquidated when below maintenance
test_adl_pro_rata                    // Winners haircut proportional to equity
test_funding_rate_direction          // Longs pay when P_mark > P_oracle
test_oracle_envelope_bound           // Crank rejects price moves > envelope
```

### Integration Tests (Localnet)

```
test_full_lifecycle_long             // Open → fund → close long (profit)
test_full_lifecycle_short            // Open → fund → close short (profit)
test_liquidation_flow                // Open → price crash → keeper liquidates
test_multi_user_imbalance            // 5 longs, 2 shorts → ψ behavior
test_keeper_crank_batch              // 50 liquidations in one tx
```

### Simulation Tests

```
test_psi_convergence_monte_carlo     // 10K paths, verify martingale property
test_dpmm_vs_static_pnl              // Compare DPMM vs fixed-spread LP PnL
test_adl_efficiency                  // Compare ψ-weighted vs equal-weight ADL
```

---

## 9. Open Questions

1. **ψ(t) update frequency:** Every crank? Every oracle update? Every trade? Trade-off between accuracy and compute. Default: every price-changing event.

2. **DPMM k selection:** How to choose k without historical data? Start at k=0.01, run simulation sweeps, let market admin adjust.

3. **Bounded envelope vs. ψ accuracy:** If oracle moves 5% but envelope caps at 2%, ψ(t) sees a truncated price path. Does this break the martingale property? Likely no — the SDE uses actual (capped) prices, so ψ reflects the pool's real experience.

4. **Funding rate period:** 8 hours is standard but arbitrary. Kim & Park's path-dependent formula works for any δ. Keep 8h for MVP.

5. **Single-slab vs multi-slab:** Percolator's slab design isolates risk per market. Single slab for MVP (one market). Multi-slab adds cross-market contagion analysis (open problem from lit review).

6. **ψ(t) initialization:** Start at ψ(0) = 1. First trade: ψ(t₁) = w₁(t₁)/m₁ (from PvpAMM two-position example). This means first position _determines_ initial ψ — is that correct? Yes: with one position, ψ = w/m, and the position's PvpAMM value w̃ = w·ψ(0)/ψ = w/ψ = m. The solo trader always gets exactly their collateral back at the same price. Fair.

---

## 10. Architecture Decision Record: Why NOT vAMM or CLOB

### 10.1 Alternative Considered: Oracle + vAMM (Two-Phase)

Proposal: Ship vAMM + Percolator first, upgrade to ψ(t) later. Rationale: vAMM is proven on Solana (Drift v2), ψ(t) is paper-only.

**Rejected.** Reasoning:
- vAMM intermediate step = rework. Building a vAMM means implementing x·y=k curve math, LP token accounting, and pool rebalancing — all of which gets ripped out in Phase 2.
- ψ(t) is SIMPLER than vAMM. It's a cumulative product, not a curve invariant. The SDE discretization is one multiply-add per update.
- vAMM has known failure modes at scale (PvpAMM paper's GMX critique: exponential position growth under frequent updates).
- DPMM ψ(t) is the strategic differentiator. Ship it first.

### 10.2 Alternative Considered: CLOB

Proposal: On-chain order book + Percolator, like dYdX/Hyperliquid but fully on-chain.

**Rejected.** Reasoning:
- Solana SVM cannot run a proper matching engine on-chain. Order matching is O(n log n). Hyperliquid keeps it off-chain.
- CLOB needs professional market makers for liquidity. Cold start kills it.
- CLOB is philosophically mismatched with ψ(t) — discrete order matching against a continuous solvency field.
- If we wanted a CLOB, we'd fork Hyperliquid's off-chain matcher, not build on Solana.

### 10.3 Decision: Oracle + DPMM + ψ(t), Direct

One phase. Ship the innovation. The math is simpler than the alternatives (cumulative product vs. curve invariant vs. order matching). The risk is that ψ(t) has no production precedent — mitigated by Percolator's bounded envelope as a safety rail, and by the fact that ψ(t) degrades gracefully to standard oracle pricing when k=0.

---

## 11. References

See `perp-futures-literature-review.md` for full paper list with equations and links.

Key papers for this design:
1. **PvpAMM** (Shang, Zhao, Chen — AFT 2025): ψ(t) SDE, PLT mechanism, arbitrage, flash-loan resistance
2. **Percolator** (Yakovenko): Risk engine, slab structure, bounded envelope, pro-rata ADL
3. **Kim & Park** (2025): Funding rate BSDE design with uniqueness guarantee
4. **Mohanty et al.** (2025): DPMM for everlasting options (adaptation to perps)
5. **Singh et al.** (AFT 2025): LVR = funding fees, CI option boundaries, constant-LVR profiles
6. **Le** (2026): Funding-aware HJB market making
7. **ADL Impossibilities** (2025): Pro-rata optimality, risk-weighted ADL
