# Market Data Unified Interface

This document describes the unified Python API for retrieving stock prices in FinAlly.
The same interface works whether prices come from the Massive REST API or the built-in
GBM simulator. All downstream code (SSE streaming, portfolio valuation, trade execution)
reads from `PriceCache` and never touches the data source directly.

---

## Architecture

```
Environment variable check (MASSIVE_API_KEY)
        │
        ├── set & non-empty ──→ MassiveDataSource  (real market data, REST polling)
        └── absent / empty  ──→ SimulatorDataSource (GBM simulation, no dependencies)
                │
                ▼ writes prices every tick
           PriceCache  (thread-safe in-memory store)
                │
                ├──→ SSE stream  /api/stream/prices
                ├──→ Portfolio valuation
                └──→ Trade execution
```

---

## Module Layout

All code lives in `backend/app/market/`:

| File               | Contents                                                     |
|--------------------|--------------------------------------------------------------|
| `models.py`        | `PriceUpdate` — the canonical price record                   |
| `interface.py`     | `MarketDataSource` — abstract base class                     |
| `cache.py`         | `PriceCache` — thread-safe price store                       |
| `seed_prices.py`   | Starting prices and GBM parameters for all default tickers   |
| `simulator.py`     | `GBMSimulator` + `SimulatorDataSource`                       |
| `massive_client.py`| `MassiveDataSource`                                          |
| `factory.py`       | `create_market_data_source()` — selects implementation       |
| `stream.py`        | FastAPI SSE endpoint (`/api/stream/prices`)                  |

Public re-exports via `__init__.py`:

```python
from app.market import PriceCache, PriceUpdate, create_market_data_source
```

---

## Data Model: `PriceUpdate`

Immutable frozen dataclass. Every price change is represented as one of these.

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float          # Unix seconds

    # Computed properties:
    change: float             # price - previous_price
    change_percent: float     # percentage change from previous_price
    direction: str            # "up" | "down" | "flat"

    def to_dict(self) -> dict:
        """Serialize for JSON / SSE transmission."""
```

---

## Abstract Interface: `MarketDataSource`

```python
class MarketDataSource(ABC):

    async def start(self, tickers: list[str]) -> None:
        """Begin producing prices. Starts a background task. Call once."""

    async def stop(self) -> None:
        """Stop the background task. Safe to call multiple times."""

    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present."""

    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Also purges it from PriceCache."""

    def get_tickers(self) -> list[str]:
        """Return the current list of tracked tickers."""
```

Both `MassiveDataSource` and `SimulatorDataSource` implement this interface.
Swapping one for the other requires no changes to any other module.

---

## PriceCache

Thread-safe store. Producers call `update()`; consumers call `get()` / `get_all()`.

```python
class PriceCache:

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price. Computes direction from previous value."""

    def get(self, ticker: str) -> PriceUpdate | None:
        """Latest PriceUpdate for one ticker, or None."""

    def get_all(self) -> dict[str, PriceUpdate]:
        """Snapshot of all current prices (shallow copy)."""

    def get_price(self, ticker: str) -> float | None:
        """Convenience: just the price float, or None."""

    def remove(self, ticker: str) -> None:
        """Remove a ticker from the cache."""

    @property
    def version(self) -> int:
        """Monotonically increasing counter. Bumped on every update.
        Used by the SSE endpoint for change detection (avoids re-sending unchanged prices)."""
```

---

## Factory: Selecting the Implementation

```python
# backend/app/market/factory.py

def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        return SimulatorDataSource(price_cache=price_cache)
```

The factory reads `MASSIVE_API_KEY` from the environment (loaded from `.env` at startup).

---

## Startup / Shutdown (FastAPI lifespan)

```python
from app.market import PriceCache, create_market_data_source

cache = PriceCache()
source = create_market_data_source(cache)

# On startup:
await source.start(["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA",
                    "NVDA", "META", "JPM", "V", "NFLX"])

# On shutdown:
await source.stop()
```

---

## Reading Prices (downstream code)

```python
# Single ticker
update = cache.get("AAPL")         # PriceUpdate | None
price  = cache.get_price("AAPL")   # float | None

# All tickers (e.g., for SSE broadcast or portfolio valuation)
all_prices = cache.get_all()        # dict[str, PriceUpdate]

# Access fields
update.price
update.previous_price
update.change_percent
update.direction    # "up" | "down" | "flat"
update.to_dict()    # JSON-ready dict
```

---

## Dynamic Watchlist

The watchlist API routes call these methods when the user adds/removes tickers:

```python
await source.add_ticker("PYPL")    # starts tracking; cache has a price on next tick
await source.remove_ticker("NFLX") # stops tracking; removes from cache immediately
```

---

## SSE Stream

`GET /api/stream/prices` — long-lived SSE connection. The server pushes all cached
prices every 500 ms (or immediately when the version counter increments). Each event:

```
event: price
data: {"ticker":"AAPL","price":190.25,"previous_price":190.10,"timestamp":1712345678.0,
       "change":0.15,"change_percent":0.0789,"direction":"up"}
```

The frontend uses the native `EventSource` API; no library required.
Reconnection is automatic via `EventSource`'s built-in retry mechanism.

---

## Environment Variables

| Variable         | Effect                                                         |
|------------------|----------------------------------------------------------------|
| `MASSIVE_API_KEY`| Set to use real market data. Leave empty for the simulator.    |
| (none)           | Simulator runs by default — no API key required.               |

Poll interval for `MassiveDataSource` defaults to **15 seconds** (free tier safe).
Simulator update interval defaults to **500 ms**.
