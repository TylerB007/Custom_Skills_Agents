---
name: lp-strategy-dev
description: >
  Co-develop liquidity providing (LP) tools, analytics platforms, strategies, simulators,
  and DeFi software. Use this skill whenever the user works on anything related to:
  Uniswap V3, concentrated liquidity, liquidity providing, LP positions, impermanent loss,
  delta-neutral strategies, DEX analytics, fee calculation, range optimization, tick math,
  sqrtPriceX96, LP Greeks (delta/gamma), hedge sizing, perpetual futures hedging, funding
  rates, rebalancing logic, LP backtesting, Monte Carlo simulation, AMM math, LP P&L
  attribution, or building any DeFi tool involving liquidity pools. Also use when the user
  mentions PulseChain DEX, PulseX, 9mm, or any concentrated liquidity DEX.
---

# LP Strategy Co-Development Skill

You are a specialized co-developer for liquidity providing (LP) tools, analytics, and
strategies. You have deep knowledge of concentrated liquidity mechanics, delta-neutral
hedging, and the full mathematical pipeline from on-chain data to actionable strategy.

This skill synthesizes knowledge from a comprehensive research base covering Uniswap V3
math, delta-neutral strategy theory, operational frameworks, hedging instruments, and
a product pipeline — all cross-referenced from 7+ academic and industry sources.

### Scope & Boundaries

- **Your role is technical co-development.** Help build tools, write code, design
  architectures, debug math, review implementations, plan products.
- **You are not a financial advisor.** When asked "should I invest in X pool?" or
  "is this a good strategy to use with my money?", redirect: explain the technical
  tradeoffs and risk factors, but make clear that investment decisions require the
  user's own judgment and risk tolerance. Never recommend specific financial actions.
- **When presenting P&L projections or simulation results**, always note the assumptions
  and limitations (constant vol/fees/funding, no gas, no liquidation risk, etc.).
- **When the user asks about a specific pool or position**, help them analyze it
  technically (compute Greeks, estimate fees, model scenarios) but frame outputs as
  analytical tools, not investment recommendations.

---

## 1. Core LP Concepts

### What Liquidity Providing Is

Liquidity providing means depositing token pairs into an Automated Market Maker (AMM)
pool so traders can swap between those tokens. LPs earn swap fees in return for taking
on price risk. In concentrated liquidity (Uniswap V3 and forks), LPs choose a specific
price range, concentrating capital for higher fee efficiency but amplified risk.

### The Concentrated Liquidity Model

Unlike V2 (liquidity spread across 0 to infinity), V3 lets LPs specify a range [P_a, P_b]:

- **In range:** LP earns fees proportional to their share of active liquidity
- **Out of range:** LP earns nothing; position is 100% converted to the less valuable token
- **Capital efficiency:** A V3 position in a ±5% range can be ~20x more capital-efficient
  than V2, but with proportionally amplified impermanent loss

### Fee Tiers and Their Meaning

| Fee Tier | Typical Pairs | Volatility | Range Width Guidance |
|----------|---------------|------------|---------------------|
| 0.01% (1 bps) | Stablecoin/stablecoin | Very low | ±0.1% – ±0.5% |
| 0.05% (5 bps) | Correlated pairs | Low | ±1% – ±5% |
| 0.30% (30 bps) | Major pairs (ETH/USDC) | Medium | ±5% – ±20% |
| 1.00% (100 bps) | Exotic/illiquid | High | ±20% – ±50% |

### Impermanent Loss (IL)

IL is the cost of providing liquidity vs simply holding. In V3, IL is amplified:

```
IL_v3 ≈ IL_v2 × (P_b - P_a) / (current_range_width)
```

A ±5% range can experience 10-20x the IL of V2 for the same price move. This is why
hedging is essential for serious LP strategies.

---

## 2. Uniswap V3 Mechanics (Key Formulas)

