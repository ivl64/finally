# Market Data Backend — Implementation Design

**Status:** Fully implemented in `backend/app/market/` (8 modules, ~500 lines, 73 tests).

This document is the authoritative design reference for the market data subsystem. It
includes the actual implementation code, integration patterns, and rationale for every
major decision.

---

## 1. Architecture

```
Environment (MASSIVE_API_KEY)
        │
        ├── set & non-empty ──→ MassiveDataSource   (Polygon.io REST polling)
        └── absent / empty  ──→ SimulatorDataSource  (GBM price simulation)
                │
                ▼  writes prices on every tick
           PriceCache  (thread-safe in-memory store, version counter)
                │
                ├──→ GET /api/stream/prices  (SSE — one event per version bump)
                ├──→ GET /api/portfolio      (current position values)
                ├──→ POST /api/portfolio/trade  (fill price at execution)
                └──→ GET /api/watchlist      (prices alongside tickers)
```

**Key constraint:** All downstream code reads from `PriceCache` only. Nothing except
the two data sources ever writes to it. This decouples the data source from every
consumer completely.

---

## 2. Module Layout

```
backend/app/market/
├── __init__.py          # Public re-exports
├── models.py            # PriceUpdate — the canonical price record
├── interface.py         # MarketDataSource — abstract base class
├── cache.py             # PriceCache — thread-safe price store
├── seed_prices.py       # Seed prices and per-ticker GBM parameters
├── simulator.py         # GBMSimulator + SimulatorDataSource
├── massive_client.py    # MassiveDataSource (Polygon.io REST poller)
├── factory.py           # create_market_data_source() — env-driven selection
└── stream.py            # FastAPI SSE endpoint
```

Public API (everything else imports from here):

```python
from app.market import (
    PriceCache,
    PriceUpdate,
    MarketDataSource,
    create_market_data_source,
    create_stream_router,
)
```

---

## 3. Data Model: `PriceUpdate`

Every price change is represented as an immutable `PriceUpdate`. Producers create them;
consumers read them.

```python
# models.py
from dataclasses import dataclass, field
import time

@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float:
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> str:
        if self.price > self.previous_price:
            return "up"
        elif self.price < self.previous_price:
            return "down"
        return "flat"

    def to_dict(self) -> dict:
        """Serialize for JSON / SSE transmission."""
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
        }
```

**Design notes:**
- `frozen=True, slots=True` — immutable and memory-efficient; safe to pass across threads
- `previous_price` is always the price from the immediately preceding update, not a daily open
- `direction` is derived, not stored — stays consistent with price values
- `to_dict()` is the only serialization path; use it for SSE, REST responses, and JSON storage

---

## 4. `PriceCache` — Thread-Safe Price Store

The cache is the single source of truth for current prices. It is written by one
background task and read by many concurrent request handlers.

```python
# cache.py
from threading import Lock
from .models import PriceUpdate

class PriceCache:
    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0  # Bumped on every update; drives SSE change detection

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price   # First update: flat direction
            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None: ...
    def get_all(self) -> dict[str, PriceUpdate]: ...  # Shallow copy — safe for iteration
    def get_price(self, ticker: str) -> float | None: ...
    def remove(self, ticker: str) -> None: ...

    @property
    def version(self) -> int:
        """Monotonically increasing. The SSE generator polls this to detect changes."""
        return self._version
```

**Design notes:**
- `threading.Lock` (not `asyncio.Lock`) because `GBMSimulator.step()` runs in the async
  event loop but `get_all()` may also be called from sync FastAPI path handlers
- `get_all()` returns a shallow copy so the caller can iterate safely without holding the lock
- `version` counter is the SSE change-detection mechanism: the SSE generator wakes every
  500ms, checks if version changed since its last send, and only transmits if it did

---

## 5. Abstract Interface: `MarketDataSource`

Both implementations satisfy this contract. Downstream code only types against this ABC.

```python
# interface.py
from abc import ABC, abstractmethod

class MarketDataSource(ABC):

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing prices. Starts a background task. Call once at startup."""

    @abstractmethod
    async def stop(self) -> None:
        """Cancel background task. Safe to call multiple times."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. Immediately visible in the next tick."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Also purges it from PriceCache."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return currently tracked tickers."""
```

