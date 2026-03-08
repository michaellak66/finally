# Market Simulator Design

Approach and code structure for simulating realistic stock prices when no Massive API key is configured. The simulator is the default data source in FinAlly.

## Overview

The simulator uses **Geometric Brownian Motion (GBM)** to generate realistic stock price paths. GBM is the standard continuous-time model underlying Black-Scholes option pricing: prices evolve with random noise, can never go negative, and produce the lognormal distribution observed in real equity markets.

Key properties:
- Updates every 500ms — prices feel alive and responsive
- Correlated moves across tickers (sector-based correlation matrix)
- Per-ticker volatility calibrated to approximate real-world behavior
- Occasional random "shock events" for visual drama
- Zero external dependencies — runs entirely in-process

---

## Files

```
backend/app/market/
  simulator.py       # GBMSimulator class + SimulatorDataSource (async wrapper)
  seed_prices.py     # SEED_PRICES, TICKER_PARAMS, correlation constants
```

---

## GBM Mathematics

At each time step, a stock price evolves as:

```
S(t + dt) = S(t) * exp((mu - sigma^2/2) * dt + sigma * sqrt(dt) * Z)
```

| Symbol | Meaning |
|--------|---------|
| `S(t)` | Current price |
| `mu` | Annualized drift (expected return), e.g. 0.05 = 5% |
| `sigma` | Annualized volatility, e.g. 0.20 = 20% |
| `dt` | Time step as a fraction of a trading year |
| `Z` | Standard normal random variable drawn from N(0,1) |

**Why the `exp()` form?** Multiplicative updates ensure prices can never reach zero or go negative. The `exp()` is the exact solution to the GBM stochastic differential equation.

**Time step calibration**: 500ms expressed as a fraction of a trading year:

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800 seconds
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ~8.48e-8
```

This tiny `dt` produces sub-cent moves per tick (typically $0.001–$0.05 depending on sigma), which accumulate naturally into realistic intraday ranges over a session.

---

## Correlated Moves

Real stocks don't move independently — tech stocks tend to move together, financials have their own correlations, etc. The simulator captures this using **Cholesky decomposition** of a correlation matrix.

### How it works

Given `n` tickers with a correlation matrix `C`:

1. Compute the lower-triangular Cholesky factor: `L = cholesky(C)`
2. At each step, draw `n` independent standard normals: `Z_independent ~ N(0, I_n)`
3. Produce correlated draws: `Z_correlated = L @ Z_independent`
4. Use `Z_correlated[i]` as the Z value for ticker `i` in the GBM formula

The resulting draws have exactly the covariance structure defined by `C`.

### Correlation structure

Defined in `seed_prices.py` as constants:

```python
CORRELATION_GROUPS = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

INTRA_TECH_CORR    = 0.6   # Tech stocks move together strongly
INTRA_FINANCE_CORR = 0.5   # Finance stocks move together
CROSS_GROUP_CORR   = 0.3   # Across sectors or unknown tickers
TSLA_CORR          = 0.3   # TSLA is in tech but does its own thing
```

Pairwise correlation lookup:

```python
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

    return CROSS_GROUP_CORR     # Cross-sector or unknown
```

The Cholesky matrix is rebuilt whenever tickers are added or removed. This is O(n²) but with n < 50 tickers it is negligible.

---

## Random Shock Events

Every step, each ticker has a small probability of a sudden large move — a "news event":

```python
EVENT_PROBABILITY = 0.001   # 0.1% per tick per ticker

if random.random() < self._event_prob:
    shock_magnitude = random.uniform(0.02, 0.05)   # 2–5% move
    shock_sign = random.choice([-1, 1])             # up or down
    self._prices[ticker] *= 1 + shock_magnitude * shock_sign
```

With 10 tickers at 2 ticks/second:
- Expected events per second: `10 * 2 * 0.001 = 0.02`
- Expected interval between events: ~50 seconds

This keeps the dashboard visually active without overwhelming the user.

---

## Seed Prices and Per-Ticker Parameters

Defined in `seed_prices.py`. These calibrate the simulator to feel realistic.

### Starting Prices

```python
SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00,
    "GOOGL": 175.00,
    "MSFT": 420.00,
    "AMZN": 185.00,
    "TSLA": 250.00,
    "NVDA": 800.00,
    "META": 500.00,
    "JPM":  195.00,
    "V":    280.00,
    "NFLX": 600.00,
}
```

Tickers added dynamically that are not in `SEED_PRICES` start at a random price between $50 and $300.

### Volatility and Drift

```python
TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},  # High volatility
    "NVDA":  {"sigma": 0.40, "mu": 0.08},  # High volatility, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},  # Low volatility (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},  # Low volatility (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}
