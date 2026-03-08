# Market Data Interface Design

The unified Python interface for market data in FinAlly. Two implementations — a GBM simulator and the Massive REST API client — sit behind a single abstract interface. All downstream code (SSE streaming, portfolio valuation, trade execution) is source-agnostic.

## Architecture

```
MarketDataSource (ABC)              interface.py
├── SimulatorDataSource    ──────── simulator.py
└── MassiveDataSource      ──────── massive_client.py
        │
        ▼
   PriceCache (thread-safe)         cache.py
        │
        ├──→ SSE /api/stream/prices  stream.py
        ├──→ GET /api/portfolio      (reads current prices for P&L)
        └──→ POST /api/portfolio/trade (reads price at execution time)
```

The data source is selected at startup by `create_market_data_source()` (factory.py). Both implementations push to the same `PriceCache`; no downstream code knows or cares which source is active.

---

## Module Layout

```
backend/app/market/
  __init__.py          # Public re-exports: PriceUpdate, PriceCache, MarketDataSource,
                       #                    create_market_data_source, create_stream_router
  models.py            # PriceUpdate dataclass
  interface.py         # MarketDataSource ABC
  cache.py             # PriceCache
  factory.py           # create_market_data_source()
  simulator.py         # GBMSimulator + SimulatorDataSource
  massive_client.py    # MassiveDataSource
  seed_prices.py       # SEED_PRICES, TICKER_PARAMS, correlation constants
  stream.py            # FastAPI SSE router factory
```

---

## Core Data Model: `PriceUpdate`

The only data structure that leaves the market data layer. Defined in `models.py`.

```python
from dataclasses import dataclass, field
import time

@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time."""

    ticker: str
    price: float           # Current price (rounded to 2 decimal places)
    previous_price: float  # Price from the immediately preceding update
    timestamp: float       # Unix seconds

    @property
    def change(self) -> float:
        """Absolute price change: price - previous_price."""
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        """Percentage change from previous price."""
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> str:
        """'up', 'down', or 'flat'."""
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

`PriceUpdate` is frozen (immutable) and uses `__slots__` for memory efficiency. It is safe to pass between threads without copying.

---

## Abstract Interface: `MarketDataSource`

Defined in `interface.py`. Both implementations must implement all five methods.

```python
from abc import ABC, abstractmethod

