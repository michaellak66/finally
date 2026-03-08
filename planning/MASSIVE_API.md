# Massive API Reference (formerly Polygon.io)

Reference documentation for the Massive REST API as used in FinAlly for real market data.

## Overview

Polygon.io rebranded as **Massive** on October 30, 2025. All existing API keys, accounts, and integrations continue to work unchanged. The Python client defaults to `api.massive.com` but `api.polygon.io` remains supported.

- **Base URL**: `https://api.massive.com` (legacy `https://api.polygon.io` also works)
- **Python package**: `massive` (install via `pip install -U massive` / `uv add massive`)
- **Min Python version**: 3.9+
- **Auth**: API key passed to `RESTClient(api_key=...)` or set in the `MASSIVE_API_KEY` env var
- **Auth header**: `Authorization: Bearer <API_KEY>` (handled automatically by the client)

## Rate Limits

| Tier | Limit | Recommended Poll Interval |
|------|-------|--------------------------|
| Free | 5 requests/minute | 15 seconds |
| Paid | Unlimited (stay under ~100 req/s) | 2–5 seconds |

FinAlly uses a single batch snapshot call for all tickers, so even the free tier can sustain a useful update cadence.

## Installation

```bash
# Using uv (project standard)
uv add massive

# Using pip
pip install -U massive
```

## Client Initialization

```python
from massive import RESTClient

# Reads MASSIVE_API_KEY from environment automatically
client = RESTClient()

# Or pass explicitly
client = RESTClient(api_key="your_key_here")
```

The `RESTClient` is **synchronous**. When used inside an async FastAPI app, run it in a thread pool via `asyncio.to_thread()` (see the FinAlly integration pattern below).

---

## Endpoints Used in FinAlly

### 1. Snapshot — All Tickers (Primary Endpoint)

Gets current price data for multiple tickers in a **single API call**. This is the only endpoint FinAlly uses for its polling loop.

**REST**: `GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,GOOGL,MSFT`

**Python client**:
```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient(api_key="your_key")

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA"],
)

for snap in snapshots:
    print(f"{snap.ticker}: ${snap.last_trade.price}")
    print(f"  Day change: {snap.day.change_percent:.2f}%")
    print(f"  Previous close: ${snap.day.previous_close}")
    print(f"  Trade timestamp (ms): {snap.last_trade.timestamp}")
```

**Response structure** (per ticker):
```json
{
  "ticker": "AAPL",
  "day": {
    "open": 188.50,
    "high": 192.10,
    "low": 187.30,
    "close": 190.75,
    "volume": 58234700,
    "volume_weighted_average_price": 190.12,
    "previous_close": 189.20,
    "change": 1.55,
    "change_percent": 0.82
  },
  "last_trade": {
    "price": 190.75,
    "size": 100,
    "exchange": "XNAS",
    "timestamp": 1709823600000
  },
  "last_quote": {
    "bid_price": 190.74,
    "ask_price": 190.76,
    "bid_size": 300,
    "ask_size": 200,
    "spread": 0.02,
    "timestamp": 1709823600500
  },
  "prev_daily_bar": { "...": "previous day OHLCV" }
}
```

**Fields extracted by FinAlly**:
- `snap.ticker` — ticker symbol
- `snap.last_trade.price` — current price for display and trade execution
- `snap.last_trade.timestamp` — Unix milliseconds; divide by 1000 for Unix seconds
- `snap.day.previous_close` — for computing day change % (not currently used but available)
- `snap.day.change_percent` — day change percentage (not currently used but available)

### 2. Single Ticker Snapshot

For getting detailed data on a single ticker.

```python
snapshot = client.get_snapshot_ticker(
    market_type=SnapshotMarketType.STOCKS,
    ticker="AAPL",
)

print(f"Price: ${snapshot.last_trade.price}")
print(f"Bid/Ask: ${snapshot.last_quote.bid_price} / ${snapshot.last_quote.ask_price}")
print(f"Day range: ${snapshot.day.low} – ${snapshot.day.high}")
```

### 3. Previous Close

Gets the previous trading day's OHLCV. Useful for seeding initial prices when the market is closed or for computing day-over-day change.

