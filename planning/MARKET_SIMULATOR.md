# Market Simulator

The GBM simulator generates realistic-looking stock price movements without any external
API dependency. It is the default data source when `MASSIVE_API_KEY` is not set.

---

## Implementation Files

| File             | Contents                                                      |
|------------------|---------------------------------------------------------------|
| `simulator.py`   | `GBMSimulator` (math engine) + `SimulatorDataSource` (async wrapper) |
| `seed_prices.py` | Starting prices, per-ticker GBM parameters, correlation config |

---

## Mathematical Model: Geometric Brownian Motion

Each tick advances every price by one GBM step:

```
S(t+dt) = S(t) * exp( (mu - sigma²/2) * dt  +  sigma * sqrt(dt) * Z )
```

Where:
- `S(t)` — current price
- `mu` — annualized drift (expected return, e.g. 0.05 = 5%/year)
- `sigma` — annualized volatility (e.g. 0.22 = 22%/year for AAPL)
- `dt` — time step as fraction of a trading year (~8.48×10⁻⁸ for 500 ms ticks)
- `Z` — correlated standard normal random variable (see below)

With `dt` this small, each tick produces sub-cent moves that accumulate naturally into
realistic-looking drifts and swings over minutes and hours.

---

## Correlated Moves (Cholesky Decomposition)

Stocks in the same sector tend to move together. The simulator models this via a
correlation matrix and its Cholesky decomposition.

**Correlation values:**

| Relationship                     | Coefficient |
|----------------------------------|-------------|
| Same tech sector (AAPL, GOOGL…)  | 0.60        |
| Same finance sector (JPM, V)     | 0.50        |
| TSLA with anything               | 0.30        |
| Cross-sector / unknown tickers   | 0.30        |

**Per-tick generation:**

```python
z_independent = np.random.standard_normal(n)   # n = number of tickers
z_correlated  = cholesky_matrix @ z_independent  # correlated draws
```

Each ticker `i` uses `z_correlated[i]` in the GBM formula instead of a raw normal draw.
The Cholesky matrix is rebuilt whenever tickers are added or removed (O(n²), n < 50).

---

## Seed Prices and Per-Ticker Parameters

Defined in `seed_prices.py`. These are the starting values when the simulator initialises:

| Ticker | Seed Price | Volatility (σ) | Drift (μ) | Notes               |
|--------|-----------|----------------|-----------|---------------------|
| AAPL   | $190      | 22%            | 5%        |                     |
| GOOGL  | $175      | 25%            | 5%        |                     |
| MSFT   | $420      | 20%            | 5%        |                     |
| AMZN   | $185      | 28%            | 5%        |                     |
| TSLA   | $250      | 50%            | 3%        | High volatility      |
| NVDA   | $800      | 40%            | 8%        | High vol, high drift |
| META   | $500      | 30%            | 5%        |                     |
| JPM    | $195      | 18%            | 4%        | Low vol (bank)       |
| V      | $280      | 17%            | 4%        | Low vol (payments)   |
| NFLX   | $600      | 35%            | 5%        |                     |

Tickers added dynamically (not in the table above) use defaults: σ=25%, μ=5%, seed=random $50–$300.

---

## Random Shock Events

On every tick, each ticker has a 0.1% chance of a sudden 2–5% price move (up or down).
With 10 tickers at 2 ticks/second, expect roughly one shock event every 50 seconds.

```python
if random.random() < 0.001:
    shock = random.uniform(0.02, 0.05)
    sign  = random.choice([-1, 1])
    price *= (1 + shock * sign)
```

This creates the visual drama of news-driven moves without modelling news.

---

## Class Structure

### `GBMSimulator` (pure math, synchronous)

```python
class GBMSimulator:
    def __init__(self, tickers: list[str], dt: float = DEFAULT_DT,
                 event_probability: float = 0.001) -> None: ...

    def step(self) -> dict[str, float]:
        """Advance all tickers by one dt. Returns {ticker: new_price}.
        Hot path — called every 500 ms."""

    def add_ticker(self, ticker: str) -> None:
        """Add ticker; rebuilds Cholesky matrix."""

    def remove_ticker(self, ticker: str) -> None:
        """Remove ticker; rebuilds Cholesky matrix."""

    def get_price(self, ticker: str) -> float | None: ...
    def get_tickers(self) -> list[str]: ...
```

`DEFAULT_DT = 0.5 / (252 * 6.5 * 3600)` — 500 ms expressed as a fraction of a trading year
(252 trading days × 6.5 hours/day × 3600 s/hour = 5,896,800 trading seconds/year).

### `SimulatorDataSource` (async wrapper, implements `MarketDataSource`)

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache: PriceCache,
                 update_interval: float = 0.5,
                 event_probability: float = 0.001) -> None: ...

    async def start(self, tickers: list[str]) -> None:
        """Create GBMSimulator, seed cache with initial prices, start background loop."""

    async def stop(self) -> None:
        """Cancel background task."""

    async def add_ticker(self, ticker: str) -> None:
        """Forward to GBMSimulator; seed cache immediately."""

    async def remove_ticker(self, ticker: str) -> None:
        """Forward to GBMSimulator; purge from PriceCache."""

    async def _run_loop(self) -> None:
        """Core loop: step() → write all prices to cache → sleep 500 ms."""
```

The background loop:

```python
while True:
    prices = self._sim.step()                        # GBM math
    for ticker, price in prices.items():
        self._cache.update(ticker=ticker, price=price)  # write to cache
    await asyncio.sleep(self._interval)              # 500 ms
```

---

## Adding a New Ticker at Runtime

```python
# SimulatorDataSource.add_ticker() does this:
self._sim.add_ticker(ticker)          # extend GBMSimulator state
price = self._sim.get_price(ticker)   # get the seed price
self._cache.update(ticker, price)     # immediately visible to SSE / portfolio
```

The new ticker appears in the next SSE broadcast (within 500 ms).

---

## "Daily Change" in the Simulator

The simulator has no concept of market open/close, so "daily change" is defined as the
percentage change from the **seed price** at process startup. The SSE payload always
includes `previous_price` (price before the last tick), so the frontend can compute
its own running change; the seed price itself is available in `seed_prices.py` for
any backend calculation that needs a stable reference.

---

## Test Coverage

The simulator is tested in `backend/tests/market/`:

| Test module              | What it covers                                              |
|--------------------------|-------------------------------------------------------------|
| `test_simulator.py`      | GBM math validity, Cholesky structure, shock events, add/remove |
| `test_simulator_source.py` | Async lifecycle, cache integration, dynamic ticker management |

Run with:

```bash
cd backend && uv run pytest tests/market/test_simulator.py -v
```
