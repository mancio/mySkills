---
name: stocks-near-support-report
description: Build from zero a Python app that generates a static HTML report listing S&P 500 (US) and STOXX Europe 600 (EU) stocks currently trading near a technical support level, sorted from cheapest to most expensive, annotated with the consensus analyst rating (Strong Buy / Buy / Hold / Sell / Strong Sell). Use when the user asks to "build the stocks-near-support app", "generate the support report", "rebuild the EU+US support screener", or similar. The skill prescribes the exact data sources, support algorithm, project layout, and HTML template to use; it must NOT invent placeholder data — every value comes from a live free API.
---

# Stocks Near Support — HTML Report Builder

## Goal

Produce a **single** `report.html` containing one sortable table of every S&P 500 (US) **and** STOXX Europe 600 (EU) constituent that:

1. is currently within a configurable distance of a computed support level, **and**
2. has a consensus analyst rating of **Buy** or **Strong Buy** (other ratings are dropped).

Rows sorted ascending by last close (cheapest → most expensive). Columns (exact order):

| Ticker | Name | Market | Last | Support | Dist % | 52w High | Drawdown | Days since high | Rating | # Analysts |

- **Market** = `US` or `EU`.
- **Drawdown** = `(52w high − last close) / 52w high × 100`, rendered as a negative percent.
- **Days since high** = trading-day distance from the bar of the 52w high to the last bar.
- **Rating** must be only `Buy` or `Strong Buy` — anything else excludes the row entirely.

No mock data. If a required secret is missing, **stop and ask the user** to provide it.

## Data sources (all free tier)

| Need | Source | Auth | Notes |
|---|---|---|---|
| S&P 500 constituents | Wikipedia `https://en.wikipedia.org/wiki/List_of_S%26P_500_companies` | none | parse with `pandas.read_html` |
| STOXX Europe 600 constituents | Wikipedia `https://en.wikipedia.org/wiki/STOXX_Europe_600` | none | parse with `pandas.read_html`; ticker column needs exchange suffix mapping (see below) |
| Daily OHLC history | `yfinance` (Yahoo Finance) | none | batch download, `period="1y"`, `interval="1d"` |
| Analyst consensus rating | Finnhub `/stock/recommendation` endpoint | **API key required** → ask user for `FINNHUB_API_KEY` | returns monthly buckets `strongBuy/buy/hold/sell/strongSell`; use most recent row |

If Finnhub rate-limits (60 req/min on free tier), throttle with `time.sleep(1.1)` between calls and cache responses to `.cache/finnhub/{symbol}.json` for 24 h.

### EU ticker → Yahoo suffix mapping (minimum set)

```
Germany     → .DE     France      → .PA     Netherlands → .AS
Switzerland → .SW     UK          → .L      Italy       → .MI
Spain       → .MC     Sweden      → .ST     Denmark     → .CO
Belgium     → .BR     Finland     → .HE     Norway      → .OL
Ireland     → .IR     Portugal    → .LS     Austria     → .VI
```

## Support algorithm

Compute per ticker on the last 252 trading days (1 year):

1. Find swing lows: a bar `i` is a swing low if `low[i] == min(low[i-5 : i+5+1])` (5-bar fractal, both sides).
2. Cluster swing-low prices that are within 1.5% of each other (simple bucketing).
3. The **support level** = the highest cluster price that is **strictly below** the last close. (i.e. nearest support from above-down view.)
4. **Distance %** = `(last_close - support) / last_close * 100`.
5. A stock is "near support" if `0 <= distance% <= NEAR_THRESHOLD_PCT` (default `5.0`, configurable via `.env`).

If no qualifying support exists below last close, drop the ticker.

## Analyst rating mapping

From the latest Finnhub recommendation row, pick the bucket with the highest count. Tie-break order: `strongBuy > buy > hold > sell > strongSell`. Display the human label and the total count `strongBuy+buy+hold+sell+strongSell`.

## Project layout

```
/
├── .env.example          # FINNHUB_API_KEY=, NEAR_THRESHOLD_PCT=5.0
├── requirements.txt      # yfinance, pandas, lxml, requests, python-dotenv, jinja2, tqdm
├── src/
│   ├── __init__.py
│   ├── config.py         # loads .env, exposes constants
│   ├── universe.py       # get_sp500() + get_stoxx600() → DataFrame[ticker, name, exchange]
│   ├── prices.py         # download_history(tickers) → dict[str, DataFrame]
│   ├── support.py        # compute_support(df) → (support, last_close, distance_pct) or None
│   ├── ratings.py        # fetch_rating(symbol) with on-disk cache
│   ├── report.py         # renders templates/report.html.j2 → report.html
│   └── main.py           # orchestrates the pipeline with tqdm progress bars
├── templates/
│   └── report.html.j2    # standalone HTML, inline CSS, sortable via small vanilla JS
└── report.html           # output (gitignored)
```

