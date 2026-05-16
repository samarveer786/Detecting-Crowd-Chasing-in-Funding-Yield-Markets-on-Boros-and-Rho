# Detecting Crowd-Chasing in Funding Yield Markets
### Research → Backtest → Live Deployment

> This repository contains the research and signal discovery process behind a live yield derivatives strategy on Boros Protocol that generated **175.5% ROI on $600 capital over 102 days** (Feb–May 2026). The strategy detects expectation failures between implied funding APR embedded in Boros Yield Units and realized perpetual funding rates.

**Live Strategy Results:** [boros-yield-strategy](../boros-yield-strategy/) — $1,053 net PnL | Sharpe 5.21 | Calmar 28.68

---

## Core Hypothesis

Forward yield markets adjust expectations with **inertia**. During leverage expansions and liquidation cascades, realized funding moves faster than implied APR — creating repeatable, predictable mispricings.

```
Gap(t) = implied_APR(t,T) − realized_funding(t)
```

When this gap becomes abnormally wide or narrow (measured by z-score), the market is either:
- **Crowd-chasing** (funding > implied) → Long YU — market will mean-revert down
- **Panic / funding collapse** (implied > funding) → Short YU — market overpriced expectations

---

## Signal Construction

```python
# Gap between Boros implied APR and Binance perpetual funding APR
gap = implied_apr - funding_apr

# EMA-normalized z-score (removes trend, captures deviation)
T = 10  # EMA window (days)
S = 30  # Rolling std window
gap_mean = gap.ewm(span=T, min_periods=T).mean()
gap_std  = gap.rolling(S, min_periods=S).std()
zscore   = (gap - gap_mean) / gap_std
```

### Trading Logic

| Z-Score | Market Condition | Action |
|---------|-----------------|--------|
| Z < −1.0 | Funding too high vs implied (crowd-chasing) | Long YU |
| Z > +1.0 | Implied too high vs funding (panic/collapse) | Short YU |
| \|Z\| < 0.05 | Mean reversion complete | Exit |

### Safety Filters

```python
# Block LONG if funding is weak or collapsing
block_long = (funding < 0) or (funding < MA3_funding and funding < 0.5 * implied)

# Block SHORT if funding has already blown out
block_short = (funding > 1.5 * implied)

# Funding shock stop-loss
shock_exit = abs(funding_today) > abs(funding_at_entry) * 2

# Curve blowoff protection (SHORT only)
curve_blowoff = implied_today > entry_implied * 1.33

# Max holding period
max_days = 90
```

---

## Research Evolution (4 Scripts)

### V1 — Simulated Data (`SimulatedData_Yield_momentum_Model_BorosYU.py`)
**Purpose:** Prove the signal works in theory before touching real data.

Synthetic implied APR simulated using:
```python
implied(T) = EMA14(funding) + risk_premium(Markov) + 0.6*sqrt(T) + noise
```
3-state Markov crowding regime: `risk_on (+3.5%) | neutral (0%) | risk_off (-2%)`

Validated that the gap z-score generates tradeable signals across 30d/90d/180d maturities.

### V1 — Empirical Data (`EmpiricalData_Yield_momentum_Model_BorosYU-2.py`)
**Purpose:** Replace simulated implied APR with real Boros on-chain data.

- Loaded `BTC_Boros_Daily_Implied_WDATE.csv` — actual Boros protocol implied APR per market address
- Merged with Binance BTC perpetual funding rate history
- First empirical validation: signal holds on real data (Jul–Dec 2025)

### V2 — Simulated Data (`V2_SimulatedData_Yield_momentum_Model_BorosYU.py`)
**Purpose:** Improve PnL model to match how Boros actually settles.

Key improvement: **mark-to-market repricing of remaining cashflow stream on exit**
```python
# When exiting before maturity, reprice the remaining days
repricing_pnl = direction * notional * (implied_exit - implied_entry) * (remaining_days / 365)
```
This correctly captures the fixed-vs-floating cashflow structure of Boros YU instruments.

### V2 — Empirical Data (`V2_EmpiricalData_Yield_momentum_Model_BorosYU.py`)
**Purpose:** Final model on real data with all V2 improvements.

Additional risk layer: **curve blowoff protection** for SHORT_YU positions — exits if implied APR rises >33% above entry implied, protecting against convex losses in rising-rate environments.

---

## PnL Attribution Model

```python
# Daily cashflow for an open position
if position == LONG_YU:
    cashflow = notional * (realized_today - entry_implied) * (1/365)
else:  # SHORT_YU
    cashflow = notional * (entry_implied - realized_today) * (1/365)

# On exit before maturity: mark-to-market repricing
repricing_pnl = direction * notional * (implied_exit - entry_implied) * (remaining_days/365)
```

---

## From Research to Live

This research directly informed the live Boros strategy:

| Research Finding | Live Implementation |
|-----------------|---------------------|
| Z-score gap signal works on real data | Core entry signal in live bot |
| Funding shock stop-loss | `abs(funding) > 2x entry_funding` → force exit |
| Crowd-chasing regime detection | Regime classification in signal engine |
| 90-day maturity horizon | Position sizing and holding period limits |
| Block-long filter | Safety filter in live execution |

**Research period:** Jul–Dec 2025 (backtested on real Boros data)
**Live deployment:** Feb 2026 → present
**Live result:** $1,053 PnL on $600 capital (175.5% ROI, 102 days)

---

## Data Sources

| File | Source |
|------|--------|
| `BTC_Boros_Daily_Implied_WDATE.csv` | Boros Protocol on-chain data (self-collected) |
| `binance_btc_funding_daily_2020.csv` | Binance BTC perpetual funding rate history |

---

## Technology Stack

Python · NumPy · Pandas · Matplotlib

---

## Repository Structure

```
├── SimulatedData_Yield_momentum_Model_BorosYU.py    # V1 — Proof of concept
├── EmpiricalData_Yield_momentum_Model_BorosYU-2.py  # V1 — Real data validation
├── V2_SimulatedData_Yield_momentum_Model_BorosYU.py # V2 — Improved PnL model
├── V2_EmpiricalData_Yield_momentum_Model_BorosYU.py # V2 — Final production model
└── README.md
```

---

## Related Work

- [boros-yield-strategy](../boros-yield-strategy/) — Live deployment of this strategy (175.5% ROI)
- [crypto-stat-arb-engine](../crypto-stat-arb-engine/) — Live Binance Futures stat arb engine
- [solana-yield-vault](../solana-yield-vault/) — Regime-based DeFi vault design
