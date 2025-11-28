# Stock Analytics API

_Last updated: 2025-11-28 05:15:56_  


This document describes all HTTP Cloud Functions / API endpoints in this stock analytics system, along with the **data they touch** and the **calculations they perform**.

---

## 1. Import Stock Data

### 1.1 Endpoint

**HTTP method:** `GET` or `POST`  
**URL:**  
```text
https://us-central1-brilliant-tide-478610-b2.cloudfunctions.net/import-stock-data
```

### 1.2 Parameters

You can pass **one or multiple symbols**, via query string or JSON body.

#### Query parameters

- `symbol` – Single symbol, e.g. `AAPL`  
  - Example: `?symbol=AAPL`
- `symbols` – Comma-separated symbols  
  - Example: `?symbols=AAPL,MSFT,GOOG`

#### JSON body (POST)

```json
{ "symbol": "AAPL" }
```

or

```json
{ "symbols": ["AAPL", "MSFT", "GOOG"] }
```

### 1.3 Behavior

For each symbol:

1. **Fetch profile from FMP**  
   - Endpoint: `https://financialmodelingprep.com/api/v3/profile/{symbol}`  
   - Extracts:
     - `companyName`, `symbol`, `exchangeShortName`, `currency`, `sector`, `industry`, `country`

2. **Upsert into `tickers` table**  
   - On-conflict key: `symbol`  
   - Returns `ticker_id` to be used for all downstream data.

3. **Fetch historical prices from FMP**  
   - Endpoint:  
     ```text
     https://financialmodelingprep.com/api/v3/historical-price-full/{symbol}?from=2020-01-01
     ```
   - Reads:
     - `date`, `open`, `high`, `low`, `close`, `adjClose`, `volume`

4. **Write raw prices into `prices_daily`**  
   - Converted to:
     - `ticker_id`, `date`, `open`, `high`, `low`, `close`, `adj_close`, `volume`
   - Upsert on: `ticker_id,date`

5. **Compute & write weekly metrics into `weekly_metrics`**  
   - See **Section 1.4 Weekly Calculations**.

6. **Compute & write daily metrics into `daily_metrics`**  
   - See **Section 1.5 Daily Calculations**.

### 1.4 Weekly Calculations

Given the full daily price DataFrame `df`:

#### 1.4.1 Weekly Resampling

```python
weekly = (
    df.set_index("date")
      .resample("W-FRI")
      .agg({"close": "last", "volume": "sum"})
      .dropna(subset=["close"])
)
```

- `week_end_date` = index after resampling (Friday-based weeks)
- `last_price` = weekly close
- `volume` = sum over the week
- `week_number` = simple index from 0, 1, 2, ...

#### 1.4.2 Weekly Moving Averages

For each row `i`:

- `ma_4w` = mean of last 4 weekly closes (requires at least 4)
- `ma_10w` = mean of last 10 weekly closes
- `ma_13w` = mean of last 13 weekly closes
- `ma_26w` = mean of last 26 weekly closes
- `ma_40w` = mean of last 40 weekly closes

### 1.4.3 26-Week Weighted Relative Strength (RSR)

1. Take contiguous array of weekly prices:  
   `arr = weekly["last_price"].to_numpy(dtype=float)`

2. Use sliding windows of length 26 (RSR_WINDOW):  
   ```python
   rolling_windows = np.lib.stride_tricks.sliding_window_view(arr, 26)[:-1]
   current_prices = arr[26:]
   rs = current_prices[:, None] / rolling_windows  # relative strength vs each prior week
   ```

3. Weights and scaling:  
   ```python
   RSR_WINDOW = 26
   RSR_SCALE  = 1_000_000
   RSR_WEIGHTS = (np.arange(1, RSR_WINDOW + 1) / 351.0).reshape(1, RSR_WINDOW)
   ```

4. Final RSR at each point:  
   ```python
   rsr_values = (rs * RSR_WEIGHTS * RSR_SCALE).sum(axis=1)
   ```

5. First 26 rows are `None` (insufficient lookback).

### 1.4.4 Rolling Highs & New-High Flags

For N in {13, 26, 52}:

- `high_Nw` = rolling max of `last_price` over N weeks  
- `is_Nw_high` = `"Yes"` if `last_price == high_Nw` else `"No"`

### 1.4.5 Rate of Change (ROC)

For N in {13, 26, 52} weeks:

```python
roc_Nw = (current_last_price / last_price.shift(N) - 1.0) * 100.0
```

