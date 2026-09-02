# Percolator Product Ideation — What To Build That Is NOT a percolator.trade Copy

**Date:** 2026-09-02
**Method:** 10 agents — 5 research (competitor teardown, engine primitive deep-dive, landscape pain inventory, market trends, academic papers) → brainstorm (18 concept candidates) → 3 judges (uniqueness/grant, market/usability, build feasibility) → synthesis.
**Recommendation:** **ShortStack** — native onchain leveraged shorts on Pyth-fed mid-cap memes, distributed through TG bots, with creator-seeded bounded-loss vaults. Sequenced: build kit (weeks 1–6) → Trading Battles as first real money (6–8) → audit → venue post-audit.

---

## 1. percolator.trade teardown — what they actually are (verified 2026-09-02)

### Claims inventory

**Verified:**
- Fork of Toly's Apache-2.0 Percolator engine; "Pump.fun for Perps"; 3 slab tiers (256/1024/4096 user slots).
- Team: 2 founders (Khubair @dcc_crypto, David @0xSquid_Sol), AI-leveraged — "Claude (lead agent)" listed as co-owner in their PIVOT_PLAN.md.
- 220 devnet markets, 71 creators, 8,000+ waitlist signups, 6,500+ X followers, 420 Kani harnesses, 22 repos — all self-reported in a public claims ledger (`docs/PITCH-CLAIMS.md`).
- Mainnet = **closed beta, one SOL/USDC market**. Their own ledger: **"slab has no successful txs since May 12"** — effectively zero mainnet volume.
- **No audit** — "quotes received, not yet engaged" (June 11). Their matcher repo opens with "NOT AUDITED — do not use with real funds."
- **No funding** — "zero outside capital." Lost Colosseum Frontier 2026 (not among 26 winners).
- Toly engagement real but limited: bounties, retweets, bug patches. Toly is also an angel in Bulk (a competing curated perp venue) — their own landmine.

**Weak/unverified claims:**
- "700–780 SPL tokens with $50K+ daily volume have no perp" — CoinGecko estimate, removed from deck.
- ">95% gross margin, $0 MM spend" — illustrative only.
- Traction: devnet markets are rent-funded (0.44–6.87 SOL) and gameable; waitlist = emails.

### Actual architecture (code-verified)

- Frontend: Next.js 14 App Router, Vercel, Supabase/Privy. Backend: percolator-api (Hono), percolator-keeper, percolator-indexer.
- Chain side: fork of percolator-prog v16 (single 12,647-line Pinocchio file) + ~51 added instructions, 4 programs on mainnet.
- **Matcher**: external program via CPI. Two modes: **vAMM** = constant-product virtual AMM from pool reserves; **passive LP** = fixed ±50bps around oracle. Reference matcher: oracle ± spread_bps (default 30bps, clamp 500bps). No depth-adaptive pricing.
- **Oracles**: three modes — `pyth-pinned` (on-chain Pyth), `hyperp` (DEX-pool EMA cranked onchain; PumpSwap/Raydium/Meteora), `admin` (keeper-pushed). **No TWAP, no depth gating, no deviation clamps** — their own ledger marks depth-gated marks + deviation clamps GATED. Thin-memecoin DEX oracles = the manipulation hole.
- **Contradiction:** pitch says coin-margined ("deposit BONK to trade BONK perp"); shipped mainnet configs are USDC-margined.

### What they wired vs skipped (from their ledger)

**Wired:** per-market slab isolation; LP vault as counterparty; parimutuel tail haircut; warmup-H gate; per-market insurance scalar; permissionless KeeperCrank; liquidations at oracle; pause/unpause; admin renounce; NFT positions.

