# Port Interfaces — Clean Architecture Boundaries

> These are the abstract base classes that enforce EISEN's core architectural invariant:
> **Trading logic never imports concrete exchange code.**

## Execution Port

The `ExecutionPort` defines the contract for any trading execution adapter — Coinbase, Binance, paper trading, or backtesting. Strategy code only sees this interface.

```python
from abc import ABC, abstractmethod
from typing import List
from src.domain.models import AccountBalance, Ticker, OrderRequest, OrderExecutionSummary

class ExecutionPort(ABC):
    """
    Interface for any trading execution adapter (Coinbase, Binance, PaperTrading).
    """

    @abstractmethod
    def get_balances(self) -> List[AccountBalance]:
        """Fetch all non-zero balances."""
        pass

    @abstractmethod
    def get_ticker(self, product_id: str) -> Ticker:
        """Fetch latest price data."""
        pass

    @abstractmethod
    def place_order(self, order: OrderRequest) -> str:
        """Place an order and return the Order ID."""
        pass

    @abstractmethod
    def get_fill_summary(self, order_id: str, product_id: str) -> OrderExecutionSummary:
        """Fetch post-trade execution details including actual fees."""
        pass
```

## Market Data Port

The `MarketDataPort` abstracts historical OHLCV data sources — production API, CSV files, or cached SQLite.

```python
from abc import ABC, abstractmethod
from datetime import datetime
from typing import List
from src.domain.models import Candle

class MarketDataPort(ABC):
    """Interface for historical OHLCV data sources."""

    @abstractmethod
    def fetch_candles(
        self, product_id: str, start: datetime, end: datetime, granularity: str
    ) -> List[Candle]:
        """
        Return candles sorted in ascending timestamp order.

        Args:
            product_id: Standard product identifier (e.g., BTC-USD).
            start: Inclusive start time in UTC.
            end: Exclusive end time in UTC.
            granularity: Standard timeframe (e.g., 1m, 5m, 1h, 1d).
        """
        raise NotImplementedError
```

## Why This Matters

In a system where a misrouted order can lose real money:

1. **Adapter swappability**: Switch between paper trading, backtesting, and live execution by swapping a single adapter — zero strategy code changes
2. **Testability**: The entire strategy engine can be tested against mock adapters with deterministic market data
3. **Risk isolation**: A bug in the Coinbase WebSocket adapter cannot corrupt the strategy state machine
4. **Deployment flexibility**: The same strategy code runs on local dev, CI, and production VPS without modification

### Concrete Adapters (private — not shown)

| Adapter | Implements | Purpose |
|---------|-----------|---------|
| `CoinbaseAdapter` | `ExecutionPort` | Live trading via Coinbase Advanced Trade API |
| `PaperAdapter` | `ExecutionPort` | Simulated execution with realistic fee modeling |
| `BacktestAdapter` | `ExecutionPort` | Deterministic replay for controlled experiments |
| `CoinbaseMarketData` | `MarketDataPort` | REST API historical candles |
| `CsvMarketData` | `MarketDataPort` | Offline datasets (Kaggle, exports) |
| `SqliteMarketData` | `MarketDataPort` | Cached local data for fast iteration |