### 1.4.6 Backward Returns (ret_mNw)

For N in {1, 4, 13, 26, 52} weeks:

```python
ret_mNw = (last_price / last_price.shift(N) - 1.0) * 100.0
```

### 1.4.7 Forward Returns (ret_pNw)

For N in {1, 4, 13, 26, 52} weeks:

```python
ret_pNw = (last_price.shift(-N) / last_price - 1.0) * 100.0
```

### 1.4.8 Trend Signal (`trend_signal`)

Depends on:

- `price = last_price`
- `ma = ma_40w` (current week)
- `prev_ma = ma_40w` (previous week)

Logic:

- `++` if `price > ma` and `ma > prev_ma` (bullish, MA rising)
- `+-` if `price > ma` and `ma < prev_ma` (above MA but MA falling)
- `-+` if `price < ma` and `ma > prev_ma` (below MA but MA rising)
- `--` if `price < ma` and `ma < prev_ma` (bearish, MA falling)
- `None` otherwise (e.g., missing MA)

### 1.4.9 Trend Setup (`trend_setup`)

Uses short/medium/long weekly MAs:

- `S = ma_4w`
- `M = ma_13w`
- `L = ma_26w`

Cases:

- **U-series (bullish stacking)**  
  - `U1` if `L > S > M`  
  - `U2` if `S > L > M`  
  - `U3` if `S > M > L`
- **D-series (bearish stacking)**  
  - `D1` if `M > S > L`  
  - `D2` if `M > L > S`  
  - `D3` if `L > M > S`
- `None` otherwise or if MA values are missing.

---

### 1.5 Daily Calculations (`daily_metrics`)

Given full daily `df` sorted by `date`:

1. `last_price = close`

2. **Cumulative max**  
   ```python
   max_to_date = close.cummax()
   ```

3. **Drawdown in %**  
   ```python
   drawdown_pct = (close / max_to_date - 1.0) * 100.0
   ```

4. **Daily moving averages**  
   - `ma_20d` = 20-day simple moving average of `close`
   - `ma_50d` = 50-day SMA
   - `ma_100d` = 100-day SMA
   - `ma_200d` = 200-day SMA

5. **Deviation from 200-day MA**  
   ```python
   dev_from_200d_ma = (close / ma_200d - 1.0) * 100.0
   ```

All of these are stored in `daily_metrics` with `ticker_id` and `date`.

---

## 2. Export Stock Data

> **Note:** The implementation code for this endpoint was not shown, but based on naming and your earlier URLs, it likely returns the data already stored in Supabase for a given symbol.

### 2.1 Endpoint

**HTTP method:** `GET`  
**URL:**  
```text
https://us-central1-brilliant-tide-478610-b2.cloudfunctions.net/export-stock-data
```

### 2.2 Parameters

- `symbol` (required) – e.g. `GOOG`

### 2.3 Expected Behavior

Typical behavior (aligned with the import pipeline):

- Look up `ticker_id` for the requested `symbol` from `tickers`.
- Query `prices_daily` (and optionally `daily_metrics` / `weekly_metrics`).
- Return data as either:
  - JSON array of rows, or
  - CSV / Excel file (depending on implementation).

If you share the actual implementation later, this section can be updated with:
- Exact response schema
- Example responses.

---

## 3. Correlation Excel

**Python entry point:** `correlation_from_request`  
**Core async builder:** `build_correlation_excel_async`

### 3.1 Endpoint

**HTTP method:** `GET` or `POST`  
**URL:**  
```text
https://us-central1-brilliant-tide-478610-b2.cloudfunctions.net/correlation
```

### 3.2 Parameters

- `symbols` – Exactly **two** symbols, comma-separated or JSON list.

#### Examples

- Query string:  
  ```text
  ?symbols=AAPL,GOOG
  ```

- JSON body:  
  ```json
  { "symbols": ["AAPL", "GOOG"] }
  ```

### 3.3 Validations

- If no symbols: returns `400` with error.  
- If fewer than 2 symbols: `400` with `"At least TWO symbols are required"`.  
- If more than 2 symbols: `400` with `"Currently support 2 symbols only"`.  
- If ticker cannot be found: `400` with detailed error.  

### 3.4 Data Fetch & Calculations

1. **Fetch `ticker_id` for each symbol** via Supabase REST on `tickers`.
2. **Fetch weekly_metrics** for each ticker:
   - Fields: `week_end_date`, `last_price`
   - Ordered ascending by date.

3. **Align weekly data by dates**
   - Inner join on `week_end_date`.