**Skipped (load-bearing):** funding rates **hard-disabled** (`funding_rate_e9 != 0` rejected); **no hard OI caps** (unreachable code path); insurance sub-vaults; depth-gated marks; deviation clamps; **one LP vault backs one side only** (other side rests on insurance — their landmine #2); audit not engaged; **mainnet V1 curated — permissionless listings pushed to 2027**.

**Bottom line:** the headline product (permissionless any-token perps on mainnet) is not shipped. Shipped = devnet playground + waitlist + closed beta with ~zero volume. Their real moat is story (Toly halo, claims discipline), not product.

### What NOT to copy (their 7 anti-patterns)

1. Slab-monolith re-engineering (12.6k-line single file, dynamic offsets, user caps).
2. External per-LP matcher CPI before one pricing mode actually works.
3. Raw DEX-spot oracles without TWAP/depth gates — the manipulation vector on thin tokens.
4. Coin-margined marketing (even they ship USDC-margined).
5. No-funding / no-MM bootstrap ("negative results" in their own design tournament).
6. Waitlist + pitch-deck GTM before a shippable artifact.
7. Marketing "permissionless" while running a curated closed beta.

---

## 2. Percolator engine — the underused superpowers

Primitive inventory (from spec v16.9.0 + percolator-prog, read 2026-09-02):

| Primitive | perc.trade uses? | Any Solana venue equivalent? |
|---|---|---|
| Source-credit parimutuel settlement `min(1, backing/claims)` | Yes | **No** — everyone else socializes via ADL/vault detonation |
| Insurance ledger + per-domain budgets + credit reservations | Yes | No |
| A/K/F O(1) lazy indices | Yes | Yes (common) |
| Warmup profit vesting | [unverified] | **No** — spec-only in v16.9.0, real build needed |
| Activation envelope proofs (price/funding/liq budgets) | Partially | No |
| Crank-forward permissionless liveness (spec compliance item) | Yes | No |
| Permissionless terminal recovery / force-close-abandoned / oracle-free stale resolve | Yes | No |
| Exclusive close serialization (`max_close_slot`) | Yes | Internal only |
| O(1)-in-N slab isolation (5,834 assets/10 MiB proven) | Yes | No |
| Permissionless asset creation, fee → insurance flywheel | Yes | No |
| Burnable per-asset admins (Drift-killer: markets that can't be rugged, even by creator) | Partial | **No** — Drift's $285M exploit was exactly admin capture |
| **293 Kani proofs + 51 function contracts + 17 kernels** double-entry conservation | Inherits, doesn't market | **No production Solana perp has anything close** |
| Design A: paid bounded seed + OI-cap-from-seed + leverage ratchet (13 Kani proofs) | [unverified] | No |
| Composite oracle + HYBRID_AFTER_HOURS EWMA (Bounty 7, STOXX/SOL 3-leg) | No | No |
| Deterministic LP reward counters (`cumulative_loss_atoms`) | Not advertised | No |

**Top underused:** (1) the conservation-proof corpus as a product — 293 Kani proofs, nobody on Solana ships formal solvency; (2) Design A bounded paid seed — the LP-sourcing answer; (3) admin-burn + permissionless wind-down — the Drift-inversion trust guarantee; (4) fee-to-insurance flywheel; (5) warmup (spec-only — building it = real differentiation).

**Engine limitations to respect:** v16.9.0 unaudited (audit predates rewrite, 1 active bug + 2 defects); Pinocchio not Anchor (zero-copy pods, LiteSVM, steep curve for Anchor devs); warmup + recovery-fallback pricing not implemented; liquidity not solved by the engine ("prevents insolvency, not emptiness"); one vault per instance, no cross-instance margin; parser-priced marks are "solvency unsafe (research-stage)" per Percolator's own threat model.

---

## 3. Solana perp landscape — the pain inventory (what's actually broken)

### Venue-by-venue (Sept 2026)

- **Jupiter Perps** (~62–84% share): Pyth-priced with simulated impact, no real book; majors-only (SOL/ETH/wBTC — no memes per Aug 2026 guide); JLP eats raw directional risk; ~92% of spot routing via private offchain "Prop AMMs."
- **GMTrade** (GMX fork): pooled LP inherits toxic-flow; volume points-farmed (Galaxy: ~74% share May 2026 partly GT points-driven); 500x; FX/commodities, not crypto long-tail.
- **Pacifica** (ex-FTX-COO): **offchain matching, onchain settlement** — contradicts SF RFP; no public audit as of May 2026; $146B cumulative volume visibly points-engine-driven; only ~$86M OI on claimed $1B/day.
- **Phoenix** (Ellipsis Labs, Toly-backed): fully onchain CLOB but $16.8M OI, ~1,300 traders; incentive-driven volume ($420k Flight Club); the honest test of "does onchain matching attract real flow" — and it's not yet winning.
- **Velocity** (ex-Drift): zero volume since Apr 1 2026 exploit; funds replaced by recovery tokens; USDT settlement backed by Tether $127.5M credit line; relaunch **drops Isolated Markets**; meme-perp history (WIF/BONK, $130M/day peak) gone.
- **Dead:** Flash Trade (shut Aug 2026, $520k lifetime revenue), Zeta (died May 2025 on fee competition alone, $15B lifetime volume).

### Structural problems (with evidence)

1. **Admin-key risk is the #1 killer** — Drift: $285M drained in 12 min via social-engineered 2/5 multisig + fake collateral token + oracle wash-trade from $500 seed pool.
2. **Oracle manipulation** — dYdX YFI: $9M insurance drain (40% of fund); GMX AVAX: $565k in an hour.
3. **Liquidation under failure** — dYdX chain halted 8h during $19B cascade; single keeper ~80% of HL/dYdX liquidation volume.
4. **Vault-inherited toxic positions** — HL JELLY: HLP auto-inherited short, ~$12–13.5M loss, validators force-settled at 1/5 market price.
5. **Cross-margin contagion** — patched post-incident by shrinking product (raise MM, raise IM), not by design.
6. **Keeper centralization / liquidation MEV** — GMX team keeper 0.12% price discretion; front-running pervasive.
7. **Funding manipulation** — no Solana venue publishes game-proof funding.
8. **Cold start** — Zeta died on fees; Flash at $520k; OBSDN at $6k TVL; every surviving venue's volume is points-farmed.
9. **LP economics never solved** — every Solana venue puts LPs against trader PnL; Hindenrank grades HL "C", GMX "C", dYdX "B-".

### Percolator fit matrix (strong fits)

H haircut vs ADL/socialized losses — **strong**. Envelope vs oracle manipulation — strong containment. Immutable-at-creation + per-market vaults vs admin kills — **strong (the Drift lesson)**. Permissionless crank vs keeper centralization — strong. Warmup vs spike-and-withdraw — strong. OI-cap-from-seed vs listing gates — partial (solves economics, not demand). ψ(t) public solvency invariant — **strong, nobody else ships it**.

### Unsolved gaps nobody serves

1. **Leveraged mid-cap memecoin exposure onchain** — Drift dead, Velocity dropped it, Jupiter majors-only. $130M/day Drift peak demand now sits in Hyperliquid custody. Contested by Perps.fun, Uranus, wounded Velocity — but no credible, audited venue exists.
2. **Bounded-risk creator listing** — everyone uses governance, multisig, or $20M HIP-3 stake. Design A replaces both.
3. **Provable public solvency** — post-Drift, "can't happen again" is the strongest claim in the market and only a formally-bounded engine can make it honestly.
4. **Fair exits instead of socialized exits.**
5. **LPs who don't eat directional risk.**
6. **Honest non-incentivized volume** — a real (even small) volume differentiator post-points-hangover.

---

## 4. Market trends that shape the product (Sept 2026)

- **Memecoin meta: V-shaped, not dead.** pump.fun: ~$7.49M fees/24h, ~$946M volume (~19% of Solana DEX volume), ~391k addresses. But H1 2026 Solana revenue $141M, −87% YoY. Memecoin share of DEX volume recovered to ~34% from 18% trough. ~95% of launchpad tokens fail in 90 days.
- **AI agents are the emerging buyer: agent wallets = 34% of Solana memecoin DEX volume (~120k agent-only wallets), Mar–Jun 2026.** Agents get leverage today via Hyperliquid custody bridges (Byreal) or **unaudited percolator deployments** (outsmart-agent ships 15 Percolator tools). No audited, onchain, Solana-settled leveraged venue for agents exists.
- **TG bots route 30–40% of Solana DEX volume** with zero leverage. Copy trading is spot-only (GMGN ~$88k/day fees).
- **RWA perps = the big new thing** ($1.32T in 5 months) — but Solana's are offchain-matched (Pacifica) or GMX-style (GMTrade). SF RFP demands fully onchain. Dragonfly's Haseeb argues CLOBs can't cold-start RWA liquidity (RFQ/brokerage wins).
- **Points farming is now a liability narrative** (Arichain collapse, Printr cancellation).
- **Prediction markets**: Jupiter Forecast (15-min crypto windows), Polymarket × Jupiter, World's 8-day exit. Short-window "perp-priced events" format unowned on Solana.
- **Onchain vs offchain**: volume winners are hybrid/offchain (HL ~35–40% share; Solana ~12–14%); but the SF grant money sits on the opposite side (fully onchain, orderbook/RFQ price discovery, no offchain matching).

---

## 5. Papers that seed ideas (mechanism → idea → Percolator mapping)

| Paper | Mechanism | Idea | Percolator mapping |
|---|---|---|---|
| Kim & Park 2506.08573 | Funding engineered to track any target; δ-averaged → instantaneous | Funding-as-primitive; choose F's anchor deliberately | Lazy F index accrues per trade already |
| ADL Impossibility 2512.01112 | No ADL = solvency + revenue + fairness; HL Oct 2025 = $45–51.7M excess haircut | Bounded, disclosed, **per-market** socialization | Parimutuel haircut IS an ADL policy — per-market isolation justified |
| He et al. 2212.06888 | Persistent perp-spot deviations, Sharpe 3.5 | Funding-carry products validated; naive funding leaks alpha | F index pre-computable per block |
| Milionis LVR 2208.06046 + 2305.14604 | LP PnL = market risk + LVR; faster blocks + lower gas strictly reduce LP losses | LVR dashboard; "Solana's 400ms slots cut LP adverse selection" | Passive LP mode is LVR-exposed; crank-tight matching |
| 2606.21769 (2026) | Optimal fee increases in instantaneous variance | Vol-scaled dynamic spreads | Recompute LP spread per crank from realized-vol estimator |
| Frongillo 2302.00196 | CFMMs ≡ prediction markets; parimutuel = budget-balanced scoring rule | Event/binary perps, resolution-only, no oracle to exploit | Parimutuel settlement native |
| Gerzon IMC'25 | 521,903 Solana sandwiches, $7.7M | Assume DEX-oracle inputs get sandwiched | Per-slot batch clearing removes intra-slot ordering rent |
| Budish FBA QJE'15 + EC'25 | Uniform-price batches convert speed→price competition; but not MEV-immune | Slot-batch perps (~400ms natural batch) | Crank aggregates orders; envelope = circuit breaker |
| Qin IMC'21 | Fixed liquidation discount over-discounts + front-running | Dutch-auction liquidations | Small change on liq instruction |
| Mackinga ICBC'22 + Ormer 2410.07893 | TWAP manipulation sublinear in window; median-based resists | Median-of-venues DEX oracle, not mean EMA | 5-slot median fits crank memory |
| Red-Black FC'21 + Klages-Mundt | Proportional-claim collateral makes liquidations optional; cap leverage vs spirals | No-liquidation lane ≤3x | Parimutuel haircut IS red-black; OI caps = spiral guard |
| Listing effects CESifo 2025 | ~10% 7-day abnormal returns, 15–20% sustained activity | Listing-as-acquisition channel | Design A permissionless listing + team seeds |

**Debunked ideas (don't build on):** "Solana has no MEV" · "batch auctions eliminate MEV" · "long TWAP = safe oracle" · "ADL can be fair" · "funding pins price" · "leveraged AMM/oracle-free vAMM as novel primitive" (no academic validation exists) · "fixed liq discounts are fine."

---

## 6. The 18 concept candidates (full catalog)

**Group A — Listing & liquidity:**
1. **First Light** — leverage on Pyth-fed mid-cap memes (WIF/BONK/POPCAT class), per-market isolated risk. The vacant Drift lane.
2. **SeedMarket** — bounded-loss permissionless listing as a paid product token teams buy. Listing customer IS the counterparty vault.
3. **Hardstop Vaults** — "HLP with a hard stop": bounded-loss per-market LP product.
4. **Redlight** — the no-liquidation lane (parimutuel haircut instead of liq cascades, ≤3x).

**Group B — Product UX:**
5. **Agent-native perps** — API/MCP-first venue for AI agents (34% of memecoin volume is agent wallets; they currently use unaudited perc markets or HL custody).
6. **ShortStack** — one-click native short for every pump, TG-bot distributed. USDC-margined shorts need no token borrow.
7. **FollowChain** — leveraged copy trading with provable parimutuel PnL attribution.
8. **Position Market** — marketplace for transferable NFT positions (exit liquidity for warmup-locked PnL).
9. **CarryBot** — one-click funding-carry autopilot.

**Group C — Infra/primitives:**
10. **Percolator-as-a-Service** — white-label the provably-solvent engine.
11. **Solvency Oracle** — live ψ(t) risk attestation other protocols query.
12. **Liquidation-as-a-Service** — Dutch-auction liquidations + permissionless crank network.
13. **Anchor-native build kit** — pinned v12-audited lineage + differential harness vs engine crate + docs/CI + thin Anchor wrapper.

**Group D — Wildcards:**
14. **SlotBatch** — slot-batch uniform-price perp clearing (FBA on Solana, paper-backed).
15. **Flatline Markets** — oracle-free parimutuel event perps.
16. **Crash Cover** — parimutuel protection markets (insurance that can't go insolvent).
17. **Trading Battles** — parimutuel trading competitions (trustless points replacement).
18. **After-Hours Index Perps** — RFP-compliant RWA lane (composite Pyth + HYBRID_AFTER_HOURS EWMA, bounty-proven upstream).

---

## 7. Judge verdicts

| Judge | Winner | Key kills |
|---|---|---|
| Uniqueness/grant | **18 After-Hours Index Perps** (RFP-defensive-by-criteria; runner-up 14 SlotBatch) | 9 CarryBot, 10 PaaS, 12 LiqaaS, 6 ShortStack (UI wedge alone), 17 Battles (copyable) |
| Market/usability | **A1 First Light + B6 ShortStack as one stack** (the trade exists nowhere else; TG-bot = distribution; runner-up A2 SeedMarket) | 3 Hardstop (no named LP source), 4 Redlight (haircuts on wins get ratioed), 8 Position Market, 11 Solvency Oracle, 16 Crash Cover |
| Build feasibility | **C13 build kit + D17 Trading Battles** as first-money deployment (zero-custody safety taxonomy; runner-up 17) | 7 FollowChain (12-person scope), 16 Crash Cover (fresh math), 14 SlotBatch (mechanism risk), 12 LiqaaS, 9 CarryBot |

### Conflict resolution (synthesis)

- Judge 1 right that grant-defensiveness matters; wrong that RWA is a product — RWA traders won't switch venues for onchain purity (Pacifica/GMTrade own that flow).
- Judge 2 right about the product (vacant trade = onchain leveraged mid-caps; nobody sells native shorts); but routing real funds through unaudited engine in 8 weeks = project-ending risk.
- Judge 3 right about sequencing (engine truth: unaudited v16, $40–100k audit wall → anything with real funds is a devnet demo in 8 weeks); wrong that a build kit is a product.
- **Merge: Judge 2's product direction + Judge 3's sequencing + Judge 1's grant positioning.**

---

## 8. FINAL RECOMMENDATION: ShortStack

**Pitch:** *The short side of the pump, onchain.* Native, USDC-margined leveraged shorts on Pyth-fed mid-cap memes (WIF/BONK/POPCAT class) — the first venue where a degen can short a mid-cap pump without bridging to Hyperliquid custody. Markets are creator-seeded with hard-capped loss; solvency provable per market; exits bounded and disclosed, never exchange-wide ADL.

**Target users:** degen shorts reached through TG bots (where 30–40% of Solana DEX volume lives, with zero leverage today); token teams buying a listing event (SeedMarket supply side folded in).

**Why this survives the "not a copy" test — every axis inverted vs percolator.trade:**

| Axis | percolator.trade | ShortStack |
|---|---|---|
| Asset scope | Parser-priced long-tail (manipulable) | Curated Pyth-fed mid-caps (WIF/BONK/POPCAT) |
| Counterparty | Passive LP pool oracle±spread | Creator-seeded bounded vaults (Design A) — listing customer IS the vault |
| Distribution | Waitlist + pitch decks | TG-bot short button (where volume already lives) |
| Risk engine | Unauthored v16 fork + 51 instructions | Pinned v12-audited lineage + differential harness vs engine crate |
| Pitch | "Any-token perps" (shipped nowhere) | "Short the pump natively, no HL custody bridge" (a trade, not a story) |
| Solvency | Silent tail haircut in UI | ψ(t) public invariant crank-updated onchain — the post-Drift claim |

**Percolator wiring:** per-market slab isolation (zero contagion) · Design A paid bounded seed + OI-cap-from-seed (loss hard-capped at seed — converts existential LP-sourcing into revenue) · warmup gate (blocks pump-and-dump PnL extraction) · H haircut + A/K/F O(1) settlement (fair, per-market, disclosed exits — never market "no liquidations", market "your exit is bounded and yours") · curated Pyth CPI marks with staleness checks only in v1 (DEX-parser long-tail gated to post-audit) · PermissionlessCrank · Token-2022 NFT positions.

**8-week sequencing (build kit → battles → demo):**

| Weeks | Work |
|---|---|
| 1–2 | Recover/pin v12-era audited lineage from git history; differential harness vs percolator-engine crate; CI |
| 3–4 | Thin Anchor wrapper (wrapper + harness only — never a 12.6k-LOC macro rewrite); devnet reference market; docs |
| 5–6 | **Trading Battles** — first real money through full trade path (open/close, A/K/F accrual, H), payouts parimutuel-capped at paid-in losses; small caps so engine bug costs contest fees, not a vault drain; live ψ(t) dashboard |
| 7–8 | ShortStack devnet demo (mid-cap marks, short-first UI, TG-bot command); SF RFP + Superteam grant applications; Colosseum demo |

**Risks + mitigations:**
1. *Unaudited engine; no grant disburses pre-audit* → in-window artifacts route no uninsured funds; RFP ask backed by shipped code; audit scopes to thin wrapper post-grant.
2. *Pinocchio vs Anchor curve* → differential tests inherit ~293 Kani proofs; wrapper-only keeps audit surface minimal.
3. *LP/seed sourcing (existential)* → SeedMarket makes listing customer the vault; validate with 1–2 mid-cap teams pre-mainnet.
4. *Demand fickleness* (memecoin share 45%→18% dip; Flash died at $520k) → the short is the uncovered half (longs exist via spot); honest non-incentivized volume as differentiator, not points.
5. *Lane contested* (Perps.fun, Uranus, Velocity) → Pyth-fed mid-caps + provable solvency + named auditable team = position none holds.

**Runner-ups to revisit:** After-Hours Index Perps (18) post-audit if RFP feedback pushes RWA lane; Redlight (4) as settlement-marketing layer once real markets exist — never standalone.

**Killed by judges (do not build):** CarryBot, PaaS, LiqaaS standalone, FollowChain, Crash Cover, SlotBatch (mechanism risk for 3 devs), Hardstop Vaults standalone, Solvency Oracle standalone (ship ψ(t) as venue feature), Position Market (before live positions), points farming.

---

## 9. Grant positioning (why a reviewer funds THIS and not "another perp")

1. **RFP criteria fit:** fully onchain execution, onchain price discovery (RFQ/CLOB matcher slab), open source, "verifiable, transparent code" — backed by the only formal proof corpus in the market (293 Kani proofs + differential harness artifact).
2. **The Drift inversion:** immutable-at-creation params, no admin listing path for collateral, burnable admins, permissionless wind-down — "markets that can't be rugged, even by their creator" is the single most credible post-Drift claim available.
3. **Bounded-loss listing economics:** replaces governance and HIP-3's $20M stake with paid seed + OI-cap-from-seed — listing without the moat, risk bounded by math.
4. **A trade, not jargon:** "short the pump onchain" is feelable in 30 seconds of demo; ψ(t) ticker + battle lifecycle = demo gold.
5. **Honest volume:** post-points-hangover, a venue whose volume is real — however small — is itself a differentiator reviewers recognize after watching Flash die.

---

## 10. Immediate next actions

1. Pin v12.19.13 lineage from percolator git history (commit fe1d2d81f3 era); stand up differential harness vs `percolator-engine` crate — week 1 gate.
2. Write the thin Anchor wrapper plan (wrapper + harness only).
3. Design Trading Battles spec: entry fee, parimutuel payout = min(1, backing/claims), sybil mitigations, gambling-law review (entry fees!).
4. Start SF RFP + Superteam India applications with the shipped-artifact story (not slides).
5. Validate SeedMarket demand: approach 1–2 mid-cap token teams ("would you pay to seed a shortable market for your token?") — the existential-dependency check before any mainnet commitment.
6. Audit quotes now ($40–100k, months lead) — scope = thin wrapper, post-grant.
7. Register Colosseum Fall (Sep 28–Nov 2) — optional tail.

---

## Sources (key)

- percolator.trade repos: https://github.com/dcccrypto/percolator-launch · https://github.com/dcccrypto/percolator-match · https://outsmartchad.github.io/outsmart-cli/
- Percolator: https://github.com/aeyakovenko/percolator · prog: https://github.com/aeyakovenko/percolator-prog · audit: https://github.com/Copenhagen0x/percolator-audit-2026-04 · Design A: https://github.com/dcccrypto/percolator-perp-liquidity
- Landscape: https://cleansky.io/blog/jupiter-solana-superapp-2026/ · https://www.galaxy.com/insights/research/solana-incentives-phoenix-perpetuals · https://www.chainalysis.com/blog/lessons-from-the-drift-hack/ · https://www.dydx.xyz/blog/october-2025-dydx-chain-incident-review-community-update · https://www.coindesk.com/markets/2025/03/26/hyperliquid-delists-jellyjelly-after-vault-squeezed-in-usd13m-tussle · https://perpfinder.com/perps/pacifica · https://hindenrank.com/blog/is-hyperliquid-hlp-safe
- Trends: https://www.cryptopolitan.com/es/pump-fun-98-percent-launchpad-revenue/ · https://coinmarketcap.com/community/fi/articles/6a282df611a9ab1bd7aa1888/ (agent wallets 34%) · https://openliquid.io/blog/telegram-trading-bots-landscape-2026/ · https://www.mitrade.com/au/insights/news/live-news/article-3-1783745-20260605 (Jupiter Forecast) · https://www.blockscholes.com/premium-research/2026---the-year-of-rwa-perps
- Papers: https://arxiv.org/abs/2506.08573 (Kim-Park) · https://arxiv.org/abs/2512.01112 (ADL trilemma) · https://arxiv.org/abs/2212.06888 (He et al.) · https://arxiv.org/abs/2208.06046 (LVR) · https://arxiv.org/abs/2606.21769 (vol-scaled fees) · https://arxiv.org/abs/2302.00196 (CFMM≡PM) · https://dl.acm.org/doi/10.1145/3730567.3764493 (Solana MEV) · https://ericbudish.org/publication/the-high-frequency-trading-arms-race-frequent-batch-auctions-as-a-market-design-response/ · https://arxiv.org/abs/2106.06389 (liquidations) · https://arxiv.org/abs/2410.07893 (Ormer) · https://link.springer.com/article/10.1007/s10203-021-00323-0 (listing effects)
