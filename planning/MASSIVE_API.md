# Massive (formerly Polygon.io) API Reference

Massive rebranded from Polygon.io on October 30, 2025. All existing API keys, accounts,
and integrations continue to work unchanged. The Python SDK package is `massive`.

---

## Authentication

```python
from massive import RESTClient
client = RESTClient(api_key="YOUR_MASSIVE_API_KEY")
```

The SDK wraps `https://api.polygon.io`. All raw HTTP calls require `?apiKey=<key>`.

---

## Endpoints Used by This Project

### 1. Full Market Snapshot (primary polling endpoint)

`GET /v2/snapshot/locale/us/markets/stocks/tickers`

Fetches the latest price data for multiple tickers in a **single** request.
This is the right endpoint for watchlist polling — one call covers all tickers.

**Query parameters:**

| Parameter     | Type              | Notes                                              |
|---------------|-------------------|----------------------------------------------------|
| `tickers`     | comma-sep string  | e.g. `AAPL,TSLA,NVDA`. Omit to get all US stocks. |
| `include_otc` | boolean           | Include OTC securities; default `false`.           |

**Response shape:**

```json
{
  "count": 2,
  "status": "OK",
  "tickers": [
    {
      "ticker": "AAPL",
      "todaysChange": 2.31,
      "todaysChangePerc": 1.22,
      "updated": 1712345678000,
      "day":     { "o": 187.50, "h": 191.20, "l": 186.80, "c": 190.25, "v": 54321000, "vw": 189.10 },
      "prevDay": { "o": 185.00, "h": 188.50, "l": 184.30, "c": 187.94, "v": 61200000, "vw": 186.75 },
      "min":     { "o": 190.10, "h": 190.40, "l": 189.90, "c": 190.25, "v": 12340, "t": 1712345640000 },
      "lastTrade": { "p": 190.25, "s": 100, "t": 1712345678000, "x": 4 },
      "lastQuote": { "P": 190.26, "Q": 200, "p": 190.24, "S": 300, "t": 1712345679000 }
    }
  ]
}
```

`lastTrade` fields: `p` = price, `s` = size, `t` = timestamp (Unix ms), `x` = exchange ID.
`lastQuote` fields: `P` = ask price, `p` = bid price, `Q` = ask size, `S` = bid size.

**Python SDK:**

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient(api_key=api_key)

# The RESTClient is synchronous — use asyncio.to_thread in async contexts
snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "TSLA", "NVDA"],
)

for snap in snapshots:
    price = snap.last_trade.price
    ts = snap.last_trade.timestamp / 1000.0  # ms → seconds
    change_pct = snap.todays_change_perc
    print(f"{snap.ticker}: ${price:.2f}  ({change_pct:+.2f}%)")
```

**Rate limits:**

| Plan      | Requests/min | Recommended poll interval |
|-----------|--------------|---------------------------|
| Free      | 5            | 15 s                      |
| Starter   | 100          | 5 s                       |
| Developer | 1 000        | 2 s                       |
| Advanced  | unlimited    | < 1 s                     |

Snapshot data clears at 3:30 AM EST and refreshes from ~4:00 AM EST.

---

### 2. Previous Day Bar (end-of-day reference price)

`GET /v2/aggs/ticker/{ticker}/prev`

Returns OHLCV for the last completed trading session. Used to compute the "daily change"
percentage shown in the watchlist (today's price vs. yesterday's close).

**Parameters:** `adjusted` (boolean, default `true`) — split-adjusted prices.

**Response:**

```json
{
  "ticker": "AAPL",
  "status": "OK",
  "results": [
    { "T": "AAPL", "o": 185.00, "h": 188.50, "l": 184.30, "c": 187.94,
      "v": 61200000, "vw": 186.75, "t": 1712188800000, "n": 523401 }
  ]
}
```

Fields: `T` = ticker, `o/h/l/c` = OHLC, `v` = volume, `vw` = VWAP,
`t` = session start (Unix ms), `n` = transaction count.

**Python SDK:**

```python
aggs = client.get_previous_close_agg(ticker="AAPL")
prev_close = aggs[0].close   # float
prev_open  = aggs[0].open    # float
```

---

### 3. Single Ticker Snapshot

`GET /v2/snapshot/locale/us/markets/stocks/tickers/{ticker}`

Same shape as one element of the full snapshot. Useful for a quick price lookup
on a single ticker without fetching the whole watchlist.

---

### 4. Last Trade

`GET /v2/last/trade/{ticker}`

Most recent trade tick. Lighter than a full snapshot.
Note: timestamp `t` here is **nanoseconds** (unlike snapshot's milliseconds).

```json
{ "status": "OK", "results": { "T": "AAPL", "p": 190.25, "s": 100, "t": 1712345678000000000, "x": 4 } }
```

---

## Async Usage Pattern (FastAPI)

The Massive `RESTClient` is **synchronous**. Always wrap in `asyncio.to_thread`:

```python
import asyncio
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient(api_key=api_key)

snapshots = await asyncio.to_thread(
    client.get_snapshot_all,
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "NVDA"],
)
```

---

## Error Handling

| HTTP | Meaning                                  |
|------|------------------------------------------|
| 401  | Invalid/missing API key                  |
| 403  | Endpoint requires a higher plan          |
| 404  | Ticker not found                         |
| 429  | Rate limit exceeded — back off and retry |

Standard pattern: catch broadly, log, and let the poll loop retry:

```python
try:
    snapshots = client.get_snapshot_all(
        market_type=SnapshotMarketType.STOCKS,
        tickers=tickers,
    )
except Exception as e:
    logger.error("Massive poll failed: %s", e)
    # Don't re-raise — the background loop will retry on the next interval
```

---

## Installation

```bash
uv add massive
```
