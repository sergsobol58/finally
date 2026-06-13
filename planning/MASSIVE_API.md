# Massive API (formerly Polygon.io) — Reference

> **Massive.com** is the rebrand of Polygon.io (announced Oct 2025). Existing
> Polygon.io API keys, REST endpoints, and response shapes are unchanged — only
> branding and the Python package name (`polygon-api-client` → `massive`)
> changed. This document covers the REST endpoints relevant to FinAlly: real-time
> snapshots and end-of-day / historical aggregates for multiple tickers.

## 1. Authentication

All requests are authenticated with an API key, either as a query parameter or
an `Authorization: Bearer` header:

```
GET https://api.polygon.io/v2/aggs/.../?apiKey=YOUR_API_KEY
```

```python
from massive import RESTClient

client = RESTClient(api_key="YOUR_API_KEY")  # or set POLYGON_API_KEY env var
```

The base host (`api.polygon.io`) remains valid post-rebrand.

## 2. Rate Limits by Tier

| Tier | Requests/min | Notes |
|---|---|---|
| Free (Stocks Basic) | 5 | 15-minute delayed data, end-of-day aggregates |
| Starter | ~100 | 15-minute delayed snapshots |
| Developer | unlimited | Real-time data |
| Advanced/Business | unlimited | Real-time + WebSocket |

For FinAlly's polling architecture (free tier), **poll every 15 seconds** to
stay safely under 5 req/min for a single combined snapshot call.

## 3. The `massive` Python Package

```bash
pip install massive   # successor to `polygon-api-client`
# or with uv:
uv add massive
```

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient(api_key="YOUR_API_KEY")
```

`RESTClient` is **synchronous**. In an asyncio app, run calls via
`asyncio.to_thread(...)` to avoid blocking the event loop (this is what
`MassiveDataSource` in `backend/app/market/massive_client.py` does).

## 4. Real-Time / Latest Price — Unified Snapshot (multiple tickers)

**Endpoint:** `GET /v3/snapshot` (also reachable via the legacy
`GET /v2/snapshot/locale/us/markets/stocks/tickers`, which the `massive`
client wraps as `get_snapshot_all()`)

**Key query parameters:**

| Param | Description |
|---|---|
| `ticker.any_of` | Comma-separated list of tickers, max 250 |
| `type` | Asset class filter (e.g. `CS` for common stock) |
| `limit` | Max results (default 10, max 250) |

**Python client call:**

```python
snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT"],
)

for snap in snapshots:
    print(snap.ticker, snap.last_trade.price, snap.last_trade.timestamp)
```

**Response shape (per ticker):**

```json
{
  "ticker": "AAPL",
  "market_status": "open",
  "last_trade": {
    "price": 190.42,
    "size": 100,
    "exchange": 11,
    "timestamp": 1718000000000   // Unix milliseconds
  },
  "last_minute": {
    "o": 190.10, "h": 190.50, "l": 190.05, "c": 190.42, "v": 12000, "vw": 190.30
  },
  "session": {
    "open": 188.00,
    "high": 191.00,
    "low": 187.50,
    "close": 190.42,
    "previous_close": 187.90,
    "change": 2.52,
    "change_percent": 1.34,
    "volume": 5400000
  }
}
```

Notes:
- `last_trade.timestamp` is **Unix milliseconds** — divide by 1000 for seconds
  (the project's `PriceUpdate.timestamp` is in seconds).
- `session.previous_close` is the field to use for a true "daily change %",
  as opposed to `PriceUpdate.change_percent` which is tick-over-tick.
- Snapshot data resets at 12:00 AM ET and populates as the market opens.

## 5. End-of-Day / Historical — Aggregates (Bars)

**Endpoint:**
```
GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}
```

| Param | Example | Description |
|---|---|---|
| `ticker` | `AAPL` | Case-sensitive symbol |
| `multiplier` | `1` | Size of the timespan window |
| `timespan` | `day`, `minute`, `hour` | Aggregation unit |
| `from` / `to` | `2026-05-01` / `2026-06-01` | Date range (YYYY-MM-DD or ms epoch) |

**Python client call:**

```python
aggs = client.get_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_="2026-05-01",
    to="2026-06-01",
)
for bar in aggs:
    print(bar.timestamp, bar.open, bar.high, bar.low, bar.close, bar.volume)
```

**Response shape:**

```json
{
  "ticker": "AAPL",
  "adjusted": true,
  "queryCount": 22,
  "resultsCount": 22,
  "status": "OK",
  "results": [
    {
      "o": 190.10,
      "h": 192.30,
      "l": 189.50,
      "c": 191.80,
      "v": 48213500,
      "vw": 190.92,
      "t": 1717200000000,
      "n": 215000
    }
  ]
}
```

Fields: `o`=open, `h`=high, `l`=low, `c`=close, `v`=volume, `vw`=VWAP,
`t`=start timestamp (ms), `n`=number of trades.

## 6. Previous Close (single ticker)

**Endpoint:** `GET /v2/aggs/ticker/{ticker}/prev`

**Python client call:**

```python
prev = client.get_previous_close_agg(ticker="AAPL")
```

Returns a single-element `results[]` array with the same OHLCV shape as
aggregates, representing the prior trading day's bar. Useful as a fallback
baseline for "daily change %" when `session.previous_close` isn't present.

## 7. Multi-Ticker Strategy for FinAlly

For the project's watchlist (≤10-20 tickers), a single `get_snapshot_all()`
call covers:
- **Latest price** → `last_trade.price` / `last_trade.timestamp`
- **Daily change %** → `session.change_percent` (or compute from
  `session.previous_close`)
- **Today's OHLC** → `last_minute` / `session`

This single call per poll cycle is sufficient — no need for separate
aggregates calls during live polling. Aggregates/previous-close endpoints are
only needed for historical chart backfill (not currently in scope).

## 8. Error Handling

| Status | Cause | Handling |
|---|---|---|
| 401 | Invalid/missing API key | Log error, fall back to simulator (see `MARKET_INTERFACE.md`) |
| 429 | Rate limit exceeded | Back off, retry on next poll interval |
| 200 with `"status": "NOT_FOUND"` per-ticker | Unknown/delisted ticker | Skip that ticker, keep others |

The current implementation (`massive_client.py`) catches all exceptions per
poll cycle, logs, and retries on the next interval — it never crashes the
background task.

---

## Sources

- [Getting Started | massive-com/client-python | DeepWiki](https://deepwiki.com/massive-com/client-python/2-getting-started)
- [GitHub - massive-com/client-python](https://github.com/polygon-io/client-python)
- [Unified Snapshot | Stocks REST API - Polygon](https://polygon.io/docs/rest/stocks/snapshots/unified-snapshot)
- [Custom Bars | Stocks REST API - Polygon](https://massive.com/docs/rest/stocks/aggregates/custom-bars)
- [Does Polygon have an endpoint that returns latest trades/quotes/aggregates in one request?](https://polygon.io/knowledge-base/article/does-polygon-have-an-endpoint-that-returns-the-latest-trades-quotes-and-aggregates-in-one-request)
