# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SECthingv2 is a comprehensive SEC (Securities and Exchange Commission) and CFTC (Commodity Futures Trading Commission) data scraper and analyzer. The tool downloads raw archives from various financial regulatory sources and processes them for analysis, particularly focused on swap transactions, equity derivatives, and insider trading data.

## Core Architecture

### Two Main Scripts

1. **Gamecock.py** (~300KB, 3000+ lines) - Primary scraper
   - Downloads archives from SEC EDGAR, CFTC swap repositories, exchange data, and insider trading records
   - Organized into download functions for each archive type (equities, credits, forex, commodities, rates, etc.)
   - Auto-installs required third-party modules on first run
   - Creates multiple subdirectories for different data types (EQUITY, CREDIT, CFTC_EQ, CFTC_CR, FOREX, etc.)

2. **analyze.py** (~60KB) - Data visualization tool
   - Processes CSV files from downloaded archives
   - Generates interactive charts with matplotlib showing swap transaction volumes, stock prices, volume, and FTD (Failed to Deliver) data
   - Supports daily, weekly, and monthly aggregation
   - Uses yfinance for stock price overlays and processes FTD archives

### Key Directory Structure

The scraper creates these subdirectories:
- `EQUITY/` - SEC equity swap data
- `CREDIT/` - SEC credit swap data
- `CFTC_EQ/`, `CFTC_CR/`, `CFTC_CO/`, `FOREX/`, `CFTC_IR/` - CFTC swap data by asset class
- `EDGAR/` - EDGAR filing archives and full-text downloads
- `EXCHANGE/` - Exchange listing/delisting data
- `INSIDERS/` - Insider trading form 4/5 filings
- `SecNport/`, `SecNcen/`, `SecNmfp/`, `Sec13F/`, `SecFormD/` - Structured SEC data sets
- `NCSR/` - N-CSR filings (fund reports)
- `FTD/` - Failed to deliver data (used by analyze.py)

## Running the Scripts

### Gamecock.py (Scraper)

```bash
# Run the main scraper - it will auto-install dependencies
python3 Gamecock.py

# Or use the filename from README
python3 gamecock.py
```

The script will:
1. Check and install required modules (chardet, pandas, requests, bs4, tqdm, lxml)
2. Display an interactive menu via `codex()` function
3. Prompt for which archive types to download
4. Create subdirectories and download archives with progress bars

### analyze.py (Visualization)

```bash
# Install dependencies first
pip3 install pandas matplotlib mplcursors yfinance

# Run the analyzer
python3 analyze.py
```

Interactive prompts will ask for:
1. Select subdirectory containing CSV files
2. Select specific CSV file to analyze
3. Chart options: execution vs expiration dates, aggregation period (daily/weekly/monthly)
4. Stock ticker for price/volume overlay
5. Whether to include price, volume, and FTD data

## Code Organization Patterns

### Gamecock.py Structure

- **Module Management** (lines 1-110): Auto-installation system for dependencies
- **Constants & Setup** (lines 22-62): Directory paths and configuration
- **Download Functions**: Each archive type has a dedicated function:
  - `download_exchange_archives()` - Exchange data
  - `download_insider_archives()` - Form 4/5 insider filings
  - `download_credit_archives()` - SEC credit swaps
  - `download_equities_archives()` - SEC equity swaps
  - `download_cftc_*_archives()` - CFTC data by asset class
  - `download_edgar_archives()` - EDGAR master indexes
  - `download_ncsr_filings()` - N-CSR fund reports
  - And many more for specific SEC structured datasets

- **Secondary Processing Functions**: Named with `_second()` suffix, used to search/filter downloaded archives
  - `edgar_second()` - Search EDGAR master files
  - `credits_second()` - Parse credit swap archives
  - `equities_second()` - Search equity swap archives by ticker/CUSIP

- **Utilities**:
  - `allyourbasearebelongtous()` - EDGAR filing downloader using idx files
  - `process_cik()` - Download all filings for a specific company CIK
  - `codex()` - Main interactive menu system

### analyze.py Structure