4. **Compute weekly returns (%):**  
   For each symbol `sym`:
   ```python
   df[f"{sym}_return"] = df[f"last_price_{sym}"].pct_change()
   ```

5. **Compute rolling correlations:**

   Using window sizes:

   - `ROLL_26 = 26` weeks
   - `ROLL_52 = 52` weeks

   ```python
   corr_26 = series1.rolling(26).corr(series2)
   corr_52 = series1.rolling(52).corr(series2)
   # First window-1 rows set to NaN
   ```

### 3.5 Excel Output Layout

The final Excel sheet contains columns:

| Column               | Description                                      |
|----------------------|--------------------------------------------------|
| Date                 | Week end date                                    |
| {SYM1}               | Weekly last_price for symbol 1                   |
| {SYM1} % Return      | Weekly percentage return for symbol 1           |
| {SYM2}               | Weekly last_price for symbol 2                   |
| {SYM2} % Return      | Weekly percentage return for symbol 2           |
| 26-Week Correlation  | Rolling 26-week correlation between returns      |
| 52-Week Correlation  | Rolling 52-week correlation between returns      |

The file is returned as:

- Content-Type:  
  `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`
- Download name:  
  `correlation_{SYM1}_{SYM2}.xlsx`

---

## 4. Daily SMA Status (Streaks, Excel)

**Python entry point:** `daily_sma_status`  
**Core async function:** `process_symbol`

### 4.1 Endpoint

**HTTP method:** `GET`  
**URL:**  
```text
http://us-central1-brilliant-tide-478610-b2.cloudfunctions.net/dailySmaStatus
```

### 4.2 Purpose

Build an **Excel file** that shows days when price is above/below a given SMA, grouped into **streaks**, and filtered by **min/max streak length**.

### 4.3 Parameters

#### Required

- `symbol` – e.g. `AAPL`

#### Optional

- `start` – Start date: `YYYY-MM-DD` (filter price window)  
- `end` – End date: `YYYY-MM-DD` (filter price window)  
- `SMA` or `sma` – SMA period; supported:
  - `50` → uses `ma_50d` column from `daily_metrics`
  - `200` → uses `ma_200d`
- `above` – Minimum streak length (in days).  
  - If provided, default `direction` becomes `"above"`.
- `below` – Maximum streak length (in days).  
  - If provided, default `direction` becomes `"below"`.
- `direction` – `"above"` or `"below"`.
  - Explicitly sets whether we measure streaks of price **above** or **below** SMA.
  - If omitted, inferred from `above`/`below`, else default `"above"`.

### 4.4 Validations

- Must provide `symbol`.  
- Cannot provide **both** `above` and `below` simultaneously.  
- `SMA` must be one of the supported values (50, 200).  
- `direction` must be `"above"` or `"below"`.

### 4.5 Data Fetch & Streak Logic

1. **Resolve SMA column**

   ```python
   VALID_SMA_COLUMNS = { 50: "ma_50d", 200: "ma_200d" }
   ma_column = VALID_SMA_COLUMNS[sma]
   ```

2. **Fetch ticker_id from `tickers`**.

3. **Fetch daily_metrics** for that ticker:

   - Columns: `date`, `last_price`, `{ma_column}`
   - Filter by `start` / `end` if supplied.
   - Ensure numeric types for calculations.

4. **Compute streaks (vectorized)**

   Using `compute_streak_fast(df, ma_column, direction)`:

   - Compute binary flag:
     - If `direction == "above"`: `filter = (last_price > ma_column)`
     - If `direction == "below"`: `filter = (last_price < ma_column)`
   - Determine `streak_id`:
     - Increments each time `filter` goes from 0 → 1
     - Zero when not in a streak
   - Compute `days_in_state`:
     - `groupby("streak_id").cumcount() + 1` for rows inside streaks
     - 0 outside streaks

5. **Filter streaks by length**

   Using `extract_highest_per_streak(df, min_length, max_length)`:

   - For each `streak_id > 0`, keep the row where `days_in_state` is maximum (i.e., last day of the streak).  
   - If `min_length` is set: only keep streaks `>= min_length`.  
   - If `max_length` is set: only keep streaks `<= max_length`.

6. **Column renaming**

   - If `direction == "above"`:
     - `filter` → `is_above_ma`
     - `days_in_state` → `days_above`
   - If `direction == "below"`:
     - `filter` → `is_below_ma`
     - `days_in_state` → `days_below`

### 4.6 Excel Output Layout

