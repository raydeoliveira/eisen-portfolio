# Engineering Standards

## Architecture Guardrails

- **Boundary enforcement**: Strict Ports & Adapters. Trading logic NEVER imports concrete exchange code.
- **Composition over frameworks**: Prefer small service classes over monolithic frameworks.
- **Persistence**: Favor local SQLite for reproducibility. All timestamps UTC.
- **Transparency**: Public methods must have docstrings explaining intent.

## Strategy Specification

- **Precision**: Define strict entry/exit triggers (e.g., "RSI < 30 AND Close > EMA", not "oversold")
- **Reset**: Every strategy must have a clear "do nothing" state
- **Time**: Explicitly document timeframes and session windows

## Temporal Hygiene Contract

- Always verify feature cadence vs decision cadence before experiments
- Treat mismatched update windows as **adversarial risk**, not a benign warning
- Any hypothesis using mixed-frequency features must document:
  - Decision bar (`granularity`), analysis window (`start/end`)
  - Feature freshness assumptions
  - Why slower features are still causal/useful at that bar size
- No performance claim without explicit note that temporal hygiene passed

## Risk & Capital

- Enforce max position size and daily loss limits (hard kill switch)
- Dynamic sizing must degrade gracefully — never to zero allocation in a recoverable regime
- Reserve at least 2% cash for market-order slippage
- Never enable live trading without explicit approval

## Code Hygiene

### Change Rules
- One concern per commit. Combine only if mechanically inseparable.
- New files require `__init__.py` update if inside a package.
- Deleted files require grep for stale imports.
- Config changes require pillar integrity check.

### Anti-Pattern Registry

Known anti-patterns that are **blocked on review**:

| Anti-Pattern | Why It's Bad | Correct Pattern |
|-------------|-------------|-----------------|
| `list.pop(0)` for ring buffers | O(n) per pop, silent perf degradation | `collections.deque(maxlen=N)` |
| Naive `datetime.now()` | Timezone-unaware, breaks temporal alignment | `datetime.now(timezone.utc)` |
| Missing `is not None` guards | `if x:` fails for `x=0` or `x=0.0` | `if x is not None:` |
| `print()` for logging | No level control, no structured output | `src.services.logger` |
| Hardcoded paths | Breaks across dev/CI/VPS | `pathlib.Path` with config roots |

### Feature Enablement Protocol

When enabling a new router feature:

1. **Verify dual-path coverage**: Feature must fire from BOTH predictive-hold AND signal-generation paths
2. **Temporal hygiene**: Check feature cadence vs decision granularity
3. **Ablation test**: Run with feature ON vs OFF, verify >1% delta
4. **OOS validation**: Bidirectional walk-forward must pass
5. **Factorial check**: No negative interaction with existing features

## Testing Standards

- 128 test modules covering domain models, port contracts, adapter behavior, strategy logic
- `ruff` for linting, `mypy` for type checking, `pytest` for unit/integration tests
- All three must pass before any code change is reported as complete
- Backtest results logged to `data/backtests/` — no exceptions

## Logging

- Use `src.services.logger` — never `print()`
- Structured logging with level control (DEBUG, INFO, WARNING, ERROR)
- Production logs written to `logs/` directory with rotation
- Backtest logs include config hash for reproducibility

## Configuration Management

- All strategy configs are JSON presets in `config/`
- Pydantic validates configs at load time — no silent misconfigs
- Config metadata fields (`_comment`, `_hypothesis`, `_supersedes`, `_granularity`) are documentation, NOT router parameters
- Config count >30 triggers archive cleanup