**REST**: `GET /v2/aggs/ticker/{ticker}/prev`

```python
for agg in client.get_previous_close_agg(ticker="AAPL"):
    print(f"Previous close: ${agg.close}")
    print(f"OHLC: O={agg.open} H={agg.high} L={agg.low} C={agg.close}")
    print(f"Volume: {agg.volume}")
    print(f"Timestamp (ms): {agg.timestamp}")
```

### 4. Aggregates (Historical Bars)

Historical OHLCV bars over a date range. Not used in FinAlly's live polling but useful if historical charting is added.

**REST**: `GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}`

```python
aggs = list(client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_="2025-01-01",
    to="2025-03-01",
    limit=50000,
))

for a in aggs:
    print(f"t={a.timestamp} O={a.open} H={a.high} L={a.low} C={a.close} V={a.volume}")
```

---

## How FinAlly Uses the API

The `MassiveDataSource` runs as a background asyncio task:

1. On `start()`, do an **immediate first poll** so the price cache is populated right away
2. Launch a background loop that calls `_poll_once()` every `poll_interval` seconds (default 15s)
3. Each poll: call `get_snapshot_all()` with all current tickers (one API call)
4. Parse `last_trade.price` and `last_trade.timestamp` from each snapshot
5. Write to the shared `PriceCache`
6. Sleep; repeat

Because `RESTClient` is synchronous, each poll runs via `asyncio.to_thread()` to avoid blocking the event loop:

```python
import asyncio
from massive import RESTClient
from massive.rest.models import SnapshotMarketType
from app.market.cache import PriceCache

async def poll_once(client: RESTClient, tickers: list[str], cache: PriceCache) -> None:
    # Run synchronous client in thread pool — never block the event loop
    snapshots = await asyncio.to_thread(
        client.get_snapshot_all,
        market_type=SnapshotMarketType.STOCKS,
        tickers=tickers,
    )
    for snap in snapshots:
        price = snap.last_trade.price
        timestamp = snap.last_trade.timestamp / 1000.0  # ms → seconds
        cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)
```

See `backend/app/market/massive_client.py` for the full production implementation.

---

## Error Handling

The client raises standard HTTP exceptions. FinAlly's poller catches all exceptions and logs them — a failed poll is silently retried on the next interval so transient errors don't crash the app.

| Status | Cause | FinAlly behavior |
|--------|-------|-----------------|
| 401 | Invalid API key | Logged as error; poller keeps running |
| 403 | Plan doesn't include endpoint | Logged as error; poller keeps running |
| 429 | Rate limit exceeded (free tier) | Logged as error; next attempt after sleep |
| 5xx | Massive server error | Logged as error; built-in retry in client |

```python
try:
    snapshots = await asyncio.to_thread(self._fetch_snapshots)
    # process...
except Exception as e:
    logger.error("Massive poll failed: %s", e)
    # Don't re-raise — loop retries on next interval
```

---

## Notes

- **One call for all tickers**: `get_snapshot_all()` fetches every watched ticker in a single HTTP request. This is critical for staying within free-tier rate limits.
- **Timestamps are Unix milliseconds**: divide by 1000 to get Unix seconds (`snap.last_trade.timestamp / 1000.0`).
- **After-hours prices**: during market-closed hours, `last_trade.price` reflects the last traded price, which may be from after-hours or pre-market.
- **Day object resets at market open**: `snap.day.previous_close` may be from the previous session during pre-market hours.
- **Invalid tickers**: the API returns an error for unrecognized tickers. FinAlly surfaces this as a 422 response when adding to the watchlist. The simulator accepts any ticker gracefully.
- **Backward compatibility**: `api.polygon.io` endpoints continue to work. No migration needed for code that targets the old domain.

---

## References

- [Massive Python Client (GitHub)](https://github.com/massive-com/client-python)
- [Stocks REST API Docs](https://massive.com/docs/rest/stocks)
- [Full Market Snapshot Endpoint](https://massive.com/docs/rest/stocks/snapshots/full-market-snapshot)
- [Massive Blog: Polygon is Now Massive](https://massive.com/blog/polygon-is-now-massive)