**Strategy pattern** — swapping `SimulatorDataSource` for `MassiveDataSource` (or any
future source) requires no changes in any other file.

---

## 6. Factory: Environment-Driven Source Selection

```python
# factory.py
import os

def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        logger.info("Market data source: Massive API (real data)")
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        logger.info("Market data source: GBM Simulator")
        return SimulatorDataSource(price_cache=price_cache)
```

The `.env` file in the project root is loaded into the environment before this runs.
The factory is the only place that inspects `MASSIVE_API_KEY`.

---

## 7. Simulator: `GBMSimulator` + `SimulatorDataSource`

### Mathematical Model

Each tick advances every price by one Geometric Brownian Motion step:

```
S(t+dt) = S(t) * exp( (mu - sigma²/2) * dt  +  sigma * sqrt(dt) * Z )
```

| Symbol  | Meaning                                         | Example (AAPL)         |
|---------|-------------------------------------------------|------------------------|
| S(t)    | Current price                                   | $190.00                |
| mu      | Annualized drift (expected return)              | 0.05 (5%/year)         |
| sigma   | Annualized volatility                           | 0.22 (22%/year)        |
| dt      | Tick duration as fraction of a trading year     | ~8.48×10⁻⁸ (500ms)     |
| Z       | Correlated standard normal draw                 | (see §7.2)             |

`dt` is calibrated to trading time, not clock time:

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ~8.48e-8
```

With this `dt`, each tick produces sub-cent moves. Over an hour (7,200 ticks), the
diffusion term accumulates to realistic intraday swings.

### Correlated Moves (Cholesky Decomposition)

Tech stocks tend to move together; finance stocks also correlate. The simulator
models this with a correlation matrix decomposed via Cholesky.

**Correlation coefficients:**

| Pair                                    | Coefficient |
|-----------------------------------------|-------------|
| Two tech stocks (AAPL, MSFT, GOOGL…)   | 0.60        |
| Two finance stocks (JPM, V)             | 0.50        |
| TSLA with any other ticker              | 0.30        |
| Cross-sector or unknown pairs           | 0.30        |

**Per-tick generation:**

```python
def step(self) -> dict[str, float]:
    n = len(self._tickers)
    z_independent = np.random.standard_normal(n)
    z_correlated  = self._cholesky @ z_independent   # correlated draws

    for i, ticker in enumerate(self._tickers):
        mu, sigma = params["mu"], params["sigma"]
        drift     = (mu - 0.5 * sigma**2) * self._dt
        diffusion = sigma * math.sqrt(self._dt) * z_correlated[i]
        self._prices[ticker] *= math.exp(drift + diffusion)
```

The Cholesky matrix is rebuilt whenever tickers are added or removed. With n < 50 this
is fast enough (O(n²)) to do synchronously.

### Seed Prices and Per-Ticker Parameters

```python
# seed_prices.py
SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00, "AMZN": 185.00,
    "TSLA": 250.00, "NVDA": 800.00,  "META": 500.00, "JPM":  195.00,
    "V":    280.00, "NFLX": 600.00,
}

TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL": {"sigma": 0.22, "mu": 0.05},
    "TSLA": {"sigma": 0.50, "mu": 0.03},  # High vol
    "NVDA": {"sigma": 0.40, "mu": 0.08},  # High vol + strong drift
    "JPM":  {"sigma": 0.18, "mu": 0.04},  # Low vol (bank)
    "V":    {"sigma": 0.17, "mu": 0.04},  # Low vol (payments)
    # ... (all 10 default tickers)
}

DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}  # For dynamically added tickers
```

Tickers not in `TICKER_PARAMS` get default params and a random seed price in [$50, $300].

### Random Shock Events

On every tick, each ticker has a 0.1% chance of a sudden 2–5% move:

```python
if random.random() < 0.001:
    shock_magnitude = random.uniform(0.02, 0.05)
    shock_sign = random.choice([-1, 1])
    self._prices[ticker] *= (1 + shock_magnitude * shock_sign)
