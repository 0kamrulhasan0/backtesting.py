# ConstrainedBacktest Specification

## Overview

Add `ConstrainedBacktest` class for regulated exchanges (DSE, etc.) that enforce:
- Long-only trading (no short selling)
- Lot size enforcement
- Circuit breakers (daily price limits)
- Settlement delays (T+N)
- Configurable commission

## Goals

- [x] Long-only enforcement: `Strategy.sell()` raises `NotImplementedError` when constraints active
- [x] Lot sizing: Orders rounded up to nearest `lot_size` (default=1)
- [x] Circuit breakers: Market orders clipped to ±`price_limit_pct` (default 10%); limit orders rejected outside bounds
- [x] Settlement delay: Cash from closed trades held for `settlement_days` bars (default 2)
- [x] Commission: Default 0.05% per trade
- [x] Drop-in compatibility: Existing long-only strategies run unchanged
- [x] Preserve core features: `optimize()`, `plot()`, `SignalStrategy`, `resample_apply`, heatmaps

## Non-Goals

- [ ] Separate package (stays in same repo, exported from `backtesting`)
- [ ] Full rewrite (minimal diff on top of existing codebase)
- [ ] Support for derivatives/options
- [ ] Multiple account types

## Architecture

### New Class: `ConstrainedBacktest`

```python
class ConstrainedBacktest(Backtest):
    def __init__(self,
                 data: pd.DataFrame,
                 strategy: Type[Strategy],
                 *,
                 cash: float = 1_000_000,
                 commission: Union[float, Tuple[float, float]] = (0, 0.0005),
                 lot_size: int = 1,
                 price_limit_pct: float = 0.10,
                 settlement_days: int = 2,
                 spread: float = 0.0,
                 margin: float = 1.0,
                 trade_on_close: bool = False,
                 hedging: bool = False,
                 exclusive_orders: bool = True,
                 finalize_trades: bool = False):
```

### Modified `_Broker` Behavior

| Feature | Standard | Constrained |
|---------|----------|-------------|
| Short orders | Allowed via `sell()` | Rejected with error |
| Position sizing | Fractional or absolute | Rounded up to `lot_size` |
| Price validation | None | Clipped to circuit breaker bounds |
| Cash availability | Immediate | Delayed by `settlement_days` |
| Hedging | Optional | Forced `False` |
| Margin | Configurable | Forced `1.0` |

### Strategy API Changes

```python
class Strategy:
    def sell(self, **kwargs):
        """Open SHORT position - NOT ALLOWED on constrained exchanges."""
        if getattr(self._broker, '_price_limit_pct', 0) > 0 or \
           getattr(self._broker, '_settlement_days', 0) > 0 or \
           getattr(self._broker, '_lot_size', 1) > 1:
            raise NotImplementedError(
                "Short selling is prohibited on this exchange. "
                "To exit a long position, use: self.position.close() or trade.close(). "
                "Use ConstrainedBacktest for markets with long-only constraints."
            )
        # ... standard short selling logic

    def buy(self, *, size=..., **kwargs):
        """Open LONG position. Size rounded up to lot_size."""
        # ... standard logic with lot size rounding
```

## DSE-Specific Features

### 1. Lot Sizing
- Parameter: `lot_size` (default=1, some mutual funds/debentures: 10, 100)
- Applied in `Strategy.buy()` and `_Broker.new_order()`
- Fractional equity sizing → compute shares → round UP to nearest lot

### 2. Circuit Breakers (±10%)
- Parameter: `price_limit_pct` (default=0.10)
- On each bar: `limit_up = prev_close * 1.10`, `limit_down = prev_close * 0.90`
- Market orders: execution price clipped to `[limit_down, limit_up]`
- Limit orders: rejected if `limit_price` outside bounds

### 3. T+2 Settlement
- Parameter: `settlement_days` (default=2)
- Track `unsettled_cash` per trade exit
- Cash from `Trade.close()` available after `settlement_days` bars
- Equity curve shows unrealized P&L immediately; cash for new orders delayed

### 4. Dividend-Adjusted Data
- Expect data from `dsebd.download(adjusted=True)`
- Classmethod: `ConstrainedBacktest.from_data_source(ticker, strategy, period, data_source)`

### 5. Commission Model
- Default: `commission=(0, 0.0005)` (0.05% per side, no fixed fee)
- Applied on entry AND exit

## API Compatibility

### Works Unchanged (Long-Only Strategies)
```python
class MyStrategy(Strategy):
    def init(self):
        self.sma = self.I(SMA, self.data.Close, 20)
    def next(self):
        if self.data.Close[-1] > self.sma[-1]:
            self.buy()
        else:
            self.position.close()
```

### Explicit Error (Short Attempts)
```python
def next(self):
    self.sell()  # NotImplementedError with clear message
```

