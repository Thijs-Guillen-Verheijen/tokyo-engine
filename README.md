# tokyo-engine

> **Note:** This repository documents the methodology, falsified hypotheses, and execution architecture of a personal systematic trading project. Signal generation, exact parameters, and order-routing code remain private while the system trades live.

---

## Overview

`tokyo-engine` is an event-driven research pipeline and live execution daemon for synchronized cross-sectional mean reversion across 12 liquid crypto perpetual futures. It runs 24/7 on a Tokyo (`ap-northeast-1`) VPS (~2–12 ms RTT to the exchange gateway) to keep round-trip latency low during tight bar-close execution windows.

Backtest and live execution share the same bar-close evaluation, VIP maker/taker fee schedule, adverse-selection modeling, and stop-loss slippage assumptions across an 8.3-month dataset (Feb–Oct 2026, 253.6 trading days).

---

## 1. Empirical Ablation & Historical Performance

Base indicator lookbacks were fixed ex-ante; structural regime filters were added sequentially to isolate each component's marginal impact. All results are evaluated across fixed H1 (Feb–Jun 2026) and H2 (Jun–Oct 2026) calendar windows, net of fees and conservative stop-loss slippage, normalized to $1.00R$ average risk per trade:

| Model Stage | Window | Trades ($N$) | Clusters ($G$) | Win Rate | $\mathbb{E}[R]_{1R}$ | $t_{\text{Clust}}$ | Profit Factor | $\sum R_{\text{H1}}$ | $\sum R_{\text{H2}}$ | $\sum R_{\text{Tot}}$ | Max DD ($R$) | Calmar |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **Stage 1** — Core Synchronized Baseline | 8.3M | 147 | 78 | 55.8% | +0.341R | +2.64 *** | 1.83 | +18.12R | +32.01R | **+50.13R** | 12.10R | 4.14 |
| **Stage 2** — + Non-Losing 2nd Tranche Scaling | 8.3M | 203 | 99 | 55.2% | +0.300R | +2.49 ** | 1.68 | +21.15R | +39.77R | **+60.92R** | 13.85R | 4.40 |
| **Stage 3** — + Climax Absorption Filter | 8.3M | 233 | 113 | 54.9% | +0.303R | +2.60 *** | 1.69 | +23.44R | +47.24R | **+70.68R** | 14.41R | 4.90 |

> **Methodological Notes:** In **Stage 2**, non-losing second tranches are logged as independent positions with their own exit brackets while keeping portfolio risk normalized to $1.00R$. Structural filters were refined in-sample on the Feb–Oct 2026 dataset, with forward performance tracked via live execution telemetry. Cluster-block bootstrapping (5,000x) for **Stage 3** yields $\mathbb{P}(\sum R > 0) = 99.46\%$.

---

## 2. Falsified Hypotheses

Standard mean-reversion assumptions broke down under correlated liquidity shocks and realistic fill modeling:

| Tested Hypothesis | Failure Mechanism | Empirical Evidence | Resolution |
|---|---|---|---|
| **Standardized Z-Score Triggers** | Shrinking rolling variance during quiet consolidations triggers premature entries on bar 1 of trend breakouts (denominator compression). | $-76.27R$ loss ($27.2\%$ WR) during the March 2026 regime shift. | Added trend-slope and volatility-normalized distance filters. |
| **Pure Passive (`POST_ONLY`) Execution** | Adverse selection: 100% fill rate on losing breakouts, missed fills on rapid V-shape bounces. | Up to 60 missed winners in a 4M stress test; WR dropped from $58.5\%$ to $52.5\%$ ($t_{\text{Clust}}: 3.06 \to 2.43$). | Built a hybrid router that chases L1 BBO passively before bar close with an `IOC` taker fallback. |
| **Single-Asset Signals** | Isolated spikes typically reflect asset-specific news or liquidation cascades rather than market-wide mean reversion. | $40.2\%$ WR, $\sum R = -41.03R$ across $N = 316$ isolated signals. | Required directional, synchronous multi-asset co-jumps across liquid core pairs. |
| **Breakeven Stop Locks** | Moving stops to breakeven mid-trade clips winners during normal price retests. | 16 winning trades stopped at $0.00R$; total return dropped from $+70.68R$ to $+65.82R$. | Kept a single-stage dynamic mean-reversion target paired with a fixed time-stop. |

---

## 3. Econometric Validation

Standard i.i.d. $t$-statistics overstate significance in crypto because multiple assets trigger simultaneously during liquidity shocks. The validation pipeline enforces four controls:

* **Cameron–Gelbach–Miller Time-Clustered Inference:** Concurrent trades are grouped into non-overlapping chronological clusters ($g \in \{1, \dots, G\}$) to account for cross-sectional correlation:

```math
\widehat{\text{Var}}_{\text{Clust}}(\bar{y}) = \frac{G}{G - 1} \cdot \frac{1}{N^2} \sum_{g=1}^{G} \left( \sum_{i \in g} (y_i - \bar{y}) \right)^2, \qquad t_{\text{Clust}} = \frac{\bar{y}}{\sqrt{\widehat{\text{Var}}_{\text{Clust}}(\bar{y})}}
```

* **Cluster-Block Bootstrapping:** 5,000x resampling with replacement at the cluster level ($G$ blocks) to estimate non-parametric confidence intervals for $\mathbb{E}[R]$, Profit Factor, and maximum drawdown.
* **Fixed Calendar Splits:** Partitioned on fixed UTC calendar dates (H1 vs. H2) rather than sample-median timestamps to avoid look-ahead contamination across volatility regimes.
* **Serial Independence Testing:** Wald–Wolfowitz runs tests ($Z = +0.14, p = 0.8849$) and Ljung–Box ARCH tests ($Q(5)_{R^2} = 0.31, p = 0.9975$) on co-jump clusters confirmed serial independence—refuting post-loss drawdown scaling in favor of constant $1.00\times$ risk sizing.

---

## 4. Execution Architecture

The daemon runs 24/7 on a Tokyo (`ap-northeast-1`) VPS, decoupling historical feature computation from order dispatch:

* **Two-Stage Feature Pipeline:** A background worker (`asyncio.to_thread`) pre-computes rolling multi-timeframe indicator states on closed bars, allowing the forming bar to be evaluated incrementally in `<3 ms` just before candle close.
* **Hybrid L1 BBO Entry Router:** Manages entries over a dedicated L1 best-bid-offer WebSocket feed (`uvloop`, keep-alive TLS, token-bucket rate limiters):
  1. **Passive Entry:** Posts `POST_ONLY` limit orders ahead of candle close to capture maker fees.
  2. **Re-Pegging & Partial Fills:** Catches immediate spread-cross rejects, re-pegs resting orders on price drift, and tracks partial fills in real time.
  3. **IOC Fallback:** Sweeps only the unfilled remainder via an `IOC` taker order if the passive window expires, bounded by a max-chase drift cap.
* **State & Telemetry:** Atomic JSON state snapshots for crash-safe position reconciliation, SQLite WAL-mode trade logs, and per-gate audit trails for live-vs-backtest slippage attribution.
* **Stack:** Python 3.12, `asyncio`, `uvloop`, `websockets`, `aiohttp`, NumPy, Pandas, SciPy, SQLite (WAL).

---

**Author:** Thijs Guillén Verheijen — BSc Econometrics & Operations Research, Erasmus School of Economics
