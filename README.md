# tokyo-engine: Systematic Market Microstructure & Econometric Validation

> **Note:** This repository documents the econometric methodology, empirical falsification log, and live execution architecture of a personal quantitative trading project (`live_m26_daemon.py`). Because the system is actively traded live, the signal generation code, exact thresholds, and order-routing logic are kept private.

---

## Overview

`tokyo-engine` is an event-driven research framework and live execution daemon built to model, validate, and trade synchronized cross-sectional mean-reversion and co-jump dynamics across liquid crypto perpetual futures.

Rather than treating backtesting and live execution as separate stages, the project integrates **order-book microstructure, queue-penetration dynamics, and non-stationary regime shifts** directly into the statistical validation pipeline over an **8.3-month dataset** (Feb–Oct 2026, 253.6 trading days).

---

## 1. Final Synthesis & Walk-Forward Performance (Phases 25–26)

The table below does not represent isolated backtests, but the **final architectural synthesis (Phases 25–26)** of a 26-phase empirical research pipeline. Phases 1–24 focused on isolating the core Data Generating Process (DGP), falsifying microstructure assumptions (see Section 2), diagnosing the March 2026 regime shift, and freezing base indicator parameters ($df_{\text{ind}} = 0$).

All results are evaluated over the full 8.3-month dataset (H1: Feb–Jun 2026 vs. H2: Jun–Oct 2026), net of VIP taker/maker fees and conservative stop-loss slippage, normalized to $1.00R$ average risk per trade:

| Model Stage / Execution Architecture | Window | $N_{\text{Tr}}$ | $G_{\text{Cl}}$ | $\text{WR}\%$ | $\mathbb{E}[R]_{1R}$ | $t_{\text{Clust}}$ | $\text{PF}$ | $\sum R_{\text{H1}}$ | $\sum R_{\text{H2}}$ | $\sum R_{\text{Tot}}$ | $\text{DD}_{1R}$ | $\text{Calmar}_{1R}$ |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **M25.0** — Core Orthogonal 3-Grid Baseline | 8.3M | 147 | 78 | 55.8% | +0.341R | +2.64 *** | 1.83 | +18.12R | +32.01R | **+50.13R** | 12.10R | 4.14 |
| **M25.3** — + Non-Losing Orthogonal 2nd Tranche | 8.3M | 203 | 99 | 55.2% | +0.300R | +2.49 ** | 1.68 | +21.15R | +39.77R | **+60.92R** | 13.85R | 4.40 |
| **M25.4** — + Absorbed Climax Gate (`U12` Core) | 8.3M | 233 | 113 | 54.9% | +0.303R | +2.60 *** | 1.69 | +23.44R | +47.24R | **+70.68R** | 14.41R | 4.90 |
| **M26.1a** — + `U12`-Anchored High-Beta Expansion* | 8.3M | 286 | 138 | 56.3% | +0.330R | +3.32 *** | 1.77 | +22.83R | +71.50R | **+94.34R** | 15.25R | 6.18 |

> **Universe Coverage Note:** The core `U12` universe spans the full 8.3-month dataset (H1 + H2). The 8 high-beta expansion symbols (`EXTRA_8`, filtered ex-ante via a 7-day rolling volatility gate $\ge 14.0\text{ bps}$) are available in the local cache from June 22, 2026 onward, contributing exclusively to H2. Cluster-block bootstrapping (5,000x) for **M26.1a** yields $\mathbb{P}(\sum R > 0) = 99.94\%$.

---

## 2. Microstructure Findings & Falsified Hypotheses

Most retail trading assumptions break down under correlated liquidity shocks and realistic fill modeling. Below is a log of hypotheses that failed during regime and execution stress-testing:

| Tested Hypothesis | Empirical Failure Mode | Empirical Evidence | Architectural Resolution |
|---|---|---|---|
| **Unfiltered Z-Scores (Latent Covariate Shift)** | Tightening rolling variance ($\sigma_{60}$) during low-volatility consolidations triggers premature entries on bar 1 of trend breakouts. | $-76.27R$ loss ($27.2\%$ WR) during the March 2026 H1 regime shift. | Enforced ex-ante 30m SMA displacement and physical $\text{ATR}_{14}$ over-extension gates. |
| **Pure Passive `POST_ONLY` Execution** | Severe adverse selection: $100\%$ fill rate on losing breakouts, but missed fills on rapid V-shape mean-reversion bounces. | Up to $60$ missed winners in 4M stress test; WR dropped from $58.5\%$ to $52.5\%$ ($t_{\text{Clust}}: 3.06 \to 2.43$). | Built **AS4 Hybrid Routing**: BBO-pegged `POST_ONLY` chaser at $T - 2.0\text{s}$ with deterministic `IOC` taker fallback at $T + 1.5\text{s}$. |
| **Isolated Single-Asset Triggers ($W = 1$)** | Idiosyncratic shocks reflect asset-specific news or liquidation cascades rather than market-wide mean reversion. | $40.2\%$ WR, $\sum R = -41.03R$ across $N = 316$ isolated signals. | Required directional, same-grid synchronous co-jumps ($W \ge 2$) anchored to core liquid pairs. |
| **Asynchronous Cross-Grid Co-Jumps** | Counting co-jumps across misaligned 9m phase offsets ($\phi_1 \neq \phi_2$) introduces false confirmation. | $20.0\%$ WR across both H1 & H2 ($\sum R = -22.35R, t_{\text{Clust}} = -3.54$). | Restricted macro discretization strictly to orthogonal 3-minute grids ($\phi \in \{0, 3, 6\}$). |
| **9-Phase Dense Discretization ($\phi \in \{0..8\}$)** | Non-divisible minute offsets cannibalize portfolio slots prematurely on minute 1 of structural shocks. | 4M Taker $\sum R$ dropped from $+51.86R$ to $+16.89R$; 8.3M $\text{Calmar}_{1R}$ collapsed from $4.90$ to $0.52$. | Locked orthogonal 3-grid rotation ($\phi \in \{0, 3, 6\}$). |
| **Mid-Trade Breakeven Stop Locks** | Moving stops to breakeven at $70\%$ of TP distance clips legitimate winners during normal retests. | $16$ winning trades stopped at $0.00R$; 8.3M $\sum R$ dropped from $+70.68R$ to $+65.82R$. | Kept single-stage $\text{SMA}_{60}$-bounded TP with a 60-minute smart-bracket time-stop. |

