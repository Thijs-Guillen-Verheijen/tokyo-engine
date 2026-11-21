# tokyo-engine: Systematic Market Microstructure & DGP Validation

**Note:** *This repository serves as a methodological overview of a proprietary econometric research project and live execution telemetry daemon. The underlying Python engine, order-routing logic, and specific alpha parameters are kept private.*

## Overview
`tokyo-engine` is a systematic research framework designed to model, test, and execute directional mean-reversion and co-jump behavior across crypto perpetual futures. The primary focus is strict econometric rigor: treating market microstructure, latency, and regime-shifts as integral parts of the statistical validation process.

## 1. Econometric Rigor & Out-of-Sample Validation
Standard $t$-statistics fail in highly correlated cross-sectional markets. This framework enforces severe constraints over an 8.3-month dataset to prevent curve-fitting and false discoveries:

*   **Time-Clustered Inference:** Significance is evaluated using Rogers time-clustered standard errors, grouping concurrent trades into independent 9-minute clusters to measure true edge.

$$
\text{Var}_{\text{Clust}}(\bar{y}) = \frac{G}{G-1} \cdot \frac{1}{N^2} \sum_{g=1}^{G} \left( \sum_{i \in g} (y_i - \bar{y}) \right)^2, \quad t_{\text{Clust}} = \frac{\bar{y}}{\sqrt{\text{Var}_{\text{Clust}}(\bar{y})}}
$$

*   **Non-Parametric Block Bootstrapping:** System robustness is verified through 5,000x cluster-block bootstrapping to establish strict confidence intervals for the expected value ($\mathbb{E}[R]$) and maximum drawdown.
*   **Fixed Walk-Forward Splits:** The dataset relies on strict chronological boundaries rather than sample-median splits, preventing the contamination of in-sample data by extreme volatility regimes (e.g., the March 2026 crash).

## 2. Microstructure & Falsified Hypotheses
An unexecuted signal is merely a hypothesis. Through iterative execution telemetry, several retail-level assumptions were falsified and physically corrected:

*   **Latent Covariate Shift (Denominator Compression):** Early iterations suffered during sudden market consolidations due to a tightening variance $\sigma_{60}$ triggering premature $Z$-score entries. This was resolved by implementing a physical 30-minute slope and $\text{ATR}_{14}$ extension gate ex-ante.
*   **Adverse Selection in Pure Maker Execution:** Relying exclusively on passive `POST_ONLY` orders resulted in a 100% fill-rate on losing trades, but significant missed fills on rapid V-shape recoveries.
*   **AS4 Hybrid Order Routing:** The production architecture solves this through a hybrid execution model, deploying a smart `POST_ONLY` chaser at $T-2.0\text{s}$ with an automated taker-fallback at $T+1.5\text{s}$ based on queue-penetration metrics.

## 3. Production Architecture
The live telemetry daemon (`live_m26_daemon.py`) is engineered for sub-100ms latency out of Tokyo/Singapore VPS infrastructure, featuring:
*   **Asynchronous Feature Building:** Vectorized calculation of all 9-minute and 1-minute state-features across a 20-symbol universe.
*   **L1 BBO Smart Chasing:** Real-time best-bid-offer tracking to dynamically re-peg resting orders before candle-close.
*   **Tech Stack:** `Python 3.12`, `asyncio`, `uvloop`, `websockets`, `Pandas`, `NumPy`.

---
*Author: Thijs Guillén Verheijen — BSc Econometrics & Operations Research, Erasmus School of Economics*
