# Domain Models — Pydantic Data Contracts

> EISEN's domain layer contains pure Python models with no infrastructure dependencies.
> All models use Pydantic for runtime validation, serialization, and schema generation.

## Core Trading Models

```python
from decimal import Decimal
from typing import Optional, Dict
from pydantic import BaseModel
from datetime import datetime

class AccountBalance(BaseModel):
    """Exchange account balance for a single currency."""
    currency: str
    available: Decimal
    hold: Decimal

class Ticker(BaseModel):
    """Real-time price snapshot for a trading pair."""
    product_id: str
    price: Decimal
    timestamp: datetime

class OrderRequest(BaseModel):
    """Order placement request — market if limit_price is None."""
    product_id: str
    side: str  # "BUY" or "SELL"
    size: Decimal
    limit_price: Optional[Decimal] = None

class OrderExecutionSummary(BaseModel):
    """Actual execution telemetry from the exchange API."""
    order_id: str
    product_id: str
    side: str
    filled_size: Decimal
    filled_value: Decimal  # size * average filled price
    average_filled_price: Decimal
    total_fees: Decimal
    fee_currency: str
    status: str  # 'FILLED', 'OPEN', 'CANCELLED', etc.
    created_at: datetime
```

## Market Data Models

```python
class Candle(BaseModel):
    """OHLCV candlestick for a single time period."""
    product_id: str
    timestamp: datetime
    open: Decimal
    high: Decimal
    low: Decimal
    close: Decimal
    volume: Decimal
```

## Signal & Feature Models

```python
class SignalRecord(BaseModel):
    """Recorded trading signal with provenance."""
    timestamp: datetime
    product_id: str
    signal_type: str
    score: float
    reason: str
    strategy: str
    session_id: Optional[str] = None  # Pillar session tag
    metadata: Optional[Dict[str, str]] = None

class FeatureRow(BaseModel):
    """Point-in-time feature vector for a product."""
    product_id: str
    timestamp: datetime
    features: Dict[str, Optional[float]]
```

## Sentiment Models

```python
class SocialPost(BaseModel):
    """Social media post for sentiment analysis."""
    source: str
    timestamp: datetime
    text: str
    author_id: Optional[str] = None
    followers: Optional[int] = None
    engagement: Optional[int] = None
    url: Optional[str] = None

class SentimentScore(BaseModel):
    """Aggregated sentiment signal from a single source."""
    source: str
    timestamp: datetime
    score: float
    confidence: float
    magnitude: float
    weight: float = 1.0
```

## Design Decisions

### Why Pydantic?

- **Runtime validation**: Decimal precision for financial amounts, datetime enforcement for temporal ordering
- **Serialization**: Models serialize to/from JSON and SQLite without custom code
- **Schema generation**: Auto-generated OpenAPI schemas for dashboard API contracts
- **Immutability signals**: BaseModel fields encourage treating domain objects as values, not mutable state

### Why Decimal?

Financial calculations with `float` accumulate rounding errors. At 87 trades over 4 years, the cumulative error from float arithmetic would be small — but in a system that compares P&L to the cent for controlled experiments, Decimal precision eliminates a class of bugs entirely.

### Why Optional metadata?

The `SignalRecord.metadata` field is the extensibility point for experiment provenance. When running controlled tests (CT-xxx), metadata carries the hypothesis ID, config hash, and evaluation contract — making every signal traceable to the experiment that generated it.