### Migration Guide
| Old Code | New Code |
|----------|----------|
| `self.sell()` | `self.position.close()` |
| `self.sell(size=0.5)` | `self.position.close(portion=0.5)` |
| `trade.close()` | `trade.close()` (unchanged) |
| `Backtest(..., margin=0.1)` | `ConstrainedBacktest(...)` (margin ignored) |
| `Backtest(..., hedging=True)` | `ConstrainedBacktest(...)` (hedging ignored) |

## File Changes

| File | Change Type | Description |
|------|-------------|-------------|
| `backtesting/backtesting.py` | Modify | Add `ConstrainedBacktest` class; modify `_Broker` for constraints |
| `backtesting/__init__.py` | Modify | Export `ConstrainedBacktest` |
| `backtesting/test/_test.py` | Add | Tests for constraints (10 tests) |
| `setup.py` | Modify | Add `dse` extra with `dsebd` |
| `README.md` | Modify | DSE usage examples |
| `AGENTS.md` | Modify | Document new class and extras |

## Testing Strategy

### Unit Tests (in `backtesting/test/_test.py`)

```python
class TestConstrainedBacktest(TestCase):
    def test_sell_raises_not_implemented(self):
        # Strategy.sell() raises NotImplementedError with helpful message
        
    def test_lot_sizing_rounds_up(self):
        # size=150 with lot_size=100 → 200 shares (2 lots)
        
    def test_lot_sizing_fractional_equity(self):
        # size=0.1 equity → rounded to nearest lot
        
    def test_circuit_breaker_clips_market_order(self):
        # Market order on 15% gap up → filled at limit_up (10%)
        
    def test_circuit_breaker_rejects_limit_order(self):
        # Limit order beyond limit_up → rejected, no trade
        
    def test_settlement_delay(self):
        # Cash from trade at bar 4 available at bar 6 (T+2)
        
    def test_constrained_defaults(self):
        # Verify default parameters
        
    def test_position_close_works(self):
        # Position.close() works correctly
        
    def test_optimize_works(self):
        # Optimization works with ConstrainedBacktest
        
    def test_plot_works(self):
        # Plotting works with ConstrainedBacktest
```

### Integration Tests
- Run existing test suite with `ConstrainedBacktest` (should pass for long-only)
- Example notebooks updated to use `ConstrainedBacktest` + `dsebd`

## Configuration Defaults

| Parameter | Default | Rationale |
|-----------|---------|-----------|
| `cash` | 1,000,000 | Typical retail account (BDT) |
| `commission` | (0, 0.0005) | 0.05% DSE standard |
| `lot_size` | 1 | Most liquid stocks |
| `price_limit_pct` | 0.10 | DSE circuit breaker |
| `settlement_days` | 2 | T+2 settlement |
| `margin` | 1.0 (forced) | No margin trading |
| `hedging` | False (forced) | No short selling |
| `exclusive_orders` | True (forced) | One position at a time |

## Example Usage

```python
import dsebd
from backtesting import ConstrainedBacktest, Strategy
from backtesting.lib import crossover, SMA

dsebd.update()
data = dsebd.download('SQURPHARMA', period='5y', adjusted=True)

class SmaCross(Strategy):
    def init(self):
        self.ma1 = self.I(SMA, self.data.Close, 10)
        self.ma2 = self.I(SMA, self.data.Close, 20)
    def next(self):
        if crossover(self.ma1, self.ma2):
            self.buy()
        elif crossover(self.ma2, self.ma1):
            self.position.close()

bt = ConstrainedBacktest(data, SmaCross, cash=1_000_000, lot_size=1)
stats = bt.run()
bt.plot()

# Or convenience method:
bt = ConstrainedBacktest.from_data_source('SQURPHARMA', SmaCross, 
                                          period='5y', cash=1_000_000,
                                          data_source=dsebd)
```

## Acceptance Criteria

- [x] `ConstrainedBacktest` runs long-only strategies identically to `Backtest`
- [x] `Strategy.sell()` raises `NotImplementedError` with helpful message when constraints active
- [x] Lot sizing enforced on all orders (round UP to nearest lot)
- [x] Price limits enforced (±10% from prev close)
- [x] T+2 settlement delays cash availability
- [x] `dsebd.download(adjusted=True)` works end-to-end
- [x] All existing tests pass with `ConstrainedBacktest`
- [x] New tests cover all constraint behaviors
- [x] Optimization and plotting work unchanged
- [x] Documentation includes DSE quickstart

## Open Decisions (Resolved)

| Decision | Resolution |
|----------|------------|
| Lot size per-ticker? | Global param; per-ticker via `from_data_source` |
| Circuit breaker: reject or clip? | Clip market orders, reject limit orders |
| Settlement: affect equity curve? | Delayed cash only; equity shows unrealized immediately |
| Package structure? | Same repo, exported from `backtesting` |

## Implementation Notes

### Ponytail Comments (Deliberate Simplifications)

```python
# ponytail: global settlement tracking, per-account tracking if multi-account needed
# ponytail: circuit breaker uses prev_close only; could use reference price if exchange specifies
# ponytail: lot size rounding always UP; could add round-to-nearest option if requested
```