class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a shared PriceCache on their
    own schedule. Downstream code never calls the data source for prices —
    it reads from the cache.

    Lifecycle:
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL", ...])
        await source.add_ticker("TSLA")
        await source.remove_ticker("GOOGL")
        await source.stop()
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates for the given tickers.

        Starts a background task. Must be called exactly once.
        The cache should have initial prices immediately after this returns.
        """

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task and release resources.

        Safe to call multiple times. After stop(), no further cache writes occur.
        """

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present.

        The next update cycle will include this ticker.
        For the simulator, the cache is seeded immediately with an initial price.
        For Massive, the price appears on the next poll.
        """

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the active set. No-op if not present.

        Also removes the ticker from the PriceCache.
        """

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of actively tracked tickers."""
```

---

## Price Cache: `PriceCache`

Defined in `cache.py`. Single shared instance in the application. Thread-safe — uses a `threading.Lock` because the Massive client writes from a thread pool via `asyncio.to_thread()`.

```python
from threading import Lock
from .models import PriceUpdate
import time

class PriceCache:
    """Thread-safe in-memory cache of the latest price for each ticker.

    Writers: SimulatorDataSource or MassiveDataSource (one at a time).
    Readers: SSE streaming endpoint, portfolio valuation, trade execution.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0  # Monotonic counter; bumped on every update

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price. Returns the PriceUpdate.

        On the first update for a ticker, previous_price == price (direction='flat').
        """
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price

            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        """Latest price for one ticker, or None."""
        with self._lock:
            return self._prices.get(ticker)

    def get_price(self, ticker: str) -> float | None:
        """Convenience: just the price float, or None."""
        update = self.get(ticker)
        return update.price if update else None

    def get_all(self) -> dict[str, PriceUpdate]:
        """Snapshot of all current prices. Returns a shallow copy."""
        with self._lock:
            return dict(self._prices)

    def remove(self, ticker: str) -> None:
        """Remove a ticker (e.g., when removed from watchlist)."""
        with self._lock:
            self._prices.pop(ticker, None)

    @property
    def version(self) -> int:
        """Monotonic version counter. Used by SSE for change detection."""
        return self._version
```

The `version` property enables efficient SSE change detection: the streaming loop compares the current version to the last-seen version and only emits an event when prices have changed.

---

## Factory: `create_market_data_source()`

Defined in `factory.py`. Called once at app startup.

```python
import os
from .cache import PriceCache
from .interface import MarketDataSource
from .massive_client import MassiveDataSource
from .simulator import SimulatorDataSource

def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Select data source from environment.

    - MASSIVE_API_KEY set and non-empty → MassiveDataSource (real market data)
    - Otherwise → SimulatorDataSource (GBM simulation, default)

    Returns an unstarted source. Caller must await source.start(tickers).
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        return SimulatorDataSource(price_cache=price_cache)
```

---

## `MassiveDataSource` Implementation

Defined in `massive_client.py`. Polls the Massive REST API every `poll_interval` seconds.

```python
import asyncio
import logging
from massive import RESTClient
from massive.rest.models import SnapshotMarketType
from .cache import PriceCache
from .interface import MarketDataSource

class MassiveDataSource(MarketDataSource):
    """Polls GET /v2/snapshot/locale/us/markets/stocks/tickers for all tickers
    in a single API call, then writes results to PriceCache.

    poll_interval:
      - Free tier (5 req/min): 15s (default)
      - Paid tiers: 2–5s
    """

    def __init__(self, api_key: str, price_cache: PriceCache, poll_interval: float = 15.0):
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._client: RESTClient | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)
        await self._poll_once()                                    # Immediate first poll
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        self._client = None

    async def add_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        if ticker not in self._tickers:
            self._tickers.append(ticker)           # Appears on next poll

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    async def _poll_loop(self) -> None:
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        if not self._tickers or not self._client:
            return
        try:
            # RESTClient is synchronous — run in thread to avoid blocking event loop
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
            for snap in snapshots:
                try:
                    self._cache.update(
                        ticker=snap.ticker,
                        price=snap.last_trade.price,
                        timestamp=snap.last_trade.timestamp / 1000.0,  # ms → seconds
                    )
                except (AttributeError, TypeError) as e:
                    logger.warning("Skipping snapshot for %s: %s", getattr(snap, "ticker", "???"), e)
        except Exception as e:
            logger.error("Massive poll failed: %s", e)
            # Don't re-raise — loop retries on next interval

    def _fetch_snapshots(self) -> list:
        """Synchronous Massive API call. Runs inside asyncio.to_thread()."""
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

---

## `SimulatorDataSource` Implementation

Defined in `simulator.py`. Wraps `GBMSimulator` in an async loop that calls `step()` every 500ms.

```python
import asyncio
from .cache import PriceCache
from .interface import MarketDataSource
from .simulator import GBMSimulator

class SimulatorDataSource(MarketDataSource):
    """Wraps GBMSimulator in an asyncio background task.

    Calls GBMSimulator.step() every update_interval seconds and writes
    the resulting prices to PriceCache.
    """

    def __init__(self, price_cache: PriceCache, update_interval: float = 0.5,
                 event_probability: float = 0.001):
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        # Seed cache immediately so SSE has data before first step()
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)  # Immediate seed

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        while True:
            try:
                if self._sim:
                    prices = self._sim.step()
                    for ticker, price in prices.items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)
```

---

## SSE Integration

Defined in `stream.py`. A FastAPI router factory that injects the `PriceCache`.

```python
import asyncio
import json
from collections.abc import AsyncGenerator
from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse
from .cache import PriceCache

def create_stream_router(price_cache: PriceCache) -> APIRouter:
    """Returns an APIRouter with GET /api/stream/prices."""
    router = APIRouter(prefix="/api/stream", tags=["streaming"])

    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        return StreamingResponse(
            _generate_events(price_cache, request),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",  # Disable nginx buffering
            },
        )

    return router


async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:
    yield "retry: 1000\n\n"   # Client retries after 1 second on disconnect

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

The version-based check means an event is only emitted when prices actually changed — no redundant payloads.

---

## Application Lifecycle

```python
from app.market import PriceCache, create_market_data_source, create_stream_router

# 1. App startup
cache = PriceCache()
source = create_market_data_source(cache)    # Reads MASSIVE_API_KEY
initial_tickers = ["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA",
                   "NVDA", "META", "JPM", "V", "NFLX"]
await source.start(initial_tickers)          # Cache seeded immediately

# 2. Register SSE route
stream_router = create_stream_router(cache)
app.include_router(stream_router)

# 3. Watchlist changes (called from watchlist API endpoints)
await source.add_ticker("PYPL")
await source.remove_ticker("NFLX")

# 4. Reading prices (called from portfolio and trade endpoints)
update = cache.get("AAPL")          # PriceUpdate | None
price  = cache.get_price("AAPL")    # float | None
all_prices = cache.get_all()        # dict[str, PriceUpdate]

# 5. App shutdown
await source.stop()
```

---

## Public API (from `__init__.py`)

```python
from app.market import (
    PriceUpdate,               # models.py
    PriceCache,                # cache.py
    MarketDataSource,          # interface.py
    create_market_data_source, # factory.py
    create_stream_router,      # stream.py
)
```

These are the only symbols that downstream code should import. Internal modules (`simulator.py`, `massive_client.py`, `seed_prices.py`) are implementation details.
