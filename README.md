# Estimating Market Impact From Position Accumulation

**Temporary price impact of aggressive orders in crypto spot markets, estimated from 100M+ trades across SOL-USDT and ETH-USDT.**

---

## Project Overview

When you need to buy $10M worth of ETH, you don't just hit the ask. The price moves against you. Understanding *how much* it moves, *as a function of what you're trying to do*, is the difference between a good fill and lighting money on fire.

This project estimates the temporary market impact function $I(V)$ from high-frequency trade data across multiple exchanges. We fit two competing models from the market microstructure literature, cross-validate them rigorously, and show that Bouchaud's square-root law holds remarkably well even in crypto, a market that barely existed when the theory was written.

The core question: if you sweep $V$ shares off the book, how far does the price move? And how quickly does it come back?

---

## Impact Model Specifications

We estimate two functional forms for the relationship between trade size $V_n$ and the subsequent return $r_n$:

**Square-root model** (Bouchaud et al.):

$$r_n \sim \mu_0 \sqrt{V_n}, \quad \mu_0 > 0$$

This is the "universal" impact law. The intuition: if resting limit orders are roughly uniformly distributed near the best price, eating through $V$ shares requires sweeping a price range proportional to $\sqrt{V}$. It is a remarkably clean result that falls out of the latent order book framework and has been documented across equities, futures, FX, and (as we show here) crypto.

**Power-law model** (general):

$$r_n \sim \mu_1 V_n^{\mu_2}, \quad \mu_1 > 0, \; 0 < \mu_2 < 1$$

This nests the square-root as the special case $\mu_2 = 0.5$. Estimating $\mu_2$ freely lets the data tell us whether square-root is right or whether the true concavity is different. Values of $\mu_2 < 0.5$ mean impact is even more concave than square-root (common in very liquid instruments); $\mu_2 > 0.5$ means impact is closer to linear (thinner books).

---

## Trade-Count Clock Design

A subtle but important design choice: we do not measure "future price" using wall-clock time ($\Delta t$). Instead, we use a **trade-count clock**:

$$r_n = \frac{P_{n+M}}{P_n} - 1$$

where $M$ is a number of trades, not seconds. This is natural for several reasons. Trade arrival is highly non-uniform: there might be 500 trades per second during a liquidation cascade and 2 trades per second at 3am UTC. Wall-clock intervals would mix very different information environments. The trade clock normalizes for activity and avoids the clock-skew issues inherent in multi-exchange data where recording timestamps do not perfectly align with exchange clocks.

We estimate impact across $M \in \{1, 5, 10, 25, 50, 100\}$ and show how the estimated parameters decay, confirming the temporary nature of the impact we are measuring.

---

## Data

High-frequency trade messages for **SOL-USDT** and **ETH-USDT** across multiple exchanges (Binance, OKX, and others), totaling 100M+ individual trades. The full dataset is 11GB+ in memory; we work with a 1/6 subsample (~17M trades) stored as Parquet files.

Each record contains a nanosecond-resolution timestamp, trade side (aggressive buyer or seller), quantity, price, and exchange identifier.

### Sweep Aggregation

Raw trade data contains many rows that are actually fragments of a single aggressive order sweeping through multiple resting limit orders. A trader submitting one market buy for 10 ETH might generate 15 separate fill records as it eats through successive price levels.

We aggregate these sweeps: consecutive same-side trades within 1ms on the same exchange are collapsed into a single parent trade with summed quantity and VWAP price. This recovers the economically meaningful unit, the aggressive order, from the mechanical fills.

---

## Methodology

### Estimation

For each pair x exchange x look-ahead $M$:

1. **Filter** to the quantile range $[0.80, 0.9975]$, focusing on the larger trades where impact is measurable while excluding the top 0.25% as potentially misleading outliers (crossed spreads, erroneous prints, etc.)

2. **Fit the square-root model** via constrained nonlinear least squares ($\mu_0 > 0$)

3. **Fit the power-law model** via constrained NLS ($\mu_1 > 0$, $0 < \mu_2 < 1$), plus a log-log OLS alternative for robustness

4. **Compute binned impact curves** (50 quantile bins) for visualization and model comparison

### Cross-Validation

Impact functions are not useful if they only describe history. We validate out-of-sample using two cohort strategies:

- **Leave-one-day-out**: train on all days except one, predict on the held-out day. This tests temporal stability: does yesterday's impact function predict today's?

- **Quantile-range holdout**: train on some trade-size bins, predict on others. This tests whether the functional form extrapolates across the size distribution, critical for practical use since execution algorithms need to predict impact for sizes they have not seen.

### Diagnostics

Beyond point estimates, we run full residual diagnostics (histogram + residual-vs-fitted), QQ plots for tail behavior, and separate the analysis by trade direction (buy vs sell) to confirm impact symmetry.