Final Excel columns:

| Column         | Description                                           |
|----------------|-------------------------------------------------------|
| date           | Date of last day in the streak                       |
| last_price     | Price on that date                                   |
| ma_XXd         | The SMA used (50 or 200) on that date                |
| is_above_ma / is_below_ma | 1 if in state (above/below), 0 otherwise |
| days_above / days_below   | Length (in trading days) of the streak    |

- Sheet name: `SMA{sma}_{direction}` (trimmed to 31 chars if needed)  
- Filename: `{symbol}_SMA{sma}_{direction}.xlsx`

---

## 5. Daily Above MA50 (Backward-Compatible Wrapper)

**Python entry point:** `daily_above_ma50`

### 5.1 Endpoint

Same as `dailySmaStatus` (HTTP function wrapper).

### 5.2 Behavior

Simply delegates to:

```python
return daily_sma_status(request)
```

Historically, this endpoint meant “days above 50-day MA”. Now it supports the full SMA/threshold/streak logic via the shared implementation.

---

## 6. Daily Seasonality – Monthly (Day-of-Month Pattern)

**Python entry point:** `seasonality_monthly_http`  
Core logic built around **day-of-month** returns.

### 6.1 Endpoint

**HTTP method:** `GET` or `POST`  
**URL:**  
```text
https://us-central1-brilliant-tide-478610-b2.cloudfunctions.net/seasonality-monthly
```

### 6.2 Parameters

- `symbol` – Ticker symbol (e.g. `AAPL`)  
- `start` – Start date `YYYY-MM-DD`  
- `end` – End date `YYYY-MM-DD`  

Can be passed as:

- Query string: `?symbol=AAPL&start=2020-01-01&end=2025-11-30`
- Or JSON body: `{ "symbol": "AAPL", "start": "...", "end": "..." }`

### 6.3 Validations

- All three (`symbol`, `start`, `end`) are required.
- `start < end` must hold.  

### 6.4 Data Fetch

1. Validate inputs → `symbol.upper(), start_dt, end_dt`.  
2. Fetch daily prices via Supabase `prices_daily`:
   - Some buffer of ~1 month before start for internal calculations.
   - Columns: `date`, `close`
3. Filter to window `[start_dt, end_dt]` for returns.

### 6.5 Calculations – Day-of-Month Seasonality

1. **Daily returns**

   ```python
   prices = df["close"].sort_index()
   returns = prices.pct_change()
   window = returns.loc[(index >= start_dt) & (index <= end_dt)].dropna()
   ```

   Each row becomes:

   ```text
   return, month, day  (where day is day-of-month: 1–31)
   ```

2. **Average by (day, month)**

   Group by `day` and `month` and compute mean return:

   ```python
   grouped = returns.groupby(["day", "month"])["return"].mean()
   ```

3. **Pivot to table (rows = day-of-month)**

   ```python
   pivot = grouped.unstack("month").reindex(index=1..31, columns=1..12)
   pivot.columns = ["Jan","Feb",...,"Dec"]
   ```

This yields a 31×12 table of expected daily returns by **day-of-month vs month**.

### 6.6 Row-Level Summaries

For each `Day` row:

- `Average` – Mean of monthly returns across all months
- `Median` – Median of monthly returns
- `Up` – Count of months with positive average return
- `Down` – Count of months with negative average return
- `% Positive` – `Up / (Up + Down)`
- `Maximum` – Best month’s average return for that day
- `Minimum` – Worst month’s average return for that day

These are appended as extra columns.

### 6.7 Formatting

The table is formatted to display:

- Percentage columns (`Jan`–`Dec`, `Average`, `Median`, `% Positive`, `Maximum`, `Minimum`) as `"±xx.xx%"`
- Count columns (`Up`, `Down`, `Day`) as integers
- Missing values shown as `--`

### 6.8 PDF Output

`seasonality_monthly_http` builds a **single-page PDF**:

- Title:  
  ```text
  {SYMBOL} Daily Seasonality
  ({start} to {end})
  ```
- Body: table with columns:

  `Day, Jan, Feb, ..., Dec, Average, Median, Up, Down, % Positive, Maximum, Minimum`

- Footer: “Reported on: YYYY-MM-DD HH:MM:SS”

File is returned as:

- MIME type: `application/pdf`
- Filename: `{SYMBOL}_{start}_{end}_monthly.pdf`

---

## 7. Yearly Seasonality – Monthly & Annual (Yearly Table)