## Build steps (execute in order)

1. **Ask the user for the Finnhub API key** before writing any code. Do not proceed without it. Tell them to register at `https://finnhub.io/register` (free) and paste the key. Save to `.env`.
2. Create `requirements.txt` and install: `python -m venv .venv && .venv\Scripts\pip install -r requirements.txt` (Windows / PowerShell).
3. Implement modules in the order: `config → universe → prices → support → ratings → report → main`.
4. Each module gets a `__main__` block with a tiny smoke test (e.g. `python -m src.universe` prints the first 10 rows).
5. `main.py` flow:
   - load universe (US + EU concatenated)
   - batch-download prices via `yfinance.download(tickers, period="1y", group_by="ticker", auto_adjust=False, threads=True)`
   - for each ticker compute support; keep only "near support"
   - compute 52w high, drawdown %, and days-since-high from the same history (no extra fetch)
   - fetch ratings for the survivors only (minimises API calls)
   - **drop every row whose rating is not `Buy` or `Strong Buy`**
   - sort ascending by last close
   - render the single `report.html`
6. Print final summary: `"Wrote report.html with N rows (X US, Y EU)"`.

## HTML template requirements

- Single self-contained file (no external CSS/JS).
- Sticky header, zebra rows, monospace numerics, right-aligned numbers.
- Color the rating cell: Strong Buy = `#0a7d28`, Buy = `#3aa856`, Hold = `#8a8a8a`, Sell = `#d97706`, Strong Sell = `#b91c1c` (white text on dark backgrounds).
- Tiny vanilla-JS click-to-sort on every column header (no libraries).
- Footer shows generation timestamp (UTC) and the `NEAR_THRESHOLD_PCT` used.

## Hard rules

- **Never** fabricate prices, ratings, or constituents. If a fetch fails for a ticker, log a warning and skip it.
- **Never** commit `.env` or the `.cache/` folder — add both to `.gitignore`.
- If Wikipedia layout changes and parsing fails, raise a clear error pointing at the offending URL; do not fall back to a hard-coded list.
- Keep dependencies to the ones in `requirements.txt`; do not add paid SDKs.
- All network calls must have a 15 s timeout and one retry with exponential backoff.

## Done = 

Running `python -m src.main` from a clean checkout (after `pip install -r requirements.txt` and a populated `.env`) produces a non-empty `report.html` whose first row is the cheapest qualifying stock, whose **Rating** column contains only `Buy` or `Strong Buy`, and whose **Drawdown** / **52w High** / **Days since high** columns are populated for every row.

## Optional follow-up: extra drawdown filter (`src/fallen_near_support.py`)

The drawdown columns are already in `report.html`. An auxiliary module exists
to produce a **secondary, narrower** HTML (`report_fallen.html`) that keeps
only rows with `drawdown% >= --min-drawdown` (default `25.0`) — useful when
the user wants "fallen + near support" pullback candidates only.

$$\text{drawdown\%} = \frac{\text{52w high} - \text{last close}}{\text{52w high}} \times 100$$

### Module

```
src/fallen_near_support.py
```

- Parses `report.html` (regex-strips `<tr>/<td>` cells; tolerates the inner
  `<span class="pill ...">` / `<span class="rating ...">` wrappers).
- Re-uses `src.prices.download_history` to recompute the 52w high — no
  separate data source, no new API key.
- Writes `report_fallen.html` with the **same column set** as `report.html`,
  sorted by drawdown descending.

### CLI

```powershell
# default: ≥25% off 52w high
python -m src.fallen_near_support

# stricter (deep crashes only)
python -m src.fallen_near_support --min-drawdown 40

# any pullback ≥15%
python -m src.fallen_near_support --min-drawdown 15

# top N most-drawn-down
python -m src.fallen_near_support --min-drawdown 25 --top 20
```

### Hard rules

- No new data source — must reuse `download_history`.
- Never fabricate the 52w high; if history is missing for a ticker, drop it.
- Drawdown is computed from raw (not adjusted) `High` to match the support
  algorithm; if `auto_adjust` is ever flipped on, update this module in sync.
- Output is sorted by drawdown desc; ties broken by ticker asc.

