# Permissionless Perp for Any Token on Monad — Percolator Risk Engine

**Date:** 2026-09-02
**Status:** In progress — agent reports landing

## Goal

Feasibility + easiness assessment: build new perp platform on Monad (EVM) with Percolator risk math at its heart. Target: **permissionless perps for any token on Monad**. Constraint: align with [Monad Metropolis](https://www.monad.xyz/developers/hackathons/metropolis) Track 01 "Onchain Finance & Trading" criteria (build 1 Sep–13 Oct 2026, 6 weeks, $30k track).

## Prior context

See `monad-metropolis-percolator-feasibility.md` in this folder — already covers: Percolator ground truth (repos, audit, spec v16.9.0, A/K/F/H), Monad specs (0.3–0.4s blocks, gas-limit billing, 128KB contracts), Perpl/Kuru/Drake basics, portability verdict (~2–3k LOC Solidity port, mine uniperpapp/Percolator-Ethereum).

---

## 1. Monad perp landscape — open-source architecture analysis

Monad mainnet since 24 Nov 2025 (chainId 143). DeFi TVL ~$959M. ~10 perp venues; 3 with meaningful volume (PerpFinder, Sept 2026).

### 1.1 Volume leaderboard

| Protocol | 24h vol | 7d vol | Share |
|---|---|---|---|
| Perpl | $25.3M | $411.3M | 74.9% |
| LeverUp | $8.4M | $63.6M | 25.0% |
| Drake | $47.7k | $564.3k | 0.1% |
| Narwhal, Pingu, Monday Trade, OBSDN | ~0 | ~0 | — |

### 1.2 Protocol catalog

**Perpl — live, market leader.** Isolated-margin CLOB (O(1) post/cancel, ~100k gas), AUSD collateral, 6 markets, invite-gated. Risk stack = dynamic margin, partial+full onchain liquidation, per-perp insurance fund, ADL, withdrawal rate-limiting (10% TVL/h, $1M floor) — **closest live analogue to Percolator's feature set.** Funding: block-number-scheduled ~hourly (8,571 blocks). **Core contracts closed-source.** Public: [dex-sdk](https://github.com/PerplFoundation/dex-sdk) (Rust, order-intent API) + docs. Oracle: Chainlink.

**LeverUp — live, #2.** "LP-free perps" — no LP pool, OI not capped by TVL, up to 1001x, zero fees when PnL<0, profit-sharing curve. AnyCollateral: ecosystem tokens as margin. Two public audits. Closed-source [unverified]. Reuse: none; curious LP-free PnL backing model.

**Monday Trade — live but ~0 volume.** SynFutures Builder Program first launch; AMM-based (SynFutures V3 synthetic AMM), 424 pairs, "Entropy" limit orders Apr 2026. Open-source status [unverified].

**Drake — mainnet mid-2026, tiny volume.** Hybrid CLOB+AMM, 50x, unified cross+isolated portfolio margin, USDC/ETH/BTC collateral. Chainlink Data Streams. Unique: **frUSDC funding-harvest vault** (delta-neutral cash-and-carry, auto-rebalance), dUSDC LP vault. Closed-source [unverified]. Concept worth studying: frUSDC overlaps our funding-arb tool idea.

**AtlantisDEX (Orbs × Symmio) — live Jan 2026.** Intent/RFQ model, NOT onchain CLOB. Orbs L3 validators run Hedger (MM using CEX liquidity), Liquidator, Price Oracle. Symmio core = **BUSL-1.1 → GPLv2 31 Dec 2027 — not forkable commercially now**. Opposite philosophy to Percolator's crank-forward permissionless design.

**OBSDN — live, dying.** Off-chain matching + onchain settlement, USDC. TVL ~$6k, down 97%/30d. Strategy: list new narratives faster than CEXs — "early Hyperliquid playbook" failed. Cautionary tale.

**Zaros — THE open-source find.** Perps for LST/LRT collateral, 30–40 pairs, Arbitrum-deployed, Monad "planned". **Only audited open-source EVM perps code found.**
- [Cyfrin/2024-07-zaros](https://github.com/Cyfrin/2024-07-zaros) — PerpsEngine root proxy, 2,878 nSloc, Solc 0.8.25, **Foundry**. Branch modules: LiquidationBranch, OrderBranch, PerpMarketBranch, SettlementBranch, TradingAccountBranch, GlobalConfigurationBranch. ERC-721 AccountNFT. Chainlink push oracle.
- [Cyfrin/2025-01-zaros-part-2](https://github.com/Cyfrin/2025-01-zaros-part-2) — Market Making Engine: ZLP vault, liquidity delegation, dynamic OI/skew caps, ADL.
- Liquidation: MMR<1, LiquidationKeeper (Chainlink Automation or allowlisted EOA). Listing permissioned (eDAO multisig).
- Cyfrin audit + [CodeHawks contest](https://codehawks.cyfrin.io/c/2024-07-zaros). License [unverified — check before mining].
- **Reuse verdict: best clean-room reference for EVM perps branch decomposition, keeper patterns, margin config. Risk math still rewritten (no A/K/F, no haircut, no envelope).**

**Bean — live.** Spot DLMM (Liquidity Book) + vault-backed perps (GMX-synthetics style), Pyth, 50x, cross-margin. [github.com/BeanExchange](https://github.com/BeanExchange): **spot infra open (bean-spot MIT, dlmm-contract-interface, planker-contracts)** — perps core closed. bean-spot (MIT) useful if bin-book mechanics wanted.

**Celeris — testnet.** PLOB: one shared liquidity pool backs all markets simultaneously, marketed on MonadDB multi-market state sync. Closed [unverified]. Concept adjacent to Percolator MarketGroup.

**Narwhal — dead (2024-era, zero volume). Pingu — shut down 31 Jul 2026.** Oracle-priced (Pyth), zero-spread, 100x, 70+ markets — failed on Monad: ~$80M cumulative volume in 6 months, <$70k TVL. **Best data point on cold-start economics: even 100x + 70 markets + zero spread couldn't sustain a non-top-3 venue.**

**Composite — pre-launch, closed.** CLOB unifying lend+spot+perps, portfolio margin. Concept only.

### 1.3 Open-source reuse landscape

| Source | License | Reusable | Relevance |
|---|---|---|---|
| Cyfrin Zaros (both repos) | [unverified] | Branch-structured perps engine, keeper design, liquidation branch, skew caps/ADL | **Highest** — only audited open EVM perps code |
| BeanExchange/bean-spot | MIT | Liquidity Book Solidity | Only if CLOB/bin-book route |
| PerplFoundation/dex-sdk | SDK public | Rust order-intent API design, funding/liq parameterization via `/v1/pub/context` | API/bounty route only |
| Symmio core | BUSL-1.1 → GPLv2 2027 | Intent-based settlement | Not forkable now |
| Kuru SDK | unverified | Onchain CLOB linked lists | Spot-only |

### 1.4 Key conclusions

1. **No Monad-native perps publishes core contracts.** All live leaders closed-source. Open-source EVM reference = Zaros (mine for structure, not risk math).
2. **Percolator port remains greenfield** — nothing to fork wholesale. Perpl productized Percolator's feature set but closed; its docs = best public parameterization reference.
3. **Funding cadence**: Perpl's block-number-scheduled hourly funding = production answer; our lazy-F per-block accrual differentiates on mechanism (O(1), no scheduled write).
4. **Market validation**: 2 venues absorb ~100% of Monad perps volume; everything else ~0. Drake under $50k/24h mid-2026 launch; Pingu died at $80M cumulative. **Cold start brutal — lead with risk-engine/observability story, not volume.**
5. **Oracle**: Chainlink (push/Data Streams) = incumbent choice (Perpl, Drake, Zaros); Pyth pull used by Bean-perps, Pingu. Both live on Monad. Pyth pull fits lazy-accrual/fail-closed envelope better; staleness guards mandatory either way.

---

## 2. Permissionless market creation + oracle coverage for long-tail tokens

### 2.1 Permissionless-listing mechanics across protocols

| Protocol | Chain | Mechanism | Permissionless? | Gate / cost | Oracle |
|---|---|---|---|---|---|
| Hyperliquid HIP-1/2 (spot) | HyperEVM | Dutch auction for ticker (2x→500 HYPE over 31h), 5-step SpotDeployAction; HIP-2 = protocol grid MM at 0.3% spread per block | Yes | HYPE auction; HIP-2 seed capital | Internal (book + oracles) |
| Hyperliquid HIP-3 (perps) | HyperEVM | Any builder deploys perp contracts/clearinghouses, no committee approval | Yes | **500k HYPE (~$20M) slashable stake**; Dutch auction per 31h; isolated margin, OI caps, halt function | Deployer-chosen |
| Hyperliquid HIP-6 (proposed) | HyperEVM | On-chain token launch, per-block uniform CCA | Proposed | 10 HYPE fee | — |
| dYdX v4 | dYdX chain | Market-listing widget, ~7 clicks | Semi | 2,000 DYDX deposit (burned if ≥33.4% NoWithVeto), ~4-day vote | Auto-configured list; 6+ CEXes required for mid-cap |
| GMX v2 | Arbitrum | Governance vote (10k tokens propose, 50k quorum) | No | Fast-track needs ≥$1B mcap, ≥$100M ADV, live Chainlink | Chainlink |
| Drift | Solana | Admin-initialized (`handle_initialize_*`) | No (admin multisig) | — | Pyth primary, Switchboard secondary |
| Jupiter Perps | Solana | Team/multisig listed [unverified] | No [unverified] | — | Pyth [unverified] |
| Perpl | Monad | 6 markets (BTC, MON, ETH, SOL, HYPE, ZEC). Process not public [unverified] | No (team) | **One Chainlink Data Streams feed per market** → listing gated on live Chainlink feed | Chainlink Data Streams (since 2025-11-25) |
| Kuru (spot) | Monad | `MonadDeployer.deployTokenAndMarket()` — public, no whitelist | Yes (spot only) | Gas only | N/A |
| **Percolator "Design A" (research)** | Solana spec | Permissionless no-deep-liquidity perps for ANY token: P2P matching first, paid bounded seed as residual counterparty, **parimutuel settlement — haircut `credit_rate = min(1, backing/claim)`**, per-market isolation, OI caps sized to seed | Design only | Creator seeds bounded paid counterparty | Pyth + staleness + confidence filters |

Sources: [Hyperliquid deploy docs](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/deploying-hip-1-and-hip-2-assets), [HIP-3](https://cryptoslate.com/anyone-can-now-create-hyperliquid-perp-contracts-with-20m-is-defi-about-to-break/), [HIP-6](https://www.kucoin.com/news/flash/hyperliquid-hip-6-proposes-on-chain-token-auction-mechanism), [dYdX widget](https://www.dydx.foundation/blog/market-listing-widget-tutorial), [GMX framework](https://gov.gmx.io/t/new-assets-listing-framework/3660/2), [Drift discussion](https://github.com/orgs/drift-labs/discussions/631), [Perpl docs](https://docs.perpl.xyz/resources/for-developers/overview), [Kuru deploy-market](https://docs.kuru.io/sdk/deploy-market), [percolator-perp-liquidity](https://github.com/dcccrypto/percolator-perp-liquidity), [Percolator threat model](https://github.com/dcccrypto/percolator/blob/main/THREAT_MODEL.md)

### 2.2 Protections inventory for tiny-mcap perps (who does what)

- **Price bands**: Drift/Velocity — reject orders if oracle-mark diverges >10% from 5-min TWAP; single trade capped at 2% impact; per-tx base-asset caps. Percolator — per-slot price-move circuit breaker (`SetOraclePriceCap`), max_crank_staleness, confidence filter. [Drift risk params](https://drift-protocol-v2-docs.vercel.app/protocol/risk-and-safety/risk-parameters)
- **Halts**: HIP-3 halt function → settle at fair value, freeze trading. Percolator fail-closed on envelope breach.
- **Stakes**: HIP-3 ~$20M slashing stake (brutal but effective). Percolator alternative = paid bounded seed + parimutuel haircut (protocol cannot go insolvent by construction).
- **PnL warmup**: Percolator holds unrealized spike profit in per-account reserve with vesting before it enters payout denominator — attacker can't spike-and-withdraw.
- **Isolation + OI caps**: HIP-3 isolated clearinghouses; Percolator per-market vaults ("a blown market dies cheaply"); dYdX isolated margin planned for long-tail.
- **Cautionary tales**: JELLY (HLP ~$230M near-drain), SK Hynix (HIP-3 deployer manipulated mark-price inputs — **deployer-chosen mark inputs = weak spot**), Drift Apr 2026 ($250M+ via compromised admin key listing fake collateral asset). Lesson: **risk params protocol-enforced, never owner-editable; listing permission never extends to collateral assets.**

### 2.3 Oracle coverage on Monad mainnet

**Pyth — live push feeds ("Pyth Lazer/Bolt"), 60 feeds total:**
- 4 premium (1h heartbeat, 0.02% dev): BTC, ETH, MON, SOL
- 56 standard (1h heartbeat, 0.05% dev): majors (AAVE, ADA, BNB, LINK, SUI, XRP…), stables (AUSD, USDC, USDT…), LSTs/wrappers (WBTC, WEETH, WSTETH, EZETH…), DOGE, XAU/XAG, ~9 ratio pairs
- PriceFeed `0x2880aB155794e7179c9eE2e38200202908C17B43`, Entropy `0xD458261E832415CFd3BAE5E416FdF3230ce6F134`. Feed list: [monad-crypto/protocols pyth.jsonc](https://github.com/monad-crypto/protocols/blob/main/mainnet/pyth.jsonc)
- Note: 60 = push set. Pyth **pull** model is chain-agnostic (~400+ crypto feeds via Hermes) — pull catalog on Monad potentially much larger than 60 [moderate confidence, verify against Pyth Monad docs].

**Chainlink — live since mainnet launch day:**
- Data Streams (pull, sub-second, onchain-verified) + Price Feeds + CCIP via SCALE. MON/USD live; verifier proxy `0xC539169910DE08D237Df0d73BcDa9074c787A4a1`; 15+ protocols integrated at launch. [Monad oracles doc](https://monad-docs.vercel.app/tooling-and-infra/oracles), [launch](https://www.cryptowisser.com/index.php/news/chainlink-ccip-data-streams-and-price-feeds-launch-on-monad/)
- **Perpl = Chainlink Data Streams, one feed per market.** Spot index writes onchain on 0.1% move or 10s-to-stale; marks = median of external CEX mids, basis-adjusted fair value, impact mid, book mid. [price indices](https://docs.perpl.xyz/exchange/price-indices)

**Alternatives** (Stork, Switchboard, Redstone, Edge): no evidence of Monad deployment found. [unverified]

**Existing Monad perps:** Perpl = Chainlink Data Streams (confirmed). Drake/Bean/Celeris [unverified].

### 2.4 Honest answer: "perp for ANY token" feasible on Monad?

**No — honest scope = "perp for any token WITH a credible oracle." Oracle is the binding constraint, not listing mechanics.**

1. Most permissionless perp listing in production (HIP-3) costs ~$20M slashing stake and still produced manipulation incidents (SK Hynix mark-input spike, JELLY). Nobody shipped long-tail perps safely without (a) huge deployer stakes or (b) Percolator-style structural constraints — and Percolator's own research repo says its any-token design is "solvency unsafe-as-specified (research-stage)".
2. Monad credible-oracle universe today ≈ 60 Pyth push feeds (majors, stables, liquid alts — no real memecoin long-tail beyond DOGE) + Chainlink feed set. Perpl's 6 markets = one Chainlink feed each.
3. New feed for novel token = oracle provider BD step, weeks-months, not hackathon step. **True gate.**
4. Below credible-oracle tokens, no safe mark → margin/liquidation/ADL all manipulable. Bands/staleness/halts mitigate, cannot substitute.

**So: listing can be permissionless on Monad today; feed creation cannot.** Pitch = "any token with a Monad oracle feed" (~60 assets day one, larger if Pyth pull catalog used). Percolator protections (envelope, PnL warmup, haircut, OI caps, isolation) = exactly what makes listing those without governance defensible.

### 2.5 Recommended MVP design

**Permissionless factory:**
1. `PerpFactory.createMarket(pythFeedId, params)` — one tx; params **immutable at creation** (leverage cap, OI cap, envelope budget, H) — never owner-editable (Drift lesson).
2. Asset universe = on-chain whitelist of live Monad oracle feed IDs (60 Pyth push IDs; extensible to pull catalog + Chainlink). Factory reads feed registry, not team address.
3. Creator seeds small insurance fund ($1–5k USDC); **OI cap sized to seed** (Percolator Design A: OI cap ∝ paid seed). Zero seed = P2P-only mode or reject. Replaces HIP-3's $20M stake with bounded-loss economics — the differentiator.
4. Permissionless keepers: liquidation + funding crank (lazy F index), fail-closed on envelope breach → auto trading halt until oracle sanity.

**Oracle design:**
- Primary: Pyth on Monad. Pull via Hermes for breadth, or 60 push feeds for simplicity (1h heartbeat + 0.05% dev is coarse — envelope budget must dominate intra-heartbeat moves).
- Percolator guards regardless: staleness (max age in blocks), confidence filter, per-block max-move envelope (re-derived for ~0.4s blocks — `price_budget_bps = cfg_max_price_move_bps_per_slot × cfg_max_accrual_dt_slots` transfers directly), PnL warmup vesting, H haircut on profit exits.
- Chainlink Data Streams (verifier `0xC5391699…`) optional second source for MON — also the Chainlink $3k sponsor-bounty hook.

**Demo narrative:** "We made listing permissionless; the risk engine (envelope + haircut + warmup + OI-cap-from-seed) is what lets us do it without governance or a $20M stake. The remaining constraint — which token has a live oracle — is shown on-screen, not hidden."

**Unverified items to close before submission:** (1) Pyth pull catalog (~400+ feeds) consumable on Monad or only 60 push? (2) Chainlink Data Streams breadth on Monad beyond MON/USD. (3) Jupiter/Perpl exact listing processes. (4) Pyth publisher economics for new feeds. (5) Drake/Bean/Celeris oracle + listing mechanics. All ~10-min doc checks.

---

## 3. Percolator math → EVM translation deep-dive

Method: cloned both repos, read 100% of EVM port src (1,444 LOC), spec v16.9.0 (1,685 lines), spec v12.19.13 (from git history), Rust engines (current 16.8k + v12-era 7.2k LOC). **Compiled and ran the full EVM port test suite locally.**

### 3.1 Critical framing discovery

**uniperpapp port is NOT a port of current spec v16.9.0.** Its formulas match the **pre-v16 "slab" spec v12.19.13** (commit `fe1d2d81f3`) and old `percolator.rs`. v16.9.0 (rewritten 2026-05/06) replaced that with "source-domain credit" architecture (liens, backing buckets, insurance reservations, ~265 Kani artifacts) — engine uses different accrual math (K delta without A-scaling, no FUNDING_DEN merge). **Correction: the formulas quoted in `monad-metropolis-percolator-feasibility.md` §3.1 as v16.9.0 are v12-era formulas mislabeled. Target v12.19.13 for the hackathon port** — it's what the EVM port implements, spec'd, tested, audited in its era. Don't attempt v16's lien machinery.

### 3.2 What the EVM port has (verified by running it)

| File | LOC | Component | Notes |
|---|---|---|---|
| Constants.sol | 34 | Spec hard bounds, scales | `POS_SCALE=1e6`, `ADL_ONE=1e15`, `FUNDING_DEN=1e9`, `MAX_ORACLE_PRICE=1e12` |
| Types.sol | 86 | Packed `Globals` + Account mapping | `SideState{k:int256, fNum:int256}` |
| FixedPointMath.sol | 98 | Rust wide_math replacement | 512-bit mulDivDown/Up + `divFloorSigned` (floor-to-−∞, matches Rust exactly) |
| Accrual.sol | 110 | Spec §5.3 accrue + §1.7 staircase | Per-second re-expression of per-slot budgets |
| Settlement.sol | 76 | Effective pos + pnl_delta | Exact v12 formulas |
| RiskEngine.sol | 152 | H haircut, equity lanes, risk notional, margin | `h = min(Residual, maturedPosTot)/maturedPosTot` |
| SolvencyProof.sol | 284 | §1.6 solvency envelope | Faithful port of Rust `validate_exact_solvency_envelope`; bounded bisection ≤96 intervals |
| PerpMarket.sol | 309 | Custody, config validation | **trade/liquidate/accrual/touch = stubs** (273-308) |
| PerpFactory.sol | 52 | EIP-1167 factory | Scaffold only, `createMarket` reverts |
| adapters/ | 100 | PushOracleAdapter, DefaultMatcher | Raw push, no clamp wired |

Stack: Solidity 0.8.28, via-IR, evm_version=cancun, **MIT license**. Status: **compiles clean, 90/90 tests pass** (needs Foundry ≥1.x; local forge 0.2.0 choked on foundry.toml fmt section). Measured: `initialize` with solvency validation ~1.77M gas on full-bisection reject path; DESIGN.md claims 6k–15k pass path.

**Red flags:** (1) milestone inflation — accrual/settlement exist as pure libs but not wired (`_accrueMarket()`, `_touch()` are no-ops); (2) staircase has no caller, oracle adapter doesn't clamp; (3) per-second granularity caveat — on Monad 0.4s blocks, consecutive blocks share timestamp (dt=0), 1 bps/s = min nonzero rate → needs e9-scaled rate change (DESIGN.md's own recommendation, mandatory for Monad); (4) token↔engine unit normalization unresolved (uint128 wei caps); (5) missing: min_funding_lifetime bound, OI symmetry machinery, **warmup admission gating (MUST for public wrappers — v12.19 §0 req 4, §9 req 2)**, trade-slippage neutralization; (6) unmaintained since 2026-06-01. **Verdict: mine the ~830 LOC math libs (MIT), don't finish their code — but verify each lib against Rust vectors independently.**

### 3.3 Exact formulas for EVM implementation (from spec v12.19.13)

**H haircut / fair exits (spec §3):**
```
Residual = V - (C_tot + I)
PosPNL_i = max(PNL_i, 0);  ReleasedPos_i = PosPNL_i - R_i   (Live)
if PNL_matured_pos_tot == 0: h = (1, 1)
else h = (min(Residual, PNL_matured_pos_tot), PNL_matured_pos_tot)
PNL_eff_matured_i = floor(ReleasedPos_i * h.num / h.den)

Eq_withdraw_raw_i = C_i + min(PNL_i,0) + PNL_eff_matured_i - FeeDebt_i
Eq_trade_raw_i    = C_i + min(PNL_i,0) + PNL_eff_trade_i   - FeeDebt_i
Eq_maint_raw_i    = C_i + PNL_i                            - FeeDebt_i
Eq_net_i          = max(0, Eq_maint_raw_i)
```

**A/K/F effective position (spec §5.1):**
```
if epoch_snap_i != epoch_s: effective_pos_q(i) = 0
else effective_abs_pos_q = floor(abs(basis_pos_q_i) * A_s / a_basis_i)
effective_pos_q = sign(basis_pos_q_i) * effective_abs_pos_q
```

**A/K/F pnl delta (spec §5.2; Rust `compute_kf_pnl_delta`):**
```
den = a_basis_i * POS_SCALE
combined = (K_now - k_snap) * FUNDING_DEN + (F_now - f_snap)
pnl_delta = floor_to_neg_inf(|basis| * |combined| / (den * FUNDING_DEN))  with sign of combined
```
Floor-to-−∞ on negative numerators is **load-bearing** (losses round against the account). Solidity `/` truncates toward zero → `divFloorSigned` required. Rust `mag = q+1 if rem≠0` reproduced exactly.

**Accrual (spec §5.3; `accrue_market_to`):**
```
funding_active     = rate≠0 && OI_long≠0 && OI_short≠0 && fund_px_last>0
price_move_active  = P_last>0 && oracle≠P_last && (OI_long≠0 || OI_short≠0)
if either: require dt <= cfg_max_accrual_dt_slots
if price_move_active, BEFORE any mutation:
    abs(oracle - P_last) * 10_000 <= cfg_max_price_move_bps_per_slot * dt * P_last
    (fires at dt==0 too — closes same-slot mark-through bypass)
ΔP = oracle - P_last
if OI_long  > 0: K_long  += A_long  * ΔP
if OI_short > 0: K_short -= A_short * ΔP
fund_num_total = fund_px_last * funding_rate_e9_per_slot * dt
F_long_num  -= A_long  * fund_num_total
F_short_num += A_short * fund_num_total
```

**Staircase / wrapper clamp (spec §1.7):** `next_price = clamp_toward(P_last, target, floor(P_last * max_move_bps_per_slot * dt / 10_000))`. Wrapper MUST-rule: raw target stored separately from effective price; risk-increasing user ops rejected or dual-price shadow-checked while `target ≠ P_last`.

**Quantity ADL / K-shift (spec §5.4):**
```
delta_K_abs = ceil(D_rem * A_old * POS_SCALE / OI_before)   // applied to opposing side's K
OI_post = OI_before - q_close_q
A_candidate = floor(A_old * OI_post / OI_before)
// A_candidate==0 with OI_post>0 → zero both OI sides + schedule both resets
```

**Margin/liquidation (spec §7):**
```
RiskNotional_i = ceil(abs(effective_pos_q(i)) * oracle_price / POS_SCALE)   // ceil load-bearing
MM_req_i = 0 if flat else max(floor(RiskNotional_i * maint_bps/10_000), min_nonzero_mm)
Liquidatable iff nonzero eff position and Eq_net_i <= MM_req_i (after authoritative touch)
```

**Solvency envelope / budgets (spec §1.6 — mandatory init-time proof):**
```
price_budget_bps  = max_price_move_bps_per_slot * max_accrual_dt_slots
funding_budget_num = max_abs_funding_e9_per_slot * max_accrual_dt_slots * 10_000
loss_budget_num = price_budget_bps * FUNDING_DEN + funding_budget_num
for every 1 <= N <= MAX_ACCOUNT_NOTIONAL (=1e20):
    price_funding_loss_N = ceil(N * loss_budget_num / (10_000 * FUNDING_DEN))
    worst_liq_notional_N = ceil(N * (10_000 + price_budget_bps) / 10_000)
    liq_fee_N = min(max(liq_fee_raw_N, min_liq_abs), liq_fee_cap)
    mm_req_N = max(floor(N * maint_bps / 10_000), min_nonzero_mm)
    require price_funding_loss_N + liq_fee_N <= mm_req_N
```
Proven ∀N via analytic decomposition + bounded bisection — **the port already carries this (`SolvencyProof.sol`) — reuse, re-parameterize per-second for Monad.**

**Crank (spec §8.7):** keeper_crank accrues exactly once; Phase 1 = keeper-supplied liquidation candidates (bounded `max_revalidations`); Phase 2 = round-robin touch sweep from persistent cursor. Spec §0 req 35: "any state that only a privileged actor can advance is non-compliant"; req 34: "public instructions MUST NOT scan all accounts."

**Global invariants (spec §5 — encode as Foundry invariants):**
```
C_tot <= V <= MAX_VAULT_TLV;  I <= V;  V >= C_tot + I
0 < effective_price <= MAX_ORACLE_PRICE
0 < A_side <= ADL_ONE;  A_side >= MIN_A_SIDE when current-epoch positions exist
0 <= OI_eff_side <= MAX_OI_SIDE_Q;  if Live: OI_eff_long == OI_eff_short
abs(K_side) + A_side * MAX_ORACLE_PRICE <= int256::MAX (was i128 on Solana)
```

### 3.4 Translation plan (minimal faithful port, Monad)

**Storage:** packed `Globals` (vault, cTot, insurance, pnlPosTot, maturedPosTot, 2× SideState {a,k,fNum,epoch,oiEffQ,posCount}, pLast/fundPxLast/slotLast/mode) + `mapping(uint256→Account)`. Price move mutates Globals only — **no account iteration ever** (req 34). One market = one EIP-1167 clone (~50k gas, trivial). Add: Pyth pull feed address, spread bps, minTradeSize, heartbeat cursor.

**Fixed-point: keep custom scales, do NOT use wad.** Wad (1e18) would invalidate every bound and proof — the scales are expressed in the §1.6 proof and A/K/F rounding parity with Rust. int256 gives ~1e40× the headroom the spec's i128 bounds assumed (never binds). Wad only at the token boundary: prices e6, collateral raw ERC-20 wei; one 512-bit mulDiv at deposit/withdraw/trade-fee boundaries; PnL stays in quote atoms via `basis·ΔP/POS_SCALE`.

**Gas on Monad ($2.5e-9/gas, gas billed on LIMIT, reverts pay):**

| Op | Gas | Cost |
|---|---|---|
| Lazy `accrueMarket()` in user tx | 15–25k | $0.00004–0.00006 |
| + Pyth pull update in-tx | +30–60k | +$0.000075–0.00015 |
| No-op accrual (price==pLast, rate==0) | 5–10k | $0.000013–0.000025 |
| Open/close position | 80–150k | $0.0002–0.00038 |
| `liquidate()` full close | 150–300k | $0.00038–0.00075 |
| Keeper heartbeat | 20–40k | ~$11–22/day/market at every-block; ~$0.07–0.14/day at 1/min |
| Market creation (clone + init + proof) | ~2–3M | ~$0.005–0.0075 one-time |

Liquidation profitable for any position with maintenance deficit > ~$0.001. Per-block funding crank economically free; accrual happens lazily in every trade/liquidate tx anyway — heartbeat only prevents `slotLast` falling > `maxAccrualDtSec` behind.

### 3.5 Mandatory vs cut

**Mandatory (solvency core):**
1. `V ≥ C_tot + I` conservation asserted after every mutation (rounding residue → explicit sink).
2. H haircut with matured-PnL gating + three equity lanes.
3. Lazy A/K/F with exact floor-to-−∞ settlement (the differentiator).
4. Risk-notional ceil + MM/IM + strict liquidatable (`Eq_net ≤ MM_req`).
5. Per-step envelope checked BEFORE mutation + maxAccrualDtSec window + staircase clamp.
6. Init-time §1.6 solvency proof (reuse SolvencyProof.sol, per-second budgets).
7. **Warmup admission with `admit_h_min > 0`** — without it, H haircut defeatable by oracle-manipulate-and-withdraw. Two-bucket reserve ~120 LOC.
8. Permissionless liquidation + quantity-ADL K-shift + OI symmetry maintenance (`OI_long == OI_short` — the bookkeeping bug factory).
9. Dual-price shadow rule while `target ≠ P_last` (reject risk-increasing trades).

**Cut (document as simplifications):** v16 source-credit model (entire §2 — biggest cut, ~4–6k LOC equivalent); stale-account certificates; resolved payout ledger (simplify to privileged resolve at last effective price); DrainOnly/ResetPending state machine (collapse to drain flag); hedge credit, dead-leg forfeit, recurring fees, reclaim machinery; ERC-4626 LP vault (single LP + simple share ledger); multi-asset portfolios (N=1).

### 3.6 Module breakdown

| Module | LOC | Status |
|---|---|---|
| Constants + Types + FixedPointMath | 220 | **Reuse as-is** (verify divFloorSigned vs Rust vectors) |
| RiskEngine + Settlement + PerpMath | 350 | **Reuse** — audit 5 gaps in §3.2 first |
| Accrual (incl. staircase wiring) | 150 | Exists as lib; wire into market + oracle adapter |
| SolvencyProof (per-second re-derivation) | 300 | **Reuse**; re-run with Monad params |
| OracleAdapter (Pyth pull + staleness + clamp) | 150 | New — Pyth live on Monad mainnet |
| PerpMarket: custody + trade + warmup + liquidate + ADL | 900–1,100 | New — the core build; M1 custody ~200 LOC reusable |
| Vault/insurance (simple LP + I ledger) | 250 | Simplified |
| PerpFactory (clone + init) | 80 | Finish the stub |
| Keeper bot (TS/Python, tight gasLimit) | 400–600 off-chain | New |
| **Total** | **~2,400–2,700** | ≈1,300–1,500 new + ~1,100 reused |

### 3.7 Testing strategy (Foundry as Kani substitute)

Kani's ~265 proofs do NOT transfer. Substitute: fuzz + invariant handlers + **differential tests vs Rust engine** (both engines deterministic pure math — extract vector pairs via tiny Rust harness → JSON → Foundry unit tests; highest-signal test possible).

Invariants to encode: (1) `vault ≥ cTot + insurance` after every handler; (2) `OI_eff_long == OI_eff_short` after every side-mutating op; (3) `h ∈ [0,1]`, effective matured payout never exceeds released; (4) `pnlPosTot == Σ max(PNL_i,0)` on touched subset (targeted aggregate proofs — full scan banned); (5) price cap fires BEFORE any mutation (wild price → assert no partial state on revert); (6) `0 < A_side ≤ ADL_ONE`; (7) ADL conserves quantity; (8) rounding direction: kfPnlDelta matches Rust floor-to-−∞ bit-for-bit on random tuples (differential); (9) solvency envelope: configs that pass validate hold at sampled breakpoints, failing configs fail; (10) no-op invariance: accrual with unchanged price/zero funding leaves settled equity unchanged. Plus reentrancy tests on every payout path (EVM-only threat class), foT deposit test, gas snapshots with explicit gasLimit assertions (Monad billing quirk).

### 3.8 Verdict

**Yes — 2 devs can port the core faithfully in 2–3 weeks** starting from the port's math libs (830 LOC, compiles, 90/90 tests, carries the hardest artifact: the §1.6 solvency proof). New build ≈1,300–1,500 LOC: trade path, warmup, liquidation+ADL, oracle wiring, factory, tests. 6-week window leaves real slack for keeper bot, dashboard (Perpl bounty), demo.

**Hardest 20% (where weeks 2–3 go):**
1. Warmup admission + reserve-bucket lifecycle interacting with H — not hard math, hard bookkeeping; aggregate invariants fail silently, then haircut pays wrong amounts.
2. Liquidation + quantity-ADL + resets — the only place OI sides legitimately diverge and must re-converge.
3. Trade path's slippage neutralization (`Eq_trade_open_raw`) — without it, trader manufactures positive PnL on the fill to pass IM. Real solvency hole, subtle.
4. Oracle stack: Pyth pull in-tx, staleness, staircase clamp, dual-price shadow while lagged — integration risk > math risk.
5. Verification layer — budget a full week; it makes "faithful port" defensible to judges.

**Process notes:** (a) pin port target to **spec v12.19.13**, not v16.9.0; (b) mine MIT port for math only, verify each lib against Rust engine independently (port pre-alpha, unmaintained).

---

## 4. Metropolis criteria alignment + MVP positioning

### 4.1 Track 01 criteria (exact)

- Track 01 "Onchain Finance & Trading" — **$30,000**. Description: *"Fast settlement makes new financial instruments and fully onchain markets more practical."*
- Best fit: *"Teams who have shipped a trading, lending, or market-making product before."*
- Suggested builds: (1) fully onchain order books without offchain matching engine, (2) **perps with funding that updates every block**, (3) undercollateralised lending priced on onchain credit history.
- **No track-specific judging rubric published.** Pool: $30k × 4 tracks + $25k grand champion; bounties stack.
- **Hard criterion: judges verify work was built during the 6 weeks** → public repo with in-window commits, live testnet (+mainnet) deployment, video showing onchain txns.
- Deliverable: working product + public profile (demo, short write-up, code link). Open source "encouraged, not required."

### 4.2 Competitive gap analysis (Monad perp incumbents)

| Feature | Perpl (live $20M+/day) | Drake (50x hybrid) | Bean (vault perps) | Orbs/Atlantis Hub Ultra | GAP on Monad today |
|---|---|---|---|---|---|
| Permissionless market creation | No — curated 6 markets, no public listing process | No | No — curated vault markets | No — team-deployed, L3-assisted | **REAL. Nobody offers it.** Proof-of-mechanic: percolator-launch (51 devnet markets) |
| Margin model | Isolated only | Cross + isolated, multi-collateral | Cross, multi-asset | n/a | Not a gap. Don't lead with it |
| Funding granularity | Block-number-scheduled, **~hourly** (every 8,571 blocks) | Standard + frUSDC vault | Standard | n/a | **REAL.** Nobody accrues per block (0.3–0.4s). Lazy F index = O(1), $0 idle |
| Public solvency observability (ψ-style) | Insurance fund + ADL + rate-limits, no public formal metric | ADL + OI caps, opaque | Vault ratios, opaque | n/a | **REAL.** Nobody publishes live solvency invariant |
| Profit-haircut fair exit vs forced ADL | Forced ADL + insurance fund | Forced ADL | Vault backstop | n/a | **REAL.** Nobody has H-haircut "junior PnL" exit |
| Fully-onchain L1 execution | Yes (CLOB on chain) | Yes | Yes | **No** — L3 aggregation | Table stakes; foil: "no offchain matching, no L3" |

**Gap real or hype?** Mechanism gap real + verified: no Monad venue allows permissionless perp creation; spot is permissionless (Kuru public `deployMarket()`) but perps universally curated. Analogues: HIP-1 (own L1), percolator-launch (Solana devnet, 51 markets). **Demand side = honest risk**: long-tail perps without liquidity = cold start; only tokens with Pyth/Chainlink feed can be listed (oracle moat replaces listing moat). Pitch = "first to remove the listing moat + formal fair-exit risk"; demo de-risks demand by live-trading a real long-tail Monad token.

### 4.3 MVP feature spec (6 weeks)

**Working title: "Percolator.Monad — permissionless perps with per-block funding and provable solvency."** Solidity 0.8 + Foundry, EIP-1167 clone per market (per-market isolation → parallel execution across markets).

**In (core):**
1. **Permissionless market factory** — `deployMarket(feedId, fundingParams, envelopeBps, seedAmount)`; creator seeds backstop vault. No whitelist. Headline differentiator — first 15 seconds of video.
2. **Ported Percolator risk math** (spec v16.9.0 → Solidity; mine uniperpapp DESIGN.md after license check): A index (coverage scaling), K (coverage accumulator), **F lazy funding index — accrues per block on any touching tx** (the literal suggested build), **H fair-exit haircut on profit withdrawals** when coverage < target, solvency invariant `V >= C_tot + I`, bounded price envelope (per-block move bps × max accrual slots, fail closed), maintenance margin, PnL warmup.
3. **Execution** — oracle-mark market orders (Pyth pull + confidence/staleness bounds; Chainlink Data Streams adapter = second oracle path, CRE bounty hook). Optional thin post-only book only if Week 3 gate passes (floor = oracle-mark engine).
4. **Permissionless liquidations** — keeper crank, bounded loop + cursor + liq bonus; lazy accrual (trader pays own accrual; idle markets cost $0). Keeper bot with tight explicit gas limits (Monad bills gas_limit).
5. **Public solvency observability** — on-chain ψ(t) coverage ratio crank-updated per block; live dashboard (doubles as Perpl $3k risk tool).
6. **Trading UI** — consumer-grade: create market → seed → trade → watch per-block funding tick → trigger liquidation → watch H haircut.
7. **Proof-of-correctness pack** — Foundry invariant tests (solvency under fuzz; A/K/F order-independence), optional differential test vs `percolator-engine` Rust crate. Gas audit.

**Out:** CLOB v1, cross-margin legs, LP yield vault, DPMM, governance, cross-chain, TWAP/limit orders, mobile app. Say so in write-up.

**90-second demo:** (0–10s) block counter + F index accruing per block — "funding every block, not hourly." (10–30s) deploy market for long-tail token Perpl/Drake don't list; seed vault. (30–55s) trade; simulated oracle jump → envelope fails closed, zero bad fills. (55–75s) keeper liquidates on camera; ψ public; profit withdrawal during stress pays H haircut — no ADL surprise. (75–90s) "Solvency invariant `V >= C_tot + I` verifiable every block."

### 4.4 Judge-objection matrix

| Objection | Counter |
|---|---|
| Oracle manipulation on small caps | Envelope fails closed on per-block move beyond bps budget; Pyth confidence + staleness; OI caps scale with seeded vault ($100 seed can't back $1M OI); honest framing — "we remove the listing moat, not the price-feed moat." v2: Kuru MON-USD book mid as mark input for thin-oracle tokens |
| No liquidity cold start | Creator-seeded backstop vault + bootstrap mode (vault-backed execution, Bean's model); non-zero funding attracts capture-arb automatically. Honest: "liquidity attraction = business problem; mechanism = the contribution" |
| "Percolator not production ready / just a port" | Not shipping Percolator. Novelty = (1) first EVM realization with public per-block solvency invariant, (2) permissionless factory, (3) per-block lazy funding on 0.4s blocks. Disclose audit status; our invariant tests are the evidence |
| Crowded lane | Incumbents curated, ADL-based, AUSD-only or L3-assisted. Adjacent, not head-on. Video shows 3 things none can demo |
| "Why not build on Perpl API?" | Both lanes: $3k risk tool reads Perpl API; core product needs permissionless markets + per-block funding — Perpl's architecture doesn't offer either (hourly funding) |
| "Per-block funding = gimmick" | It's the track's own suggested build; ~$0.0001/crank or $0 lazy. Benefits: no keeper-clock trust, exact PnL attribution, funding-arb granularity. Show gas numbers on screen |
| "Built during the window?" | Public repo with in-window commits, testnet + mainnet deploys, video over live testnet |
| "Solidity port already exists" | uniperpapp port: pre-alpha, unmaintained, missing execution/oracle/liquidation/LP, no factory, no per-block funding. Mine math (after license check), don't reuse code |

### 4.5 Bounty prioritization (same codebase, ascending effort)

1. **Perpl $3k risk/analytics tool** — ψ/A/K/F dashboard already built for own protocol; add Perpl API data source. ~2–3 days.
2. **Envio $1k** — index our events with HyperIndex for dashboard. ~1–2 days.
3. **Chainlink $3k CRE** — Data Streams as second oracle path for one market (Drake already uses it on Monad). Medium effort; confirm "CRE" scope on platform.
4. **Perpl $5k API use** — ψ-driven funding-capture/arb tool trading through Perpl API (perpl-sdk). ~1 week after $3k dashboard lands.
5. **Kuru $5k new assets/markets** — opportunistic: deploy spot market for demo token (~30 min via SDK) + use Kuru MON-USD book mid as mark input to make it substantive.
6. **Kuru $5k consumer app** — only if UI genuinely on Kuru SDK; else skip.
7. **Aurora $5k / Agora $10k×2 / Nansen / Alchemy** — don't chase. Use AUSD collateral (free Agora narrative).

**Open items before Week 2:** (1) exact Perpl/Kuru bounty criteria (unpublished — ask on platform/Discord), (2) verify uniperpapp license before mining math.

---

## 5. Synthesis — architecture proposal + build plan

### 5.1 Feasibility verdict

**Feasible. Easiest faithful path = port Percolator spec v12.19.13 (not v16.9.0) to Solidity, starting from uniperpapp's MIT math libs (~830 LOC verified working).** Core ≈1,300–1,500 new LOC + ~1,100 reused. 2 devs, ~3 weeks core + 1 week verification + 2 weeks product/polish.

**"Permissionless perp for any token" honest scope:** any token WITH a live Monad oracle feed (~60 Pyth push feeds + pull catalog + Chainlink where available). Oracle = the real gate, not listing mechanics. That's the pitch: remove the listing moat, keep the price-feed moat, and let Percolator's envelope + warmup + H haircut + OI-cap-from-seed make it safe without governance or HIP-3's $20M stake.

### 5.2 Proposed architecture

```
PerpFactory (EIP-1167 clones, permissionless)
  └─ PerpMarket per token (packed Globals: A/K/F/H/envelope/ψ)
       ├─ OracleAdapter: Pyth pull (primary) + Chainlink Data Streams (secondary, MON)
       │    staleness + confidence + staircase clamp + dual-price shadow rule
       ├─ RiskEngine: v12.19 H haircut, 3 equity lanes, risk-notional ceil, MM/IM
       ├─ Accrual: lazy A/K/F, per-block granularity on any touching tx (O(1))
       ├─ Warmup: two-bucket reserve, admit_h_min > 0 (mandatory, spec §9 req 2)
       ├─ Liquidate: permissionless, quantity-ADL K-shift, OI symmetry maintenance
       ├─ Insurance: creator-seeded backstop, OI cap ∝ seed (Design A bounded-loss economics)
       └─ ψ observability: public coverage ratio crank-updated per block
Keeper bot (TS): Phase-1 liquidations + heartbeat, tight explicit gasLimits
Dashboard: live ψ/A/K/F + funding tick → Perpl $3k risk-tool bounty (Perpl API data source)
```

Execution model: oracle-mark market orders (floor), optional thin post-only book (upside, Week 3 gate). No forced ADL — H haircut fair exits. Fail-closed envelope on any per-block move beyond bps budget.

### 5.3 Why this wins Metropolis Track 01

- **Matches suggested build verbatim**: perps with funding updating every block (lazy F index, per-block granularity on 0.3–0.4s blocks — incumbents do hourly scheduled funding).
- **Unique gap on Monad**: permissionless market creation — nobody offers it (all incumbents curated). Proof-of-mechanic exists (percolator-launch: 51 devnet markets).
- **Unique risk story**: public per-block solvency invariant `V ≥ C_tot + I` + ψ coverage ratio + H fair-exit haircut vs incumbents' opaque forced ADL. Formal lineage: Toly's Percolator, Apache-2.0, Kani-proven in Rust, our Foundry invariants + differential tests in Solidity.
- **Why Monad**: per-block funding economically trivial here ($11–22/day/market active crank, $0 lazy); envelope math gets better on sub-second blocks; gas-limit billing handled from day one.
- **Bounty stack (same codebase)**: Perpl $3k risk tool → Envio $1k → Chainlink $3k (Data Streams adapter) → Perpl $5k API use (funding-capture tool) → Kuru $5k new markets (spot listing + book-mid as mark input).

### 5.4 6-week plan

| Week | Scope |
|---|---|
| 1 | Pin spec v12.19.13; verify port math libs vs Rust vectors (differential harness); Foundry scaffold; hello-world on Monad testnet; read Monad gas docs; register hackathon + confirm Perpl/Kuru bounty criteria |
| 2 | Wire Accrual + staircase + Pyth adapter into market; per-second budgets re-derived for Monad; SolvencyProof re-run; Foundry invariant handlers (solvency, OI symmetry, rounding) |
| 3 | Trade path (slippage neutralization) + warmup admission + custody; fuzz the fill edges |
| 4 | Liquidate + quantity-ADL + resets + insurance/OI-cap-from-seed + ψ observability field |
| 5 | Factory permissionless createMarket + keeper bot (tight gasLimits) + dashboard; deploy testnet + mainnet; Envio indexing |
| 6 | Invariant-test pass + gas audit + differential test suite final; demo video; write-up (disclose: v12 lineage, audit status, cut features); submission |

**Fallback triggers:** Week 2 gate — math libs fail differential vs Rust → port math from spec directly (adds ~1 week, still feasible). Week 3 gate — trade path + warmup not solid → drop book/extra oracle, ship oracle-mark-only + stronger dashboard story.

### 5.5 Open questions to close before building

1. Perpl/Kuru bounty exact criteria (unpublished — ask on platform/Discord).
2. uniperpapp port license = MIT (verified in foundry config) — double-check no copy-left deps (OpenZeppelin MIT, forge-std — fine).
3. Pyth pull catalog consumable on Monad beyond the 60 push feeds? Verify with Pyth Monad docs.
4. Chainlink Data Streams breadth on Monad beyond MON/USD (for the CRE bounty hook).
5. Collateral choice: AUSD (18dp, Agora narrative, Perpl parity) vs USDC (6dp). Pin before Week 3 (token↔engine normalization).
6. Zaros license before mining structure patterns [unverified].

---

## Sources

- PerpFinder Monad: https://perpfinder.com/chains/monad
- Perpl docs: https://docs.perpl.xyz/ · funding: https://docs.perpl.xyz/exchange/funding · SDK: https://docs.perpl.xyz/resources/for-developers/sdk/install · price indices: https://docs.perpl.xyz/exchange/price-indices
- PerplFoundation dex-sdk: https://github.com/PerplFoundation/dex-sdk
- Zaros audited repos: https://github.com/Cyfrin/2024-07-zaros · https://github.com/Cyfrin/2025-01-zaros-part-2 · CodeHawks: https://codehawks.cyfrin.io/c/2024-07-zaros
- BeanExchange org: https://github.com/BeanExchange · Bean docs: https://docs.bean.exchange/
- Drake: https://docs.drake.exchange/ · analysis: https://hakresearch.com/drake-exchange-la-gi-perp-dex-onchain-kieu-cex-tren-monad
- LeverUp: https://leverup.gitbook.io/docs
- Orbs × Atlantis: https://www.orbs.com/Atlantis-Brings-Onchain-Perpetuals-to-Monad-via-Orbs-Perpetual-Hub/ · Symmio license: https://docs.symm.io/legal-and-brand/terms-of-service-and-licensing/contract-license
- Monday Trade: https://blog.synfutures.com/monday-trade-synfutures-builder-program-monad-mainnet/
- Pingu shutdown: https://docs.pingu.exchange/pingu-exchange-docs · OBSDN: https://defillama.com/protocol/obsdn
- Pyth on Monad: https://github.com/monad-crypto/protocols/blob/main/mainnet/pyth.jsonc · https://docs.pyth.network/price-feeds/core/contract-addresses/evm
- Chainlink on Monad: https://monad-docs.vercel.app/tooling-and-infra/oracles
- Monad gas: https://docs.monad.xyz/developer-essentials/gas-pricing
- Metropolis: https://www.monad.xyz/developers/hackathons/metropolis · platform: https://hackathon.monad.xyz/
- Percolator: https://github.com/aeyakovenko/percolator · EVM port: https://github.com/uniperpapp/Percolator-Ethereum · audit: https://github.com/Copenhagen0x/percolator-audit-2026-04 · percolator-launch: https://github.com/dcccrypto/percolator-launch · perp-liquidity design A: https://github.com/dcccrypto/percolator-perp-liquidity · threat model: https://github.com/dcccrypto/percolator/blob/main/THREAT_MODEL.md
- Hyperliquid listing: https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/deploying-hip-1-and-hip-2-assets · HIP-3: https://cryptoslate.com/anyone-can-now-create-hyperliquid-perp-contracts-with-20m-is-defi-about-to-break/
- dYdX listing widget: https://www.dydx.foundation/blog/market-listing-widget-tutorial · GMX listing framework: https://gov.gmx.io/t/new-assets-listing-framework/3660/2
- Drift: https://github.com/orgs/drift-labs/discussions/631 · risk params: https://drift-protocol-v2-docs.vercel.app/protocol/risk-and-safety/risk-parameters
