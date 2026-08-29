[![](https://i.imgur.com/E8Kj69Y.png)](https://kernc.github.io/backtesting.py/)

Backtesting.py
==============
[![Build Status](https://img.shields.io/github/actions/workflow/status/kernc/backtesting.py/ci.yml?branch=master&style=for-the-badge)](https://github.com/kernc/backtesting.py/actions)
[![Code Coverage](https://img.shields.io/codecov/c/gh/kernc/backtesting.py.svg?style=for-the-badge&label=Covr)](https://codecov.io/gh/kernc/backtesting.py)
[![Source lines of code](https://img.shields.io/endpoint?url=https%3A%2F%2Fghloc.vercel.app%2Fapi%2Fkernc%2Fbacktesting.py%2Fbadge?filter=.py%26format=human&style=for-the-badge&label=SLOC&color=skyblue)](https://ghloc.vercel.app/kernc/backtesting.py)
[![Backtesting on PyPI](https://img.shields.io/pypi/v/backtesting.svg?color=blue&style=for-the-badge)](https://pypi.org/project/backtesting)
[![PyPI downloads](https://img.shields.io/pypi/dd/backtesting.svg?style=for-the-badge&label=D/L&color=skyblue)](https://pypistats.org/packages/backtesting)
[![Total downloads](https://img.shields.io/pepy/dt/backtesting?style=for-the-badge&label=%E2%88%91&color=skyblue)](https://pypistats.org/packages/backtesting)
[![Stars](https://img.shields.io/github/stars/kernc/backtesting.py?color=silver&style=for-the-badge&label=%e2%ad%90)](https://github.com/kernc/backtesting.py)
[![GitHub Sponsors](https://img.shields.io/github/sponsors/kernc?color=pink&style=for-the-badge&label=%E2%99%A5)](https://github.com/sponsors/kernc)

Backtest trading strategies with Python.

[**Project website**](https://kernc.github.io/backtesting.py) + [Documentation] &nbsp;&nbsp;|&nbsp; [YouTube]

[Documentation]: https://kernc.github.io/backtesting.py/doc/backtesting/
[YouTube]: https://www.youtube.com/results?q=%22backtesting.py%22

Installation
------------

    $ pip install git+https://github.com/0kamrulhasan0/backtesting.py

Usage
-----
```python
from backtesting import Backtest, Strategy
from backtesting.lib import crossover

def SMA(values, n):
    import pandas as pd
    return pd.Series(values).rolling(n).mean().values


class SmaCross(Strategy):
    def init(self):
        price = self.data.Close
        self.ma1 = self.I(SMA, price, 10)
        self.ma2 = self.I(SMA, price, 20)

    def next(self):
        if crossover(self.ma1, self.ma2):
            self.buy()
        elif crossover(self.ma2, self.ma1):
            self.position.close()


# For DSE (Dhaka Stock Exchange) use ConstrainedBacktest:
# bt = ConstrainedBacktest(data, SmaCross, cash=1_000_000, lot_size=1)
bt = Backtest(SQURPHARMA, SmaCross, commission=.0005,
              exclusive_orders=True)
stats = bt.run()
bt.plot()
```

Results in:

```text
Start                     2004-08-19 00:00:00
End                       2013-03-01 00:00:00
Duration                   3116 days 00:00:00
Exposure Time [%]                       94.27
Equity Final [$]                     68935.12
Equity Peak [$]                      68991.22
Return [%]                             589.35
Buy & Hold Return [%]                  703.46
Return (Ann.) [%]                       25.42
Volatility (Ann.) [%]                   38.43
CAGR [%]                                16.80
Sharpe Ratio                             0.66
Sortino Ratio                            1.30
Calmar Ratio                             0.77
Alpha [%]                              450.62
Beta                                     0.02
Max. Drawdown [%]                      -33.08
Avg. Drawdown [%]                       -5.58
Max. Drawdown Duration      688 days 00:00:00
Avg. Drawdown Duration       41 days 00:00:00
# Trades                                   93
Win Rate [%]                            53.76
Best Trade [%]                          57.12
Worst Trade [%]                        -16.63
Avg. Trade [%]                           1.96
Max. Trade Duration         121 days 00:00:00
Avg. Trade Duration          32 days 00:00:00
Profit Factor                            2.13
Expectancy [%]                           6.91
SQN                                      1.78
Kelly Criterion                        0.6134
_strategy              SmaCross(n1=10, n2=20)
_equity_curve                          Equ...
_trades                       Size  EntryB...
dtype: object
```
[![plot of trading simulation](https://i.imgur.com/xRFNHfg.png)](https://kernc.github.io/backtesting.py/#example)

Find more usage examples in the [documentation].


Features
--------
* Simple, [well-documented API](https://kernc.github.io/backtesting.py/doc/backtesting/backtesting.html)
* Blazing fast execution
* Built-in [optimizer](https://kernc.github.io/backtesting.py/doc/examples/Quick%20Start%20User%20Guide.html#Optimization)
  based on [SAMBO](https://sambo-optimization.github.io)
* [Library of composable base strategies](https://kernc.github.io/backtesting.py/doc/examples/Strategies%20Library.html)
  and related utilities
* Indicator-library-agnostic (BYO)
* Supports _any_ financial instrument with OHLC(V) candlestick data
* [Detailed trade results](https://kernc.github.io/backtesting.py/doc/examples/Quick%20Start%20User%20Guide.html#Trade-data)
  provided as simple Series/DataFrame objects
* [Interactive visualizations](https://kernc.github.io/backtesting.py/#example)

![xkcd.com/1570](https://imgs.xkcd.com/comics/engineer_syllogism.png)


Dhaka Stock Exchange (DSE) Support
----------------------------------
Backtesting.py includes `ConstrainedBacktest` for regulated markets like Bangladesh (DSE):
- Long-only trading (no short selling)
- Lot size enforcement (e.g., 1, 10, 100 shares)
- Circuit breakers (±10% daily price limits)
- T+2 settlement delay
- Dividend-adjusted prices (critical for bonus shares like 200% stock dividends)
- Default commission: 0.05% per trade

```python
import dsebd
from backtesting import ConstrainedBacktest, Strategy
from backtesting.lib import crossover

def SMA(values, n):
    import pandas as pd
    return pd.Series(values).rolling(n).mean().values


# Update local cache
dsebd.update()

# Download dividend-adjusted data (critical for DSE bonus shares)
data = dsebd.download('SQURPHARMA', period='5y', adjusted=True)

class SmaCross(Strategy):
    def init(self):
        self.ma1 = self.I(SMA, self.data.Close, 10)
        self.ma2 = self.I(SMA, self.data.Close, 20)
    def next(self):
        if crossover(self.ma1, self.ma2):
            self.buy()                    # Enter long
        elif crossover(self.ma2, self.ma1):
            self.position.close()         # Exit long (NOT self.sell())

bt = ConstrainedBacktest(data, SmaCross, cash=1_000_000, lot_size=1)
stats = bt.run()
bt.plot()
```

Or use the convenience method with any data source:

```python
bt = ConstrainedBacktest.from_data_source('SQURPHARMA', SmaCross,
                                          period='5y', cash=1_000_000,
                                          data_source=dsebd)
stats = bt.run()
```

**Key differences from standard Backtest:**
- `Strategy.sell()` raises `NotImplementedError` (use `position.close()`)
- Orders rounded to nearest lot size
- Market orders clipped to ±10% circuit breaker
- Cash from trades unavailable for 2 days (T+2)

Bugs
----
Before reporting bugs or posting to the
[discussion board](https://github.com/kernc/backtesting.py/discussions),
please read [contributing guidelines](CONTRIBUTING.md), particularly the section
about crafting useful bug reports and ```` ``` ````-fencing your code.
The maintainers thank you!


Alternatives
------------
See [alternatives.md] for a list of alternative Python
backtesting frameworks and related packages.

[alternatives.md]: https://github.com/kernc/backtesting.py/blob/master/doc/alternatives.md