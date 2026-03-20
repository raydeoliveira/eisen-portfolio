# Capital Event Tracking — Deposit-Aware Equity

## Problem

A live trading system must handle deposits and withdrawals without corrupting performance metrics. When a user deposits $500 into a $500 portfolio, the equity doubles — but PnL should stay the same. Ratio calculations (Sortino, Calmar, Sharpe) must be computed against the **cost basis**, not the raw equity.

Most retail trading dashboards get this wrong. They either:
1. Treat deposits as PnL (inflating returns)
2. Require manual recalibration of the starting capital
3. Reset all metrics on each capital change

## Design

EISEN tracks capital flows as first-class events in the persistence layer:

```sql
CREATE TABLE capital_events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TEXT NOT NULL,
    event_type TEXT NOT NULL,  -- 'deposit' | 'withdrawal' | 'adjustment'
    amount REAL NOT NULL,      -- positive for deposits, negative for withdrawals
    note TEXT,
    metadata TEXT              -- JSON: pillar target, tx_id, etc.
);
```

### Equity Computation

```
capital_base = initial_seed + Σ(capital_events.amount)
portfolio_pnl = current_equity - capital_base
pnl_pct = portfolio_pnl / capital_base × 100
```

The `capital_base` is the denominator for all ratio calculations. This ensures:

| Scenario | Equity | Capital Base | PnL | PnL % |
|----------|--------|-------------|-----|-------|
| Start: $500 seed | $500 | $500 | $0 | 0% |
| Deposit $500 | $1,000 | $1,000 | $0 | 0% |
| Market gains $50 | $1,050 | $1,000 | $50 | 5% |
| Withdraw $200 | $850 | $800 | $50 | 6.25% |

### Integration Points

```
capital_events table (SQLite)
         ↓
get_net_capital_flow() — sums all events
         ↓
capital_base = starting_equity + net_flow
         ↓
compute_live_weighted_portfolio_snapshot()
  - Uses capital_base for weighted allocation math
  - PnL = portfolio_equity - capital_base (not raw starting_equity)
         ↓
Dashboard /api/pillars endpoint
  - Returns: portfolio_equity, portfolio_pnl, capital_base,
             net_capital_flow, total_deposits, total_withdrawals
```

### CLI Interface

```bash
# Record events
python3.11 scripts/record_capital_event.py deposit 500 --note "Scale-up funding"
python3.11 scripts/record_capital_event.py withdrawal 200 --note "Partial cash-out"
python3.11 scripts/record_capital_event.py adjustment 50 --note "Fee rebate"

# View history
python3.11 scripts/record_capital_event.py list
python3.11 scripts/record_capital_event.py summary
```

## Dynamic Reallocation Events

Beyond user-initiated deposits and withdrawals, the allocator generates
internal capital reallocation events:

- **Idle-realloc (H308)**: When P7 Whipsaw is dormant (no crash-recovery
  opportunity) for 6+ bars, 65% of its allocated capital redirects to
  P5 PAXG as a safe-haven. When P7 re-activates, capital flows back.
  These internal shifts are tracked as allocation state changes, not as
  capital events, preserving the invariant that `capital_base` only
  reflects external flows.

- **IDLE event overrides (H309)**: During scheduled macro events (FOMC,
  NFP, CPI, OPEX, etc.), the allocator applies regime-specific weight
  adjustments. 8 event overrides for the IDLE regime shift capital
  defensively before known volatility windows.

## Key Invariants

1. **Deposits never inflate PnL**: A $500 deposit increases both equity and capital_base by $500 — PnL is unchanged
2. **Withdrawals don't create phantom losses**: A $200 withdrawal decreases both equity and capital_base
3. **Ratio denominators use capital_base**: Sortino, Calmar, drawdown calculations use the deposit-adjusted base
4. **Deterministic reconstruction**: Portfolio state can be fully reconstructed from `capital_events + trade signals`
5. **Pillar-agnostic**: Capital events apply at the portfolio level, not per-pillar (the allocator distributes across pillars)

## Why Not Auto-Detect Deposits?

Exchange APIs could theoretically detect deposits by watching balance changes that don't correspond to trades. EISEN uses manual recording instead because:

- **Reliability**: Exchange API balance queries can fail, return stale data, or miss pending transactions
- **Auditability**: Manual events create an explicit audit trail with timestamps and notes
- **Control**: The user decides when a capital change is "official" — pending transfers, fee rebates, and margin calls all have different semantics
- **Simplicity**: Polling balance changes introduces race conditions with concurrent trade execution
