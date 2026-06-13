# Market Data Interface — Unified Design

This document describes the unified Python interface FinAlly uses to retrieve
stock prices, abstracting over two interchangeable backends:

- **`MassiveDataSource`** — real data via the Massive (Polygon.io) API, used
  when `MASSIVE_API_KEY` is set (see `MASSIVE_API.md`)
- **`SimulatorDataSource`** — GBM-based price simulation, used otherwise (see
  `MARKET_SIMULATOR.md`)

**Status:** This interface is already implemented in
`backend/app/market/` (`interface.py`, `cache.py`, `factory.py`,
`massive_client.py`, `simulator.py`, `models.py`, `stream.py`). This document
is the design reference for that implementation.

## 1. Goals

- Downstream code (portfolio valuation, trade execution, SSE streaming) never
  needs to know which backend is active
- A single in-memory cache is the source of truth for "current price"
- Selection between real data and simulation happens at startup, based purely
  on environment configuration
- Both backends push updates on their own schedule into the shared cache;
  nothing pulls directly from the backend

## 2. Component Diagram

```
                 create_market_data_source(cache)
                              │
              ┌───────────────┴───────────────┐
              │ MASSIVE_API_KEY set?           │
              ▼ yes                            ▼ no
      MassiveDataSource                SimulatorDataSource
   (polls Massive REST API          (steps GBM model every
    every 15s, runs in thread)        500ms)
              │                                │
              └───────────────┬────────────────┘
                               ▼
                          PriceCache
                    (thread-safe, versioned)
                               │
        ┌──────────────────────┼──────────────────────┐
        ▼                       ▼                       ▼
  SSE stream             Portfolio valuation      Trade execution
 /api/stream/prices      (positions, P&L)         (fill price)
```

## 3. Core Types

### `PriceUpdate` (immutable dataclass — `models.py`)

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float: ...          # price - previous_price
    @property
    def change_percent(self) -> float: ...  # tick-over-tick %
    @property
    def direction(self) -> str: ...          # "up" | "down" | "flat"
    def to_dict(self) -> dict: ...           # JSON-serializable
```

`change`/`change_percent`/`direction` are **tick-over-tick** (vs. the
previous cache write), not session-relative. This is sufficient for flash
animations; a true "daily change %" would need `session.previous_close` from
the Massive snapshot (see `MASSIVE_API.md` §4).

### `PriceCache` (thread-safe — `cache.py`)

```python
class PriceCache:
    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate
    def get(self, ticker: str) -> PriceUpdate | None
    def get_price(self, ticker: str) -> float | None
    def get_all(self) -> dict[str, PriceUpdate]
    def remove(self, ticker: str) -> None
    @property
    def version(self) -> int   # increments on every update
```

- A single `Lock` guards all reads/writes (cheap — dict ops only)
- `version` lets the SSE endpoint detect "did anything change since I last
  sent an event" without diffing the whole dict
- On the *first* update for a ticker, `previous_price == price` →
  `direction == "flat"`

### `MarketDataSource` (ABC — `interface.py`)

```python
class MarketDataSource(ABC):
    async def start(self, tickers: list[str]) -> None: ...
    async def stop(self) -> None: ...
    async def add_ticker(self, ticker: str) -> None: ...
    async def remove_ticker(self, ticker: str) -> None: ...
    def get_tickers(self) -> list[str]: ...
```

Lifecycle contract:

```python
source = create_market_data_source(cache)
await source.start(["AAPL", "GOOGL", ...])   # called exactly once
await source.add_ticker("TSLA")              # dynamic watchlist changes
await source.remove_ticker("GOOGL")          # also evicts from cache
await source.stop()                          # safe to call multiple times
```

Both implementations run a background `asyncio.Task` that writes to the
shared `PriceCache` on their own cadence — `start()`/`stop()` manage that
task's lifecycle.

### `create_market_data_source()` (factory — `factory.py`)

```python
def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    return SimulatorDataSource(price_cache=price_cache)
```

Returns an **unstarted** source — the caller is responsible for
`await source.start(tickers)`.

## 4. Backend Comparison

| | `SimulatorDataSource` | `MassiveDataSource` |
|---|---|---|
| Trigger | `MASSIVE_API_KEY` unset/empty | `MASSIVE_API_KEY` set |
| Update cadence | 500ms (configurable) | 15s poll (free tier) |
| Mechanism | In-process GBM model (`GBMSimulator`) | Threaded REST call (`get_snapshot_all`) |
| Initial cache seed | Immediate, on `start()` | Immediate (`_poll_once()` before loop) |
| Failure mode | N/A (always succeeds) | Logs + retries next interval; cache simply goes stale |
| Dependencies | `numpy` | `massive` package + network |

## 5. Usage by Downstream Code

```python
from app.market import PriceCache, create_market_data_source, create_stream_router

cache = PriceCache()
source = create_market_data_source(cache)
await source.start(watchlist_tickers)

# SSE endpoint
app.include_router(create_stream_router(cache))

# Portfolio valuation
price = cache.get_price("AAPL")  # float | None

# Watchlist add/remove (from API or LLM action)
await source.add_ticker("PYPL")
await source.remove_ticker("NFLX")
```

### Tracked-ticker set

Tickers passed to `start()`/`add_ticker()` should be the union of the
**watchlist** and any **open positions**, so a held-but-unwatched ticker
still gets priced for portfolio valuation.

## 6. Error / Edge Cases

- **`cache.get_price(ticker)` returns `None`** if the ticker has never been
  updated (e.g. just added, first poll/tick hasn't happened) or after
  `remove_ticker`. Callers (trade execution, portfolio valuation) must handle
  `None` explicitly — e.g. reject a trade with "price unavailable" rather than
  treating it as zero.
- **Massive API failure** (bad key, rate limit, network) does not crash the
  app — the poller logs and retries. Cached prices simply stop updating until
  the next successful poll.
- **Switching backends** is purely an environment/restart concern — there is
  no runtime fallback from Massive to Simulator if the API key turns out to be
  invalid after `start()`. (A future enhancement could detect repeated
  failures and fall back, but this is out of scope for the current design.)