---

## Notebook Architecture

The analysis is structured as a single self-contained Jupyter notebook:

| Section | Content |
|---|---|
| **1-2** | Configuration, helper classes, all reusable utilities defined upfront |
| **3** | Data loading via Polars (fast), EDA with trade size distributions, price series, exchange breakdowns |
| **4** | Sweep aggregation with VWAP, per-exchange then combined |
| **5** | Forward returns on trade-count clock using `shift()`, signed by trade direction |
| **6** | Core estimation loop: per-exchange and combined, all $M$ values |
| **7** | Parameter tables for all estimated coefficients with standard errors and $R^2$ |
| **8.1** | Binned impact curves with both fitted models overlaid |
| **8.2** | Log-log plots with slope = 0.5 reference line |
| **8.3** | How the impact exponent and coefficient evolve with look-ahead $M$ |
| **8.4** | Side-by-side exchange comparison |
| **8.5** | Multi-$M$ overlay on same axes, the clearest view of impact decay |
| **8.6** | Return distributions: histogram + KDE + QQ + skew/kurtosis |
| **8.7** | Residual diagnostics for both models |
| **8.8** | Impact by quantile bucket (Q80-90, Q90-95, Q95-99.75) |
| **8.9** | Buy vs sell signed impact, confirming directionality |
| **9** | Cross-validation by day and by quantile range |
| **10** | Impact decay: mean absolute impact vs $M$ |
| **11** | Full interpretation of every visualization |
| **12** | Theoretical discussion: Bouchaud's framework, Kyle's lambda, limitations |

---

## Empirical Results

**The square-root law works.** Across both assets and multiple exchanges, the estimated power-law exponent $\mu_2$ clusters near 0.5, consistent with the Bouchaud prediction. This is striking given that these are crypto spot markets with very different microstructure than the equity markets where the law was originally documented.

**Impact is unambiguously temporary.** The multi-$M$ analysis and decay plots show a clear decline in estimated impact as we look further ahead in the trade clock. Most of the initial displacement reverts within 50-100 subsequent trades.

**Exchange liquidity matters for level, not shape.** Different exchanges show different *magnitudes* of impact (reflecting order book depth), but the *functional form* is consistent. The same $\sqrt{V}$ law fits whether you are trading on Binance or OKX, just with different coefficients.

**The data is extremely noisy at the individual trade level.** Raw $R^2$ values are low, which is expected. A single trade's return is dominated by noise from subsequent unrelated order flow. The signal emerges only in conditional expectations (binned averages), which is the correct way to think about impact functions. We are estimating $E[r|V]$, not predicting individual $r$.

---

## Reproducing the Analysis

```
project/
  data/
    ETH-USDT_fewer_trades.parquet
    SOL-USDT_fewer_trades.parquet
  HW7_Market_Impact_Estimation.ipynb
  README.md
```

Drop the Parquet files in `data/`, open the notebook, and run all cells. The notebook is designed to complete in under 30 minutes on the subsampled data.

### Dependencies

```
pandas numpy polars scipy matplotlib seaborn joblib scikit-learn
```

Standard scientific Python stack. Polars is used for fast Parquet I/O; everything else runs in Pandas.

---

## Extensions and Production Considerations

If this were a production system rather than an analysis project, the natural extensions would be:

**Dynamic impact models.** Impact is not stationary. It varies with volatility, time of day, and market regime. A model that conditions on recent realized volatility or bid-ask spread would be more accurate for real-time execution.

**Permanent vs temporary decomposition.** We measure total forward impact at various horizons, but a proper Almgren-Chriss decomposition would separate the temporary component (which decays) from the permanent component (which persists). This requires modeling the full impact decay curve, not just point estimates at discrete $M$ values.

**Cross-exchange impact propagation.** When a large trade hits Binance, how quickly does the price on OKX adjust? The lead-lag structure of cross-venue impact is directly relevant for execution algorithms that route across multiple venues.

**Transaction cost analysis integration.** Impact is one component of execution cost. A complete TCA framework would combine impact estimates with spread costs, timing risk, and opportunity cost to optimize execution schedules for real portfolio transitions.

---

## References

- Bouchaud, J.-P., Farmer, J.D., and Lillo, F. (2009). *How Markets Slowly Digest Changes in Supply and Demand.* Handbook of Financial Markets.
- Kyle, A.S. (1985). *Continuous Auctions and Insider Trading.* Econometrica, 53(6), 1315-1335.
- Almgren, R. and Chriss, N. (2001). *Optimal Execution of Portfolio Transactions.* Journal of Risk, 3(2), 5-39.
- Toth, B., Lemperiere, Y., Deremble, C., et al. (2011). *Anomalous Price Impact and the Critical Nature of Liquidity in Financial Markets.* Physical Review X, 1(2).
