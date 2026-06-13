# Market Simulator — Approach & Code Structure

This document describes the design of FinAlly's built-in market data
simulator, used whenever `MASSIVE_API_KEY` is not set. It is the default mode
— most users (and all CI/E2E tests) run on the simulator.

**Status:** Implemented in `backend/app/market/simulator.py` and
`backend/app/market/seed_prices.py`. This document is the design reference.

## 1. Goals

- Produce realistic, continuously-moving prices for an arbitrary set of
  tickers with **no external dependencies**
- Prices should feel "alive": correlated sector moves, occasional dramatic
  events, per-ticker volatility personalities
- Cheap enough to step every 500ms for ~10-50 tickers without perceptible CPU cost
- Conform to the same `MarketDataSource` interface as the real data backend

## 2. Mathematical Model: Geometric Brownian Motion (GBM)

Each ticker's price evolves via the standard GBM discretization:

```
S(t+dt) = S(t) * exp((mu - sigma²/2) * dt + sigma * sqrt(dt) * Z)
```

- `S(t)` — current price
- `mu` — annualized drift (expected return)
- `sigma` — annualized volatility
- `dt` — time step, expressed as a fraction of a trading year
- `Z` — standard normal random draw (correlated across tickers — see §3)

### Time step

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR  # ≈ 8.48e-8, for 500ms ticks
```

This tiny `dt` produces sub-cent moves per tick that accumulate into
realistic intraday volatility over minutes/hours.

## 3. Correlated Moves via Cholesky Decomposition

Real markets move in sectors — tech stocks tend to rise/fall together.
The simulator builds a **correlation matrix** across all tracked tickers and
uses its **Cholesky decomposition** to transform independent normal draws
into correlated ones:

```python
z_independent = np.random.standard_normal(n)
z_correlated = cholesky_matrix @ z_independent
```

### Correlation rules (`seed_prices.py`)

| Pair | Correlation |
|---|---|
| Two tech tickers (`AAPL, GOOGL, MSFT, AMZN, META, NVDA, NFLX`) | 0.6 |
| Two finance tickers (`JPM, V`) | 0.5 |
| Anything involving `TSLA` | 0.3 (TSLA "does its own thing") |
| Cross-sector / unknown tickers | 0.3 |

The correlation matrix (and its Cholesky factor) is rebuilt whenever a ticker
is added or removed — O(n²) but n stays small (< 50), so this is cheap.

## 4. Per-Ticker Parameters & Seed Prices

`seed_prices.py` defines realistic starting points and personalities for the
default watchlist:

```python
SEED_PRICES = {"AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00, ...}

TICKER_PARAMS = {
    "AAPL": {"sigma": 0.22, "mu": 0.05},
    "TSLA": {"sigma": 0.50, "mu": 0.03},   # high volatility
    "NVDA": {"sigma": 0.40, "mu": 0.08},   # high vol, strong drift
    "JPM":  {"sigma": 0.18, "mu": 0.04},   # low vol (bank)
    ...
}

DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}  # for dynamically-added tickers
```

Any ticker not in `SEED_PRICES`/`TICKER_PARAMS` (added at runtime via the
watchlist) gets a random seed price in `[$50, $300]` and `DEFAULT_PARAMS`.

## 5. Random "Event" Shocks

To add visual drama (matching the "prices flash dramatically" UX goal), each
tick has a small independent chance of a sudden jump per ticker:

```python
event_probability = 0.001  # ~0.1% per tick per ticker

if random.random() < event_probability:
    shock_magnitude = random.uniform(0.02, 0.05)   # 2-5%
    shock_sign = random.choice([-1, 1])
    price *= 1 + shock_magnitude * shock_sign
```

With 10 tickers at 2 ticks/sec, expect roughly one event every ~50 seconds —
frequent enough to be noticeable, rare enough not to dominate.

## 6. Code Structure

### `GBMSimulator` — pure simulation logic (no async, no I/O)

```python
class GBMSimulator:
    def __init__(self, tickers: list[str], dt: float = DEFAULT_DT,
                 event_probability: float = 0.001): ...

    def step(self) -> dict[str, float]:
        """Advance all tickers by one tick. Returns {ticker: new_price}.
        Hot path — called every 500ms, must stay fast."""

    def add_ticker(self, ticker: str) -> None     # rebuilds Cholesky
    def remove_ticker(self, ticker: str) -> None  # rebuilds Cholesky
    def get_price(self, ticker: str) -> float | None
    def get_tickers(self) -> list[str]
```

Internally tracks `_prices`, `_params`, `_tickers` (ordered, for the
correlation matrix index), and `_cholesky` (rebuilt on any ticker change).

### `SimulatorDataSource` — `MarketDataSource` adapter

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache: PriceCache,
                 update_interval: float = 0.5,
                 event_probability: float = 0.001): ...

    async def start(self, tickers: list[str]) -> None:
        # creates GBMSimulator, seeds cache immediately,
        # spawns background _run_loop() task

    async def _run_loop(self) -> None:
        while True:
            prices = self._sim.step()
            for ticker, price in prices.items():
                self._cache.update(ticker=ticker, price=price)
            await asyncio.sleep(self._interval)
```

- **Immediate seed on `start()`** — writes initial prices to the cache before
  the loop begins, so SSE clients get data on the very first connection.
- **`add_ticker`/`remove_ticker`** also immediately update the cache (for adds)
  or evict (for removes), independent of the loop's cadence.
- Exceptions inside `_run_loop` are caught and logged per-iteration — a
  transient error never kills the background task.

## 7. Tuning Knobs (for future adjustment)

| Parameter | Location | Effect |
|---|---|---|
| `update_interval` | `SimulatorDataSource.__init__` | Tick frequency (default 500ms) |
| `event_probability` | `SimulatorDataSource.__init__` / `GBMSimulator` | Frequency of shock events |
| `sigma`, `mu` per ticker | `seed_prices.TICKER_PARAMS` | Volatility/drift personality |
| Correlation values | `seed_prices.py` constants | How tightly sectors move together |

## 8. Testing

The simulator is fully deterministic-testable by seeding `numpy`/`random` and
asserting statistical properties (e.g. mean reversion of `step()` outputs
over many iterations, correlation matrix validity, Cholesky success for any
ticker set). See `backend/tests/market/test_simulator.py` (17 tests, 98%
coverage per `planning/MARKET_DATA_SUMMARY.md`).
