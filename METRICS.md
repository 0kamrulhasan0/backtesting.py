# Backtesting.py Metrics Reference

Complete documentation of all metrics returned by `Backtest.run()` and `Backtest.optimize()`, with mathematical formulas, interpretations, and Bangladeshi market (DSE) examples.

---

## Table of Contents

1. [API Reference](#api-reference)
2. [Period Metrics](#period-metrics)
3. [Performance Metrics](#performance-metrics)
4. [Risk & Drawdown Metrics](#risk--drawdown-metrics)
5. [Annualized Metrics](#annualized-metrics)
6. [Risk-Adjusted Metrics](#risk-adjusted-metrics)
7. [Trade Quality Metrics](#trade-quality-metrics)
8. [Activity Metrics](#activity-metrics)
9. [Bangladeshi Market Context](#bangladeshi-market-context)

---

## API Reference

### Core Classes

#### `Backtest`
```python
class Backtest:
    def __init__(self, data: pd.DataFrame, strategy: Type[Strategy], *,
                 cash: float = 10_000,
                 spread: float = 0.0,
                 commission: Union[float, Tuple[float, float]] = 0.0,
                 margin: float = 1.0,
                 trade_on_close: bool = False,
                 hedging: bool = False,
                 exclusive_orders: bool = False,
                 finalize_trades: bool = False)
```

#### `ConstrainedBacktest` (for DSE/regulated exchanges)
```python
class ConstrainedBacktest(Backtest):
    def __init__(self, data: pd.DataFrame, strategy: Type[Strategy], *,
                 cash: float = 1_000_000,
                 commission: Tuple[float, float] = (0, 0.0005),
                 lot_size: int = 1,
                 price_limit_pct: float = 0.10,
                 settlement_days: int = 2,
                 **kwargs)

    @classmethod
    def from_data_source(cls, ticker: str, strategy: Type[Strategy], *,
                         period: str = '5y',
                         data_source=None, **kwargs)
```

#### `Strategy` (base class)
```python
class Strategy:
    def init(self) -> None:          # Override: declare indicators
    def next(self) -> None:          # Override: main logic per bar
    def I(self, func, *args, **kwargs) -> np.ndarray:  # Declare indicator
    def buy(self, *, size, limit=None, stop=None, sl=None, tp=None, tag=None) -> Order
    def sell(self, *, size, limit=None, stop=None, sl=None, tp=None, tag=None) -> Order  # Not allowed in ConstrainedBacktest

    @property
    def equity(self) -> float
    @property
    def data(self) -> _Data
    @property
    def position(self) -> Position
    @property
    def orders(self) -> Tuple[Order]
    @property
    def trades(self) -> Tuple[Trade]
    @property
    def closed_trades(self) -> Tuple[Trade]
```

#### `Position`
```python
class Position:
    @property
    def size(self) -> float          # Positive = long
    @property
    def pl(self) -> float            # Unrealized P&L in BDT
    @property
    def pl_pct(self) -> float        # Unrealized P&L %
    @property
    def is_long(self) -> bool
    def close(self, portion: float = 1.0) -> None
```

#### `Order`
```python
class Order:
    @property
    def size(self) -> float          # Positive = long, negative = short
    @property
    def limit(self) -> float
    @property
    def stop(self) -> float
    @property
    def sl(self) -> float
    @property
    def tp(self) -> float
    @property
    def is_long(self) -> bool
    @property
    def is_short(self) -> bool
    @property
    def is_contingent(self) -> bool
    def cancel(self) -> None
```

#### `Trade`
```python
class Trade:
    @property
    def size(self) -> int
    @property
    def entry_price(self) -> float
    @property
    def exit_price(self) -> float
    @property
    def entry_time(self) -> pd.Timestamp
    @property
    def exit_time(self) -> pd.Timestamp
    @property
    def entry_bar(self) -> int
    @property
    def exit_bar(self) -> int
    @property
    def pl(self) -> float
    @property
    def pl_pct(self) -> float
    @property
    def value(self) -> float
    @property
    def is_long(self) -> bool
    @property
    def is_short(self) -> bool
    @property
    def sl(self) -> float
    @sl.setter
    def sl(self, price: float)
    @property
    def tp(self) -> float
    @tp.setter
    def tp(self, price: float)
    def close(self, portion: float = 1.0) -> None
```

---

## Period Metrics

### `Start`
**Definition**: First timestamp in the data.
```python
s['Start'] = index[0]
```

### `End`
**Definition**: Last timestamp in the data.
```python
s['End'] = index[-1]
```

### `Duration`
**Definition**: Total calendar time covered by the backtest.
```python
s['Duration'] = s['End'] - s['Start']
```

**Bangladeshi Example**:
- Data: SQURPHARMA daily from `2020-01-01` to `2024-12-31`
- `Duration` = `1826 days 00:00:00` (5 years)

---

## Performance Metrics

### `Equity Final [৳]`
**Definition**: Account equity at the end of the backtest (cash + position value).
```python
s['Equity Final [$]'] = equity[-1]
```

**Bangladeshi Example**:
- Initial cash: ৳10,00,000 (10 lakh)
- Final equity: ৳14,50,000
- Profit: ৳4,50,000 (45%)

### `Equity Peak [৳]`
**Definition**: Highest equity value reached during the backtest.
```python
s['Equity Peak [$]'] = equity.max()
```

**Bangladeshi Example**:
- Equity reached ৳16,00,000 in March 2023 during bull run
- Later declined to ৳14,50,000 at end
- Peak used for drawdown calculations

### `Commissions [৳]`
**Definition**: Total commission paid across all trades (entry + exit).
```python
commissions = sum(t._commissions for t in trades)
```

**Bangladeshi Example**:
- DSE commission: 0.05% per trade (৳50 per ৳1,00,000)
- 20 trades × ৳1,00,000 avg size = ৳10,000 total commission
- Reduces net profit significantly for high-frequency strategies

### `Return [%]`
**Definition**: Total percentage return on initial capital.
```python
s['Return [%]'] = (equity[-1] - equity[0]) / equity[0] * 100
```

**Formula**: 
$$\text{Return \%} = \frac{\text{Equity Final} - \text{Initial Cash}}{\text{Initial Cash}} \times 100$$

**Bangladeshi Example**:
- Start: ৳10,00,000
- End: ৳14,50,000
- Return = (4,50,000 / 10,00,000) × 100 = **45%**

### `Buy & Hold Return [%]`
**Definition**: Return from simply holding the asset (long-only) from first trading bar.
```python
first_trading_bar = _indicator_warmup_nbars(strategy_instance)
c = ohlc_data.Close.values
s['Buy & Hold Return [%]'] = (c[-1] - c[first_trading_bar]) / c[first_trading_bar] * 100
```

**Bangladeshi Example**:
- SQURPHARMA: ৳200 → ৳250 over 5 years = **25%**
- Your strategy: 45% → **Outperformed buy & hold by 20%**
- If strategy < buy & hold → strategy added no value

### `Exposure Time [%]`
**Definition**: Percentage of bars where a position was open (time in market).
```python
have_position = np.repeat(0, len(index))
for t in trades_df[['EntryBar', 'ExitBar']].itertuples(index=False):
    have_position[t.EntryBar:t.ExitBar + 1] = 1
s['Exposure Time [%]'] = have_position.mean() * 100
```

**Bangladeshi Example**:
- 1,826 trading days in 5 years
- Position open for 1,500 days
- Exposure = (1500/1826) × 100 = **82.1%**
- High exposure = more market risk, low = timing the market

---

## Risk & Drawdown Metrics

### `Max. Drawdown [%]`
**Definition**: Largest peak-to-trough decline in equity.
```python
dd = 1 - equity / np.maximum.accumulate(equity)
max_dd = -np.nan_to_num(dd.max())
s['Max. Drawdown [%]'] = max_dd * 100
```

**Step-by-Step Mathematical Breakdown**:

1. **`np.maximum.accumulate(equity)`** — Running peak (high-water mark)
   - Returns an array where each element is the maximum of all previous equity values up to that point
   - `accumulate` vs `max`: `np.maximum()` is element-wise between two arrays; `accumulate()` is cumulative max along a single array (expanding window, no fixed size)
   - Example: equity = [100, 110, 105, 108, 115] → peaks = [100, 110, 110, 110, 115]

2. **`equity / peak`** — Current equity as fraction of peak
   - 1.0 = at all-time high
   - 0.8 = 20% below peak
   - Example: equity=105, peak=110 → 105/110 = 0.9545

3. **`1 - equity/peak`** — Drawdown fraction
   - 0 = at peak (no drawdown)
   - 0.2 = 20% drawdown
   - Example: 1 - 0.9545 = 0.0455 (4.55% drawdown)

4. **`np.nan_to_num(dd.max())`** — Handle edge cases
   - Converts NaN to 0 (happens if equity starts at 0 or is all zeros)
   - `max()` finds the worst drawdown across entire period

5. **`-max_dd * 100`** — Convert to positive percentage
   - Drawdown is negative by convention; we report as positive %

**Formula**: 
$$\text{Max DD \%} = \max\left(\frac{\text{Peak Equity} - \text{Current Equity}}{\text{Peak Equity}}\right) \times 100$$

**Why `accumulate`?**
- `np.maximum()` = element-wise max between **two arrays**
- Rolling max = fixed **window** (looks back N periods only)
- **`accumulate()` = expanding window**, full history (no window limit)
- Drawdown needs **all-time high to date**, not a 20-day high

**Bangladeshi Example**:
- Equity peaked at ৳16,00,000 (March 2023)
- Fell to ৳12,00,000 (October 2023)
- Max DD = (16,00,000 - 12,00,000) / 16,00,000 × 100 = **25%**
- DSE context: 2020 COVID crash saw 30-40% drawdowns

### `Avg. Drawdown [%]`
**Definition**: Average of all drawdown peaks.
```python
s['Avg. Drawdown [%]'] = -dd_peaks.mean() * 100
```

**Bangladeshi Example**:
- Drawdown peaks: 10%, 25%, 15%, 5%
- Average = (10+25+15+5)/4 = **13.75%**

### `Max. Drawdown Duration`
**Definition**: Longest time (calendar days) to recover from a drawdown.
```python
s['Max. Drawdown Duration'] = _round_timedelta(dd_dur.max())
```

**Bangladeshi Example**:
- Drawdown started March 2023 (৳16L peak)
- Recovered January 2024 (back to ৳16L)
- Duration = **10 months**
- DSE bear markets can last 1-3 years

### `Avg. Drawdown Duration`
**Definition**: Average recovery time across all drawdowns.
```python
s['Avg. Drawdown Duration'] = _round_timedelta(dd_dur.mean())
```

---

## Annualized Metrics

### `Return (Ann.) [%]`
**Definition**: Compound annual growth rate based on daily returns.
```python
gmean_day_return = geometric_mean(day_returns)
annualized_return = (1 + gmean_day_return)**annual_trading_days - 1
s['Return (Ann.) [%]'] = annualized_return * 100
```

**Formula** (compounding):
$$\text{Annual Return} = (1 + \text{Daily Geometric Mean})^{\text{Trading Days}} - 1$$

**Bangladeshi Example**:
- Daily geometric mean return: 0.05%
- DSE trading days: ~240/year (excluding weekends/holidays)
- Annual = (1.0005)^240 - 1 = **12.7%**

### `Volatility (Ann.) [%]`
**Definition**: Annualized standard deviation of daily returns (compounding-aware).
```python
s['Volatility (Ann.) [%]'] = np.sqrt(
    (day_returns.var(ddof=1) + (1 + gmean_day_return)**2)**annual_trading_days
    - (1 + gmean_day_return)**(2 * annual_trading_days)
) * 100
```

**Mathematical Breakdown** (compounding-aware formula):

The standard `std × √N` assumes simple returns. For log/compound returns:

1. **`day_returns.var(ddof=1)`** — Sample variance of daily returns
   - `ddof=1` = Bessel's correction (unbiased estimator)

2. **`(1 + gmean_day_return)^2`** — Squared growth factor
   - `gmean_day_return` = geometric mean of daily returns
   - Accounts for compounding effect on variance

3. **`(var + (1+gmean)^2)^N - (1+gmean)^(2N)`** — Variance of compounded returns
   - First term: expected value of squared compounded return
   - Second term: squared expected compounded return
   - Difference = variance of compounded annual return

4. **`np.sqrt(...)`** — Standard deviation (volatility)
   - Square root of variance gives standard deviation
   - Multiply by 100 for percentage

**Why not simple `std × √240`?**
- Simple formula assumes returns are independent and additive
- Compounding means returns multiply: `(1+r1)(1+r2)...`
- This formula correctly handles the variance of the product

**Bangladeshi Example**:
- Daily return std: 1.2%
- Simple annualized: 1.2% × √240 = **18.6%**
- Compounding-aware (this formula): ~19.2%
- DSE is more volatile than developed markets (15-20% typical)

### `CAGR [%]`
**Definition**: Compound Annual Growth Rate based on total period.
```python
time_in_years = (duration.days + duration.seconds/86400) / 365.25
s['CAGR [%]'] = ((Equity_Final / equity[0])**(1/time_in_years) - 1) * 100
```

**Formula**: 
$$\text{CAGR} = \left(\frac{\text{Final}}{\text{Initial}}\right)^{1/\text{Years}} - 1$$

**Bangladeshi Example**:
- ৳10L → ৳14.5L in 5 years
- CAGR = (1.45)^(1/5) - 1 = **7.7%**
- Differs from `Return (Ann.)` due to compounding method

---

## Risk-Adjusted Metrics

### `Sharpe Ratio`
**Definition**: Excess return per unit of total risk (volatility).
```python
s['Sharpe Ratio'] = (s['Return (Ann.) [%]'] - risk_free_rate * 100) / s['Volatility (Ann.) [%]']
```

**Formula**:
$$\text{Sharpe} = \frac{R_p - R_f}{\sigma_p}$$

Where:
- $R_p$ = Portfolio annual return
- $R_f$ = Risk-free rate (Bangladesh: ~6-8% for govt bonds)
- $\sigma_p$ = Portfolio annual volatility

**Bangladeshi Example**:
- Strategy return (ann.): 12%
- Risk-free (Bangladesh 10Y bond): 7%
- Volatility: 18%
- Sharpe = (12 - 7) / 18 = **0.28** (low)
- Good: >1, Excellent: >2

### `Sortino Ratio`
**Definition**: Excess return per unit of downside risk only.
```python
s['Sortino Ratio'] = (annualized_return - risk_free_rate) / (
    np.sqrt(np.mean(day_returns.clip(-np.inf, 0)**2)) * np.sqrt(annual_trading_days)
)
```

**Mathematical Breakdown**:

1. **`day_returns.clip(-np.inf, 0)`** — Keep only negative returns
   - Positive returns → 0
   - Negative returns → unchanged
   - Example: [0.02, -0.01, 0.03, -0.015] → [0, -0.01, 0, -0.015]

2. **`clip(...)**2`** — Square the negative returns
   - [-0.01, 0, -0.015] → [0.0001, 0, 0.000225]

3. **`np.mean(...)`** — Average of squared negative returns
   - This is the **downside variance** (semi-variance)
   - Only penalizes bad volatility, not good volatility

4. **`np.sqrt(...) * np.sqrt(annual_trading_days)`** — Annualized downside deviation
   - Square root = standard deviation
   - × √N = annualize

**Why Sortino > Sharpe for DSE?**
- Sharpe penalizes ALL volatility (including big winning days)
- DSE has asymmetric returns: few big winners, many small losers
- Sortino only penalizes the downside → fairer for trend strategies

**Formula**:
$$\text{Sortino} = \frac{R_p - R_f}{\sigma_{\text{downside}}}$$

**Bangladeshi Example**:
- Same strategy: 12% return, 7% risk-free
- Downside deviation: 10% (only negative returns)
- Sortino = (12-7)/10 = **0.5**
- Better than Sharpe for asymmetric DSE returns

### `Calmar Ratio`
**Definition**: Annual return divided by maximum drawdown.
```python
s['Calmar Ratio'] = annualized_return / (-max_dd or np.nan)
```

**Mathematical Breakdown**:

1. **`annualized_return`** — Compounded annual return (already computed)
   - `(1 + gmean_day_return)^annual_trading_days - 1`
   - Same as `Return (Ann.) [%] / 100`

2. **`-max_dd`** — Max drawdown as positive decimal
   - `max_dd` is negative (e.g., -0.25 for 25%)
   - `-max_dd` = 0.25
   - `or np.nan` handles case where max_dd = 0 (no drawdown)

3. **Division** — How many years of return per max loss
   - Calmar = 0.12 / 0.25 = 0.48
   - Means: you earn 0.48 years of returns for each max drawdown risk

**Interpretation**:
- **Calmar > 1**: Annual return > max drawdown (good)
- **Calmar = 1**: Break-even (return = max loss)
- **Calmar < 1**: Max loss exceeds annual return (risky)

**Why use Calmar?**
- Sharpe/Sortino use volatility (statistical risk)
- Calmar uses **realized worst-case loss** (actual risk)
- Better for investors who fear drawdowns more than volatility

**Bangladeshi Example**:
- Annual return: 12%
- Max DD: 25%
- Calmar = 12/25 = **0.48**
- Good: >1 (return exceeds max loss)

### `Alpha [%]`
**Definition**: Jensen's Alpha - excess return vs CAPM prediction.
```python
beta = cov_matrix[0, 1] / cov_matrix[1, 1]
s['Alpha [%]'] = s['Return [%]'] - rf*100 - beta * (s['Buy & Hold Return [%]'] - rf*100)
```

**Mathematical Breakdown**:

1. **`np.cov(equity_log_returns, market_log_returns)`** — Covariance matrix
   - Returns 2×2 matrix: [[var_portfolio, cov], [cov, var_market]]
   - Uses **log returns** (continuously compounded) for CAPM accuracy

2. **`beta = cov_matrix[0, 1] / cov_matrix[1, 1]`** — Covariance / Market Variance
   - Measures how much portfolio moves with market
   - β = 1 → moves with market; β = 1.2 → 20% more volatile

3. **CAPM Expected Return**: `R_f + β(R_m - R_f)`
   - What you SHOULD earn given your market risk
   - Risk-free + risk premium for your beta

4. **Alpha = Actual - Expected**
   - `Return[%] - rf - β × (BuyHold[%] - rf)`
   - Positive = skill/outperformance; Negative = underperformance

**Bangladeshi Example**:
- Strategy return: 45% (5 years) → ~7.7% annual
- Market (DSEX) return: 25% (5 years) → ~4.6% annual
- Beta: 0.8
- Risk-free: 7%
- Alpha = 7.7 - 7 - 0.8×(4.6-7) = **2.62% annual**
- Positive alpha = skill, negative = luck/benchmark exposure

### `Beta`
**Definition**: Sensitivity to market (DSEX index) movements.
```python
cov_matrix = np.cov(equity_log_returns, market_log_returns)
beta = cov_matrix[0, 1] / cov_matrix[1, 1]
```

**Mathematical Breakdown**:

1. **Log Returns**: `np.log(equity[1:] / equity[:-1])`
   - Continuously compounded returns
   - Additive over time (unlike simple returns)

2. **Covariance Matrix** (2×2):
   ```
   [[var(portfolio), cov(portfolio, market)],
    [cov(market, portfolio), var(market)]]
   ```

3. **Beta = cov / var_market**:
   - Slope of regression line: portfolio return vs market return
   - β = 1.0 → portfolio mirrors market
   - β = 1.5 → portfolio amplifies market moves by 50%
   - β = 0.0 → no correlation with market

**Bangladeshi Example**:
- Beta = 1.2 → 20% more volatile than DSEX
- Beta = 0.8 → 20% less volatile (defensive)
- DSE financial stocks: β ~ 1.3-1.5
- DSE pharma (SQURPHARMA): β ~ 0.6-0.8

---

## Trade Quality Metrics

### `# Trades`
**Definition**: Total number of completed trades.
```python
s['# Trades'] = n_trades = len(trades_df)
```

**Bangladeshi Example**:
- 50 trades in 5 years = 10 trades/year
- DSE T+2 settlement limits frequency
- High frequency = more commissions (0.05% each side)

### `Win Rate [%]`
**Definition**: Percentage of profitable trades.
```python
win_rate = (pl > 0).mean()
s['Win Rate [%]'] = win_rate * 100
```

**Formula**:
$$\text{Win Rate} = \frac{\text{Profitable Trades}}{\text{Total Trades}} \times 100$$

**Bangladeshi Example**:
- 50 trades, 32 profitable
- Win Rate = 32/50 × 100 = **64%**
- >55% is good for trend following on DSE

### `Best Trade [%]`
**Definition**: Largest single trade return.
```python
s['Best Trade [%]'] = returns.max() * 100
```

**Bangladeshi Example**:
- Bought GP at ৳280, sold at ৳380
- Return = (380-280)/280 × 100 = **35.7%**
- Can be skewed by one lucky trade

### `Worst Trade [%]`
**Definition**: Largest single trade loss.
```python
s['Worst Trade [%]'] = returns.min() * 100
```

**Bangladeshi Example**:
- Bought BEXIMCO at ৳120, sold at ৳85
- Loss = (85-120)/120 × 100 = **-29.2%**
- DSE circuit breakers limit single-day loss to ~10%

### `Avg. Trade [%]`
**Definition**: Geometric mean of trade returns (compounding).
```python
mean_return = geometric_mean(returns)
s['Avg. Trade [%]'] = mean_return * 100
```

**Formula**:
$$\text{Avg Trade} = \left(\prod(1 + r_i)\right)^{1/n} - 1$$

**Bangladeshi Example**:
- Trade returns: +10%, -5%, +15%, -8%, +12%
- Geometric mean = (1.1×0.95×1.15×0.92×1.12)^(1/5) - 1 = **4.3%**
- Arithmetic mean would be 4.8% (overstates)

### `Profit Factor`
**Definition**: Gross profits divided by gross losses.
```python
s['Profit Factor'] = returns[returns > 0].sum() / abs(returns[returns < 0].sum())
```

**Mathematical Breakdown**:

1. **`returns[returns > 0].sum()`** — Sum of all winning trade returns
   - Only positive returns included
   - Example: [+10%, +5%, +15%] → sum = 30%

2. **`abs(returns[returns < 0].sum())`** — Absolute sum of all losing trade returns
   - Only negative returns, made positive
   - Example: [-8%, -5%, -12%] → sum = 25%

3. **Division** — Ratio of gross profit to gross loss
   - PF = 30% / 25% = 1.2
   - PF = 1.0 → break-even (wins = losses)
   - PF > 1.0 → profitable system

**Interpretation**:
- **PF < 1.0**: Losing system
- **PF = 1.0-1.5**: Marginal
- **PF = 1.5-2.0**: Good
- **PF > 2.0**: Excellent

**Relationship to Win Rate**:
- PF = (Win Rate / (1 - Win Rate)) × (Avg Win / Avg Loss)
- Can have high Win Rate but low PF (small wins, big losses)
- Can have low Win Rate but high PF (big wins, small losses)

**Bangladeshi Example**:
- Winning trades total return: +120%
- Losing trades total return: -60%
- Profit Factor = 120/60 = **2.0**
- >1.5 is good, >2 is excellent

### `Expectancy [%]`
**Definition**: Average return per trade (arithmetic mean).
```python
s['Expectancy [%]'] = returns.mean() * 100
```

**Mathematical Breakdown**:

1. **`returns.mean()`** — Arithmetic average of trade returns
   - Sum of all trade returns ÷ number of trades
   - NOT compounded (unlike Avg Trade %)

2. **What it tells you**: Expected profit per trade in %
   - Positive = profitable on average
   - Negative = losing on average

**Difference from Avg Trade [%]:**
| Metric | Mean Type | Compounding |
|--------|-----------|-------------|
| Expectancy | Arithmetic | No |
| Avg Trade % | Geometric | Yes |

- Expectancy overstates if returns are volatile
- Avg Trade % is what you'd actually get from compounding

**Bangladeshi Example**:
- 50 trades, total return 45%
- Expectancy = 45%/50 = **0.9% per trade**
- × 10 trades/year = 9% annual (before compounding)

### `SQN` (System Quality Number)
**Definition**: Van Tharp's metric: √n × mean(P&L) / std(P&L)
```python
s['SQN'] = np.sqrt(n_trades) * pl.mean() / (pl.std() or np.nan)
```

**Mathematical Breakdown**:

1. **`pl.mean()`** — Average P&L per trade in BDT (not %)
   - Raw profit/loss in currency units
   - Unlike other metrics which use %

2. **`pl.std()`** — Standard deviation of P&L
   - Measures consistency of trade outcomes
   - Lower std = more consistent

3. **`np.sqrt(n_trades)`** — Sample size adjustment
   - More trades → higher SQN (statistical significance)
   - √N accounts for law of large numbers

4. **Ratio** = Signal-to-noise ratio scaled by sample size
   - High mean, low std, many trades → high SQN

**Van Tharp's Interpretation Scale**:
| SQN | Rating | Action |
|-----|--------|--------|
| < 1.0 | Poor | Abandon |
| 1.0-1.5 | Below avg | Needs work |
| 1.5-2.0 | Average | Acceptable |
| 2.0-2.5 | Good | Trade it |
| 2.5-3.0 | Excellent | Scale up |
| > 3.0 | Superb | Maximum size |

**Why SQN matters for DSE**:
- DSE has high commissions (0.05% each side = 1% round-trip)
- Small sample (240 trading days/year) → need high SQN to be confident
- Low commission strategies need >2.0; high frequency needs >2.5

**Bangladeshi Example**:
- 50 trades, avg P&L ৳8,000, std ৳25,000
- SQN = √50 × 8,000 / 25,000 = 7.07 × 0.32 = **2.26 (Good)**

### `Kelly Criterion`
**Definition**: Optimal position sizing fraction.
```python
s['Kelly Criterion'] = win_rate - (1 - win_rate) / (pl[pl > 0].mean() / -pl[pl < 0].mean())
```

**Mathematical Breakdown**:

1. **Variables**:
   - `p = win_rate` (e.g., 0.64)
   - `q = 1 - p` (loss rate, e.g., 0.36)
   - `b = avg_win / avg_loss` (payoff ratio, e.g., 1.5)

2. **Formula**: `f* = p - q/b`
   - Fraction of bankroll to bet per trade
   - Maximizes geometric growth rate (log utility)

3. **Derivation** (simplified):
   - Expected log growth = p·log(1+f·b) + q·log(1-f)
   - Maximize w.r.t f → f* = p - q/b

**Critical Assumptions**:
- Known, fixed probabilities (p, b)
- Infinite sequence of identical bets
- No transaction costs (violated in DSE: 0.05% × 2 = 0.1%)
- No correlation between trades

**Why Half-Kelly for DSE?**:
- Kelly assumes perfect knowledge of p and b
- Real markets: p and b are estimated with error
- DSE commissions (0.1% round-trip) reduce effective payoff
- Full Kelly → 50% chance of 50% drawdown
- Half-Kelly (f*/2) → 75% of Kelly growth, much less risk

**Bangladeshi Example**:
- Win rate: 64% (0.64)
- Avg win: ৳15,000, Avg loss: ৳10,000
- b = 1.5
- Kelly = 0.64 - 0.36/1.5 = **0.40 (40%)**
- Meaning: Optimal position = 40% of equity per trade
- **DSE reality**: Use 10-20% (half-Kelly) due to estimation error

---

## Activity Metrics

### `Max. Trade Duration`
**Definition**: Longest holding period for a single trade.
```python
s['Max. Trade Duration'] = _round_timedelta(durations.max())
```

**Bangladeshi Example**:
- Trend following on SQURPHARMA: held 180 days
- DSE context: Medium-term trends last 3-6 months

### `Avg. Trade Duration`
**Definition**: Average holding period across all trades.
```python
s['Avg. Trade Duration'] = _round_timedelta(durations.mean())
```

**Bangladeshi Example**:
- Avg hold: 25 days
- DSE swing trading: 15-40 days typical
- Day trading not practical due to T+2

---

## Bangladeshi Market Context

### DSE-Specific Considerations

| Factor | Impact on Metrics |
|--------|-------------------|
| **T+2 Settlement** | Cash from trade unavailable for 2 days; affects `Exposure Time` and `Equity Final` |
| **Circuit Breakers (±10%)** | Limits single-day loss/gain; caps `Best/Worst Trade` |
| **Lot Sizes (1, 10, 100)** | Rounding affects position sizing; small accounts constrained |
| **Commission 0.05%** | 20 trades = 1% annual drag; hurts `Return`, `Expectancy`, `Profit Factor` |
| **No Short Selling** | `Strategy.sell()` disabled; only long + position.close() |
| **Dividend Adjustments** | Bonus shares (200% = 3:1) distort prices; MUST use `adjusted=True` |
| **Trading Hours** | 10:00-14:30 BST; 240 trading days/year |
| **Market Cap Tiers** | A/B/N/Z categories have different liquidity/volatility |

### Example: Complete SQURPHARMA Backtest Output

```text
Start                     2020-01-02 00:00:00
End                       2024-12-30 00:00:00
Duration                   1823 days 00:00:00
Exposure Time [%]                       78.45
Equity Final [৳]                  14,52,340
Equity Peak [৳]                   16,05,200
Commissions [৳]                       12,450
Return [%]                             45.23
Buy & Hold Return [%]                  22.15
Return (Ann.) [%]                       7.74
Volatility (Ann.) [%]                  18.56
CAGR [%]                                7.72
Sharpe Ratio                             0.15
Sortino Ratio                            0.28
Calmar Ratio                             0.31
Alpha [%]                                3.45
Beta                                     0.72
Max. Drawdown [%]                      -24.8
Avg. Drawdown [%]                       -6.2
Max. Drawdown Duration      198 days 00:00:00
Avg. Drawdown Duration       42 days 00:00:00
# Trades                                   42
Win Rate [%]                            61.90
Best Trade [%]                          38.5
Worst Trade [%]                        -18.2
Avg. Trade [%]                           1.08
Max. Trade Duration         176 days 00:00:00
Avg. Trade Duration          38 days 00:00:00
Profit Factor                            2.15
Expectancy [%]                           1.08
SQN                                      2.34
Kelly Criterion                        0.38
```

**Interpretation for DSE**:
- ✅ **Beat buy & hold** (45% vs 22%)
- ⚠️ **Low Sharpe (0.15)** — high volatility relative to return
- ✅ **Good Profit Factor (2.15)** — wins exceed losses 2:1
- ✅ **Win Rate 62%** — solid for trend following
- ⚠️ **Max DD 25%** — large drawdown; consider position sizing
- ✅ **SQN 2.34** — "Good" system quality
- 📊 **Kelly 38%** — use 15-20% position size (half-Kelly)

---

## Key Formulas Summary

| Metric | Formula |
|--------|---------|
| **Return %** | `(Final - Initial) / Initial × 100` |
| **CAGR** | `(Final/Initial)^(1/years) - 1` |
| **Max DD** | `max((Peak - Current)/Peak) × 100` |
| **Sharpe** | `(R_p - R_f) / σ_p` |
| **Sortino** | `(R_p - R_f) / σ_downside` |
| **Calmar** | `Annual Return / Max DD` |
| **Profit Factor** | `Gross Wins / \|Gross Losses\|` |
| **SQN** | `√N × mean(PnL) / std(PnL)` |
| **Kelly** | `p - q/b` where `b = avg_win/avg_loss` |

---

## References

- Van Tharp, "Trade Your Way to Financial Freedom" (SQN)
- Sharpe, "The Sharpe Ratio" (1966, 1994)
- Sortino & van der Meer, "Downside Risk" (1991)
- Young, "Calmar Ratio" (1991)
- DSE Trading Rules: https://www.dsebd.org/displayAboutUs.php
- Bangladesh Bank T-bill rates for risk-free proxy

---

*Generated from `backtesting/_stats.py:compute_stats()` source code.*