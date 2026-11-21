# tokyo-engine: Systematic Market Microstructure & Econometric Validation

> **Note:** This repository summarizes the econometric methodology, falsified hypotheses, and live execution architecture of a personal quantitative trading project. Because the system is actively traded live, the signal generation code, exact parameters, and order-routing logic are kept private.

---

## Overview

You might wonder why a student in Rotterdam named this project `tokyo-engine`. Surprise, surprise: the matching engines for Asian crypto perpetuals sit in Tokyo, and trying to win maker queue priority from a Dutch laptop is a losing game, so the daemon runs 24/7 on a Tokyo (`ap-northeast-1`) VPS.

`tokyo-engine` is an event-driven research framework and live execution daemon built to model, validate, and trade synchronized cross-sectional mean-reversion across 12 liquid crypto perpetual futures. Rather than treating backtesting and execution as separate worlds, the pipeline explicitly models **execution friction, adverse selection, and slippage** over an **8.3-month dataset** (Feb–Oct 2026, 253.6 trading days).

---

## 1. Empirical Ablation & Walk-Forward Performance

The table below shows the final synthesis of a 26-phase research process on the 12-asset core universe. Earlier phases focused on isolating the core market mechanism, falsifying naive assumptions (Section 2), and freezing base indicator parameters ex-ante ($df_{\text{ind}} = 0$). Each row adds one structural component to the baseline to isolate its exact marginal impact.

All results are evaluated across fixed H1 (Feb–Jun 2026) and H2 (Jun–Oct 2026) calendar windows, net of VIP taker/maker fees and conservative stop-loss slippage, normalized to $1.00R$ average risk per trade:

| Model Stage | Window | Trades ($N$) | Clusters ($G$) | Win Rate | $\mathbb{E}[R]_{1R}$ | $t_{\text{Clust}}$ | Profit Factor | $\sum R_{\text{H1}}$ | $\sum R_{\text{H2}}$ | $\sum R_{\text{Tot}}$ | Max DD ($R$) | Calmar |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **Stage 1** — Core Synchronized Baseline | 8.3M | 147 | 78 | 55.8% | +0.341R | +2.64 *** | 1.83 | +18.12R | +32.01R | **+50.13R** | 12.10R | 4.14 |
| **Stage 2** — + Non-Losing 2nd Tranche Scaling | 8.3M | 203 | 99 | 55.2% | +0.300R | +2.49 ** | 1.68 | +21.15R | +39.77R | **+60.92R** | 13.85R | 4.40 |
| **Stage 3** — + Climax Absorption Filter | 8.3M | 233 | 113 | 54.9% | +0.303R | +2.60 *** | 1.69 | +23.44R | +47.24R | **+70.68R** | 14.41R | 4.90 |

>  **Methodological Notes:** In **Stage 2**, non-losing 2nd tranches are logged as independent positions with their own exit brackets; all portfolio returns remain normalized to $1.00R$ average risk per trade. Non-parametric cluster-block bootstrapping (5,000x) for **Stage 3** yields $\mathbb{P}(\sum R > 0) = 99.46\%$.

---

## 2. What Failed: Microstructure & Falsified Hypotheses

Most textbook trading assumptions break down under correlated liquidity shocks and realistic fill modeling. Below is a log of hypotheses that failed during stress-testing and how they were resolved conceptually:

| Tested Hypothesis | Why It Failed | Empirical Evidence | Resolution |
|---|---|---|---|
| **Standardized Z-Score Triggers** | During quiet consolidations, shrinking rolling variance causes premature entries on the very first bar of a real trend breakout (covariate shift). | $-76.27R$ loss ($27.2\%$ WR) during the March 2026 regime shift. | Added ex-ante trend-displacement and volatility-normalized over-extension filters. |
| **Pure Passive (`POST_ONLY`) Execution** | Severe adverse selection: 100% fill rate on losing breakouts, while missing fills on rapid V-shape bounces. | Up to 60 missed winners in 4M stress test; WR dropped from $58.5\%$ to $52.5\%$ ($t_{\text{Clust}}: 3.06 \to 2.43$). | Built a **hybrid router** that actively chases L1 BBO prior to bar close with a deterministic `IOC` taker fallback. |
| **Single-Asset Signals** | Isolated spikes usually reflect asset-specific news or liquidation cascades rather than market-wide mean reversion. | $40.2\%$ WR, $\sum R = -41.03R$ across $N = 316$ isolated signals. | Required directional, synchronous multi-asset co-jumps across liquid core pairs. |
| **Breakeven Stop Locks** | Moving stops to breakeven mid-trade clips legitimate winners during normal price retests. | 16 winning trades stopped at $0.00R$; total return dropped from $+70.68R$ to $+65.82R$. | Kept a single-stage dynamic mean-reversion target paired with a fixed time-stop. |

