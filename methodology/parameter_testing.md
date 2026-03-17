# Parameter Testing Framework

## Core Principle

> Every parameter change must be treated as a scientific hypothesis.
> No change is accepted without out-of-sample evidence.

## Decision Framework

```
Hypothesis → Temporal Gate → Liveness Check → Sweep → 6 Analysis Gates → OOS → Accept/Reject
```

## The 6 Gates (Sequential — Stop at First Failure)

| Gate | Question | Fail Action |
|------|----------|------------|
| **1. Gradient** | Does the param produce ANY P&L variation? | Mark as DEAD, never sweep again |
| **2. Surface Shape** | Plateau (safe) vs spike (dangerous) vs bimodal (suspicious)? | If spike: reject (overfit) |
| **3. In-Sample** | Is improvement >2% over baseline? | Reject (noise) |
| **4. Bidirectional OOS** | Both forward AND reverse improve? | Reject (direction-dependent) |
| **5. Era Decomposition** | Gain spread across eras or concentrated? | Accept w/ risk if concentrated |
| **6. Factorial** | Multi-param changes — independent or interactive? | Interaction = reject unless justified |

## Temporal Gate (Required Before Gate 1)

Before any sweep or factorial, explicitly document and validate:

1. **Decision granularity** (`1m/5m/15m/1h/...`)
2. **Native cadence** of each enabled feature family
3. **Staleness budget** when external adjustments are enabled
4. **Bar-count constants** converted back to real time

Red flags that **block** testing:
- L2 signals at 1h → 99.97% information loss → **STOP**
- Daily features at 5m without lag → look-ahead bias → **STOP**
- 1.5 year window when 5 years available → weak validation → **WARN**

## Accept/Reject Table

| Evidence | Decision |
|----------|----------|
| Bidirectional OOS pass + 3+ eras | **STRONG ACCEPT** |
| Bidirectional OOS pass + concentrated era | **ACCEPT w/ risk** |
| Mixed OOS + net >20% improvement | **PROVISIONAL** |
| Mixed OOS + net <10% | **REJECT** |
| Net negative OOS | **REJECT** |

## A/B/C/D Sweep Design (For New Data Signals)

| Sweep | Config | Purpose |
|-------|--------|---------|
| **A** | Baseline (current champion) | Control |
| **B** | Baseline + new signal ON | Raw signal impact (must show >1% delta) |
| **C** | Pareto-optimized with signal | Best possible with signal |
| **D** | Sweep C winner, signal OFF | Ablation proof |

**Gate**: `|P&L(B) - P&L(A)| / P&L(A) > 0.01` before proceeding to C.

This design prevents the common trap of optimizing *around* a useless signal — if the raw signal (B) shows no effect, no amount of parameter tuning (C) can create real alpha.

## Tiered Testing Protocol (Mandatory for Sweeps >5 Configs)

### Phase 1: Quick Kill (1 window, ~3 min/config)
- Run **ONLY** on the hardest window for the hypothesis type:
  - Bear-protection → 2022 bear window
  - Bull-capture → 2020 or 2023 bull window
  - Regime-transition → 2021 sideways window
- **Kill threshold**: Underperforms baseline by >5pp → eliminate immediately

### Phase 2: Validation (3 windows, ~10 min/config)
- Run survivors on full eval-contract 3-window set
- **Kill threshold**: Composite score < champion - 10 → eliminate

### Phase 3: Walk-Forward OOS (6 windows, ~25 min/config)
- Yearly windows 2020–2025 with forward degradation check
- **Final gate**: 2025_YTD delta must be > -5pp

### Efficiency
- 80%+ of configs are eliminated in Phase 1
- 20-config Phase 1: ~60 min vs ~6 hours for full sweep
- False negative risk is low because we test on the HARDEST window first

## Crypto-Appropriate Ratio Optimization

| Priority | Metric | Rationale |
|----------|--------|-----------|
| **Primary** | Sortino(γ=2) | Penalizes only downside vol; rewards upside skew |
| **Secondary** | Omega(0) | Full-distribution probability-weighted gain/loss |
| **Tertiary** | Calmar | Return / max drawdown |
| **NEVER** | Sharpe | Penalizes upside vol which is *desirable* in crypto |

## Dead Parameter Management

Parameters confirmed to have zero gradient are permanently cataloged. 36+ parameters have been declared dead across 398+ controlled tests. Examples of zero-gradient mechanisms:

- Dynamic threshold variants (0/9 success rate)
- Volatility-target sizing (clips tail gains)
- Regime-conditional exit modifications (structurally negative)
- Monthly drawdown circuit breakers (kill upside capture)
- Partial sizing below macro EMA (bleeds via whipsaw)

Dead parameters are never re-swept unless the market microstructure fundamentally changes (e.g., the 2025 CHOP regime shift).
