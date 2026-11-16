# Indian Stock Market Dashboard

**NSE & BSE | Historical Market Analysis**

A Power BI report that tracks intraday price and volume movement for 15 large-cap Indian stocks across the NSE and BSE. It covers market-wide gainers and losers, per-stock price action, and exchange-level trading volume.

---

## Contents

- [Report pages](#report-pages)
- [Data](#data)
- [Data model](#data-model)
- [DAX measures](#dax-measures)
- [Getting started](#getting-started)
- [Project structure](#project-structure)
- [Known limitations](#known-limitations)

---

## Report pages

The report has four pages (1920 × 1080, dark `#101827` background). Each page has the same header and a set of slicers.

### 1. Market Overview
A high-level snapshot of the market.
- KPI cards: Total Stocks, Total Listings, Latest Change %, Gainers Count, Losers Count
- Donut chart: total listings by exchange
- Clustered bar chart: latest change % by symbol, split by exchange (tooltips show latest price and volume)
- Tables: **Top 5 Gainers** and **Top 5 Losers** (symbol, exchange, latest price, latest change %)
- Slicers: Date, Exchange, Symbol

### 2. Individual Stock Analysis
A drill-down view of one stock.
- KPI cards: Latest Price, Open, High, Low, Change %
- Line chart: **Intraday Stock Price Movement** (last price over time; tooltips show open, high, low, volume)
- Column chart: **Trading volume** by trade date
- Slicers: Date, Exchange, Symbol

### 3. Gainers and Losers
- KPI cards: Gainers Count, Losers Count, Unchanged Count
- Bar chart: **Latest Change %** by symbol
- Tables: Top 5 Gainers and Top 5 Losers
- Slicers: Date, Exchange, Symbol

### 4. Trading Volume Analysis
- KPI card: Total Latest Volume
- 100% stacked bar: share of volume by exchange
- Column charts: **Exchange volume comparison** and **Daily volume trend**
- Table: **Volume details** (symbol, exchange, latest volume, average stock change %, latest change %)
- Slicers: Date, Exchange, Symbol

---

## Data

| | |
|---|---|
| **Source file** | `stock_history.csv` (loaded through Power Query) |
| **Rows** | 6,080 |
| **Granularity** | 5-minute intraday snapshots, about 304 per listing |
| **Coverage** | 15 symbols; 20 listings (15 on NSE, 5 on BSE) |
| **Period** | 9–12 Nov 2026 |

**Symbols:** AXISBANK, BHARTIARTL, HDFCBANK, HINDUNILVR, ICICIBANK, INFY, ITC, LT, MARUTI, NHPC, RELIANCE, SBIN, TATAMOTORS, TCS, WIPRO

### CSV columns

| Column | Type | Description |
|---|---|---|
| `symbol` | text | Ticker symbol |
| `exchange` | text | `NSE` or `BSE` |
| `last_price` | decimal | Price at the snapshot |
| `open_price` | decimal | Day's opening price |
| `high_price` | decimal | Day's high so far |
| `low_price` | decimal | Day's low so far |
| `previous_close` | decimal | Previous session's close |
| `change_percent` | decimal | % change vs. previous close |
| `volume` | integer | Cumulative traded volume |
| `recorded_at` | datetimezone | Snapshot timestamp |

### Columns added in Power Query

| Column | Logic |
|---|---|
| `TradeDate` | `Date.From([recorded_at])` |
| `CustomTradeTime` | `Time.From([recorded_at])` |

---

## Data model

Two tables:

- **`StockData`**: the fact table, with the CSV columns above plus `TradeDate`, `CustomTradeTime` and `Comparison` (a constant `"Exchange Volume"` used as the category axis of the 100% stacked bar).
- **`Calendar`**: a DAX date table built from `StockData[TradeDate]`.

```dax
Calendar =
CALENDAR(
    MIN(StockData[TradeDate]),
    MAX(StockData[TradeDate])
)
```

It has `Date`, `Year`, `Month`, `Month Number` and `Day` columns.

> **Note:** there is currently **no relationship** between `Calendar` and `StockData`. See [Known limitations](#known-limitations).

---

## DAX measures

All measures live in `StockData`. The "Latest" measures find the most recent `recorded_at` in the current filter context and read the value at that moment.

| Measure | Purpose |
|---|---|
| `Total Stocks` | Distinct count of symbols |
| `Total Listings` | Distinct symbol + exchange pairs |
| `Total Records` | Row count |
| `NSE Listings` / `BSE Listings` | Listings filtered to one exchange |
| `Latest Timestamp` | `MAX(recorded_at)` |
| `Latest Price` | Last price at the latest timestamp |
| `Latest Open` / `Latest High` / `Latest Low` | OHLC values at the latest timestamp |
| `Latest Change %` | Change % at the latest timestamp |
| `Latest Volume` | Volume at the latest timestamp |
| `Average Stock Change %` | Average of each listing's latest change % |
| `Gainers Count` | Listings with latest change % > 0 |
| `Losers Count` | Listings with latest change % < 0 |
| `Unchanged Count` | Listings with latest change % = 0 |
| `Total Latest Volume` | Sum of each listing's latest volume |

Pattern used by the "Latest" measures:

```dax
Latest Price =
VAR LatestTime = [Latest Timestamp]
RETURN
    CALCULATE(
        MAX(StockData[last_price]),
        REMOVEFILTERS(StockData[recorded_at]),
        StockData[recorded_at] = LatestTime
    )
```

---

## Getting started

**Requirements:** [Power BI Desktop](https://powerbi.microsoft.com/desktop/) (a recent release; the file uses the PBIR report format).

1. Open `Stocks.pbix`.
2. Point the data source at your copy of the CSV. The file currently loads from a local path:
   `C:\Users\Dell\Downloads\stock_history.csv`
   - Go to **Home → Transform data → Data source settings → Change Source**, or edit the `Source` step of the `StockData` query.
3. Click **Refresh**.

---

## Project structure

A `.pbix` file is a zip archive. Its main parts:

```
Stocks.pbix
├── DataModel                 # compressed tabular model (tables, DAX, Power Query)
├── DiagramLayout             # model view layout
├── Report/
│   ├── definition/
│   │   ├── report.json       # theme and report-level settings
│   │   └── pages/            # 4 pages, each with a visuals/ folder
│   └── StaticResources/      # Fluent2 base theme
├── Settings, Metadata, Version
└── SecurityBindings
```

---

## Known limitations

- **Date slicers may not filter the data.** `Calendar` has no relationship to `StockData`, so Date slicers sourced from `Calendar[Date]` have no effect on the `StockData` measures. Create a many-to-one relationship `StockData[TradeDate]` → `Calendar[Date]` to fix this. (The "Intraday" and "Trading volume" charts use `StockData` columns directly, so they are unaffected.)
- **Hard-coded file path.** The CSV path is local to one machine; update it (or use a parameter) before sharing.
- **Small typo.** One table title reads "TOP 5 Gainners" on the Market Overview page.
- **Static dataset.** The data covers a few days and does not refresh from a live feed.