```

With 10 tickers at 2 ticks/second: expect ~1 shock per 50 seconds. These create the
visual drama of news-driven moves without modelling news.

### `SimulatorDataSource` — Async Wrapper

```python
# simulator.py (SimulatorDataSource)
class SimulatorDataSource(MarketDataSource):

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        # Seed the cache immediately so SSE has data on first connection
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

    async def _run_loop(self) -> None:
        while True:
            try:
                prices = self._sim.step()
                for ticker, price in prices.items():
                    self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)  # 500ms default

    async def add_ticker(self, ticker: str) -> None:
        self._sim.add_ticker(ticker)          # Extends GBM state + rebuilds Cholesky
        price = self._sim.get_price(ticker)
        if price is not None:
            self._cache.update(ticker, price) # Immediately visible to SSE consumers
```

---

## 8. Massive API Client: `MassiveDataSource`

### Authentication

```python
from massive import RESTClient

client = RESTClient(api_key="YOUR_MASSIVE_API_KEY")
# Wraps https://api.polygon.io — all existing Polygon keys work unchanged
```

### Primary Endpoint: Full Snapshot

One API call fetches all watched tickers simultaneously:

```
GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,TSLA,NVDA
```

Response shape (relevant fields):

```json
{
  "tickers": [
    {
      "ticker": "AAPL",
      "todaysChangePerc": 1.22,
      "lastTrade": {
        "p": 190.25,
        "t": 1712345678000
      },
      "day":     { "o": 187.50, "h": 191.20, "l": 186.80, "c": 190.25 },
      "prevDay": { "c": 187.94 }
    }
  ]
}
```

Python SDK (synchronous — must be wrapped in `asyncio.to_thread`):

```python
from massive.rest.models import SnapshotMarketType

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "TSLA", "NVDA"],
)

for snap in snapshots:
    price     = snap.last_trade.price
    timestamp = snap.last_trade.timestamp / 1000.0  # ms → seconds
    change_pct = snap.todays_change_perc
```

### `MassiveDataSource` Implementation

```python
# massive_client.py
class MassiveDataSource(MarketDataSource):

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)
        await self._poll_once()   # Immediate first poll — cache populated before SSE starts
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")

    async def _poll_loop(self) -> None:
        while True:
            await asyncio.sleep(self._interval)  # 15s default (free tier safe)
            await self._poll_once()

    async def _poll_once(self) -> None:
        if not self._tickers or not self._client:
            return
        try:
            # RESTClient is synchronous — avoid blocking the event loop
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
            for snap in snapshots:
                price     = snap.last_trade.price
                timestamp = snap.last_trade.timestamp / 1000.0
                self._cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)
        except Exception as e:
            logger.error("Massive poll failed: %s", e)
            # Don't re-raise — loop retries on next interval

    def _fetch_snapshots(self) -> list:
        """Synchronous REST call. Runs in a thread pool via asyncio.to_thread."""
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

### Rate Limits and Poll Intervals

| Plan      | Requests/min | Recommended `poll_interval` |
|-----------|--------------|-----------------------------|
| Free      | 5            | 15.0s (default)             |
| Starter   | 100          | 5.0s                        |
| Developer | 1,000        | 2.0s                        |
| Advanced  | unlimited    | 1.0s or lower               |

Override the default:

```python
MassiveDataSource(api_key=key, price_cache=cache, poll_interval=5.0)
```

### Error Handling

| HTTP | Cause                              | Behavior                        |
|------|------------------------------------|---------------------------------|
| 401  | Invalid API key                    | Logged, loop retries in 15s     |
| 403  | Endpoint requires higher tier      | Logged, loop retries in 15s     |
| 404  | Ticker not found                   | Snapshot skipped (per-ticker)   |
| 429  | Rate limit exceeded                | Logged, loop retries in 15s     |
| N/A  | Network error                      | Logged, loop retries in 15s     |

All failures are non-fatal. The cache retains the last known prices until the next
successful poll. The SSE stream continues to push stale (but still present) prices.

### Daily Change % with Massive

Use `snap.todays_change_perc` directly — Massive computes this from today's open vs.
the last trade price. No additional API call needed.

---

## 9. SSE Streaming Endpoint

### Event Format

The SSE endpoint sends all cached prices as a single JSON blob per event:

```
retry: 1000

data: {"AAPL":{"ticker":"AAPL","price":190.25,"previous_price":190.10,"timestamp":1712345678.0,"change":0.15,"change_percent":0.0789,"direction":"up"},"MSFT":{...},...}

data: {"AAPL":{"ticker":"AAPL","price":190.27,...},...}
```