---

## 3. Econometric Validation Protocol

Standard i.i.d. $t$-statistics severely overstate significance in crypto markets because multiple assets trigger simultaneously during liquidity shocks. The validation pipeline enforces four statistical safeguards:

* **Rogers / Cameron–Gelbach–Miller Time-Clustered Inference:** Concurrent trades are grouped into non-overlapping chronological clusters to account for cross-sectional correlation:

```math
\widehat{\text{Var}}_{\text{Clust}}(\bar{y}) = \frac{G}{G - 1} \cdot \frac{1}{N^2} \sum_{g=1}^{G} \left( \sum_{i \in g} (y_i - \bar{y}) \right)^2, \qquad t_{\text{Clust}} = \frac{\bar{y}}{\sqrt{\widehat{\text{Var}}_{\text{Clust}}(\bar{y})}}
```

* **Cluster-Block Bootstrapping:** Resampling 5,000x with replacement at the *cluster* level ($G$ blocks) to derive non-parametric confidence intervals for $\mathbb{E}[R]$, Profit Factor, and maximum drawdown.
* **Fixed Calendar Splits:** Datasets are split on fixed UTC calendar dates (H1 vs. H2) rather than sample-median timestamps, preventing look-ahead contamination across shifting volatility regimes.
* **Serial Independence & Risk-Shield Ablation:** Wald–Wolfowitz runs tests ($Z = +0.14, p = 0.8849$) and Ljung–Box ARCH tests ($Q(5)_{R^2} = 0.31, p = 0.9975$) on purified co-jump clusters confirmed that cluster outcomes are serially independent—empirically refuting post-loss drawdown scaling shields in favor of constant $1.00\times$ risk allocation.

---

## 4. Live Execution Architecture

The production daemon runs 24/7 on Tokyo (`ap-northeast-1`) VPS infrastructure (~2–12 ms RTT to the Asian API gateway), decoupling heavy historical feature computation from the real-time execution path:

* **Two-Stage Incremental Pipeline (`<3 ms` Decision Hot-Path):** A background worker thread (`asyncio.to_thread`) pre-computes rolling multi-timeframe indicator states on closed bars, allowing the live forming bar to be evaluated incrementally in sub-3ms right before candle close.
* **Active L1 BBO Order Chaser:** Rather than firing naive market orders on candle close, the daemon manages entries through a three-step execution loop over a dedicated L1 best-bid-offer WebSocket feed (`uvloop`, keep-alive TLS, token-bucket rate limiters):
  1. **Spread-Aware Entry:** Places passive `POST_ONLY` limit orders just ahead of candle close to capture maker fees and queue priority.
  2. **Dynamic Re-Pegging:** Catches immediate spread-cross rejects, re-pegs resting orders on price drift, and accounts for partial fills in real time.
  3. **Deterministic Fallback:** Sweeps only the unfilled remainder via an `IOC` taker order if the passive window expires, guarded by a flash-spike circuit breaker.
* **Telemetry & Crash Recovery:** Persists atomic JSON state snapshots (per-tranche position reconciliation), SQLite WAL-mode trade logs, and per-gate audit trails for live-vs-backtest slippage attribution.
* **Tech Stack:** Python 3.12, `asyncio`, `uvloop`, `websockets`, `aiohttp`, NumPy, Pandas, SciPy, SQLite (WAL).

---

**Author:** Thijs Guillén Verheijen — BSc Econometrics & Operations Research, Erasmus School of Economics
