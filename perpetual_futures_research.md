# Perpetual Futures on Solana: Finding the Next "xy=k" Moment

This research report synthesizes findings from over 20 cutting-edge academic papers (2021-2026), protocol whitepapers, and the codebase of Anatoly Yakovenko's **Percolator** risk engine. The goal is to identify mathematical innovations that can be leveraged to build a novel perpetual futures DEX on Solana.

---

## 1. The Search for the "xy=k" of Perpetuals

In spot AMMs, the constant product formula ($x \cdot y = k$) defined a generation of DeFi. For perpetual futures, the ecosystem has fractured into several competing models. Our research identified the leading mathematical equivalents for derivatives:

### The Fundamental Pricing Theorem
The most fundamental equation for perpetual futures, derived by Ackerer, Hugonnier, and Jermann (2026) [arXiv:2310.11771], defines a perpetual future's no-arbitrage price as the risk-neutral expectation of the spot price at a random time:

$$ F_t = E_t^Q [S_\tau] $$

Where $\tau$ is a random time governed by the funding rate intensity. The funding mechanism ($\text{Mark} - \text{Index}$) is what forces the continuous price anchoring.

### The Replicating Market Maker (RMM) Theorem
Angeris, Evans, and Chitra (2021) proved a profound result: **Any concave derivative payoff can be replicated by a Constant Function Market Maker (CFMM).**
Using Fenchel conjugacy, they showed a one-to-one mapping between trading functions $\phi(R)$ and derivative payoffs $V(p)$. This means AMMs don't just *facilitate* derivatives; geometrically, they *are* derivatives.

---

## 2. Evolution of Perpetual DEX Architectures

The field has evolved through distinct architectural paradigms, moving away from pure AMMs toward oracle-driven models:

1.  **Virtual AMMs (vAMM):** (e.g., Perpetual Protocol v1). Uses $x_{\text{virtual}} \cdot y_{\text{virtual}} = k$ for price discovery without real liquidity. **Verdict:** Elegant but suffers from capital inefficiency and funding rate skew risks.
2.  **Oracle-Pool Models (NAV pricing):** (e.g., GMX, Jupiter). Price is strictly defined by an oracle. $P = \text{AUM} / \text{Supply}$. **Verdict:** Highly capital efficient, zero traditional slippage, but transfers directional risk entirely to the LP pool.
3.  **Perpetual Demand Lending Pools (PDLPs):** A formalized model of modern oracle pools (Chitra et al., 2025). They use a **Target Weight Mechanism (TWM)** acting as a PID controller to adjust fees based on asset utilization and weight deviations.
4.  **Adaptive Curve AMMs:** (Nadkarni et al., 2024). Uses Kalman filtering and the Glosten-Milgrom model to dynamically adjust the bonding curve, estimating price *without external oracles* to minimize LP Loss-Versus-Rebalancing (LVR).

---

## 3. Advanced Derivatives: The Convex Frontier

Standard perpetuals are linear ($p=1$). The most significant innovation in derivative product design is the generalization to non-linear payoffs.

### Power Perpetuals & Squeeth (Paradigm / Opyn, 2021)
Power perpetuals track $S^p$. "Squeeth" (Squared ETH) tracks $S^2$, offering constant gamma (convexity) without strike prices or expirations.
$$ \text{Funding Fee} = \text{Mark} - \text{Index}(S^p) $$

### Spanning with Power Perpetuals (Clark, 2023)
Using a Taylor series expansion, **any payoff function** can be approximated by a weighted sum of power perpetuals.
$$ V(S) \approx V(S_0) + \Delta(S-S_0) + \frac{\Gamma}{2}(S-S_0)^2 + ... $$
This allows for perfect hedging of AMM Impermanent Loss.

### Everlasting Options & Dynamic Proactive Market Making (DPMM)
Perpetual options where $\text{Funding} = \text{Mark} - \text{Payoff}(S, K)$. Recent research (Mohanty et al., 2026) shows that DPMMs can make LPs profitable in these markets by dynamically adjusting pricing and delta-hedging.

---

## 4. Toly's Percolator: A Risk Engine Masterclass

**Percolator** is an open-source, Solana-native risk engine designed by Anatoly Yakovenko. It is structurally unique compared to existing platforms (Drift, Hyperliquid, Mango).

### Core Innovations in Percolator

1.  **The Sharded "Slab" Architecture:** Every market is an independent, zero-copy Solana account ("slab"). Risk is perfectly isolated. A global **Router** manages cross-slab collateral.
2.  **Deterministic Haircuts over ADL:** Auto-Deleveraging (ADL) forces specific users to close profitable positions to save the system. Percolator abandons ADL. Instead, it uses a global **Haircut Ratio ($H$)**:
    $$ H = \min(\text{Residual}, \text{PNL}_{\text{pos\_tot}}) / \text{PNL}_{\text{matured\_pos\_tot}} $$
    If the system is undercollateralized, *all* profitable users take a proportional haircut. Capital is senior; profit is junior.
3.  **Lazy A/K Side Indices:** Handles bankruptcy overhangs efficiently. Operations touch only bounded portfolio views, ensuring O(1) compute cost per event, which is vital for Solana's compute limits.
4.  **Coin-Margined, Permissionless Listing:** "Pump.fun for perps." Users deposit the underlying token to trade its perpetual.

---

## 5. Combining Percolator with Bleeding-Edge Math (The Innovation Blueprint)

How can we build a generational protocol by combining the Percolator risk engine with recent academic breakthroughs?

### Opportunity A: The Convexity Router (Power Perps on Percolator)
Because Percolator's "Slabs" isolate risk perfectly, it is the ideal environment for Power Perpetuals.
*   **Design:** Launch $S^1$ (linear), $S^2$ (Squeeth), and $S^3$ as separate slabs.
*   **The Magic:** The Percolator Router handles cross-margining. Users can construct Taylor-series hedges (Spanning) across slabs to perfectly hedge LP positions in Raydium/Orca, with margin netted globally but risk isolated locally.

### Opportunity B: Ergodic Optimal Liquidations
Standard liquidations cause massive price cascades. Cao & Šiška (2024) [arXiv:2411.19637] modeled liquidations as an ergodic optimal control problem, deriving HJB equations that balance execution speed against price impact.
*   **Design:** Replace standard aggressive liquidator cranks in Percolator with an "Ergodic Crank" that gradually disposes of bankrupt positions, minimizing the Haircut ($H$) applied to profitable users.

### Opportunity C: LVR-Aware Dynamic Funding Rates
Milionis et al. defined Loss-Versus-Rebalancing (LVR). Kim & Park (2025) and Zhang (2026) proved optimal funding rate algorithmic feedback rules.
*   **Design:** Implement a path-dependent funding rate mechanism (using Delayed BSDEs) inside the Percolator slab that dynamically adjusts the funding multiplier based on real-time LVR estimation of the underlying spot AMM.

---

## Recommended Reading List

1.  **Autodeleveraging: Impossibilities and Optimization** (Chitra, Dec 2025). Proves the ADL trilemma. Explains why Percolator's $H$ haircut model is superior.
2.  **Perpetual Demand Lending Pools** (Chitra et al., 2025). The math behind modern oracle pools (GMX, Jupiter).
3.  **Power Perpetuals** (White et al., 2021) & **Spanning** (Clark, 2023). The frontier of non-linear perpetuals.
4.  **A Primer on Perpetuals** (Angeris et al., 2023). The mathematical foundation of funding rates.
5.  **Funding-Aware Optimal Market Making for Perpetual DEXs** (Le, 2026). HJB modeling for market makers.
