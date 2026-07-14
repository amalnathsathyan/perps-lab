# Solana Perps Deep Research — Chat Export
## July 2026

---

## User Request 1: Perp futures implementations and repos on Solana

### Response: Full research compiled

#### Reference Implementations (Start Here)

**1. solana-labs/perpetuals** — Canonical reference
- Repo: `github.com/solana-labs/perpetuals`
- Stack: Anchor (Rust), Pyth oracles, Apache 2.0
- Architecture: Multi-custody pool — each pool holds token custodies (assets, stables, synthetic). Pool-to-trader model, LPs earn real yield.
- Status: Mature, well-documented, forks everywhere. **Best starting point.**

**2. Percolator** — Anatoly Yakovenko's perps engine
- Repos:
  - `github.com/dcccrypto/percolator` — Core risk engine (no_std Rust, v16.8.3 spec)
  - `github.com/dcccrypto/percolator-launch` — "Pump.fun for Perps" one-click market deployer
  - `github.com/dcccrypto/percolator-sdk` — TypeScript SDK
  - `github.com/HaidarIDK/PERColator` — Web + CLI with 116/116 Kani formal proofs
- Architecture: Coin-margined, vAMM initial liquidity, novel risk model (haircut ratio H, side indices A/K instead of ADL queues). Permissionless listing — no governance, any SPL token. Up to 20x.
- Status: Research-grade, not production audited. **Best for studying formal verification of perps risk engine.**

**3. alxn787/perp-dex** — Simplest full implementation
- Repo: `github.com/alxn787/perp-dex`
- Stack: Anchor (Rust)
- Architecture: Hybrid orderbook + AMM. Market/limit orders Long/Short. PDA-based accounts. Direct matching → AMM fill → hybrid execution.
- Status: Smaller codebase, easier to grok. **Best for understanding the full flow end-to-end.**

#### Production Protocols

| Protocol | Repo | Architecture | Open Source |
|---|---|---|---|
| Drift Protocol v2 | `drift-labs/protocol-v2` | CLOB + JIT + vAMM hybrid | Full |
| Mango Markets v4 | `blockworks-foundation/mango-v4` | On-chain CLOB | Full (AGPL-3.0) |
| Jupiter Perps | `jup-ag/cli` (CLI only) | JLP pool model | Partial |
| Flash Trade | `flash-trade/flash-perpetuals` | Pool-to-peer (fork of solana-labs) | Full (Apache 2.0) |
| Adrena | `AdrenaFoundation` | Pool-based (GMX-style) | Partial (archived) |
| GMTrade | `code-423n4/gmtrade-bug-bounty` | Multi-pool (GMX-style) | Partial |
| Pacifica | Closed source | Off-chain match + on-chain settle | Closed |
| Bullet (ex-Zeta) | `bulletxyz` | Validator sidecar CLOB | Partial |
| Phoenix Perps | `Ellipsis-Labs/phoenix-v1` | On-chain CLOB (crankless) | Full (BUSL-1.1) |
| Raydium Perps | Via Orderly Network | Off-chain CLOB | Closed |
| Solayer Margin | `solayer-labs` | SVM appchain | Partial |

#### Architectural Decision Tree

```
Perps architecture on Solana:
├── vAMM/Pool-based (simplest to build)
│   ├── solana-labs/perpetuals — multi-custody pool
│   ├── Percolator — coin-margined, permissionless
│   └── Jupiter Perps — JLP basket pool
├── Orderbook/CLOB (harder, more capital-efficient)
│   ├── Drift v2 — CLOB + JIT + vAMM hybrid
│   ├── Mango v4 — Serum-style orderbook
│   └── Pacifica — fully onchain CLOB
└── Oracle-based (simplest, centralization risk)
    └── SportsPerp — oracle-driven composite indices
```

#### What to Study for Building Your Own

| Priority | Repo | Why |
|---|---|---|
| 1 | `solana-labs/perpetuals` | Canonical. Build, deploy, trace every instruction. |
| 2 | `alxn787/perp-dex` | Smaller, readable. Hybrid orderbook+AMM in fewer lines. |
| 3 | `dcccrypto/percolator` | Formal verification. Risk engine invariants. |
| 4 | `drift-labs/protocol-v2` | Production patterns. Keeper networks, JIT auctions, cross-margin. |

---

## User Request 2: Deep research agent findings (background agent completed)

### Additional details discovered:

**Protocols initially missed:** GMTrade (volume leader, pool-based, 500x), Pacifica (fastest-growing, off-chain matching + on-chain settlement), Bullet (validator sidecar, 20ms execution), Solayer Margin Trade (SVM appchain perps), Raydium Perps (via Orderly Network, 80+ pairs).

**Architectural split:**

| Model | Protocols | Build Complexity |
|---|---|---|
| Pool-to-trader (vAMM) | Jupiter, GMTrade, Adrena, Flash Trade | Easiest |
| Hybrid (CLOB + vAMM) | Drift v2 | Hard |
| On-chain CLOB | Mango v4, Phoenix | Hardest |
| Off-chain match + on-chain settle | Pacifica, Raydium Perps | Medium |
| Infrastructure-level | Bullet, Solayer | Hardest |

---

## User Request 3: Beginner's guide to perp DEX + Percolator

### Part 1: What is a Perpetual Future?

You bet on price direction. No expiry. No delivery of the actual asset.

**Example:**
- SOL is $100. You think it goes up.
- You deposit $100 (collateral). You want 5x leverage.
- Protocol "lends" you $400. Your position = $500 (5 SOL worth).
- SOL goes to $120. You made $100 profit on $100 deposit = 100% return.
- SOL goes to $80. Your $500 position lost $100 = your entire deposit gone. **Liquidated.**

The trick that makes this work without expiry dates: **funding rate.**
If perp price drifts above spot price → longs pay shorts. Below → shorts pay longs.

### Part 2: The Five Pieces of a Perp DEX

1. **Oracle** — "What's the real price?" (Pyth on Solana). #1 attack surface.
2. **Vault** — PDA holding all the money. Core invariant: `sum(all_user_balances) ≤ vault_balance`.
3. **Risk Engine** — The MATH. This is Percolator. "Can this position open? Is it underwater? How much can they withdraw?"
4. **Funding Rate** — Anchors perp price to spot price. Periodic payments between longs and shorts.
5. **Liquidation** — Keeper bots close underwater positions, take fee. ADL as fallback.

### Part 3: What Percolator Actually Is

Percolator is NOT a DEX. It has no token transfers, no accounts, no signer checks, no CPI calls. It is a **pure math library** in `no_std` Rust with zero dependencies.

**The Three Invariants:**

1. **Backed Exits (Haircut Ratio H):**
   `H = total_collateral_in_vault / total_trader_net_equity`
   H must always be ≥ 1.0. Exchange must always have enough money to pay everyone.

2. **Fair Overhang (Side Indices A/K):**
   When longs pay shorts (or vice versa), payment is distributed lazily. Two indices A/K track "who gets paid next." Processing is O(1), not O(n).

3. **Bounded Cranks (Price Envelope):**
   Each state update can only move price by a capped amount. New markets start with restricted price ranges ("two-bucket warmup"). Prevents oracle manipulation.

### Part 4: How to Build It — Step by Step

**Step 1: Understand the reference (1-2 days)**
```bash
git clone https://github.com/solana-labs/perpetuals.git
cd perpetuals
anchor build
cargo test-bpf -- --nocapture
```
Read: `state/mod.rs` → `open_position.rs` → `close_position.rs` → `math/`

**Step 2: Understand Percolator (1 day)**
```bash
git clone https://github.com/dcccrypto/percolator.git
cd percolator
# Read the .md spec files — they ARE the design docs
cargo kani --tests
```

**Step 3: Design your program's accounts**
```rust
pub struct Market {
    pub token_mint: Pubkey,
    pub vault: Pubkey,
    pub oracle: Pubkey,
    pub funding_rate: i64,
    pub open_interest_long: u64,
    pub open_interest_short: u64,
    pub percolator_state: PercolatorState,
}

pub struct Position {
    pub owner: Pubkey,
    pub market: Pubkey,
    pub direction: Direction,
    pub size: u64,
    pub collateral: u64,
    pub leverage: u64,
    pub entry_price: u64,
    pub last_funding_index: u64,
}
```

**Step 4: Build 4 instructions for MVP**
```
1. initialize_market    — Create a new perp market for a token
2. deposit_collateral   — User sends tokens to vault PDA
3. open_position        — Open long/short with leverage
4. close_position       — Close position, settle PnL, return collateral
```

**Step 5: Wire in Percolator**
```rust
require!(
    percolator.can_open(
        collateral, size, leverage,
        market.open_interest_long,
        market.open_interest_short,
        oracle_price,
    ),
    ErrorCode::RiskCheckFailed
);
```