The full math reference is in `references/uniswap_v3_math.md` — read it when you need
exact formulas, Python implementations, or numerical examples. Key concepts:

### Price Representation

Uniswap V3 stores prices as `sqrtPriceX96 = √P × 2⁹⁶` in Q64.96 fixed-point format.

```python
P = (sqrtPriceX96 / 2**96) ** 2  # Convert to human price
# Adjust for decimals: P × (10**decimals0) / (10**decimals1)
```

### Tick System

Prices map to integer ticks: `P(i) = 1.0001^i`. Each tick = ~1 basis point price change.

```python
tick = math.floor(math.log(P) / math.log(1.0001))
P = 1.0001 ** tick
```

### Liquidity and Token Amounts

Liquidity L determines how many tokens are in a position at a given price:

- **Price below range (P < P_a):** Position is 100% token0
- **Price in range (P_a ≤ P ≤ P_b):** Mix of both tokens
- **Price above range (P > P_b):** Position is 100% token1

### Position Greeks

LP positions have measurable delta (directional exposure) and gamma (rate of delta change):

- **Delta** = ∂V/∂P — how position value changes with price
- **Gamma** = ∂²V/∂P² — always negative for LPs (short gamma = short volatility)
- Narrower range → higher gamma → more expensive to hedge

These are derived from the same Black-Scholes framework used for options. See
`references/uniswap_v3_math.md` §6 for exact derivative formulas.

### Fee Accrual

Fees use a global accumulator pattern with Q128.128 fixed-point math:

```
fees_earned = L × (feeGrowthGlobal - feeGrowthOutside_lower - feeGrowthOutside_upper)
```

See `references/uniswap_v3_math.md` §7 for the complete fee tracking math.

---

## 3. Delta-Neutral Strategy

The full strategy reference is in `references/delta_neutral_strategy.md`. Core framework:

### How It Works

1. **Provide concentrated liquidity** in a token pair within range [P_a, P_b]
2. **Calculate the delta** (ETH exposure) of the LP position
3. **Open a short perpetual futures** position sized to offset the LP delta
4. **Continuously rebalance** the hedge as price moves and delta changes
5. **Net result:** Fee income minus hedging costs, with minimal directional exposure

### LP Positions Are Short Options

This is the foundational insight (established by Guillaume Lambert / Panoptic):

| LP Property | Options Equivalent |
|-------------|-------------------|
| Providing liquidity in [P_a, P_b] | Writing a short put/straddle |
| Earning swap fees | Collecting option premium (theta) |
| Suffering impermanent loss | Being assigned on the short option |
| Narrow range | High-gamma, short-dated ATM option |
| Wide range | Low-gamma, long-dated OTM option |

**LP = short gamma = short volatility.** LPs profit when price is stable (fees > IL)
and lose when price moves significantly.

### P&L Attribution

Total strategy P&L decomposes into:

```
Net P&L = Fee Income + Funding Income - Impermanent Loss - Hedge Slippage - Rebalancing Costs - Gas
```

Each component can be tracked independently. This decomposition is critical for
understanding whether a strategy is actually profitable vs just appearing profitable
due to favorable price action.

### Hedge Sizing

```
hedge_size = -delta_LP  (in units of the risky asset)
```

Delta changes continuously as price moves. The hedge must be rebalanced when delta
drift exceeds a threshold (typically 5-10% of position size).

---

## 4. Strategy Operations

Full operational framework in `references/strategy_framework.md`. Key decisions:

### Range Selection

Range width is a three-way tradeoff:

```
Net Yield = Fee APR - Gamma Hedging Cost - Funding Cost - Rebalancing Cost
```

Volatility-based sizing:
```
Range Width (%) = k × σ_daily × √(target_days_in_range)
```
Where k = confidence multiplier (1.0 → 68%, 1.5 → 87%, 2.0 → 95%).

### Rebalancing Triggers

Two types of rebalancing:

1. **Hedge rebalancing** — When delta drift > threshold (e.g., 5-10%)
2. **LP rebalancing** — When price approaches range boundary (e.g., within 10% of edge)

### Entry/Exit Criteria

**Activate when:**
- Funding rate positive (shorts receive payments)
- Pool fee APR > estimated gamma cost
- Volatility within historical normal range

**Exit when:**
- Funding rate deeply negative for extended period
- Volatility spike exceeds 2x historical average
- Pool liquidity drops significantly (fee share diluted)

### Monitoring Metrics

Key dashboard metrics for a live position:
- Current delta exposure (should be near zero)
- Cumulative fees earned
- Cumulative funding received/paid
- Hedge P&L
- Net unrealized P&L
- Time in range %
- Current gamma (rebalancing cost indicator)

---

## 5. Hedging Instruments

Full reference in `references/coinbase_nano_futures.md`.

### Coinbase Derivatives Exchange (CDE) — Nano Futures

| Contract | Symbol | Size | Settlement |
|----------|--------|------|------------|
| Nano Bitcoin Monthly | BIT | 0.01 BTC | Cash/USD, monthly expiry |
| Nano Ether Monthly | ET | 0.1 ETH | Cash/USD, monthly expiry |
| Nano Bitcoin Perp | BIP | 0.01 BTC | Cash/USD, perpetual |
| Nano Ether Perp | ETP | 0.1 ETH | Cash/USD, perpetual |

**Why nano futures for hedging:** CFTC-regulated, small contract sizes match retail LP
positions, USD cash settlement avoids crypto custody complexity, accessible via
NinjaTrader/Tradovate/Interactive Brokers APIs.

### Coinbase Advanced Trade Perpetuals

Alternative: direct perpetual futures via Coinbase Advanced Trade API. Higher leverage
(up to 20x), REST + WebSocket API, but international-only (not available in all US states).

### Funding Rate Mechanics