- **Main Workflow** (lines 76-981):
  1. Directory and CSV file selection
  2. Data loading and conversion
  3. Position processing (NEWT/MODI/TERM action types)
  4. FTD data generation from zip archives if ticker provided
  5. Chart generation with `chart_all_expiries()` function

- **Key Functions**:
  - `parse_zips_in_batches()` - Extracts FTD data for specific ticker from zip archives
  - `aggregate_notional_by_currency()` - Aggregates swap notional amounts by currency
  - `chart_all_expiries()` - Main charting function with price/volume/FTD overlays
  - `log_to_console()` - Timestamped logging

- **Data Processing**:
  - Handles swap action types: NEWT (new), MODI (modified), TERM (terminated)
  - Processes columns by index (iloc[19], iloc[21], iloc[25]) for notional amounts and currencies
  - Excludes basket swaps from notional calculations due to unclear weighting

## Important Implementation Details

### Rate Limiting & User Agents

- User agent rotation is implemented (lines 67-74 in Gamecock.py) for SEC requests
- Rate limiting varies by archive type (some use delays, others use concurrent downloads)
- SEC requires proper User-Agent headers to avoid 403 errors

### Data Format Handling

- Most SEC archives use pipe-delimited (|) CSV format
- Encoding issues handled with UTF-8 fallback to latin1
- ZIP files often contain single CSV files that must be extracted for processing
- FTD data uses YYYYMMDD format for settlement dates

### Chart Backends

analyze.py uses Qt5Agg backend for matplotlib (line 4). If charts don't display:
- Try TkAgg backend instead: `matplotlib.use('TkAgg')`
- Hover functionality requires interactive backend and mplcursors

### Memory Management

- Large files (Gamecock.py is 300KB+) should be read with offset/limit parameters
- analyze.py uses `low_memory=False` and `on_bad_lines='skip'` for pandas
- Batch processing used for FTD zip parsing (default 100 files per batch)

## Common Development Tasks

### Adding New Archive Source

1. Define source directory constant at top of Gamecock.py
2. Add to `directories` list (line 42-59)
3. Create `download_<name>_archives()` function following existing patterns
4. Add menu entry in `codex()` function
5. Optionally create `<name>_second()` for post-download processing

### Modifying Chart Output

Charts in analyze.py:
- Bar charts show NEWT/MODI/TERM action counts
- Trendlines use secondary y-axes for price (orange), volume (purple), FTD (red)
- Hover tooltips created via mplcursors callback (lines 478-542)
- Saved as PNG files with descriptive filenames in current directory

### Working with Swap Data

Swap CSVs contain columns:
- `Dissemination Identifier` - Unique swap ID
- `Action type` - NEWT, MODI, or TERM
- `Execution Timestamp` - Trade execution date
- `Expiration Date` - Swap maturity date
- `Product name` - Swap type (e.g., "Equity:Swap:PriceReturnBasicPerformance:Basket")
- `Notional amount-Leg 1` (index 19) - Contract notional value
- Currency (index 21) - Notional currency
- `Total notional quantity-Leg 1` (index 25) - Share/contract quantity

## Testing

No formal test suite exists. Manual testing workflow:
1. Run Gamecock.py and download small archive set
2. Verify files appear in correct subdirectories
3. Run analyze.py on downloaded CSVs
4. Verify charts display and save correctly

## Dependencies

**Gamecock.py** auto-installs:
- chardet - Character encoding detection
- pandas - Data manipulation
- requests - HTTP requests
- bs4 (BeautifulSoup) - HTML parsing
- tqdm - Progress bars
- lxml - XML/HTML parsing

**analyze.py** requires manual install:
- pandas, matplotlib, mplcursors, yfinance

Python 3.12 recommended (per README).

## Notes on Data Sources

- SEC EDGAR: https://www.sec.gov/Archives/edgar/
- SEC structured data: https://www.sec.gov/files/structureddata/data/
- CFTC swaps: https://pddata.dtcc.com/ (via hardcoded URL lists)
- Exchange data: nasdaq.com, nyse.com listing archives
- FTD data: sec.gov fails-to-deliver archives (used by analyze.py)