- One `data:` line per event (no `event:` type — client receives all as `message`)
- All tickers in one payload per event (not one event per ticker)
- `retry: 1000` on connection: browser auto-reconnects after 1 second on disconnect

### Implementation

```python
# stream.py
from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse
from collections.abc import AsyncGenerator

router = APIRouter(prefix="/api/stream", tags=["streaming"])

def create_stream_router(price_cache: PriceCache) -> APIRouter:
    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        return StreamingResponse(
            _generate_events(price_cache, request),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",   # Prevents nginx from buffering SSE
            },
        )
    return router

async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:
    yield "retry: 1000\n\n"
    last_version = -1
    while True:
        if await request.is_disconnected():
            break
        current_version = price_cache.version
        if current_version != last_version:
            last_version = current_version
            prices = price_cache.get_all()
            if prices:
                data = {ticker: update.to_dict() for ticker, update in prices.items()}
                yield f"data: {json.dumps(data)}\n\n"
        await asyncio.sleep(interval)
```

**Why version-based change detection?** The SSE generator wakes every 500ms regardless.
Without the version check, it would resend identical data on every wakeup. With the
check, it only transmits when at least one price changed since the last send.

### Frontend Connection Pattern

```typescript
const source = new EventSource("/api/stream/prices");

source.onmessage = (event) => {
    const prices: Record<string, PriceUpdate> = JSON.parse(event.data);
    for (const [ticker, update] of Object.entries(prices)) {
        // Flash green/red based on update.direction
        updateWatchlistRow(ticker, update);
        appendSparklinePoint(ticker, update.price);
    }
};

source.onerror = () => {
    // EventSource auto-reconnects using the retry value we sent (1000ms)
    setConnectionStatus("reconnecting");
};
```

---

## 10. FastAPI Integration (App Lifecycle)

```python
# backend/app/main.py  (example)
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.market import PriceCache, create_market_data_source, create_stream_router

DEFAULT_TICKERS = ["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA",
                   "NVDA", "META", "JPM", "V", "NFLX"]

price_cache = PriceCache()
market_source = create_market_data_source(price_cache)

@asynccontextmanager
async def lifespan(app: FastAPI):
    await market_source.start(DEFAULT_TICKERS)
    yield
    await market_source.stop()

app = FastAPI(lifespan=lifespan)
app.include_router(create_stream_router(price_cache))
```

`price_cache` and `market_source` are module-level singletons — no dependency injection
framework needed for a single-user app.

---

## 11. Dynamic Watchlist Integration

Watchlist API routes must call `market_source` to keep prices in sync:

```python
# backend/app/routes/watchlist.py  (example)
from fastapi import APIRouter

router = APIRouter(prefix="/api/watchlist")

@router.post("/")
async def add_to_watchlist(body: AddTickerRequest):
    # 1. Write to database
    await db.add_watchlist_entry(ticker=body.ticker)
    # 2. Start tracking prices immediately
    await market_source.add_ticker(body.ticker)
    return {"ticker": body.ticker, "status": "added"}

@router.delete("/{ticker}")
async def remove_from_watchlist(ticker: str):
    # 1. Remove from database
    await db.remove_watchlist_entry(ticker=ticker)
    # 2. Stop tracking — also purges from PriceCache
    await market_source.remove_ticker(ticker)
    return {"ticker": ticker, "status": "removed"}
```

After `add_ticker()`:
- **Simulator:** seed price written to cache immediately; appears in next SSE event (~500ms)
- **Massive:** ticker added to the poll list; appears after the next poll (up to 15s)

After `remove_ticker()`:
- Price removed from cache immediately; SSE stops including it on the next send

---

## 12. Portfolio Integration

Reading prices for trade execution and portfolio valuation:

```python
# Get price for a single ticker (trade execution)
price = price_cache.get_price("AAPL")
if price is None:
    raise HTTPException(400, "Price unavailable for AAPL")

# Get all prices for portfolio valuation
all_prices = price_cache.get_all()   # dict[str, PriceUpdate]
for ticker, position in positions.items():
    current_price = all_prices.get(ticker)
    if current_price:
        unrealized_pnl = (current_price.price - position.avg_cost) * position.quantity
```

**Important:** `price_cache.get_all()` returns a snapshot. Take it once per request,
not once per ticker, to get consistent prices across a portfolio valuation.

