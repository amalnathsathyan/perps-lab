# Monad Metropolis × Percolator — Feasibility Study

**Date:** 2026-09-02
**Status:** In progress — section 4 complete, sections 1–3 pending agent results

## Context

- Hackathon: [Monad Metropolis](https://www.monad.xyz/developers/hackathons/metropolis), online, 1 Sep – 13 Oct 2026, $250k+ pool
- Track 01: Onchain Finance & Trading, **$30,000**, judges incl. Keone Hon, Galaxy, Electric Capital, Paradigm, CoinFund
- Rule: work shown 13 Oct must be built during the 6 weeks. Existing Solana research repo counts as research only.
- Suggested builds (Track 01):
  1. Fully onchain order books with no offchain matching engine (sponsor: Kuru — $5k consumer trading app, $5k new assets/markets)
  2. Perpetuals with funding that updates every block (sponsor: Perpl — $5k API use, $3k analytics/risk tool)
  3. Undercollateralised lending priced on onchain credit history (sponsor: Agora — $10k mobile trading, $10k cross-border payments; plus Envio indexer bounty, Aurora Intents $5k)

## Key framing fact

Monad = EVM bytecode-compatible L1 (Cancun, Solidity + Foundry/Hardhat), mainnet live since 24 Nov 2025 (chainId 143). [Mainnet launch](https://coinpaprika.com/news/monad-mainnet-launch-evm-10000-tps-2025/), [deep dive](https://chainstack.com/deep-dive-into-monad-speed-performance-and-decentralization/). **Percolator's Rust/Anchor/Solana code transfers 0% — only design concepts transfer.**

---

## 1. Fully onchain CLOB + Percolator (Kuru)

### 1.1 Kuru facts

**Kuru = fully on-chain spot CLOB + integrated AMM backstop on Monad.** [Docs](https://docs.kuru.io/index). Every market pairs an OrderBook with discretized AMM liquidity (CPAMM discretized into book price points); liquidity types: CLOB, AMM, CLMM, strategy vaults. All state in contract storage.

- **Matching fully on-chain**: takers "crank" maker orders; match-against-resting-first, price-time priority. Orders = linked lists per price level (`Order {uint40 prev, next}`, `PricePoint {head, tail}`, `TreeUint24` active price points). Order types: GTC limit, post-only, market (IOC/FOK), flip orders, batch cancel. [OrderBook docs](https://docs.kuru.io/contracts/OrderBook)
- **MarginAccount is a misnomer**: central accounting ledger (`debitUser`/`creditUser`/`creditFee`), gated by `verifiedMarket` allowlist. **No leverage, no collateral ratios, no liquidations.** Spot only. No perps roadmap found. [MarginAccount docs](https://docs.kuru.io/contracts/MarginAccount)
- **Extension points (full inventory)**:
  1. Permissionless market creation — YES: `MonadDeployer.deployTokenAndMarket()`, `ParamCreator.deployMarket()` (public, no whitelist). [deploy-market SDK](https://docs.kuru.io/sdk/deploy-market)
  2. `verifiedMarket` allowlist — only seam resembling third-party settlement, but registering a custom market type with custom risk logic is NOT documented. Needs Kuru confirmation.
  3. **NO fill hooks/callbacks.** Integration = event indexing only (`MarketRegistered`, `Trade`). [Integration docs](https://docs.kuru.io/contracts/Integration)
  4. Read-only surface: `bestBidAsk()`, `getL2Book()`, `getMarketParams()` — enough to observe the book, not to enforce risk on fills.
- **Deployment status**: mainnet live. Router `0xd651346d...`, MarginAccount `0x2A68ba18...`. Official markets: MON-AUSD, MON-USDC. $11.6M Series A led by Paradigm. [Addresses](https://docs.kuru.io/contracts/Contract-addresses), [raise](https://blog.kuru.io/p/kuru-labs-raises-116m-led-by-paradigm)
- **Bounties**: titles only, no public criteria — "Build the Next Consumer Trading App on Kuru" ($5k), "Bring New Assets and Markets to Kuru" ($5k). Platform login-gated. [Hackathon](https://hackathon.monad.xyz/)
- **Gas** (from [kuru-sdk-py](https://github.com/Kuru-Labs/kuru-sdk-py)): place ≈160k, cancel ≈44k gas. Empirical MM heuristic: `gas = 193k + 159k×n_buys + 160k×n_sells + 44k×n_cancels`. Recommended MM cadence 5–15 s.

**Monad perp/CLOB landscape (crowded):** Perpl (live CLOB perps, $20M+/day July 2026, Dragonfly $9.25M), Drake (CLOB+AMM perps, 50x, frUSDC funding vault, testnet), Celeris (PLOB shared-liquidity perps testnet), Clober. Category proven; lane crowded.

### 1.2 Monad gas economics (matters for all options)

- Mainnet since 24 Nov 2025. Block ~0.4 s, finality ~0.8 s. Block gas limit 200M, **per-tx gas limit 30M**. Real throughput 2–3k TPS.
- **Critical quirk: Monad charges the gas LIMIT, not gas used** — keepers/traders must set explicit tight gas limits. [Gas docs](https://docs.monad.xyz/developer-essentials/gas-pricing)
- EIP-1559, min base fee 100 gwei. MON ≈ $0.025 → ~$2.5e-9/gas. 0.4 s blocks → 216,000 blocks/day.

| Op | Gas limit | Cost |
|---|---|---|
| Kuru limit place | ~160k | ≈ $0.0004 |
| Kuru cancel | ~44k | ≈ $0.00011 |
| MM bot 10 s cadence | 160k × 8,640/day | ≈ $3.5/day |
| **Per-block funding crank (lazy F index)** | 40–60k | ≈ $0.0001–0.00015/block → **~$22–32/day/market**; or $0 — F accrues lazily on any trade/liquidate tx |
| Liquidation tx | 200–350k | ≈ $0.0005–0.0009 (profitable on positions >$0.05–0.1 collateral) |
| Max tx | 30M | ≈ $0.0075 (hard cap on matching-loop iterations) |

Per-block funding accrual on an order-book perp is **economically trivial on Monad**; keeper liquidation economics sound down to dust-sized positions; friction = keeper holds MON for gas + must set explicit limits.

### 1.3 Integration options ranked

**Option B — Own minimal on-chain perp CLOB with Percolator logic ported to Solidity. RANK 1.**
Single-market order book (Kuru-style linked lists per price level, price-time priority, post-only makers, gas-bounded taker walks); **every fill runs ported Percolator margin check** (A index scaling, envelope check, maintenance margin); **funding accrues per block via lazy F index** (O(1) accumulator, no per-position loop — Percolator's core trick). Kuru's role: deploy/list a market via SDK (satisfies "New Assets and Markets"), use Kuru MON-USD book mid as a mark input, reuse kuru-sdk-py MM patterns, consumer-app bounty if UI built. ~1.5–2k lines Solidity.

**Option C — Percolator as clearing layer over existing book. RANK 2, blocked.**
Kuru has no fill hooks → would require forking Kuru's OrderBook Solidity and grafting clearing onto matching loop, or undocumented custom market registration. License unverified. On Perpl: fills settle via their API offchain → that's the $3k risk-tool bounty, not a perp product.

**Option A — Kuru spot CLOB + separate perp engine, no real integration. RANK 3.**
Two disconnected products; no "Percolator integrated with CLOB" story; wastes team's risk-engine edge. BUT the perp-engine half alone = lowest-risk build, maps to "per-block funding" suggestion — recommended fallback if book slips.

**Verdict:** (b) — CLOB for perp orders with Percolator margin-on-fill + funding + liquidation crank — realizable only on your OWN book, not on Kuru's.

### 1.4 6-week plan (Option B)

Scope day 0: ONE market (MON-PERP), AUSD or USDC collateral, fixed 5x, post-only MMs + IOC/FOK takers, A/K/F indices, bounded envelope, per-block funding, permissionless liquidator + pro-rata profit-haircut (no forced ADL v1), insurance fund = protocol buffer. Team 2–3 devs.

| Week | Scope |
|---|---|
| 1 | Foundry scaffold + hello-world on Monad testnet. Extract A/K/F invariants from percolator spec.md → Solidity pseudocode (wad/PRBMath fixed-point). Sandbox Kuru SDK, record real gas. Read Monad gas-limit-billing + deferred-execution docs. |
| 2 | Solidity `RiskEngine`: margin math, A index, F index (`accrueFunding()` per block), K index (coverage h), envelope check. Foundry fuzz + invariant tests as Kani substitute. |
| 3 | Order book: linked-list-per-price-level, price-time priority, postOnly, gas-bounded market walks with margin check on every fill, cancel, batchCancel. |
| 4 | Liquidations (permissionless, bounded loop, liq bonus), profit-haircut withdrawal gating, funding crank. Pyth pull or Chainlink Data Streams + staleness guard. TS/Python keeper bot with tight gas limits. |
| 5 | Trading UI (consumer-grade — doubles toward Kuru consumer-app bounty), risk dashboard exposing A/K/F + ψ(t) (doubles toward Perpl $3k risk-tool bounty), MM bot demo. Deploy mainnet + testnet; register market on Kuru mainnet via SDK if pursuing bounty. |
| 6 | Invariant-test pass, gas audit (explicit limits everywhere), demo video, write-up, public repo, submission. |

**Fallback trigger (end Week 3):** book not matching correctly → pivot to oracle-mark perp engine with per-block funding, no CLOB. Keeps "funding every block" build; frees Weeks 4–6 for polish + bounties.

### 1.5 Risks

1. Kuru contract license unverified → confirms/kills fork-based Option C.
2. Kuru bounty criteria unpublished — ask Kuru directly.
3. Custom market registration undocumented — the one question that could turn "no integration" into "real integration." Ask Kuru.
4. Percolator not production-ready by its own label; Apr 2026 audit = 1 active bug + 2 code defects. Port transfers zero formal guarantees — treat as spec, disclose in write-up.
5. EVM-novice team: budget Week 1 fully for ramp. Book state = hottest state (serialized re-execution → less parallel speedup on the book itself).
6. 30M gas/tx cap bounds matching walks — design fill loops with per-iteration budget from day one.
7. Oracle coverage for chosen asset on Monad — verify.
8. Crowded lane (Perpl live $20M+/day, Drake, Celeris, Clober). Differentiation must be loud: per-block funding, coverage-ratio/haircut, A/K/F + ψ observability.
9. MON price volatility (Nov unlock overhang) — keeper cost math robust to ±50% MON moves.
10. Collateral choice AUSD vs USDC — pick one, verify faucet/on-ramp.

### 1.6 Verdict

**Hardest of the three Track 01 suggestions for this team.** Own research (v1 doc) explicitly rejected on-chain CLOB as too complex for MVP; team has zero order-book code, EVM-novice; Perpl/Drake shipped exactly this, well-funded. Kuru bounties are UI/SDK work, not risk-engine work; no fill hooks → Percolator-as-clearing-layer not buildable without fork/undocumented extension. Realistic Kuru integration is shallow: list market, index events, app on SDK.

**Easy high-fit build = suggested build #2 (per-block funding perps)** — almost a literal description of Percolator's lazy F index; v2 wrapper research transfers 1:1; no order book required; per-block funding ≈ $20–30/day/market. ~4-week build + 2 weeks polish.

Strategic rec: enter on build #2 core, add thin post-only book only if Weeks 1–3 go smoothly, harvest adjacent bounties: Kuru consumer-app (UI on top of own engine + Kuru listing), Perpl $3k risk-tool (A/K/F + ψ dashboard on Perpl order flow).

---

## 2. Perps with per-block funding + Percolator (Perpl)

### 2.1 Perpl facts

**Perpl = fully on-chain, isolated-margin perpetual futures DEX on Monad.** Mental model: "Uniswap as a perpetual exchange with a CLOB." Founded by ex-FPGA devs, $9.25M Dragonfly-led raise. [Docs](https://docs.perpl.xyz/), [developer overview](https://docs.perpl.xyz/resources/for-developers/overview), [launch blog](https://blog.perpl.xyz/we-built-the-perp-dex-everyone-said-was-impossible/).

Architecture:
- **On-chain CLOB**: bit-index trees + partition list map. O(1) constant-time post/cancel, price-time priority. Price tree 256³ ≈ 16M price levels. "Change" op replaces cancel+repost in one tx. MM post-and-cancel ≈ 100k gas (vs ~200k Uniswap V2 swap). Claims ~1,000–5,000 post-and-cancels/sec at Monad's 500M gas/s.
- **Margin**: isolated only — each position has own collateral, no cross-margin contagion.
- **Risk stack: dynamic margin engine, on-chain liquidation (partial + full), per-perpetual insurance fund, ADL, withdrawal rate-limiting.** ← This is essentially Percolator's feature set, already built and live.
- **Collateral**: AUSD (Agora Dollar). Mainnet (chain 143) markets: BTC, MON, ETH, SOL, HYPE, ZEC; testnet chain 10143.
- **Interfaces**: REST + WebSocket API (`https://app.perpl.xyz/api`, `wss://app.perpl.xyz`), Rust SDK (`perpl-sdk` crate + `perpl-cli`). Live per-market params via `GET /v1/pub/context`.
- Fun.xyz integration for cross-chain/fiat deposit rails.

**Funding — block-scheduled, the track's exact suggested build already exists here:**
- Funding events scheduled **by block number**, ~hourly: every 8,571 blocks (~0.42s avg consensus). [Funding docs](https://docs.perpl.xyz/exchange/funding)
- Rate settable up to 143 blocks (~1 min) in advance, cannot be set after funding block.
- Methodology: impact-price-difference (sum of `ImpactPriceDifference / OraclePrice` samples over interval) — Hyperliquid/Binance-style, gas-optimized for on-chain.
- Clamped rate (`absFundingClampPctPer100k`, 0–15%), funding price constrained to Chainlink oracle tolerance + staleness checks.
- Docs explicitly note on-chain funding settlement is gas-prohibitive for typical DEXs — hence block scheduling + advance setting.

**Monad perp competitors:** Drake Exchange (hybrid CLOB+AMM, funding + frUSDC funding-rate vault, 50x, gas-free trading), Bean Exchange (perps + DLMM, 50x), Kuru (spot CLOB). MON perps trade on 23 venues, ~$61M OI.

### 2.2 Per-block funding mechanics — honest reading

"Funding that updates every block" — nobody literally writes a global accumulator every block; Perpl's design (block-number-scheduled funding, rate set in advance, hourly application with per-block granularity) is the production-grade reading. Lazy accrual on user interaction (GMX/Drift standard) + block-indexed scheduling is the pattern. True per-block funding = novelty without benefit; the Monad-specific win is block-number-based scheduling instead of keeper clocks.

### 2.3 Percolator integration options (ranked)

**Option 1 — Risk overlay on Perpl API. Rank #1.**
Greenfield perps = rebuilding Perpl (which already has Percolator's entire feature set). Instead: consume Perpl's public API, add Percolator-derived layer:
- **ψ(t) solvency monitor**: compute ξ (long/short OI imbalance) + price path from Perpl API/on-chain state → public pool-health metric, funding-pressure forecaster, ADL early warning. Off-chain calculator + dashboard, or small on-chain registry cranked permissionlessly.
- **DPMM mark analysis**: compare Perpl mark vs DPMM-adjusted price — LP/inventory risk tool.
- **Funding arbitrage tooling**: Perpl funding is block-scheduled → predictable. A ψ-aware funding-capture strategy/tool (Drake already ships frUSDC vault — differentiate via the formal ψ/risk lens).
Fits BOTH Perpl bounties: $5k API use (product actually trades/reads via API) + $3k analytics/risk tool. Lowest build risk: no protocol-level solidity risk engine needed, works even if greenfield scope shrinks.

**Option 2 — Greenfield Solidity perps with A/K/F port. Rank #2.**
Rewrite Percolator's margin/liquidation/ADL in Solidity with block-number-scheduled funding. ~80% of team research transfers (chain-agnostic math), but: Perpl + Drake + Bean already occupy the lane, 6 weeks for a competing engine is tight, and the "why" (judges see another perps DEX, not Percolator's differentiator).

**Option 3 — Both. Rank #3.** Only if greenfield goes smoothly early; overlay-first, engine-second.

### 2.4 6-week plan (Option 1)

Product: "ψ-Lens for Perpl — a Percolator-derived risk observability + funding tool on Perpl's API."

| Week | Scope |
|---|---|
| 1 | Perpl API integration spike: read context, OI long/short, insurance fund, funding schedule, order placement via perpl-sdk (Rust — team's strength). Define ξ, ψ update loop. |
| 2 | ψ(t) monitor backend: ingest price ticks + OI, compute ξ, discretized ψ update; store history. |
| 3 | ADL/insurance-fund analytics: fund coverage ratio, ADL probability flags, funding-pressure forecaster from ψ. Dashboard (public, live, 90-second-demo-friendly). |
| 4 | API-use product: funding-capture bot or one-click funding-arb position tool trading through Perpl API (locks the $5k bounty). |
| 5 | On-chain ψ registry (optional small contract, keeper-cranked) + Envio indexer for public verifiability. |
| 6 | Foundry tests for registry, demo video: ψ monitor catching a stress event, tool placing funded position. Write-up + submission. |

Scope guards: no greenfield engine, no cross-chain, single market focus (MON or SOL), registry contract ~300 lines max.

### 2.5 Risks

- Perpl API rate limits / stability — fallback: read chain state directly (all on-chain).
- Bounty ambiguity — confirm exact $5k API-use requirements with organizers early (unverified specifics beyond one-line descriptions).
- Drake's frUSDC vault overlaps funding-capture idea — differentiate via ψ/formal-risk narrative.
- ψ(t) is advisory only (same as v2 wrapper design) — fine for hackathon, honest in write-up.

### 2.6 Verdict

Strongest Track 01 fit. Transfers most of the team's research (funding, margin, ADL, ψ, DPMM — all chain-agnostic), directly answers "why Monad" (block-scheduled funding = Monad-native), and both Perpl bounties are in the lane. Build-on-top beats rebuild in 6 weeks.

---

## 3. Percolator + Monad reality check (portability verdict)

### 3.1 Percolator ground truth

**Real, public, actively-developed codebase by Toly Yakovenko. Reference design + engine, NOT a deployed DEX.** Surfaced 2025-10-19, [The Block coverage](https://www.theblock.co/post/375325/solana-co-founder-anatoly-yakovenko-designing-perps-dex-github). Toly: "just messing around with Claude... Please steal the idea." README: "Educational research project. Not production ready. Not audited. Do not use with real funds."

| Repo | Role | Status |
|---|---|---|
| [aeyakovenko/percolator](https://github.com/aeyakovenko/percolator) | Risk-engine library + spec.md v16.9.0 | Rust, 559★, active 2026-09-01, Apache-2.0 |
| [aeyakovenko/percolator-prog](https://github.com/aeyakovenko/percolator-prog) | Solana program (Pinocchio 0.6, 12.6k LOC) | Active |
| [percolator-cli](https://github.com/aeyakovenko/percolator-cli) / [percolator-meta](https://github.com/aeyakovenko/percolator-meta) | TS CLI/simulator, meta-program crate | Active |
| [Copenhagen0x/percolator-audit-2026-04](https://github.com/Copenhagen0x/percolator-audit-2026-04) | Independent audit | **1 active bug + 2 code defects**, 10 SAFE proofs, 305-harness baseline clean |
| [dcccrypto/percolator-launch](https://github.com/dcccrypto/percolator-launch) | Community one-click market launcher | Devnet only (51 markets, 14.5k cranks) |
| [uniperpapp/Percolator-Ethereum](https://github.com/uniperpapp/Percolator-Ethereum) | **Solidity reimplementation** | Pre-alpha, 1,444 LOC src, unmaintained since May 2026 |

- Published crate: [`percolator-engine`](https://crates.io/crates/percolator-engine) v1.0.0 on crates.io (no-std, "formally verified risk engine... fair exits (H) and O(1) overhang clearing (A/K)").
- Architecture: two-program design (engine + wrapper). **A/K/F indices confirmed**: `effective_pos(i) = floor(basis_i * A / a_basis_i)`; `pnl_delta(i) = floor(|basis_i| * ((K - k_snap_i)*FUNDING_DEN + (F - f_snap_i)) / (a_basis_i * POS_SCALE * FUNDING_DEN))`. O(1) settlement per touched account, order-independent. Envelope: `price_budget_bps = cfg_max_price_move_bps_per_slot * cfg_max_accrual_dt_slots`. Solvency precondition per notional.
- **Correction (see `monad-permissionless-perp-feasibility.md` §3.1):** these formulas are the **v12.19.13 "slab" lineage**, not v16.9.0 — v16.9.0 (rewritten 2026-05/06) uses a source-domain credit model with different accrual math. Target v12.19.13 for any port.
- "Slab" vocabulary retired in v16 → `MarketGroup` (independent vault/credit domain) + `PortfolioAccount` (legs + active_bitmap). ADL = "Quantity ADL" after residual durability; fairness via A index scaling, not priority queue. Keeper → "Crank-forward public markets" requirement (any privileged-actor-only state advance = non-compliant).
- **No live mainnet deployment exists.**

### 3.2 Monad ground truth (Sept 2026)

- Mainnet since 2025-11-24, chainId 143 (testnet 10143). Blocks **~300–400ms, single-slot finality ~600ms**. 10k TPS target; testnet sustained ~5k; ~500M gas/s. Fees sub-cent. [How Monad Works](https://blog.monad.xyz/blog/how-monad-works), [BlockEden recap](https://blockeden.xyz/de/blog/2026/04/01/monad-mainnet-mon-token-high-performance-evm-l1-10000-tps/)
- Full EVM bytecode compat (Pectra/Cancun: TSTORE/TLOAD/MCOPY, EIP-7702, P256 precompile). Full JSON-RPC compat — MetaMask/Foundry/Hardhat unchanged. Key [differences](https://monad-docs.vercel.app/developer-essentials/differences): contract size 128KB, no EIP-4844 blobs, **gas charged on gas_limit not gas_used, no refunds, reverts pay gas**, no global mempool.
- $150M TVL week one; Uniswap/Curve/Morpho live; ~304 protocols. Pyth + Chainlink both live on mainnet.
- For Solana/Anchor/Rust team: Solidity + Foundry = whole game. Zero Anchor/Pinocchio transfer; PDA/rent/CU/SPL gone.

**Metropolis verified details (2026-09-02):** intake open at [hackathon.monad.xyz](https://hackathon.monad.xyz/). Build 1 Sep–13 Oct 2026, judging 14–27 Oct, winners 3 Nov. 4 tracks × $30k + $25k grand champion. Judges: Galaxy, Electric Capital, Paradigm, Pantera, Dragonfly, CoinFund, IOSG, Castle Island. Full bounty list: Kuru $5k×2 (consumer app / new assets+markets), Perpl $5k (API use) + $3k (analytics/risk tool), Agora $10k×2 (mobile trading / cross-border payments), Aurora Intents $5k (any-chain liquidity), Envio $1k, Chainlink $3k, Privy $5k, Nansen AI $5k, Dynamic $5k, + smaller credits.

### 3.3 Portability verdict

**0% literal reuse. Port the spec, not the code.** No SVM→EVM path. But:
- **Partial Solidity port exists**: [uniperpapp/Percolator-Ethereum](https://github.com/uniperpapp/Percolator-Ethereum) — 1,444 LOC src (safety core: H haircut, conservation invariant, risk notional, margin, custody, A/K/F accrual+settlement). Incomplete: trade execution, oracle wiring, liquidation+ADL, LP vault, insurance fund pending. [Its mapping table](https://github.com/uniperpapp/Percolator-Ethereum/blob/main/docs/DESIGN.md) = best available portability analysis. Mine it for math.
- **What transfers verbatim (math chain-agnostic):** H haircut (junior PnL), solvency invariant `V >= C_tot + I`; A/K/F lazy indices (on EVM pack into one `Globals` slot → O(1) SLOAD/SSTORE per price move; **native uint256 kills ~3k lines of Rust wide-math**); bounded price/funding envelope (re-derive budgets in seconds, re-run solvency proof for 0.3–1s blocks); margin model, PnL warmup vesting, per-market isolation (slab → one EIP-1167 clone per market).
- **What mutates:** crank-forward liveness — replace continuous permissionless crank with **lazy on-demand accrual** (trader pays own accrual in own tx; idle markets cost $0) + searcher-driven liquidations + low-freq keeper heartbeat.
- **What's new on EVM:** reentrancy guards (new threat class), ERC-20/SafeERC20 token layer, Pyth pull oracle in-tx, owner checks instead of invoke_signed PDAs, Foundry/Halmos/Certora + differential tests vs Rust engine instead of Kani (244 proofs do NOT transfer).
- **Minimal port estimate: ~2,000–3,000 LOC Solidity** for coin-margined, oracle-priced single-market perp with H + A/K/F + envelope + `liquidate()`. (uniperpapp did ~70% of that scope in 1,444 LOC.)
- Gas model change: Monad charges gas_limit → tight explicit limits, batched liquidations with cursor; storage writes dominate → pack aggregates into one Globals struct; optimistic parallel execution parallelizes across markets, per-market Globals = serialization hotspot (same as slab on Solana).

---

## 4. Undercollateralised lending + Percolator

### 4.1 Onchain credit history landscape

**There is NO established, portable onchain credit-history primitive. Everyone bootstraps their own.** The best analysis of why ([Blockbooster, "onchain native credit origination"](https://www.odaily.news/post/5211270)) names the "three locks": (1) persistent identity (default and respawn with a new address), (2) cross-protocol reputation transmission (no standard for scores), (3) credible data. Any identity binding strong enough to solve this (KYC, biometrics) sacrifices permissionlessness; viable interim path = **"rewarding compliance" — ratchet collateral ratios down (150% → 130% → 110% → 80%) as repayment history accrues, and intercept on-chain cash flows automatically.**

Score providers (all Ethereum-centric, **none Monad-native**):
- [Spectral MACRO Score](https://www.htx.com/zh-cn/feed/news/272226/) — 300–850 score from Compound/Aave repayment history, wallet age, DEX behavior; ML oracle; NFC tokens; live in [Bulla's P2P lending](https://www.bulla.network/we-ve-integrated-spectral-into-bulla-s-lending-solutions). Ethereum-only.
- Cred Protocol — onchain lending history + debt-to-collateral ratios; ETH-focused.
- [7-project survey](https://mpost.io/7-projects-building-crypto-native-credit-scores-in-2026/) (Spectral, Cred Protocol, Arcx, Goldfinch, TrueFi, Credora, Guild) — **Monad not mentioned for any.**
- [ChainAware comparison](https://chainaware.ai/learn/comparisons/defi-credit-score-platforms.html) — 8 platforms; DeFi lending TVL >$50B yet >90% overcollateralized.

Standards: [ERC-7231](https://eips.ethereum.org/EIPS/eip-7231) = identity-aggregation NFT (CARV), not credit. **No credit-history ERC/EIP exists.** ERC-8004 = agent identity, not credit.

Monad-specific:
- **Klaave** — only undercollateralized-lending precedent on Monad: AI-agent credit-rail protocol, launched Feb 2026 **as a Monad hackathon entry**, live on mainnet ([build post](https://moltbook-observatory.sushant.info.np/posts/fd0ab0c8-af96-40c7-ab1a-b99d8b66317c), [submission](https://moltbook-observatory.sushant.info.np/posts/4f174e17-d739-4a39-a317-da8cf2cd2e72)). Mechanics: lenders deposit USDC; agents post **bond smaller than loan**; credit limit = f(bond, strategy equity delta, failures counter), recomputed per epoch by permissionless `updateEpoch()`; delinquency → permissionless `slashBond()`; no oracles, no governance. **0 deposits after 10 days (cold-start reality check).**
- [Gearbox on Monad](https://docs.gearbox.finance/developers/gearbox-protocol-monad-lp-opportunity) — "undercollateralized" only via composable Credit Accounts; LTV-guardrailed, not reputation-based.
- Monad 9 months old, ~35–40% of wallets are one-transaction airdrop addresses — **almost no natural wallet history to score.** Onchain credit history on Monad = history you generate yourself.
- Infra: [Envio HyperIndex](https://docs.envio.dev/blog/how-to-index-monad-data-using-envio) (official Monad indexer, Envio sponsor bounty exists); Pyth + Chainlink oracles live on Monad mainnet (Pyth pull-oracle PriceFeed `0x2880aB...17B43`).

Agora sponsor relevance: AUSD = stablecoin (VanEck/State Street reserves, Paradigm-backed, [OFT across 150+ chains](https://www.agora.finance/blog/ausd-now-borderless-onchain)); ~48% of AUSD supply on Monad; [earnAUSD yield token via Upshift](https://mpost.io/agora-and-upshift-partner-to-deploy-ausd-and-earnausd-on-monad-establishing-core-stablecoin-and-yield-infrastructure/). **$10k cross-border payments bounty = tangential for lending.** Real overlap: (i) AUSD payment history is a legit credit signal — "pay with AUSD, build payment record, unlock credit line"; (ii) AUSD = natural lending asset. But claiming bounty requires shipping payments app on top — spreads 6 weeks thin. Recommendation: use AUSD as lending asset, pitch narrative, don't chase payments bounty.

### 4.2 Design options ranked by 6-week buildability

**Option A — Progressive undercollateralization ("credit vault"): self-generated repayment history drives a collateral-ratio ratchet. Rank #1.**
Borrowers start ~120–130% collateralized; each repaid loan epoch increments onchain score accumulator (repayment count, timeliness, age, max-loan-serviced); tiers unlock lower collateralization (120% → 100% → 80% → 60%); collateralized portion liquidatable, uncollateralized portion covered by insurance fund. This is the Blockbooster "reward compliance" thesis. Only honest answer to "undercollateralized" without a dataset — the history IS the product.
- Build: ~1,500–2,000 lines Solidity + Foundry tests. Pyth pull oracle. Envio indexer. 6 weeks: yes, with slack.
- Weakness: cold start — day-one demo needs seeded/simulated history (acceptable; Klaave shipped same shape).

**Option B — Social/DAO vouching pools (Union-style trust lines). Rank #2 on buildability only.**
1-week contract set (stake, vouch, borrow, slash); canonical: [Union Protocol](https://consensys.io/blog/union-protocol-a-trust-based-solution-for-uncollateralized-defi-lending). But: Percolator transfer ≈ zero, novelty ≈ zero (Union since 2021, countless clones), stretches the "credit history" brief (vouch graph ≠ history).

**Option C — Yield-backed / self-repaying loans (Alchemix pattern with earnAUSD). Rank #3.**
Strong Agora tie-in, strong demo, but Alchemix exists since 2021 (novelty low), Percolator transfer ≈ zero, arguably off-brief (yield ≠ credit history), and at 2026 yields (2–4%) self-repayment timelines absurd.

**Option D — ZK credit passport + proof-of-personhood. Rank #4 for this team.**
The ETHGlobal winning formula — [CryptoBureau](https://ethglobal.com/showcase/cryptobureau-u1389) (Lisbon finalist; ZK proofs of Sismo/WorldID/PolygonID → lowered collateral), [DOG](https://hackathonprojects.dev/project/dog) (Istanbul; ZKML credit score), [Vouch DeFi/Veritas](https://ethglobal.com/showcase/vouch-defi-hfvw3) (ETHGlobal NY 2026 winner: World ID + TEE private AI underwriting + ERC-8004 credit passport + income routing), [Credit](https://hackathonprojects.dev/project/credit) (Bangkok; World ID + Dutch-auction interest rates + Merkle default registry). All depend on ZK/TEE/World ID infra — no evidence World ID verifier available on Monad; team has zero identity/ZK background.

### 4.3 Percolator concept transfer (honest)

| Percolator concept | Lending analog | Verdict |
|---|---|---|
| Margin model (initial/maintenance, leverage checks) | Score-tiered LTV / dynamic collateral requirements | **GENUINE** — same math shape |
| Keeper crank (permissionless, cursor, O(1)/account, fail-closed) | Permissionless health-check/default-marking loop; epoch score updates; auto-repay sweeps | **GENUINE, strongest transfer** — uncollateralized loans have no liquidation event, so a permissionless crank is the natural mechanism. Klaave independently converged on this (`updateEpoch()`, `slashBond()`), validating the pattern |
| Insurance fund (first-loss backstop) | Loss pool covering unsecured portion; lenders stake in | **GENUINE** |
| ψ(t) public solvency monitor | Public pool-health ratio (loss reserve / outstanding unsecured), crank-updated | **GENUINE as observability** — transfers nearly verbatim as "loss-reserve coverage ratio" |
| Bounded price envelope (fail-closed on oracle jumps) | LTV caps + staleness/deviation bounds in crank | **PARTIAL** — same defensive reflex, different object |
| CI option safety filter | Cheap per-borrower "is overdue / near-default" pre-check | **PARTIAL** — compute-saving pattern transfers, option math doesn't |
| A/K/F accumulators | — | **NO transfer.** Perps-specific |
| ADL (pro-rata auto-deleveraging) | — | **NO transfer** |
| DPMM mark price | — | **NO transfer** |
| Rust/Anchor codebase | Solidity/Foundry | **0% code transfer** (same cost for any Monad build) |

**~30–40% of Percolator's conceptual DNA transfers genuinely (crank, insurance fund, dynamic collateral, ψ observability, envelope reflex); 0% of code; none of the perps-domain math.** Partial genuine fit, not forced — but honest pitch is "we brought risk-engine discipline to onchain credit", not "we built a risk engine, here it is as lending".

### 4.4 6-week plan for Option A

Product: "CreditVault — undercollateralized lending on Monad priced on repayment history earned onchain. Borrowers start at 120% collateral, ratchet down to 60% as score grows; permissionless crank runs health checks; insurance fund covers unsecured portion. Lenders watch public ψ-style loss-coverage ratio."

Stack: Solidity 0.8.x + Foundry; Pyth pull oracle; Envio HyperIndex (Envio bounty eligible); AUSD lending asset (Agora tie-in).

| Week | Scope |
|---|---|
| 1 | Spec score function + collateral tiers in formal risk doc (the Percolator-style deliverable judges read). Scaffold Foundry. Contracts: `CreditVault` (ERC-4626 lender pool), `CreditAccount`. |
| 2 | Core: deposit, borrow (120% collateralized), repay, top-up, withdraw. Onchain score accumulator on repay: on-time count, lateness, largest-loan-serviced, age → tiers T0–T3. |
| 3 | Keeper crank: permissionless `crank(maxIterations)` — pull Pyth with staleness + deviation bounds (envelope reflex, fail closed), iterate borrowers via cursor, mark overdue, accrue interest, partial liquidate collateralized portion, flag defaults on unsecured portion. |
| 4 | Insurance fund: first-loss staking, pro-rata default cover, ψ-style public coverage ratio `reserve / unsecured_outstanding` updated by crank. |
| 5 | Envio indexer exposing repayment history + score timeline (GraphQL); demo dashboard; seed-history script for demo. |
| 6 | Foundry invariant tests (solvency: unsecured ≤ insurance reserve; no tier upgrade without repayments), demo video, write-up. |

Scope guards: one asset (AUSD), one score function, no governance, no interest-rate markets, no ZK/identity.

Risks: (1) Cold start — mitigate with seeded history + demo narrative. (2) Klaave precedent — differentiate via formal risk doc, insurance fund, ψ-style health metric (Klaave has none). (3) "Undercollateralized" purists — ratchet answers this: mechanism undeniably undercollateralized at upper tiers.

### 4.5 Verdict

**Weakest Track 01 fit for this team. Perps build clearly stronger.**

1. Perps → Monad perps transfers ~80% of team's research (margin models, funding, liquidation, ADL, oracle envelopes, A/K/F + ψ — all chain-agnostic math). Perps → lending transfers ~30–40%, none of the signature math. Both need same Rust→Solidity rewrite, so code isn't the differentiator — domain is.
2. "Perps with funding that updates every block" is the only suggestion leveraging Monad's defining claim (fast blocks). Undercollateralized lending is Monad-agnostic; every winning precedent built elsewhere. Judges ask "why Monad" — perps has the answer.
3. Sponsor economics: Perpl bounties ($5k + $3k) directly in perps lane; Agora bounties adjacent-at-best for lending.
4. Competition: Klaave already live on Monad mainnet + mature ETHGlobal winning formula team can't execute. Perps contested too, but formal-risk-engine differentiation demonstrable in 90-second video (envelope fail-closed, ψ monitor, keeper crank).
5. Track says "best fit: teams who have shipped a trading, lending, or market-making product" — team shipped trading risk infra, not lending.

Caveat: if locked into lending, Option A buildable in 6 weeks, transfers 3–4 Percolator concepts, wouldn't embarrass. Defensible second choice — but second choice.

---

## 5. Synthesis — which track suggestion is easiest + best fit

| Suggestion | Percolator transfer | 6-week risk | Verdict |
|---|---|---|---|
| 1. Fully onchain CLOB (Kuru) | ~50% (margin-on-fill, funding) but **no integration point** — Kuru spot-only, zero fill hooks, custom risk market registration undocumented | High — own research rejected CLOB; zero order-book code; EVM novice; Perpl/Drake shipped it | Hardest. Shallow Kuru play only (list market, app on SDK) |
| 2. Perps, per-block funding (Perpl) | **~80%** — A/K/F lazy indices IS the funding mechanism; v2 wrapper research (DPMM, ψ, CI filter) transfers 1:1; per-block funding ≈ $20–30/day/market gas | Low-Medium — 2–3k LOC Solidity port of spec (partial port exists to mine); ~4 weeks build | **Easiest + strongest fit. All 3 agents converge here** |
| 3. Undercollateralized lending | ~30–40% — crank, insurance fund, dynamic collateral tiers, ψ observability; none of signature math (A/K/F, ADL, DPMM) | Low build risk, high competition risk (Klaave live, ETHGlobal ZK formula unexecutable) | Weakest fit. Defensible fallback (Option A ratchet), but second choice |

**Recommendation — build #2, two-lane:**

1. **Core (primary): greenfield minimal Percolator port in Solidity.** Coin-margined single-market MON-PERP, oracle-priced (Pyth pull), H haircut + A/K/F + bounded envelope + permissionless `liquidate()`. 2–3k LOC, port spec not code, mine [uniperpapp/Percolator-Ethereum](https://github.com/uniperpapp/Percolator-Ethereum) for math. Funding accrues lazily per block (lazy F index) — directly answers "funding that updates every block," the only suggestion that uses Monad's defining claim. Weeks 1–4 build, 5–6 polish/submission.
2. **Bounty garnish (secondary):** risk dashboard exposing A/K/F + ψ(t) over Perpl order flow → fits Perpl $3k "Best Analytics / Risk Tool" and $5k "API use" (tool trades/reads via API). Kuru consumer-app bounty = UI on own engine + Kuru listing, if time.

**Why not the others:** CLOB needs Kuru cooperation (undocumented extension) or a fork; lending wastes the team's signature math. Both agents + lending agent independently landed on perps.

**Immediate actions:**
1. Register at hackathon.monad.xyz (intake open now).
2. Ask Perpl for exact bounty criteria; ask Kuru whether custom market registration with custom risk logic is possible.
3. Read percolator spec.md v16.9.0, extract A/K/F + H invariants → Solidity pseudocode (Week 1).
4. Verify uniperpapp port license before mining.

---

## Sources (aggregate)

- Monad Metropolis: https://www.monad.xyz/developers/hackathons/metropolis · platform: https://hackathon.monad.xyz/
- Kuru docs: https://docs.kuru.io/index · OrderBook: https://docs.kuru.io/contracts/OrderBook · MarginAccount: https://docs.kuru.io/contracts/MarginAccount · Integration: https://docs.kuru.io/contracts/Integration · addresses: https://docs.kuru.io/contracts/Contract-addresses · deploy-market SDK: https://docs.kuru.io/sdk/deploy-market
- kuru-sdk-py: https://github.com/Kuru-Labs/kuru-sdk-py · Kuru raise: https://blog.kuru.io/p/kuru-labs-raises-116m-led-by-paradigm
- Percolator repo: https://github.com/aeyakovenko/percolator · audit 2026-04: https://github.com/Copenhagen0x/percolator-audit-2026-04
- Monad gas pricing: https://docs.monad.xyz/developer-essentials/gas-pricing · post-launch: https://luganodes.com/blog/monad-mainnet-post-launch-research
- Drake: https://docs.drake.exchange/ · analysis: https://hakresearch.com/drake-exchange-la-gi-perp-dex-onchain-kieu-cex-tren-monad
- Percolator: https://github.com/aeyakovenko/percolator · prog: https://github.com/aeyakovenko/percolator-prog · launch: https://github.com/dcccrypto/percolator-launch · crates.io: https://crates.io/crates/percolator-engine · EVM port: https://github.com/uniperpapp/Percolator-Ethereum (+ https://github.com/uniperpapp/Percolator-Ethereum/blob/main/docs/DESIGN.md)
- The Block on Percolator: https://www.theblock.co/post/375325/solana-co-founder-anatoly-yakovenko-designing-perps-dex-github
- Monad docs: https://docs.monad.xyz/ · differences: https://monad-docs.vercel.app/developer-essentials/differences · how it works: https://blog.monad.xyz/blog/how-monad-works · mainnet recap: https://blockeden.xyz/blog/2026/04/01/monad-mainnet-mon-token-high-performance-evm-l1-10000-tps/
- Perpl docs: https://docs.perpl.xyz/ · developer overview: https://docs.perpl.xyz/resources/for-developers/overview · funding: https://docs.perpl.xyz/exchange/funding
- Perpl launch blog: https://blog.perpl.xyz/we-built-the-perp-dex-everyone-said-was-impossible/ · Perpl x Fun.xyz: https://blog.perpl.xyz/perpl-x-fun-xyz/
- Backpack explainer: https://learn.backpack.exchange/articles/what-is-perpl
- Drake Exchange: https://hakresearch.com/drake-exchange-la-gi-perp-dex-onchain-kieu-cex-tren-monad
- MON funding data: https://loris.tools/funding/coin/mon · https://www.coinglass.com/es/currencies/MON/futures
- Blockbooster onchain credit origination: https://www.odaily.news/post/5211270
- Spectral MACRO: https://www.htx.com/zh-cn/feed/news/272226/
- Bulla × Spectral: https://www.bulla.network/we-ve-integrated-spectral-into-bulla-s-lending-solutions
- Credit score project survey: https://mpost.io/7-projects-building-crypto-native-credit-scores-in-2026/
- ChainAware comparison: https://chainaware.ai/learn/comparisons/defi-credit-score-platforms.html
- ERC-7231: https://eips.ethereum.org/EIPS/eip-7231
- Klaave build post: https://moltbook-observatory.sushant.info.np/posts/fd0ab0c8-af96-40c7-ab1a-b99d8b66317c
- Gearbox Monad: https://docs.gearbox.finance/developers/gearbox-protocol-monad-lp-opportunity
- Envio Monad indexing: https://docs.envio.dev/blog/how-to-index-monad-data-using-envio
- Agora: https://www.agora.finance/blog/ausd-now-borderless-onchain
- Agora × Upshift on Monad: https://mpost.io/agora-and-upshift-partner-to-deploy-ausd-and-earnausd-on-monad-establishing-core-stablecoin-and-yield-infrastructure/
- Union Protocol: https://consensys.io/blog/union-protocol-a-trust-based-solution-for-uncollateralized-defi-lending
- Alchemix: https://docs.alchemix.fi/user/concepts/self-repaying-loans
- Monad mainnet launch: https://coinpaprika.com/news/monad-mainnet-launch-evm-10000-tps-2025/
- Monad deep dive: https://chainstack.com/deep-dive-into-monad-speed-performance-and-decentralization/
