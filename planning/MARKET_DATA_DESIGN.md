# Market Data Backend — Detailed Design

Implementation-ready design for the FinAlly market data subsystem. Covers the unified interface, in-memory price cache, GBM simulator, Massive API client, SSE streaming endpoint, and FastAPI lifecycle integration.

Everything lives under `backend/app/market/`.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [File Structure](#2-file-structure)
3. [Data Model — `models.py`](#3-data-model--modelspy)
4. [Price Cache — `cache.py`](#4-price-cache--cachepy)
5. [Abstract Interface — `interface.py`](#5-abstract-interface--interfacepy)
6. [Seed Prices & Ticker Parameters — `seed_prices.py`](#6-seed-prices--ticker-parameters--seed_pricespy)
7. [GBM Simulator — `simulator.py`](#7-gbm-simulator--simulatorpy)
8. [Massive API Client — `massive_client.py`](#8-massive-api-client--massive_clientpy)
9. [Factory — `factory.py`](#9-factory--factorypy)
10. [SSE Streaming Endpoint — `stream.py`](#10-sse-streaming-endpoint--streampy)
11. [Package Exports — `__init__.py`](#11-package-exports--__init__py)
12. [FastAPI Lifecycle Integration](#12-fastapi-lifecycle-integration)
13. [Watchlist Coordination](#13-watchlist-coordination)
14. [Testing Strategy](#14-testing-strategy)
15. [Error Handling & Edge Cases](#15-error-handling--edge-cases)
16. [Configuration Summary](#16-configuration-summary)

---

## 1. Architecture Overview

```
MarketDataSource (ABC)              interface.py
├── SimulatorDataSource  ────────── simulator.py     (default: GBM simulation)
└── MassiveDataSource    ────────── massive_client.py (when MASSIVE_API_KEY set)
        │
        ▼
   PriceCache (thread-safe, in-memory)    cache.py
        │
        ├──→ GET /api/stream/prices   (SSE, long-lived)
        ├──→ GET /api/portfolio       (P&L computation at request time)
        ├──→ POST /api/portfolio/trade (price snapshot at execution time)
        └──→ GET /api/watchlist       (latest price per ticker)
```

**Strategy pattern** — both data sources implement the same ABC. All downstream code is source-agnostic; only the factory reads `MASSIVE_API_KEY`.

**Single shared `PriceCache`** — the one place prices live. Producers write, consumers read. Thread-safe with a version counter for efficient SSE change detection.

**SSE over WebSockets** — one-way server→client push is all the frontend needs. Simpler, universal browser support via `EventSource`, no handshake complexity.

---

## 2. File Structure

```
backend/
  app/
    market/
      __init__.py         # Public re-exports only
      models.py           # PriceUpdate frozen dataclass
      cache.py            # PriceCache (thread-safe)
      interface.py        # MarketDataSource ABC
      seed_prices.py      # SEED_PRICES, TICKER_PARAMS, correlation constants
      simulator.py        # GBMSimulator + SimulatorDataSource
      massive_client.py   # MassiveDataSource (Polygon.io / Massive REST)
      factory.py          # create_market_data_source()
      stream.py           # FastAPI SSE router factory
  tests/
    market/
      test_models.py
      test_cache.py
      test_simulator.py
      test_simulator_source.py
      test_factory.py
      test_massive.py
```

---

## 3. Data Model — `models.py`

The only data structure that leaves the market data layer. Frozen and slotted for thread safety and memory efficiency.

```python
# backend/app/market/models.py
from dataclasses import dataclass
import time


@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time."""

    ticker: str
    price: float           # Current price, rounded to 2 decimal places
    previous_price: float  # Immediately preceding price
    timestamp: float       # Unix seconds (float)

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

**Why frozen + slots?** Immutability makes it safe to hand `PriceUpdate` objects across thread boundaries without copying. `__slots__` eliminates `__dict__` overhead — this object is created ~20 times per second.

---

## 4. Price Cache — `cache.py`

Single shared in-memory store. Thread-safe via `threading.Lock`. The `version` counter enables the SSE loop to detect changes without polling every key.

```python
# backend/app/market/cache.py
import time
from threading import Lock
from .models import PriceUpdate


class PriceCache:
    """Thread-safe in-memory store for the latest price of each ticker.

    Writers: exactly one MarketDataSource (simulator or Massive poller).
    Readers: SSE endpoint, portfolio endpoints, trade execution.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0   # Bumped on every write

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price. Returns the resulting PriceUpdate.

        On the first update for a ticker, previous_price == price (direction='flat').
        """
        with self._lock:
            ts = timestamp if timestamp is not None else time.time()
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
        """Latest PriceUpdate for one ticker, or None if not tracked."""
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
        """Remove a ticker (called when removed from watchlist)."""
        with self._lock:
            self._prices.pop(ticker, None)
            self._version += 1

    @property
    def version(self) -> int:
        """Monotonic counter. The SSE loop compares this to detect changes."""
        return self._version
```

**Version counter pattern**: the SSE generator records `last_version = price_cache.version` at each iteration. It only emits a new SSE event when `current_version != last_version`. This avoids sending duplicate payloads when the loop wakes up but no prices changed.

---

## 5. Abstract Interface — `interface.py`

Both implementations must satisfy this contract. Downstream code depends only on this ABC.

```python
# backend/app/market/interface.py
from abc import ABC, abstractmethod


class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push PriceUpdate objects into a shared PriceCache on their
    own schedule. Downstream code never calls the data source for prices directly —
    it reads from the cache.

    Lifecycle:
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL", ...])  # starts background task, seeds cache
        await source.add_ticker("TSLA")              # adds ticker to active set
        await source.remove_ticker("GOOGL")          # removes from active set + cache
        await source.stop()                          # cancels background task
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates for the given tickers.

        Starts a background asyncio task. Must be called exactly once.
        The cache must have initial prices immediately after this returns.
        """

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task and release resources.

        Safe to call multiple times. After stop(), no further cache writes occur.
        """

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present.

        Simulator: seeds the cache immediately with an initial price.
        Massive: price appears on the next poll (up to poll_interval seconds later).
        """

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the active set. No-op if not present.

        Also removes the ticker from the PriceCache so downstream endpoints
        stop seeing stale prices for this ticker.
        """

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of actively tracked tickers."""
```

---

## 6. Seed Prices & Ticker Parameters — `seed_prices.py`

Calibration constants for the simulator. Separated from `simulator.py` so they can be edited or extended without touching simulation logic.

```python
# backend/app/market/seed_prices.py

# Realistic starting prices for the default 10 tickers
SEED_PRICES: dict[str, float] = {
    "AAPL":  190.00,
    "GOOGL": 175.00,
    "MSFT":  420.00,
    "AMZN":  185.00,
    "TSLA":  250.00,
    "NVDA":  800.00,
    "META":  500.00,
    "JPM":   195.00,
    "V":     280.00,
    "NFLX":  600.00,
}

# Per-ticker GBM parameters: annualized volatility (sigma) and drift (mu)
TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},   # High vol, lower drift
    "NVDA":  {"sigma": 0.40, "mu": 0.08},   # High vol, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},   # Stable bank stock
    "V":     {"sigma": 0.17, "mu": 0.04},   # Stable payments stock
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

# Fallback for tickers not in the tables above
DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}

# Sector groupings drive the pairwise correlation matrix
CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

# Correlation coefficients used to build the Cholesky matrix
INTRA_TECH_CORR    = 0.6   # Strong: tech stocks move together
INTRA_FINANCE_CORR = 0.5   # Moderate: finance stocks
CROSS_GROUP_CORR   = 0.3   # Weak: cross-sector or unknown
TSLA_CORR          = 0.3   # TSLA is in tech but treated independently
```

---

## 7. GBM Simulator — `simulator.py`

Two classes: `GBMSimulator` (pure sync, the math engine) and `SimulatorDataSource` (async wrapper implementing the interface).

### 7.1 GBM Mathematics

At each time step a stock price evolves as:

```
S(t + dt) = S(t) * exp((mu - sigma²/2) * dt  +  sigma * sqrt(dt) * Z)
```

| Symbol | Meaning |
|--------|---------|
| `S(t)` | Current price |
| `mu` | Annualized drift (e.g. 0.05 = 5% annual return) |
| `sigma` | Annualized volatility (e.g. 0.22 = 22%) |
| `dt` | Time step as fraction of a trading year |
| `Z` | Draw from N(0,1) — standard normal |

The `exp()` form is the exact solution to the GBM SDE and guarantees prices can never reach zero or go negative.

**Time step calibration:**

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # ~5.9 million seconds
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ~8.48e-8 per 500ms tick
```

This tiny `dt` produces sub-cent moves per tick (typically $0.001–$0.05 depending on `sigma` and the current price level), which accumulate naturally into realistic intraday ranges.

### 7.2 Correlated Moves via Cholesky Decomposition

Real stocks don't move independently — tech stocks correlate at ~0.6 intraday. The simulator captures this:

1. Build correlation matrix `C` (n×n) using `_pairwise_correlation()`.
2. Compute lower-triangular Cholesky factor: `L = cholesky(C)`.
3. At each step draw `n` independent standard normals: `Z_ind ~ N(0, I_n)`.
4. Produce correlated draws: `Z_corr = L @ Z_ind`.
5. Use `Z_corr[i]` as Z for ticker `i` in the GBM formula.

The Cholesky matrix is rebuilt whenever tickers are added or removed. At n < 50 tickers this is O(n²) and negligible.

### 7.3 `GBMSimulator` Class

```python
# backend/app/market/simulator.py
import math
import random
import numpy as np
from .seed_prices import (
    SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS,
    CORRELATION_GROUPS, INTRA_TECH_CORR, INTRA_FINANCE_CORR,
    CROSS_GROUP_CORR, TSLA_CORR,
)


class GBMSimulator:
    """Generates correlated GBM price paths for multiple tickers.

    Pure synchronous Python — no asyncio. Called from SimulatorDataSource.
    """

    TRADING_SECONDS_PER_YEAR: float = 252 * 6.5 * 3600
    DEFAULT_DT: float = 0.5 / TRADING_SECONDS_PER_YEAR  # ~8.48e-8

    def __init__(
        self,
        tickers: list[str],
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
    ) -> None:
        self._dt = dt
        self._event_prob = event_probability
        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None

        for ticker in tickers:
            self._add_ticker_internal(ticker)  # Batch init — no per-ticker Cholesky rebuild
        self._rebuild_cholesky()

    # ------------------------------------------------------------------
    # Hot path — called every 500ms
    # ------------------------------------------------------------------

    def step(self) -> dict[str, float]:
        """Advance all tickers one time step. Returns {ticker: new_price}."""
        n = len(self._tickers)
        if n == 0:
            return {}

        z_ind = np.random.standard_normal(n)
        z = self._cholesky @ z_ind if self._cholesky is not None else z_ind

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            mu    = self._params[ticker]["mu"]
            sigma = self._params[ticker]["sigma"]

            drift     = (mu - 0.5 * sigma ** 2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            # Random shock event: 0.1% chance per tick of a sudden 2–5% move
            if random.random() < self._event_prob:
                shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
                self._prices[ticker] *= (1 + shock)

            result[ticker] = round(self._prices[ticker], 2)

        return result

    # ------------------------------------------------------------------
    # Dynamic ticker management
    # ------------------------------------------------------------------

    def add_ticker(self, ticker: str) -> None:
        """Add a ticker and rebuild the correlation matrix."""
        if ticker in self._prices:
            return
        self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker and rebuild the correlation matrix."""
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_price(self, ticker: str) -> float | None:
        return self._prices.get(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # ------------------------------------------------------------------
    # Internal helpers
    # ------------------------------------------------------------------

    def _add_ticker_internal(self, ticker: str) -> None:
        """Add without rebuilding Cholesky — used during batch initialization."""
        if ticker in self._prices:
            return
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
        self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return
        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
                corr[i, j] = rho
                corr[j, i] = rho
        self._cholesky = np.linalg.cholesky(corr)

    @staticmethod
    def _pairwise_correlation(t1: str, t2: str) -> float:
        tech    = CORRELATION_GROUPS["tech"]
        finance = CORRELATION_GROUPS["finance"]
        if t1 == "TSLA" or t2 == "TSLA":
            return TSLA_CORR
        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR
        return CROSS_GROUP_CORR
```

### 7.4 `SimulatorDataSource` Class

```python
# backend/app/market/simulator.py  (continued)
import asyncio
import logging
from .cache import PriceCache
from .interface import MarketDataSource

logger = logging.getLogger(__name__)


class SimulatorDataSource(MarketDataSource):
    """Runs GBMSimulator in an asyncio background task.

    Calls GBMSimulator.step() every update_interval seconds and writes
    the resulting prices to PriceCache.
    """

    def __init__(
        self,
        price_cache: PriceCache,
        update_interval: float = 0.5,
        event_probability: float = 0.001,
    ) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        # Seed the cache immediately — SSE clients get data before the first step()
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")
        logger.info("Simulator started with %d tickers", len(tickers))

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        logger.info("Simulator stopped")

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)  # Immediate seed
            logger.info("Simulator: added ticker %s", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)
        logger.info("Simulator: removed ticker %s", ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        """Core loop: step → write to cache → sleep → repeat."""
        while True:
            try:
                if self._sim:
                    prices = self._sim.step()
                    for ticker, price in prices.items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed — continuing")
            await asyncio.sleep(self._interval)
```

---

## 8. Massive API Client — `massive_client.py`

Polls the Massive (formerly Polygon.io) REST API every `poll_interval` seconds. One API call per poll fetches all tickers simultaneously, staying within free-tier rate limits.

```python
# backend/app/market/massive_client.py
import asyncio
import logging
from massive import RESTClient
from massive.rest.models import SnapshotMarketType
from .cache import PriceCache
from .interface import MarketDataSource

logger = logging.getLogger(__name__)


class MassiveDataSource(MarketDataSource):
    """Polls GET /v2/snapshot/locale/us/markets/stocks/tickers for all tickers
    in a single API call, then writes to PriceCache.

    poll_interval defaults:
      - Free tier (5 req/min): 15s
      - Paid tiers: 2–5s
    """

    def __init__(
        self,
        api_key: str,
        price_cache: PriceCache,
        poll_interval: float = 15.0,
    ) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._client: RESTClient | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)
        await self._poll_once()   # Immediate first poll — cache populated before returning
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")
        logger.info("Massive poller started, interval=%ss", self._interval)

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        self._client = None
        logger.info("Massive poller stopped")

    async def add_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        if ticker not in self._tickers:
            self._tickers.append(ticker)
            # Price will appear on the next poll (up to poll_interval seconds)
            logger.info("Massive: queued ticker %s (appears on next poll)", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)
        logger.info("Massive: removed ticker %s", ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # ------------------------------------------------------------------
    # Internal polling
    # ------------------------------------------------------------------

    async def _poll_loop(self) -> None:
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        if not self._tickers or not self._client:
            return
        try:
            # RESTClient is synchronous — run in thread pool to avoid blocking the event loop
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
            for snap in snapshots:
                try:
                    price     = snap.last_trade.price
                    timestamp = snap.last_trade.timestamp / 1000.0  # ms → seconds
                    self._cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)
                except (AttributeError, TypeError) as e:
                    logger.warning("Skipping bad snapshot for %s: %s",
                                   getattr(snap, "ticker", "???"), e)
        except Exception as e:
            logger.error("Massive poll failed: %s", e)
            # Don't re-raise — loop silently retries on next interval

    def _fetch_snapshots(self) -> list:
        """Synchronous Massive API call. Must run inside asyncio.to_thread()."""
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

### Massive API Response Fields Used

```python
# Per-snapshot fields extracted by _poll_once():
snap.ticker                    # str — ticker symbol (e.g. "AAPL")
snap.last_trade.price          # float — current traded price
snap.last_trade.timestamp      # int — Unix milliseconds (divide by 1000 for seconds)

# Available but not currently used (for future enhancement):
snap.day.previous_close        # float — prior session close
snap.day.change_percent        # float — day-over-day % change
snap.last_quote.bid_price      # float — current bid
snap.last_quote.ask_price      # float — current ask
```

---

## 9. Factory — `factory.py`

Called once at application startup. Selects the implementation based on the environment.

```python
# backend/app/market/factory.py
import os
from .cache import PriceCache
from .interface import MarketDataSource
from .massive_client import MassiveDataSource
from .simulator import SimulatorDataSource


def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Select data source from environment.

    - MASSIVE_API_KEY set and non-empty → MassiveDataSource (real market data)
    - Otherwise → SimulatorDataSource (GBM simulation, default)

    Returns an unstarted source. Caller must: await source.start(tickers)
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        logger.info("Using Massive (Polygon.io) market data source")
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        logger.info("Using GBM simulator market data source")
        return SimulatorDataSource(price_cache=price_cache)
```

---

## 10. SSE Streaming Endpoint — `stream.py`

A FastAPI router factory. Injects the `PriceCache` as a closure so the router has no global state.

```python
# backend/app/market/stream.py
import asyncio
import json
import logging
from collections.abc import AsyncGenerator
from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse
from .cache import PriceCache

logger = logging.getLogger(__name__)


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
                "X-Accel-Buffering": "no",   # Disable nginx buffering
            },
        )

    return router


async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:
    """Async generator that yields SSE-formatted strings.

    Uses version-based change detection — only emits when prices have changed.
    """
    # Tell the client to retry after 1 second on disconnect
    yield "retry: 1000\n\n"

    last_version = -1
    while True:
        if await request.is_disconnected():
            logger.debug("SSE client disconnected")
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

### SSE Event Format

Each event is a JSON object keyed by ticker. Frontend receives this and updates each watched ticker's display.

```
data: {"AAPL": {"ticker": "AAPL", "price": 190.75, "previous_price": 190.62,
       "timestamp": 1709823600.5, "change": 0.13, "change_percent": 0.0682,
       "direction": "up"}, "GOOGL": {...}, ...}

```

(Note the blank line terminates the SSE event.)

### Frontend `EventSource` Usage

```typescript
const source = new EventSource('/api/stream/prices');

source.onmessage = (event) => {
  const prices: Record<string, PriceUpdate> = JSON.parse(event.data);
  for (const [ticker, update] of Object.entries(prices)) {
    updateTickerDisplay(ticker, update);  // triggers price flash CSS animation
  }
};

source.onerror = () => {
  // EventSource auto-reconnects after the retry: 1000 directive
  setConnectionStatus('reconnecting');
};
```

---

## 11. Package Exports — `__init__.py`

Only these symbols are part of the public API. Internal modules are implementation details.

```python
# backend/app/market/__init__.py
from .models import PriceUpdate
from .cache import PriceCache
from .interface import MarketDataSource
from .factory import create_market_data_source
from .stream import create_stream_router

__all__ = [
    "PriceUpdate",
    "PriceCache",
    "MarketDataSource",
    "create_market_data_source",
    "create_stream_router",
]
```

Downstream imports:

```python
from app.market import PriceCache, create_market_data_source, create_stream_router
```

---

## 12. FastAPI Lifecycle Integration

The market data subsystem hooks into FastAPI's `lifespan` context manager. The `PriceCache` and `MarketDataSource` are stored in `app.state` so they can be injected into API routes.

```python
# backend/app/main.py
from contextlib import asynccontextmanager
import logging
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from app.market import PriceCache, create_market_data_source, create_stream_router

logger = logging.getLogger(__name__)

DEFAULT_TICKERS = ["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA",
                   "NVDA", "META", "JPM", "V", "NFLX"]


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ── Startup ────────────────────────────────────────────────────────
    cache  = PriceCache()
    source = create_market_data_source(cache)   # Reads MASSIVE_API_KEY from env

    # Load initial tickers from the database (so persisted watchlist is respected)
    # Fallback to defaults if DB unavailable on first boot
    initial_tickers = await _load_watchlist_from_db() or DEFAULT_TICKERS
    await source.start(initial_tickers)          # Cache seeded before first request

    app.state.price_cache   = cache
    app.state.market_source = source

    logger.info("Market data subsystem started with tickers: %s", initial_tickers)

    yield  # App is now running

    # ── Shutdown ───────────────────────────────────────────────────────
    await source.stop()
    logger.info("Market data subsystem stopped")


app = FastAPI(lifespan=lifespan)

# Register SSE route (injects cache via closure)
# Must be registered after cache is created — here it's done after lifespan setup
# so we use a deferred registration pattern:
def setup_routes(app: FastAPI) -> None:
    from app.market import create_stream_router
    # Cache is available at this point via app.state after startup
    # Better: create the router here and include it
    pass

# Preferred pattern — create router with cache before lifespan using a module-level cache:
# Or use FastAPI dependency injection (see below)
```

### Dependency Injection for Routes

```python
# backend/app/dependencies.py
from fastapi import Request
from app.market import PriceCache, MarketDataSource


def get_price_cache(request: Request) -> PriceCache:
    return request.app.state.price_cache


def get_market_source(request: Request) -> MarketDataSource:
    return request.app.state.market_source
```

```python
# backend/app/routers/watchlist.py
from fastapi import APIRouter, Depends
from app.dependencies import get_market_source, get_price_cache
from app.market import MarketDataSource, PriceCache

router = APIRouter(prefix="/api/watchlist", tags=["watchlist"])


@router.post("")
async def add_ticker(
    body: AddTickerRequest,
    source: MarketDataSource = Depends(get_market_source),
    cache: PriceCache = Depends(get_price_cache),
):
    ticker = body.ticker.upper().strip()
    await source.add_ticker(ticker)
    # For simulator: price is immediately in cache. For Massive: may take one poll interval.
    price = cache.get_price(ticker)
    return {"ticker": ticker, "price": price}


@router.delete("/{ticker}")
async def remove_ticker(
    ticker: str,
    source: MarketDataSource = Depends(get_market_source),
):
    await source.remove_ticker(ticker.upper())
    return {"removed": ticker.upper()}
```

---

## 13. Watchlist Coordination

The market data layer must track the **union of watchlist tickers and held position tickers** so P&L stays current even if the user removes a watched ticker.

```python
# backend/app/routers/portfolio.py — trade execution

@router.post("/trade")
async def execute_trade(
    body: TradeRequest,
    source: MarketDataSource = Depends(get_market_source),
    cache: PriceCache = Depends(get_price_cache),
    db: AsyncConnection = Depends(get_db),
):
    ticker = body.ticker.upper()
    price  = cache.get_price(ticker)

    if price is None:
        raise HTTPException(status_code=404, detail=f"No price data for {ticker}")

    # ... execute trade logic, update DB ...

    # If this ticker is now held but not on the watchlist, ensure it's being tracked
    # so its price (and therefore P&L) continues to update via SSE
    if body.side == "buy":
        tickers = source.get_tickers()
        if ticker not in tickers:
            await source.add_ticker(ticker)

    return trade_result
```

The SSE endpoint streams prices for **all tickers tracked by the source** — both watchlist entries and position tickers — so the frontend's portfolio heatmap and P&L calculations stay current even for tickers removed from the watchlist.

---

## 14. Testing Strategy

### Unit Tests

**`test_models.py`** — `PriceUpdate` invariants:

```python
def test_direction_up():
    u = PriceUpdate(ticker="AAPL", price=191.0, previous_price=190.0, timestamp=0.0)
    assert u.direction == "up"
    assert u.change == 1.0
    assert u.change_percent == pytest.approx(0.5263, rel=1e-3)

def test_direction_flat():
    u = PriceUpdate(ticker="AAPL", price=190.0, previous_price=190.0, timestamp=0.0)
    assert u.direction == "flat"
    assert u.change == 0.0

def test_to_dict_keys():
    u = PriceUpdate(ticker="AAPL", price=191.0, previous_price=190.0, timestamp=1000.0)
    d = u.to_dict()
    assert set(d.keys()) == {"ticker", "price", "previous_price", "timestamp",
                              "change", "change_percent", "direction"}
```

**`test_cache.py`** — thread safety and version counter:

```python
def test_version_increments_on_update():
    cache = PriceCache()
    v0 = cache.version
    cache.update("AAPL", 190.0)
    assert cache.version == v0 + 1

def test_first_update_sets_previous_to_same():
    cache = PriceCache()
    update = cache.update("AAPL", 190.0)
    assert update.previous_price == update.price
    assert update.direction == "flat"

def test_second_update_carries_previous():
    cache = PriceCache()
    cache.update("AAPL", 190.0)
    u2 = cache.update("AAPL", 192.0)
    assert u2.previous_price == 190.0
    assert u2.direction == "up"

def test_thread_safety():
    import threading
    cache = PriceCache()
    errors = []

    def writer():
        for i in range(1000):
            try:
                cache.update("AAPL", 190.0 + i * 0.01)
            except Exception as e:
                errors.append(e)

    threads = [threading.Thread(target=writer) for _ in range(5)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    assert errors == []
```

**`test_simulator.py`** — GBM properties:

```python
def test_step_returns_all_tickers():
    sim = GBMSimulator(["AAPL", "GOOGL", "MSFT"])
    prices = sim.step()
    assert set(prices.keys()) == {"AAPL", "GOOGL", "MSFT"}

def test_prices_always_positive():
    sim = GBMSimulator(["TSLA"])   # High volatility ticker
    for _ in range(1000):
        prices = sim.step()
        assert prices["TSLA"] > 0

def test_prices_rounded_to_cents():
    sim = GBMSimulator(["AAPL"])
    for _ in range(100):
        price = sim.step()["AAPL"]
        assert round(price, 2) == price

def test_add_ticker_appears_in_step():
    sim = GBMSimulator(["AAPL"])
    sim.add_ticker("NVDA")
    prices = sim.step()
    assert "NVDA" in prices

def test_remove_ticker_absent_from_step():
    sim = GBMSimulator(["AAPL", "GOOGL"])
    sim.remove_ticker("GOOGL")
    prices = sim.step()
    assert "GOOGL" not in prices

def test_correlation_produces_comovement():
    """Tech stocks should trend in the same direction more than 50% of the time."""
    sim = GBMSimulator(["AAPL", "MSFT"], event_probability=0.0)
    same_direction = 0
    for _ in range(1000):
        initial = {"AAPL": sim.get_price("AAPL"), "MSFT": sim.get_price("MSFT")}
        prices = sim.step()
        aapl_up = prices["AAPL"] > initial["AAPL"]
        msft_up = prices["MSFT"] > initial["MSFT"]
        if aapl_up == msft_up:
            same_direction += 1
    # With correlation 0.6 expect ~70–75% same-direction
    assert same_direction > 600
```

**`test_factory.py`** — environment-driven selection:

```python
def test_no_key_returns_simulator(monkeypatch):
    monkeypatch.delenv("MASSIVE_API_KEY", raising=False)
    source = create_market_data_source(PriceCache())
    assert isinstance(source, SimulatorDataSource)

def test_empty_key_returns_simulator(monkeypatch):
    monkeypatch.setenv("MASSIVE_API_KEY", "")
    source = create_market_data_source(PriceCache())
    assert isinstance(source, SimulatorDataSource)

def test_key_set_returns_massive(monkeypatch):
    monkeypatch.setenv("MASSIVE_API_KEY", "test_key_123")
    source = create_market_data_source(PriceCache())
    assert isinstance(source, MassiveDataSource)
```

**`test_simulator_source.py`** — async integration:

```python
import asyncio

async def test_start_seeds_cache():
    cache = PriceCache()
    source = SimulatorDataSource(price_cache=cache)
    await source.start(["AAPL", "GOOGL"])
    assert cache.get_price("AAPL") is not None
    assert cache.get_price("GOOGL") is not None
    await source.stop()

async def test_prices_update_over_time():
    cache = PriceCache()
    source = SimulatorDataSource(price_cache=cache, update_interval=0.1)
    await source.start(["AAPL"])
    price_1 = cache.get_price("AAPL")
    await asyncio.sleep(0.35)  # Wait for ~3 steps
    price_2 = cache.get_price("AAPL")
    await source.stop()
    # Prices should have moved (extremely unlikely to be identical after 3 steps)
    assert price_1 != price_2

async def test_stop_is_idempotent():
    cache = PriceCache()
    source = SimulatorDataSource(price_cache=cache)
    await source.start(["AAPL"])
    await source.stop()
    await source.stop()   # Should not raise
```

**`test_massive.py`** — with mocked API client:

```python
from unittest.mock import MagicMock, patch

async def test_poll_updates_cache():
    cache = PriceCache()
    source = MassiveDataSource(api_key="key", price_cache=cache, poll_interval=60.0)

    snap = MagicMock()
    snap.ticker = "AAPL"
    snap.last_trade.price = 195.50
    snap.last_trade.timestamp = 1709823600000

    source._client = MagicMock()
    with patch.object(source, "_fetch_snapshots", return_value=[snap]):
        await source._poll_once()

    assert cache.get_price("AAPL") == 195.50

async def test_bad_snapshot_skipped():
    cache = PriceCache()
    source = MassiveDataSource(api_key="key", price_cache=cache)
    source._client = MagicMock()

    bad_snap = MagicMock()
    bad_snap.ticker = "BAD"
    bad_snap.last_trade = None   # Missing last_trade → AttributeError

    with patch.object(source, "_fetch_snapshots", return_value=[bad_snap]):
        await source._poll_once()   # Should not raise

    assert cache.get("BAD") is None

async def test_poll_failure_does_not_propagate():
    cache = PriceCache()
    source = MassiveDataSource(api_key="key", price_cache=cache)
    source._client = MagicMock()

    with patch.object(source, "_fetch_snapshots", side_effect=RuntimeError("network error")):
        await source._poll_once()   # Should silently log and return
```

---

## 15. Error Handling & Edge Cases

| Scenario | Behavior |
|----------|----------|
| No prices in cache yet | `cache.get_price(ticker)` returns `None`; endpoints return 503 or empty state |
| Ticker not tracked by source | `cache.get("UNKNOWN")` returns `None`; watchlist POST should add it to source first |
| Massive poll fails (401, 429, 5xx) | Logged as error; loop sleeps and retries on next interval; no exception propagates |
| Bad snapshot (missing `last_trade`) | Per-snapshot `AttributeError` caught and logged; other tickers in that poll succeed |
| Simulator step throws (rare numpy error) | `logger.exception()` in `_run_loop`; loop continues; GBM itself cannot throw under normal conditions |
| Client disconnects SSE | `await request.is_disconnected()` returns `True`; generator exits cleanly |
| Ticker added while simulator running | `GBMSimulator.add_ticker()` + Cholesky rebuild + immediate cache seed |
| Ticker added while Massive running | Added to `_tickers` list; appears in next poll |
| Invalid ticker with Massive API | API returns error response; caught by `_poll_once` exception handler |
| Invalid ticker with simulator | Accepted; uses `DEFAULT_PARAMS` (sigma=0.25) and random seed price $50–$300 |
| Same ticker added twice | No-op in both implementations; idempotent |
| `stop()` called before `start()` | No-op; `_task` is `None` |
| `stop()` called twice | No-op on second call; `_task.done()` check guards cancellation |

---

## 16. Configuration Summary

| Variable | Default | Effect |
|----------|---------|--------|
| `MASSIVE_API_KEY` | (unset) | Set to use real market data; unset = GBM simulator |
| `poll_interval` | `15.0s` | Massive poll cadence; free tier max = 15s |
| `update_interval` | `0.5s` | Simulator step cadence |
| `event_probability` | `0.001` | Probability of shock event per tick per ticker |
| `LLM_MOCK` | `false` | Unrelated to market data; for chat endpoint testing |

### Simulator Behavior Reference

| Property | Value |
|----------|-------|
| Update frequency | 500ms (2 ticks/second) |
| Price precision | 2 decimal places |
| Minimum price | Bounded by `exp()` — approaches 0 asymptotically, never reaches it |
| Typical tick size | $0.001–$0.05 (varies with price and sigma) |
| Shock event frequency | ~1 event per 50 seconds with 10 tickers |
| Shock magnitude | 2–5%, random direction |
| TSLA intraday range | ~3% at 50% annualized sigma (most dramatic ticker) |
| V/JPM intraday range | ~0.8% at 17–18% annualized sigma (most stable) |
| Cholesky rebuild cost | O(n²) — negligible for n < 50 tickers |

---

*This document was generated from the planning files: `PLAN.md`, `MARKET_INTERFACE.md`, `MARKET_SIMULATOR.md`, `MASSIVE_API.md`, and `MARKET_DATA_SUMMARY.md`.*