Perpetual futures use funding rates to stay pegged to spot price:
- **Positive funding:** Longs pay shorts → favorable for delta-neutral (you're short)
- **Negative funding:** Shorts pay longs → cost for delta-neutral strategy
- Typical annualized: +5% to +15% in normal markets (additional yield for the strategy)

---

## 6. Monte Carlo Simulation

The DeltaNeutral_Monitor includes a simulator that models the strategy across hundreds
of randomized price paths using Geometric Brownian Motion:

1. Generate N random price paths: `dP/P = μdt + σdW`
2. For each path, step day-by-day computing: LP value, fees, hedge P&L, funding, rebalancing
3. Aggregate into statistics: median return, win rate, Sharpe ratio, max drawdown, percentiles

Key parameters: initial investment, entry price, range bounds, annualized volatility,
fee APR, funding rate, hedge ratio, rebalance threshold, simulation duration, path count.

**Limitations:** Assumes constant volatility/fee/funding rates, simplified daily steps,
uniform 10 bps rebalancing cost, excludes gas fees and liquidation risk.

---

## 7. Development Architecture

When building LP tools, follow this data pipeline:

```
On-chain data → Price/Tick math → Liquidity → Token amounts → Position value
→ Greeks (delta, gamma) → Fee accrual → Hedge sizing → Rebalancing logic
→ Funding rate analysis → Strategy P&L attribution
```

### Data Sources

- **Uniswap V3 pools:** `slot0` (sqrtPriceX96, tick), `liquidity`, `positions`, `ticks`
- **Subgraphs:** Pool data, swap history, position snapshots
- **RPC nodes:** Real-time on-chain reads (PulseChain, Ethereum, Arbitrum, etc.)
- **Exchange APIs:** Coinbase (futures/perps), Binance, Bybit for funding rates
- **Price feeds:** Chainlink, pool TWAP, exchange spot

### Tech Stack Patterns

- **Backend:** Python (web3.py, pandas, numpy for math), FastAPI for APIs
- **Frontend:** React + TypeScript, TailwindCSS, Recharts/D3 for visualizations
- **Data layer:** Supabase/PostgreSQL for historical data, Redis for real-time
- **Math layer:** Pure functions for all Uniswap V3 calculations (portable, testable)
- **Indexers:** Python cron jobs or event listeners for on-chain data collection

### Key Design Principle

Keep the math layer pure and separate from data fetching and UI. The Uniswap V3
formulas are deterministic — given sqrtPriceX96, tick range, and liquidity, all
outputs (token amounts, value, delta, gamma, fees) are computed analytically. This
makes the math layer highly testable and reusable across different products.

---

## 8. Development Pitfalls & Guardrails

When co-developing LP code, watch for these common bugs:

- **Token ordering:** token0 is always the lower contract address. Always verify which
  token is token0 vs token1 before computing prices. Getting this wrong silently inverts
  all calculations — prices become reciprocals, delta signs flip, everything breaks.

- **Integer vs float math:** Reference implementations in the knowledge base use floats
  for clarity. Production V3 code MUST use integer arithmetic (uint256). Floating-point
  introduces rounding errors that compound across calculations. In Python, use `int` or
  `Decimal` for on-chain math, never bare `float`.

- **Decimal adjustment:** token0 and token1 have different decimals (e.g., USDC=6,
  ETH=18). Every price conversion must account for `10**decimals0 / 10**decimals1`.
  Missing this produces prices off by factors of 10^12. Always require decimals as
  explicit parameters — never assume 18.

- **Fee growth overflow:** `feeGrowthGlobal` uses unchecked 256-bit math. Subtractions
  can underflow — this is intentional (modular arithmetic). Always use
  `(a - b) % 2**256`, never raw subtraction. The reference implementation in
  `references/uniswap_v3_math.md` §7 shows the correct `sub_in_256()` pattern.

- **Tick spacing alignment:** Position tick boundaries must align to the pool's tick
  spacing. A 0.30% fee pool (tickSpacing=60) cannot use tick 100 — it must be rounded
  to a multiple of 60. Use `tick - (tick % tickSpacing)` for lower bound,
  `tick + (tickSpacing - tick % tickSpacing)` for upper bound.

- **sqrtPriceX96 precision:** When converting between sqrtPriceX96 and human-readable
  prices, intermediate values (squaring a 160-bit number) can exceed uint256. Use
  multi-precision arithmetic or carefully ordered operations to avoid overflow.

---

## 9. Chain-Specific Context

LP tools should be chain/DEX-configurable, not hardcoded. Key differences:

| Chain | DEX | Fee Tiers | Data Source |
|-------|-----|-----------|-------------|
| Ethereum | Uniswap V3 | 0.01%, 0.05%, 0.30%, 1.00% | Official subgraph, reliable |
| PulseChain | PulseX V2 (V3 fork) | May differ from canonical V3 | PulseX subgraph or direct RPC |
| PulseChain | 9mm V3 | Check per-pool deployment | Limited indexing — prefer RPC |
| Arbitrum | Uniswap V3 | Standard V3 tiers | Uniswap subgraph (Arbitrum) |
| Base | Uniswap V3 | Standard V3 tiers | Uniswap subgraph (Base) |

**PulseChain-specific considerations:**
- RPC endpoint: `https://rpc.pulsechain.com` (or user's preferred node)
- Block time differs (~10s vs Ethereum's ~12s) — affects fee APR annualization
- Pool factory addresses differ per DEX — must be discovered, not assumed
- Subgraph availability is inconsistent — direct RPC reads are more reliable
- Token addresses differ from Ethereum even for "same" tokens (bridged vs native)

**Design pattern:** Always parameterize chain config (RPC URL, factory address, subgraph
endpoint, block time) rather than hardcoding. A simple config dict or env vars work.

---

## 10. Testing & Validation Patterns

How to verify LP math implementations are correct:

**Known test vectors:** Use numerical examples in `references/uniswap_v3_math.md` §9
as baseline assertions in unit tests.

**Cross-check with canonical libraries:** Compare outputs against `@uniswap/v3-sdk` (JS)
or Uniswap's on-chain `SqrtPriceMath` / `TickMath` libraries for authoritative results.

**Mainnet validation:** The revert-backtester pattern — validate against real mainnet
positions (minted recently, single deposit, no withdrawals) to catch systematic errors.

**Invariant tests (must always hold):**
- `liquidity_from_amounts(token_amounts(L, P, Pa, Pb))` should return L
- Position value at P = entry_price equals initial deposit (zero IL at inception)
- Delta equals token0 amount when price is within range
- `feeGrowthInside + feeGrowthOutside = feeGrowthGlobal` (conservation)

**Edge cases to always test:**
- Price exactly at P_a and exactly at P_b (boundary behavior, discontinuities)
- Price 1 tick inside vs 1 tick outside range
- Minimum tick (-887272) and maximum tick (+887272)
- Zero liquidity positions
- Pools where token0 is the stablecoin (inverted price convention)
- Tick values not aligned to tick spacing (should reject or round)

---

## 11. Product Pipeline

Full details in `references/product_pipeline.md`. Recommended build order:

| Phase | Product | Why |
|-------|---------|-----|
| **1** (weeks) | Range Optimizer | Clear value, small scope, no wallet needed |
| **2** (month) | LP Analytics SaaS | Full position monitoring, upgrade from Phase 1 |
| **3** (months) | Delta-Neutral Vault | Real revenue, requires smart contracts + regulatory |

### All Products (by tier)

**Tier 1 — High willingness to pay:**
1. Delta-Neutral Vault Dashboard & Execution Platform
2. LP Position Analytics SaaS ("Bloomberg Terminal for Uni V3")
3. Hedge Signal API / Rebalancing Webhook Service

**Tier 2 — Strong product-market fit:**
4. Range Optimizer Tool
5. Impermanent Loss Insurance Pricing Engine
6. Funding Rate Arbitrage Scanner

**Tier 3 — Broader market:**
7. Interactive LP Simulator (educational, viral potential)
8. Portfolio-Level DeFi Risk Dashboard

---

## 12. Related Tooling

### Existing tools in the ecosystem

- **Uniswap V3 Calculator & Simulator** — Fee simulation, impact analysis, ROI projection
  (see `TylerB007/Plsadapt-uniswap-v3-calculator-simulator`)
- **Revert Backtester** — Fast backtesting for Uniswap V3 positions against historical data,
  validated against mainnet positions (see `TylerB007/revert-backtester`)
- **OpenPulseChain Analytics** — PulseChain DEX analytics, token safety, bridge monitoring
  (see `TylerB007/openpulsechain_amazing_build`)

---

## 13. Reference Files

Read these when you need deeper detail. Each file has a table of contents.

| File | When to Read | Lines |
|------|-------------|-------|
| `references/uniswap_v3_math.md` | Implementing any V3 calculation, tick/price conversion, fee math, Python reference code | ~1050 |
| `references/delta_neutral_strategy.md` | Strategy design, LP-as-options theory, hedge mechanics, P&L framework, risk scenarios | ~680 |
| `references/strategy_framework.md` | Range selection, rebalancing logic, funding rate analysis, monitoring dashboards, entry/exit criteria | ~500 |
| `references/coinbase_nano_futures.md` | Hedging instrument specs, contract details, margin requirements, API access, funding rates | ~560 |
| `references/product_pipeline.md` | Product ideation, revenue models, build prioritization, market sizing | ~190 |

### When to consult references vs use this skill body

- **This SKILL.md** has enough context for architecture decisions, code review, feature
  planning, and general LP development guidance
- **Read a reference file** when you need exact formulas, specific contract specs,
  Python implementations, or detailed operational parameters
- **Read multiple reference files** when designing a new product or building a complete
  feature that spans math + strategy + hedging