**Step 6: Add funding rate** (only after Step 4 works)
```rust
let funding_payment = position.size
    * (market.funding_rate - position.last_funding_index)
    / FUNDING_PRECISION;
```

**Step 7: Add liquidation** (only after Step 6 works)
Keeper bot calls `liquidate_position` with oracle price. If underwater, close position, take fee %.

### Part 5: What to Skip for v1 MVP

- Orderbook / limit orders → pool-to-trader only, instant execution at oracle price
- Cross-margin → isolated margin only
- Multi-asset collateral → USDC only
- Keeper network → manual liquidation cranks
- Governance → single admin key
- Fees beyond open/close spread
- Multiple markets → start with SOL-PERP only

**v1 architecture:** ~500 lines of Anchor Rust. Deposit → open → close → withdraw. Ship that, then iterate.

---

## Security Incidents to Learn From

| Protocol | Date | Loss | Root Cause |
|---|---|---|---|
| Mango Markets | Oct 2022 | ~$100M | Oracle manipulation via MNGO token price |
| Drift Protocol | Apr 2026 | ~$285M | Social engineering of admin key holders (NOT smart contract bug) |

---

## Solana-Specific Implementation Notes

- **PDAs for vaults:** Every custody/trader account is a PDA. Seeds include market ID + user pubkey.
- **CPI is expensive:** Percolator's no-CPI design is deliberate. Each CPI call costs compute units.
- **Account size limits:** Solana accounts max 10KB. Perp positions must be compact.
- **Clock is approximate:** `Clock::get()?.unix_timestamp` has ~400ms granularity.
- **Atomic settlement:** Solana transactions are all-or-nothing.

---

## All Protocol Repositories

| Protocol | GitHub | Open Source | Architecture |
|---|---|---|---|
| Solana Labs Perpetuals | `solana-labs/perpetuals` | Full (Apache 2.0) | Pool-based |
| Percolator | `dcccrypto/percolator` | Full (Apache 2.0) | Risk engine only |
| Percolator Launch | `dcccrypto/percolator-launch` | Full | Permissionless deployer |
| Percolator SDK | `dcccrypto/percolator-sdk` | Full | TypeScript SDK |
| PERColator (Haidar) | `HaidarIDK/PERColator` | Full | Web + CLI |
| Drift Protocol v2 | `drift-labs/protocol-v2` | Full | Hybrid CLOB+vAMM |
| Mango Markets v4 | `blockworks-foundation/mango-v4` | Full (AGPL-3.0) | On-chain CLOB |
| Phoenix | `Ellipsis-Labs/phoenix-v1` | Full (BUSL-1.1) | On-chain CLOB |
| Flash Trade | `flash-trade/flash-perpetuals` | Full (Apache 2.0) | Pool-to-peer |
| perp-dex | `alxn787/perp-dex` | Full | Hybrid AMM+orderbook |
| Jupiter Perps | `jup-ag/cli` (CLI only) | Partial | JLP pool |
| Pacifica | Closed | Closed | Off-chain match |
| GMTrade | `code-423n4/gmtrade-bug-bounty` | Partial | Multi-pool |
| Bullet | `bulletxyz` (org) | Partial | Validator sidecar |
| Adrena | `AdrenaFoundation` | Partial (archived) | Pool-based |
| Vega Finance | `thevega-finance/thevega-perps` | Full | RWA-extended |
| Raydium Perps | Via Orderly Network | Closed | Off-chain CLOB |
| Solayer Margin | `solayer-labs` | Partial | SVM appchain |

---

Sources:
- [Solana Labs Perpetuals](https://github.com/solana-labs/perpetuals)
- [Percolator](https://github.com/dcccrypto/percolator)
- [Drift Protocol v2](https://github.com/drift-labs/protocol-v2)
- [Mango v4](https://github.com/blockworks-foundation/mango-v4)
- [Phoenix v1](https://github.com/Ellipsis-Labs/phoenix-v1)
- [Flash Trade Perpetuals](https://github.com/flash-trade/flash-perpetuals)
- [perp-dex](https://github.com/alxn787/perp-dex)
- [Percolator Launch](https://github.com/dcccrypto/percolator-launch)
- [Jupiter CLI](https://github.com/jup-ag/cli)
- [Adrena Foundation](https://github.com/AdrenaFoundation)
- [GMTrade Bug Bounty](https://github.com/code-423n4/gmtrade-bug-bounty)