---

## 3. Econometric Validation Protocol

Standard i.i.d. $t$-statistics severely overstate significance in crypto markets where multiple pairs trigger simultaneously during liquidity shocks. To limit data-snooping, all base indicator parameters are frozen ex-ante ($df_{\text{ind}} = 0$, no grid-searches), while structural regime gates are evaluated via chronological ablation:

* **Rogers / Cameron–Gelbach–Miller Time-Clustered Inference:** Concurrent trades are grouped into non-overlapping 9-minute chronological clusters to account for cross-sectional correlation:

```math
\widehat{\text{Var}}_{\text{Clust}}(\bar{y}) = \frac{G}{G - 1} \cdot \frac{1}{N^2} \sum_{g=1}^{G} \left( \sum_{i \in g} (y_i - \bar{y}) \right)^2, \qquad t_{\text{Clust}} = \frac{\bar{y}}{\sqrt{\widehat{\text{Var}}_{\text{Clust}}(\bar{y})}}
```

* **Cluster-Block Bootstrapping:** Resampling $5{,}000\times$ with replacement at the *cluster* level ($G$ blocks) to derive non-parametric confidence intervals for $\mathbb{E}[R]$, Profit Factor, and maximum drawdown.
* **Fixed Chronological Splits:** Datasets are split strictly on fixed UTC calendar dates (H1 vs. H2) rather than sample-median timestamps (`entry_time.median()`). H1 initially served as an out-of-sample stress test that exposed latent covariate shift during March 2026, motivating the physical extension gates above.
* **Cluster-Level i.i.d. Verification:** Wald–Wolfowitz runs tests ($Z = +0.14, p = 0.8849$) and Ljung–Box ARCH tests ($Q(5)_{R^2} = 0.31, p = 0.9975$) on purified co-jump clusters confirmed serial independence—empirically refuting post-loss drawdown scaling shields in favor of constant $1.00\times$ base risk allocation.

---

## 4. Live Execution Architecture (`live_m26_daemon.py`)

Deployed on low-latency Tokyo (`ap-northeast-1`) VPS infrastructure, the daemon decouples historical feature computation from the real-time execution hot-path:

* **Two-Stage Incremental Pipeline (`<3 ms` Decision Hot-Path):**
  * **Background Cache (`:10.0s`):** A worker thread (`asyncio.to_thread`) pre-computes rolling 9m Wilder RSI/StochRSI states, $\text{ATR}_{14}$ baselines, and 7-day rolling volatility screens across 20 symbols on closed bars.
  * **O(1) Scalar Hot-Path (`:58.0s`):** At $T - 2.0\text{s}$ prior to candle close, only the forming 1-minute bar is evaluated incrementally in sub-3ms before dispatching concurrent orders.
* **L1 BBO Smart Order Chaser & Rate-Limited Engine:** Maintains a dedicated L1 best-bid-offer feed alongside 1m kline ringbuffers (`uvloop`, keep-alive TLS sessions, token-bucket rate limiters). Places spread-aware `POST_ONLY` orders with $60\text{ ms}$ fast-reject recovery, partial-fill accounting, and deterministic `IOC` fallback at $T + 1.5\text{s}$.
* **White-Box Telemetry & Crash Recovery:** Persists atomic JSON state snapshots (`positionId` reconciliation for hedge-mode tranches), SQLite WAL-mode trade logs, and per-gate audit trails for live-vs-backtest slippage attribution.
* **Tech Stack:** Python 3.12, `asyncio`, `uvloop`, `websockets`, `aiohttp`, NumPy, Pandas, SciPy, SQLite (WAL).

---

**Author:** Thijs Guillén Verheijen — BSc Econometrics & Operations Research, Erasmus School of Economics