```

`sigma` (volatility) drives intraday range:
- Low (`0.17–0.22`): stable stocks like V, AAPL, MSFT
- Medium (`0.25–0.35`): GOOGL, AMZN, META, NFLX
- High (`0.40–0.50`): NVDA, TSLA — dramatically larger moves

`mu` (drift) is the long-run expected annual return. With the tiny `dt`, its effect per tick is imperceptible but it subtly biases the random walk upward over long sessions.

---

## `GBMSimulator` Class

Defined in `simulator.py`. Pure synchronous Python — no asyncio here.

```python
import math
import random
import numpy as np
from .seed_prices import (SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS,
                           CORRELATION_GROUPS, INTRA_TECH_CORR,
                           INTRA_FINANCE_CORR, CROSS_GROUP_CORR, TSLA_CORR)

class GBMSimulator:
    """Generates correlated GBM price paths for multiple tickers."""

    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600   # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR    # ~8.48e-8

    def __init__(self, tickers: list[str], dt: float = DEFAULT_DT,
                 event_probability: float = 0.001) -> None:
        self._dt = dt
        self._event_prob = event_probability
        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None

        for ticker in tickers:
            self._add_ticker_internal(ticker)
        self._rebuild_cholesky()   # Build once after batch init (more efficient)

    def step(self) -> dict[str, float]:
        """Advance all tickers by one time step. Returns {ticker: new_price}.

        Hot path — called every 500ms.
        """
        n = len(self._tickers)
        if n == 0:
            return {}

        z_independent = np.random.standard_normal(n)
        z = self._cholesky @ z_independent if self._cholesky is not None else z_independent

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            mu = self._params[ticker]["mu"]
            sigma = self._params[ticker]["sigma"]

            drift     = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            if random.random() < self._event_prob:
                shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
                self._prices[ticker] *= (1 + shock)

            result[ticker] = round(self._prices[ticker], 2)

        return result

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

---

## `SimulatorDataSource` Class

Also in `simulator.py`. Wraps `GBMSimulator` in an asyncio background task, implementing the `MarketDataSource` interface.

```python
import asyncio
import logging
from .cache import PriceCache
from .interface import MarketDataSource

class SimulatorDataSource(MarketDataSource):
    """Runs GBMSimulator in an asyncio background task.

    Calls step() every update_interval seconds and writes to PriceCache.
    """

    def __init__(self, price_cache: PriceCache, update_interval: float = 0.5,
                 event_probability: float = 0.001) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        # Seed cache immediately — SSE clients get data before the first step()
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
        """Core loop: step, write to cache, sleep."""
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

Note the exception guard in `_run_loop`: a numpy failure or other unexpected error is logged but the loop continues. The simulator is designed to never crash the app.

---

## Behavior Notes

| Property | Value |
|----------|-------|
| Update interval | 500ms (2 updates/second) |
| Price resolution | Rounded to 2 decimal places |
| Minimum price | Bounded below by GBM math (`exp()` > 0 always) |
| Typical tick size | $0.001–$0.05 (depends on sigma and current price) |
| Shock event frequency | ~0.1% per tick per ticker = ~1 event/50 seconds with 10 tickers |
| Shock magnitude | 2–5% instantaneous move, random direction |
| Cholesky rebuild cost | O(n²), negligible for n < 50 |

- Prices can drift significantly over long sessions — a 50% annualized sigma (TSLA) produces roughly 3% intraday range, which accumulates if the session runs for hours
- The correlation structure means AAPL and MSFT will visibly move together; TSLA will regularly diverge
- Unknown tickers added dynamically use `DEFAULT_PARAMS` (sigma=0.25) and start between $50–$300
- `_add_ticker_internal()` is separate from `add_ticker()` to allow batch initialization without redundant Cholesky rebuilds

---

## Demo

A Rich terminal dashboard is available for testing the simulator visually:

```bash
cd backend
uv run market_data_demo.py
```

Displays all 10 tickers with live prices, sparklines, color-coded direction arrows, and a log of shock events. Runs 60 seconds or until Ctrl+C.
