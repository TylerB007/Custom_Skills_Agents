# LP Analytics Framework — Pool Selection & Position Performance

## Reference for Evaluating Pools Before Entry and Tracking Position Performance

---

## Table of Contents

### Part 1: Pool Selection
1. [Pool Fundamentals](#1-pool-fundamentals)
2. [Fee Economics](#2-fee-economics)
3. [Liquidity Depth Analysis](#3-liquidity-depth-analysis)
4. [Volatility Profile](#4-volatility-profile)
5. [Token & Contract Risk](#5-token--contract-risk)
6. [MEV & Execution Risk](#6-mev--execution-risk)
7. [Funding Rate Viability](#7-funding-rate-viability-delta-neutral-only)
8. [Pool Selection Scorecard](#8-pool-selection-scorecard)

### Part 2: Position Performance
9. [Core Performance Metrics](#9-core-performance-metrics)
10. [Time-Series Tracking](#10-time-series-tracking)
11. [Risk Metrics](#11-risk-metrics)
12. [Comparative Benchmarks](#12-comparative-benchmarks)
13. [Dashboard Layout](#13-dashboard-layout-recommendation)

---

# Part 1: Pool Selection Framework

Use this checklist to evaluate a pool **before** committing capital. Each category
has specific metrics with pass/caution/fail thresholds.

---

## 1. Pool Fundamentals

These metrics determine whether a pool has enough activity to sustain LP returns.

### TVL (Total Value Locked)

The pool's total capital. Affects your fee share and exit liquidity.

| Threshold | Signal |
|-----------|--------|
| > $10M | Strong — deep liquidity, low position impact |
| $1M – $10M | Adequate — viable for positions up to ~$100K |
| $100K – $1M | Caution — thin liquidity, significant price impact |
| < $100K | Avoid — insufficient depth, high slippage risk |

**Scaling rule:** Your position should generally be < 5% of pool TVL to avoid
dominating the liquidity at your tick range.

### Volume (24h and 7d Average)

Trading volume drives fee income. Look at both recent and average.

| Metric | How to Compute | What It Tells You |
|--------|---------------|-------------------|
| Daily volume | Sum of swap amounts over 24h | Current activity level |
| 7-day average daily volume | Sum of 7d volume / 7 | Smoothed activity (less noise) |
| Volume trend | Compare 7d avg to 30d avg | Growing or declining pool |

### Volume/TVL Ratio (Turnover)

The most important single metric for fee sustainability.

```
Daily Turnover = 24h Volume / TVL
```

| Ratio | Signal |
|-------|--------|
| > 0.3 | Excellent — high fee generation per unit of capital |
| 0.1 – 0.3 | Good — sustainable fee income |
| 0.03 – 0.1 | Marginal — fees may not justify risk |
| < 0.03 | Poor — capital is parked, not working |

### Pool Age

| Age | Signal |
|-----|--------|
| > 6 months | Established — stable fee history, predictable |
| 1 – 6 months | Maturing — check for declining volume trends |
| < 1 month | New — high uncertainty, may be promotional volume |

---

## 2. Fee Economics

### Historical Fee APR

Never trust a single snapshot. Compute rolling fee APR at multiple windows:

```
Fee APR = (Daily Fees / Active Liquidity in Range) × 365
```

| Window | Purpose |
|--------|---------|
| 7-day rolling | Current conditions, may be noisy |
| 30-day rolling | Recent trend, smoothed |
| 90-day rolling | Baseline expectation, seasonal patterns visible |

**Compare across windows:** If 7d >> 90d, current fees may be temporarily elevated
(event-driven volume). If 7d << 90d, the pool may be declining.

### Fee Income Stability

Compute the coefficient of variation (CV) of daily fee income:

```
CV = std(daily_fees) / mean(daily_fees)
```

| CV | Signal |
|----|--------|
| < 0.5 | Stable — predictable income |
| 0.5 – 1.0 | Moderate — some variability |
| > 1.0 | Volatile — fee income is spiky and unreliable |

### Fee Tier Appropriateness

Cross-reference the pool's fee tier against pair volatility:

| Pair Volatility (daily) | Appropriate Fee Tier | If Mismatched |
|-------------------------|---------------------|---------------|
| < 0.5% | 0.01% or 0.05% | Higher tier = LPs overcompensated, may attract competition |
| 0.5% – 2% | 0.05% or 0.30% | — |
| 2% – 5% | 0.30% | Lower tier = LPs undercompensated for risk |
| > 5% | 1.00% | Lower tier = avoid (IL will dominate fees) |

---

## 3. Liquidity Depth Analysis

### Liquidity Distribution

Fetch the pool's liquidity at each initialized tick to understand where capital
is concentrated.

**Key questions:**
- Where is the bulk of liquidity relative to the current price?
- Is there a "liquidity cliff" near your target range boundaries?
- How much of the total liquidity is within ±5% of the current price?

### Your Share of Active Liquidity

```
Your Fee Share = Your Liquidity / Total Active Liquidity at Current Tick
```

If your share is very small (< 0.1%), your fee income may be negligible.
If very large (> 10%), you're a dominant LP — good for fees but watch for
front-running and your own price impact on exit.

### Competitor LP Density

Check how many other LPs have positions overlapping your target range:

- **Dense competition** at popular ranges (e.g., ±5% around current price) means
  lower fee share per LP
- **Sparse competition** at wider or offset ranges means higher fee share but
  potentially lower fee capture (less volume crosses your range)
- **Optimal:** Find ranges with good volume flow but fewer competing LPs

**Data source:** Query position events from the subgraph or scan `mint` events
from the pool contract to see where other LPs are concentrated.

---

## 4. Volatility Profile

### Multi-Window Realized Volatility

Compute annualized volatility from historical price data:

```python
returns = [log(P[i] / P[i-1]) for i in range(1, len(prices))]
daily_vol = std(returns)
annual_vol = daily_vol * sqrt(365)
```

| Window | Purpose |
|--------|---------|
| 7-day | Current regime |
| 30-day | Recent baseline |
| 90-day | Historical norm |

### Volatility Regime Assessment

| Regime | Signal | LP Implication |
|--------|--------|----------------|
| 7d vol < 30d vol < 90d vol | Declining — calming market | Favorable: lower IL, lower gamma cost |
| 7d vol > 30d vol > 90d vol | Increasing — heating up | Caution: widen range or wait |
| 7d vol >> 90d vol (2x+) | Spike — extreme conditions | Avoid: gamma costs will dominate |
| All windows similar | Stable — normal conditions | Ideal for LP entry |

### Volatility vs Fee Tier Check

The pool's fee tier should compensate for the pair's volatility. A rough check:

```
Required Fee APR > ½ × |Γ| × σ² × P² × 365 / Position_Value
```

If the pool's actual fee APR is below this threshold, gamma costs will likely
exceed fee income regardless of range width.

---

## 5. Token & Contract Risk

### Smart Contract Risk

| Factor | Check | Red Flag |
|--------|-------|----------|
| Audit status | Has the pool's DEX been audited? | No audit, or audit > 2 years old |
| Time since deployment | How long has the DEX been live? | < 3 months |
| Fork fidelity | If a Uniswap V3 fork, are there modifications? | Significant custom logic beyond cosmetic |
| Upgrade authority | Can the contract be modified? | Admin can change fees, pause, or upgrade |
| Historical exploits | Has this DEX had security incidents? | Any exploit in the last 12 months |

### Token-Specific Risk

| Factor | Check | Red Flag |
|--------|-------|----------|
| Token contract audit | Is the token itself audited? | No audit, anonymous team |
| Admin/owner functions | Can supply be minted, transfers paused? | Unlimited mint, blacklist functions |
| Bridge risk | Is the token bridged from another chain? | Bridge has < $10M TVL or is < 6 months old |
| Centralization | Top holder concentration | Top 10 holders control > 50% of supply |
| Regulatory risk | Is the token likely to face regulatory action? | Securities classification concerns |

### Quick Token Risk Score

For each token in the pair, assign:
- **Low risk:** Battle-tested (ETH, WBTC, major stablecoins), audited, decentralized
- **Medium risk:** Established DeFi token, audited, some centralization
- **High risk:** New token, unaudited, centralized, or bridged from less-secure chain
- **Pair risk = max(token0_risk, token1_risk)** — the weakest link determines risk

---

## 6. MEV & Execution Risk

### Sandwich Attack Exposure

Concentrated liquidity positions can be targeted by sandwich bots, especially
during swaps that cross your tick range.

| Factor | Lower MEV Risk | Higher MEV Risk |
|--------|---------------|-----------------|
| Chain | L2 (Arbitrum, Base) with sequencer | Ethereum mainnet (public mempool) |
| Pool size | Large TVL | Small TVL |
| Position size | Small relative to pool | Large relative to pool |
| Range width | Wide | Narrow (concentrated = higher value target) |

### Price Impact at Your Position Size

Before entering, estimate the price impact of your deposit:

```
Impact ≈ Your Deposit / (2 × Active Liquidity at Current Tick × √P)
```

If impact > 0.5%, your entry alone moves the market meaningfully.

### Gas Cost Economics

Factor gas costs into your net yield calculation:

| Operation | Ethereum L1 | L2 (Arbitrum/Base) |
|-----------|-------------|-------------------|
| Add liquidity | $20 – $100 | $0.50 – $5 |
| Remove liquidity | $20 – $100 | $0.50 – $5 |
| Collect fees | $10 – $50 | $0.25 – $2 |
| Swap (rebalance tokens) | $15 – $80 | $0.30 – $3 |
| **Total round-trip** | **$65 – $330** | **$1.55 – $15** |

**Minimum position size rule:** Total gas for entry + exit should be < 1% of position.
On Ethereum L1: minimum ~$33K position. On L2: minimum ~$1.5K position.

---

## 7. Funding Rate Viability (Delta-Neutral Only)

Skip this section for unhedged LP strategies.

### Current Funding Assessment

| Metric | Source | What to Check |
|--------|--------|---------------|
| Current 8h funding | Exchange API | Positive = favorable |
| 7-day average | Historical data | Trend direction |
| 30-day average | Historical data | Baseline expectation |
| Cross-exchange spread | Multiple venues | Arb opportunity or consensus |

### Funding Sustainability

**Ask:** Will funding likely remain positive for the duration of my position?

| Signal | Interpretation |
|--------|---------------|
| Positive funding + rising open interest | Strong bullish sentiment — likely to persist |
| Positive funding + flat OI | Neutral — funding may mean-revert |
| Positive funding + falling OI | Warning — longs exiting, funding may flip |
| Negative funding | Unfavorable — wait for normalization |

### Net Yield Pre-Check

```
Estimated Net Yield = Fee APR + Funding APR - Gamma Cost APR - Gas/Trading Costs
```

If this is < 5% annualized, the risk-adjusted return may not justify the complexity
of running a delta-neutral strategy.

---

## 8. Pool Selection Scorecard

Use this template to score pools before entry. Rate each category 0-2:

| Category | 0 (Fail) | 1 (Caution) | 2 (Pass) |
|----------|----------|-------------|----------|
| **TVL** | < $100K | $100K – $1M | > $1M |
| **Turnover** | < 0.03 | 0.03 – 0.1 | > 0.1 |
| **Fee APR (30d)** | < 10% | 10% – 25% | > 25% |
| **Fee Stability (CV)** | > 1.0 | 0.5 – 1.0 | < 0.5 |
| **Volatility Regime** | Spiking | Increasing | Stable/declining |
| **Token Risk** | High | Medium | Low |
| **MEV Risk** | L1 + small pool | L1 + large pool | L2 |
| **Funding (if DN)** | Negative | Neutral | Positive + sustainable |

**Scoring:** 12-16 = Strong candidate. 8-11 = Proceed with caution. < 8 = Skip.

---

# Part 2: Position Performance Analytics

Metrics for tracking live and historical LP positions.

---

## 9. Core Performance Metrics

These are the most important numbers for any LP to track.

### ROI vs HODL

The single most important metric — did LP'ing outperform simply holding?

```
LP Return = (Current Position Value + Collected Fees - Initial Investment) / Initial Investment
HODL Return = (Initial Token Amounts at Current Prices - Initial Investment) / Initial Investment
LP vs HODL = LP Return - HODL Return
```

**Positive LP vs HODL** = LP was worth it. **Negative** = would have been better off holding.

### Fee Yield vs IL Ratio

```
Fee/IL Ratio = Total Fees Earned / |Impermanent Loss|
```

| Ratio | Interpretation |
|-------|---------------|
| > 2.0 | Excellent — fees far exceed IL |
| 1.0 – 2.0 | Positive — fees cover IL with surplus |
| 0.5 – 1.0 | Marginal — fees partially offset IL |
| < 0.5 | Poor — IL dominates, position is underwater |

### IL Recovery Rate

```
IL Recovery = Total Fees Earned / |Impermanent Loss| × 100%
```

100% = fees fully recovered IL. > 100% = net positive after IL.

### Net APR

```
Net APR = (Net P&L / Initial Investment) × (365 / Days Held)
```

Where Net P&L = Fees + Funding - IL - Hedge Costs - Gas - Rebalancing Costs.

### Win Rate

```
Win Rate = Days with Positive Net P&L / Total Days Active
```

For delta-neutral strategies, target > 60%. For unhedged LP, this is highly
dependent on price action.

---

## 10. Time-Series Tracking

Track these metrics over time to identify trends and regime changes.

### Fee APR Trend

Plot rolling 7-day fee APR over the position's lifetime. Look for:
- **Declining trend:** Pool activity dropping, consider exit
- **Spikes:** Event-driven volume (may not persist)
- **Stable:** Ideal conditions

### Cumulative P&L Decomposition

Stacked area chart showing cumulative contributions over time:
- Fee income (positive, growing)
- Funding income (positive or negative)
- Impermanent loss (negative, growing with price moves)
- Hedge P&L (offsets IL in delta-neutral)
- Rebalancing costs (negative, step function at each rebalance)
- **Net P&L** (sum of all — the line that matters)

This is the most diagnostic chart for understanding strategy health.

### Delta Drift Over Time

Plot net delta (LP delta - hedge size) over time. Should hover near zero for
delta-neutral strategies. Persistent drift indicates rebalancing is too infrequent.

### Funding Rate History

Plot the actual funding rates received/paid over the position's lifetime alongside
the cumulative funding P&L. Helps identify whether funding conditions have changed
since entry.

---

## 11. Risk Metrics

### Maximum Drawdown

```
For each day:
  Peak = max(Peak, Portfolio Value)
  Drawdown = (Peak - Portfolio Value) / Peak
Max Drawdown = max(all Drawdowns)
```

| Drawdown | Interpretation |
|----------|---------------|
| < 3% | Excellent risk control |
| 3% – 8% | Acceptable for delta-neutral |
| 8% – 15% | Concerning — review hedge effectiveness |
| > 15% | Strategy may be broken — investigate |

### Sharpe Ratio

```
Daily Returns = (V[t] - V[t-1]) / V[t-1]
Sharpe = mean(Daily Returns) / std(Daily Returns) × √365
```

| Sharpe | Interpretation |
|--------|---------------|
| > 2.0 | Excellent risk-adjusted return |
| 1.0 – 2.0 | Good |
| 0.5 – 1.0 | Mediocre — consider alternatives |
| < 0.5 | Poor — strategy not compensating for risk |

### Time in Range

```
Time in Range = Hours Price in [P_a, P_b] / Total Hours Active × 100%
```

| % In Range | Signal |
|------------|--------|
| > 90% | Excellent range selection |
| 70% – 90% | Good — some idle time |
| 50% – 70% | Marginal — range may be too narrow |
| < 50% | Poor — re-center or widen range |

### Realized vs Implied Volatility

```
Implied Vol ≈ sqrt(2 × Fee APR / Capital Efficiency)  [rough heuristic]
Realized Vol = actual price volatility over the period
```

If realized >> implied: gamma costs are higher than fees compensate for.
If realized << implied: favorable — low gamma cost, fees are generous.

### Capital Efficiency Utilization

```
Efficiency = Active Capital / Total Deployed Capital
```

Where Active Capital = LP capital earning fees + hedge margin in use.
Idle capital (margin buffer not currently needed) reduces efficiency.

---

## 12. Comparative Benchmarks

Always compare your strategy against alternatives:

| Benchmark | Formula | Purpose |
|-----------|---------|---------|
| **vs HODL** | LP Return - HODL Return | Was LP'ing worth the complexity? |
| **vs Lending** | LP Return - Aave/Compound APY | Was LP better than simple lending? |
| **Hedged vs Unhedged** | DN Return - Unhedged LP Return | Was the hedge worth the cost? |
| **vs Pool Average** | Your Fee APR - Pool Average Fee APR | Is your range optimized? |
| **Return per Unit of Risk** | Sharpe Ratio comparison | Risk-adjusted comparison |

### When to Exit Based on Benchmarks

- LP vs HODL negative for > 14 days → Range may need recentering
- LP vs Lending negative for > 30 days → Consider simpler strategy
- Hedged underperforming unhedged for > 7 days → Hedge may be miscalibrated
- Your fee APR < 50% of pool average → Reposition to denser tick range

---

## 13. Dashboard Layout Recommendation

When building an LP analytics dashboard, organize into these panels:

### Top Bar: Position Summary
- Position ID, pool, chain, tokens, fee tier
- Current price, range [P_a, P_b], in/out of range indicator
- Position age, total invested, current value

### Row 1: Key Metrics (cards)
- Net P&L ($, %) | ROI vs HODL (%) | Fee/IL Ratio | Net APR
- Time in Range (%) | Sharpe Ratio | Max Drawdown

### Row 2: P&L Attribution Chart
- Stacked bar or waterfall: Fees + Funding - IL - Hedge Costs - Gas = Net

### Row 3: Time Series (tabbed)
- Tab 1: Cumulative P&L decomposition (stacked area)
- Tab 2: Fee APR trend (rolling 7d line)
- Tab 3: Price path with range overlay
- Tab 4: Delta drift (for delta-neutral)

### Row 4: Risk & Alerts
- Current delta exposure
- Margin utilization (if hedged)
- Active alerts (near range edge, high delta drift, negative funding, etc.)

### Row 5: Comparison Table
- Side-by-side: LP Return vs HODL vs Lending vs Unhedged

---

## Data Sources for Analytics

| Metric | Source |
|--------|--------|
| Pool TVL, volume | Subgraph `pool` entity or DEX analytics API |
| Liquidity distribution | `pool.ticks()` on-chain or subgraph tick data |
| Historical prices | Swap events from subgraph or pool `slot0` snapshots |
| Fee APR | Computed from fee growth events or swap volume × fee tier |
| Funding rates | Exchange API (Coinbase, Binance, etc.) |
| Token risk | Manual assessment + on-chain contract analysis |
| Gas costs | Etherscan/block explorer gas price APIs |
| Position data | NonfungiblePositionManager on-chain reads |
