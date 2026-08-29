# DSEBD - Dhaka Stock Exchange Data Package

A Python package for fetching and analyzing stock market data from the Dhaka Stock Exchange (DSE). This package provides a simple and intuitive API similar to `yfinance` for accessing historical price data, company information, dividend adjustments, and current market prices.

## Features

- 📊 **Historical Data**: Download OHLCV (Open, High, Low, Close, Volume) data for any DSE-listed stock
- 🏢 **Company Information**: Fetch detailed company fundamentals and financial performance
- 💰 **Current Prices**: Get real-time market data
- 📋 **Dividend Archive**: Scrape and store full dividend history from LankaBangla Financial Portal
- 🔧 **Price Adjustment**: Adjust historical prices for stock dividends (bonus shares) to prevent skew from events like 2:1 or 200% stock dividends
- 🚀 **Fast & Efficient**: Uses DuckDB for local caching and async requests for web scraping
- 🔄 **Explicit Cache Update**: Refreshes prices and dividends with `dsebd.update()`
- 📈 **Pandas Integration**: Returns data as pandas DataFrames for easy analysis

## Installation

### Set Github Token
```bash
export GH_TOKEN=your_github_personal_access_token
``` 
or Jupyter notebook
```python
import os
os.environ['GH_TOKEN'] = 'your_github_personal_access_token'
```


### From GitHub
```bash
pip install git+https://$GH_TOKEN@github.com/0kamrulhasan0/dsebd.git
```

### From Source
```bash
!git clone https://$GH_TOKEN@github.com/0kamrulhasan0/dsebd.git
cd dsebd
pip install -e .
```

## Quick Start

### Download Historical OHLCV Data for a Single Ticker

```python
import dsebd

# Refresh local DuckDB cache before reading data.
dsebd.update()

# Download 1 month of data (default)
df = dsebd.download('SQURPHARMA')

# Download with custom date range
df = dsebd.download('SQURPHARMA', start='2024-01-01', end='2024-12-31')

# Download with period shorthand
df = dsebd.download('SQURPHARMA', period='1y')  # 1 year
df = dsebd.download('SQURPHARMA', period='6mo')  # 6 months
df = dsebd.download('SQURPHARMA', period='5d')   # 5 days
```

### Download OHLCV Data for Multiple Tickers

```python
# Returns a multi-level DataFrame
df = dsebd.download(['SQURPHARMA', 'BEXIMCO', 'GP'], period='1mo')

# Access data for specific ticker
squrpharma_close = df['Close']['SQURPHARMA']
```

### Download All Available Tickers

Download all available data (OHLCV + Others) for all DSE-listed stocks

```python
df = dsebd.download_all(period='1mo')
```

### Using the Ticker Class

```python
# Create a Ticker object
ticker = dsebd.Ticker('SQURPHARMA')

# Get company information
print(ticker.info)

# Get current price data
print(ticker.current)

# Get historical OHLCV data
history = ticker.history(period='1y')
```

### Using the Tickers Class

```python
tickers = dsebd.Tickers(['SQURPHARMA', 'BEXIMCO', 'GP'])

# Access individual ticker objects
squrpharma = tickers.tickers['SQURPHARMA']
print(squrpharma.info)

# Get historical OHLCV data for all tickers
history = tickers.history(period='6mo')
```

### Dividend Data & Price Adjustment

