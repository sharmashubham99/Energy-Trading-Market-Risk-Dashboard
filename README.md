# Energy Trading Market Risk Dashboard

A Power BI report modeling daily market risk exposure, PnL, and limit utilization
across power, gas, coal, and carbon (EUA) trading desks. Built as a portfolio
project targeting Market Risk / Regulatory Reporting analyst roles.

## Why this project
Real market risk reporting means: pull trade-level data, roll it up by desk and
commodity, track exposure and PnL against risk limits, and flag breaches — daily.
This project reproduces that workflow end-to-end in Power BI, on realistic
(synthetic) energy trading data.

## Data model (star schema)
- `fact_trades.csv` — 11,338 trades, Jan 2024 – Dec 2025 (weekdays only)
  - TradeID, Date, CommodityID, TraderID, Position (Long/Short), Volume,
    Price, Notional_EUR, PnL_EUR
- `dim_date.csv` — calendar table (Year, Quarter, Month, Week, DayOfWeek)
- `dim_commodity.csv` — Power, Gas, Coal, Carbon (EUA), with unit and currency
- `dim_trader.csv` — 8 traders across 3 desks (Power / Gas / Coal & Emissions)
- `risk_limits.csv` — VaR and exposure limit per desk

Relationships: fact_trades connects to each dim table via a one-to-many
relationship on the matching ID column. dim_date connects on Date.

## How to build it
1. Open Power BI Desktop → Get Data → Text/CSV → import all 5 files from `/data`.
2. Go to Model view. Drag to connect:
   - fact_trades[CommodityID] → dim_commodity[CommodityID]
   - fact_trades[TraderID] → dim_trader[TraderID]
   - fact_trades[Date] → dim_date[Date]
   - dim_trader[Desk] → risk_limits[Desk]
3. Mark `dim_date` as a Date Table (Model view → right-click table → Mark as
   Date Table) — required for time-intelligence DAX to work correctly.
4. Add the DAX measures below in a new table called `_Measures`.
5. Build report pages (see "Suggested pages" below).
6. Publish to Power BI Service (Home → Publish) once you have a free account.

## DAX measures to write yourself (don't paste-and-forget — type these out,
they're the actual skill)

```
Total Notional Exposure =
SUMX ( fact_trades, ABS ( fact_trades[Notional_EUR] ) )

Net Position EUR =
SUMX (
    fact_trades,
    IF ( fact_trades[Position] = "Long", fact_trades[Notional_EUR], -fact_trades[Notional_EUR] )
)

Total PnL =
SUM ( fact_trades[PnL_EUR] )

Cumulative PnL =
CALCULATE (
    [Total PnL],
    FILTER ( ALLSELECTED ( dim_date ), dim_date[Date] <= MAX ( dim_date[Date] ) )
)

Daily VaR 95 (EUR) =
VAR DailyPnL =
    SUMMARIZE ( fact_trades, fact_trades[Date], "DayPnL", [Total PnL] )
RETURN
    ABS ( PERCENTILEX.INC ( DailyPnL, [DayPnL], 0.05 ) )

Exposure vs Limit % =
DIVIDE ( [Total Notional Exposure], SUM ( risk_limits[Exposure_Limit_EUR] ) )

Limit Breach Flag =
IF ( [Exposure vs Limit %] > 1, "BREACH", "OK" )
```

## Suggested report pages
1. **Overview** — KPI cards (Total Exposure, Total PnL, Daily VaR, Breach Flag),
   PnL trend line by month, exposure by commodity (donut or bar).
2. **Desk Risk** — matrix of Desk × Commodity showing Exposure, VaR, Limit,
   Utilization %, conditional formatting on breaches.
3. **Trade Explorer** — table with slicers (Date, Trader, Commodity, Position)
   for drill-down, plus a scatter of Volume vs PnL to spot outlier trades.