---

## 13. "Daily Change %" — Simulator vs. Massive

| Source    | Definition of daily change                    | Where to find it                    |
|-----------|-----------------------------------------------|-------------------------------------|
| Simulator | % change from seed price at process startup  | `(current - SEED_PRICES[t]) / SEED_PRICES[t] * 100` |
| Massive   | % change from today's market open            | `snap.todays_change_perc`           |

For the simulator, compute it in the backend using `seed_prices.SEED_PRICES` as the
reference. Include it in the SSE payload or the watchlist REST response — the frontend
should not need to know which data source is active.

Recommended: add a `daily_change_percent` field to `PriceUpdate.to_dict()` for the
watchlist response, and compute it in the data source layer:

```python
# In SimulatorDataSource, track seed prices and expose them:
daily_change_pct = (current_price - seed) / seed * 100

# In MassiveDataSource._poll_once():
daily_change_pct = snap.todays_change_perc  # Already computed by Polygon
```

---

## 14. Testing

### Running the Tests

```bash
cd backend
uv run --extra dev pytest tests/market/ -v        # Market data tests only
uv run --extra dev pytest --cov=app               # All tests with coverage
```

### Test Matrix

| Module                      | Tests | What it covers                                           |
|-----------------------------|-------|----------------------------------------------------------|
| `test_models.py`            | 11    | PriceUpdate fields, properties, direction, `to_dict()`  |
| `test_cache.py`             | 13    | Thread safety, version counter, update/get/remove        |
| `test_simulator.py`         | 17    | GBM math, Cholesky structure, shocks, add/remove tickers |
| `test_simulator_source.py`  | 10    | Async lifecycle, cache integration, dynamic tickers      |
| `test_factory.py`           | 7     | Env var detection, correct class returned                |
| `test_massive.py`           | 13    | Snapshot parsing, poll loop, error handling              |

### Example Test: Cache Thread Safety

```python
# tests/market/test_cache.py
import threading

def test_concurrent_updates_are_consistent():
    cache = PriceCache()

    def writer(ticker, prices):
        for price in prices:
            cache.update(ticker, price)

    threads = [
        threading.Thread(target=writer, args=("AAPL", range(100))),
        threading.Thread(target=writer, args=("MSFT", range(100))),
    ]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

    assert cache.get("AAPL") is not None
    assert cache.get("MSFT") is not None
    assert cache.version == 200
```

### Example Test: GBM Math Validity

```python
# tests/market/test_simulator.py
def test_gbm_prices_stay_positive():
    sim = GBMSimulator(["AAPL", "TSLA"])
    for _ in range(1000):
        prices = sim.step()
        assert all(p > 0 for p in prices.values())

def test_correlated_moves_use_cholesky():
    sim = GBMSimulator(["AAPL", "MSFT"])
    assert sim._cholesky is not None
    assert sim._cholesky.shape == (2, 2)
```

### Example Test: Factory Selection

```python
# tests/market/test_factory.py
import os

def test_returns_simulator_when_no_key(monkeypatch):
    monkeypatch.delenv("MASSIVE_API_KEY", raising=False)
    source = create_market_data_source(PriceCache())
    assert isinstance(source, SimulatorDataSource)

def test_returns_massive_when_key_set(monkeypatch):
    monkeypatch.setenv("MASSIVE_API_KEY", "test_key_123")
    source = create_market_data_source(PriceCache())
    assert isinstance(source, MassiveDataSource)
```

---

## 15. Terminal Demo

A standalone Rich terminal demo visualizes live simulated prices:

```bash
cd backend
uv run market_data_demo.py
```

Displays a live-updating table: all 10 tickers, current price, direction arrow, change %,
and a sparkline. Runs for 60 seconds or until Ctrl+C. Useful for verifying the simulator
parameters produce realistic-looking price action.

---

## 16. Adding a New Data Source

To add a third data source (e.g., a WebSocket feed):

1. Create `backend/app/market/websocket_client.py`
2. Implement `MarketDataSource` (all 5 abstract methods)
3. Add a branch in `factory.py` checking a new env var (e.g., `WS_FEED_URL`)
4. Add tests in `tests/market/test_websocket_client.py`

No changes needed in `cache.py`, `stream.py`, any route, or any consumer. The strategy
pattern pays for itself here.