Stock dividends (bonus shares) like 200% or 2:1 can severely skew historical prices. DSEBD scrapes the full dividend archive from LankaBangla Financial Portal (https://lankabd.com/Home/DividendArchive) and can adjust historical prices accordingly.

```python
from dsebd import fetch_dividends, get_adjustment_factors

# Fetch dividend records for a ticker
divs = fetch_dividends('ACI')
print(divs[['Year', 'Cash_Dividend_Pct', 'Stock_Dividend_Pct', 'Record_Date']])

# See the adjustment factors for each stock dividend event
factors = get_adjustment_factors('STYLECRAFT')
# 410% stock div → factor = 100/510 = 0.196
# 150% stock div → factor = 100/250 = 0.400

# Download with price adjustment
df_adj = dsebd.download('STYLECRAFT', period='5y', adjusted=True)

# Ticker also supports adjusted history
t = dsebd.Ticker('ACI')
hist_adj = t.history(period='2y', adjusted=True)
divs = t.dividends()
```

### Force Update Data

Use explicit cache update for normal refreshes. The update checks each source at
most once per Bangladesh calendar day:

```python
dsebd.update()
```

Force an immediate source check when needed:

```python
dsebd.update(force=True)
```

Existing `force_update` arguments remain supported for compatibility, but
`dsebd.update()` is preferred because it updates all prices and dividends
together.

### Update Local Cache

Normal reads use local DuckDB data and do not make network requests. Refresh all
price data and the dividend archive explicitly:

```python
import dsebd

dsebd.update()
```

The update checks each source at most once per Bangladesh calendar day. Repeated
calls on the same day skip source requests. Use `force=True` to bypass the daily
check:

```python
dsebd.update(force=True)
```

Price rows are refreshed only through the latest available market date. Dividend
data is normalized, hashed, and written to DuckDB only when its content changes.

### Update Dividend Archive

```bash
# Via CLI
dsebd dividends

# Or alongside price data update
dsebd update --dividends
```

### Available Tickers List

```python
from dsebd import TICKERS_LIST

print(f"Total tickers: {len(TICKERS_LIST)}")
print(f"Sample tickers: {TICKERS_LIST[:10]}")
```

## API Reference

### Functions

#### `update(force=False)`
Refresh the local DuckDB cache for all prices and the dividend archive.

The cache performs at most one source check per Bangladesh calendar day. Use
`force=True` to bypass the daily check. Normal data reads do not make network
requests.

**Returns:** dictionary with `prices` and `dividends` update status

#### `download(tickers, start=None, end=None, period='1mo', force_update=False, adjusted=False)`
Download historical OHLCV data for one or more tickers.

**Parameters:**
- `tickers` (str or list): Single ticker symbol or list of ticker symbols
- `start` (str, optional): Start date in 'YYYY-MM-DD' format
- `end` (str, optional): End date in 'YYYY-MM-DD' format
- `period` (str, optional): Period shorthand ('1d', '5d', '1mo', '3mo', '6mo', '1y', etc.)
- `force_update` (bool, optional): Legacy per-request refresh flag
- `adjusted` (bool, optional): Adjust prices for stock dividends (default: False)

**Returns:** pandas DataFrame with OHLCV data

#### `download_all(start=None, end=None, period='1mo', force_update=False, adjusted=False)`
Download historical data for all available tickers.

**Parameters:** Same as `download()` except no `tickers` parameter

#### `fetch_dividends(ticker=None)`
Fetch dividend records from the local database.

**Parameters:**
- `ticker` (str, optional): Ticker symbol. If None, returns all records.

**Returns:** pandas DataFrame with columns: Ticker, Sector, Year, Dividend_Type, Cash_Dividend_Pct, Stock_Dividend_Pct, EPS, NAV, Publish_Date, Record_Date, AGM_Date, Duration, Year_End_Date, Remarks

#### `get_adjustment_factors(ticker)`
Compute cumulative stock dividend adjustment factors.

**Returns:** List of (type, record_date, cash_val, stock_factor) tuples

#### `adjust_prices(df, ticker)`
Apply stock dividend adjustments to a price DataFrame.

### Classes

#### `Ticker(ticker, force_update=False)`
Represents a single stock ticker.

**Attributes:**
- `ticker` (str): Ticker symbol
- `info` (dict): Company information and fundamentals
- `current` (dict): Current/latest price data

**Methods:**
- `history(start=None, end=None, period='1mo', force_update=False, adjusted=False)`: Get historical OHLCV data
- `dividends()`: Get dividend records for this ticker

#### `Tickers(tickers, force_update=False)`
Represents multiple stock tickers.

**Attributes:**
- `tickers_name` (list): List of ticker symbols
- `tickers` (dict): Dictionary mapping ticker symbols to Ticker objects

**Methods:**
- `history(start=None, end=None, period='1mo', force_update=False, adjusted=False)`: Get historical data for all tickers

### Constants

#### `TICKERS_LIST`
List of all available ticker symbols on DSE.

### CLI Commands

```bash
# Update price data
dsebd update

# Update dividend archive
dsebd dividends

# Update both
dsebd update --dividends
```

## Data Storage

The package uses DuckDB to cache downloaded data locally. The database file (`stock_data.duckdb`) is stored in the package directory. Tables:

- `hist_stock_data` — OHLCV price data per ticker per date
- `company_details` — Company fundamentals (JSON blob)
- `dividends` — Full dividend archive with record dates and percentages
- `cache_metadata` — Internal daily source-check and content-hash metadata

## Requirements

- Python >= 3.10
- pandas >= 1.5.0
- numpy >= 1.20.0
- duckdb >= 0.9.0
- aiohttp >= 3.8.0
- lxml >= 4.9.0
- tqdm >= 4.65.0

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Disclaimer

This package is for educational and research purposes only. The data is sourced from publicly available information on the Dhaka Stock Exchange website and LankaBangla Financial Portal. Please verify all data independently before making any investment decisions.

## Acknowledgments

- Data sources: [Dhaka Stock Exchange (DSE)](https://www.dsebd.org/), [LankaBangla Financial Portal](https://lankabd.com/)
- Inspired by the `yfinance` package

## Support

For issues, questions, or contributions, please visit the [GitHub repository](https://github.com/0kamrulhasan0/dsebd).