**Python entry point:** `seasonality_http`  
This is the **yearly/annual seasonality** view.

### 7.1 Endpoint

**HTTP method:** `GET` or `POST`  
**URL:**  
```text
https://us-central1-brilliant-tide-478610-b2.cloudfunctions.net/seasonality-yearly
```

### 7.2 Parameters

- `symbol` – Ticker symbol (e.g. `AAPL`)  
- `start` – Start date `YYYY-MM-DD`  
- `end` – End date `YYYY-MM-DD`  

Same pattern: query string or JSON body.

### 7.3 Data Fetch

1. Validate inputs → `symbol.upper(), start_dt, end_dt`  
2. Fetch daily prices from `prices_daily` similar to monthly seasonality, with buffered start as needed.

### 7.4 Monthly Returns (for Yearly Matrix)

1. Resample daily close to **end-of-month**:

   ```python
   monthly_close = df["close"].resample("M").last()
   monthly_returns = monthly_close.pct_change().dropna()
   ```

2. Filter to `[start_dt, end_dt]`:

   ```python
   window = monthly_returns[(index >= start_dt) & (index <= end_dt)]
   ```

3. Each entry has `(Year, Month, return)`; pivot to rows=Year, columns=Month:

   ```python
   pivot = returns.to_frame(name="return")
   pivot["Year"] = index.year
   pivot["Month"] = index.month
   table = pivot.pivot(index="Year", columns="Month", values="return").reindex(columns=1..12)
   table.columns = ["Jan", "Feb", ..., "Dec"]
   ```

### 7.5 Annual Returns

For each calendar year within the window:

- Use daily prices:
  - `first` = first close in the year within window
  - `last`  = last close in the year within window
- If non-null and `first != 0`:

  ```python
  annual_return = (last / first) - 1.0
  ```

- Add as `Annual Return` column to the yearly row.

### 7.6 Summary Rows

Once the yearly matrix is built, append these summary rows at the bottom:

- `Average` – Mean across years for each column
- `Median` – Median across years
- `Up` – Count of years with positive returns
- `Down` – Count of years with negative returns
- `% Positive` – `Up / (Up + Down)`
- `Maximum` – Best yearly value
- `Minimum` – Worst yearly value

These summary row labels live in the `Year` column.

### 7.7 Formatting & Output

The final display table has columns:

| Column        | Description                             |
|---------------|-----------------------------------------|
| Year          | Calendar year or summary row label      |
| Jan … Dec     | Monthly returns as percentages          |
| Annual Return | Full-year return as percentage          |

Formatting:

- Most numeric cells rendered as `"xx.xx%"`
- In `Up` / `Down` rows, numeric cells are rendered as integer counts
- Missing values rendered as `--`

Output:

- Rendered as a PDF with a large table.
- Title:

  ```text
  {SYMBOL}
  ({start_year} - {end_year})
  ```

- Footer with timestamp.  
- Returned as:

  - MIME type: `application/pdf`
  - Filename: `{SYMBOL}_{start}_{end}.pdf`

---

## 8. Error Handling Summary

Across endpoints:

- Missing required parameters → HTTP 400 with `{ "error": "..." }`
- Invalid symbols / no Supabase data → HTTP 400 or 500 depending on context.
- Unexpected exceptions → HTTP 500 with a generic `"Internal error: ..."` or full message.

---

## 9. Quick Endpoint Reference

| # | Endpoint Name           | HTTP Method | URL Path Suffix            | Output Type | Main Calculations                         |
|---|-------------------------|-------------|----------------------------|------------|-------------------------------------------|
| 1 | Import Stock Data       | GET/POST    | `/import-stock-data`       | JSON       | Profile import, daily + weekly + metrics  |
| 2 | Export Stock Data       | GET         | `/export-stock-data`       | JSON/Excel | Data read-back (implementation-specific)  |
| 3 | Correlation             | GET/POST    | `/correlation`             | Excel      | Weekly returns & rolling correlations     |
| 4 | Daily SMA Status        | GET         | `/dailySmaStatus`          | Excel      | Above/below SMA streak logic              |
| 5 | Daily Above MA50        | GET         | wrapper → `dailySmaStatus` | Excel      | Backward compatibility wrapper            |
| 6 | Seasonality Monthly     | GET/POST    | `/seasonality-monthly`     | PDF        | Day-of-month performance table (+summary) |
| 7 | Seasonality Yearly      | GET/POST    | `/seasonality-yearly`      | PDF        | Year×Month matrix + annual returns        |

---

End of documentation.
