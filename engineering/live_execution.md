# Live Execution Architecture

## Overview

EISEN executes real market orders on Coinbase Advanced Trade via the REST API.
All 3 pillars (P1 Treasury, P5 PAXG, P7 Whipsaw Recovery) run as independent
processes on a VPS, trading against the same Coinbase account simultaneously.

## Order Routing

```
Pillar Process → OrderRequest → CoinbaseAdapter.place_order()
                                        ↓
                              Coinbase REST API (market_market_ioc)
                                        ↓
                              Fill confirmation → log to SQLite
```

### Product ID Mapping

Live orders use USDC-denominated pairs because the account holds USDC, not USD
fiat. Ticker and candle data still uses USD pairs for deeper liquidity and wider
API support.

| Purpose | Pair |
|---------|------|
| P1 orders | BTC-USDC |
| P5 orders | PAXG-USDC |
| P7 orders | BTC-USDC |
| Ticker data | BTC-USD, PAXG-USD |
| Candle data | BTC-USD |

### Quote-Size BUY Orders

Market BUY orders use `quote_size` (USD amount to spend) instead of `base_size`
(asset quantity to buy). This solves two problems:

1. **Precision**: Different products have different `base_increment` values
   (BTC: 8 decimals, PAXG: 5 decimals). Specifying USD avoids
   `INVALID_SIZE_PRECISION` rejections.

2. **Allocation math**: The allocator computes dollar amounts per pillar. Using
   `quote_size` means the order matches the allocation exactly without
   price-dependent quantity conversion.

SELL orders still use `base_size` because the seller knows exactly how much
of the asset they hold.

```python
if side == "BUY":
    order_config = {"market_market_ioc": {"quote_size": str(usd_amount)}}
else:
    order_config = {"market_market_ioc": {"base_size": str(asset_quantity)}}
```

## Multi-Pillar Coordination

Three independent processes trade the same Coinbase account. Position sizing
conflicts are prevented through file-mediated dynamic allocation:

```
capital_rebalancer.py (writer)
        ↓
allocation_state.json (shared state file)
        ↓
allocation_reader.py → get_my_allocation() (reader, per-pillar)
```

Each pillar reads its allocated capital percentage and sizes orders as a
fraction of that allocation. The rebalancer runs on a separate loop,
adjusting weights based on regime detection (CVaR interpolation + CHOP shift).

### Allocation Example (Bull Regime, EISEN S7)

| Pillar | Allocation | Capital ($1,000 base) |
|--------|-----------|----------------------|
| P1 Treasury | 0% (IDLE) | $0 (redistributed) |
| P5 PAXG | 27.4% | $274 |
| P7 Whipsaw | 48.0% | $480 |
| Cash (FH13 legacy) | 24.6% | $246 |

When P1 is in IDLE state (not deployed), its allocation is redistributed to
other pillars and cash via the regime allocation tables.

### Staleness Guard

The allocation state file has a 10-minute staleness threshold. If the
rebalancer hasn't written an update in 10 minutes, pillars fall back to
default static weights rather than trading on stale regime data.

## Historical Prewarm

On restart, each pillar feeds historical candles through its indicator
pipeline before accepting live data:

- **P1**: 500 candles through `router.on_candle(candle, indicators_only=True)`
- **P5**: Price history for momentum and vol calculation
- **P7**: 200 1h candles + 150 5m candles for peak/crash detection

The `indicators_only=True` path updates all technical indicators, the
predictive top detector, and the regime shift predictor without generating
any trade signals. This means:

- RSI, ATR, EMA, Bollinger Bands are immediately valid
- The predictive top detector has 500 observations (well above its 50-bar minimum)
- No phantom signals fire during warmup
- The system is trade-ready on the first live candle

## Error Handling

The `CoinbaseAdapter` handles three response formats from the Coinbase SDK:

```python
response = client.create_order(...)

if response.get("success"):
    return response["success_response"]["order_id"]
elif "failure_response" in response:
    raise OrderError(response["failure_response"])
elif "error_response" in response:
    raise OrderError(response["error_response"])
elif response.get("success") is False:
    raise OrderError(response.get("error_response") or response)
```

Order failures are logged but don't crash the pillar process. The next tick
will re-evaluate market conditions and may retry if the signal persists.

## Fee Model

Coinbase One membership with 25% rebate:

| Market | Taker (net) | Maker (net) |
|--------|------------|------------|
| Spot | 0.015% | 0.000% |
| Derivatives | 0.0075% | 0.00375% |

All orders are market IOC (immediate-or-cancel), so only taker fees apply.
The fee model is baked into backtest simulations to ensure forward-test
results are comparable.
