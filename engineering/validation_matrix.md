# Validation Matrix — What to Validate and When

## Primary Verification Loop

```bash
bash scripts/agent_verify.sh            # Standard: lint + type-check + tests + data status
bash scripts/agent_verify.sh --overfit   # + overfitting detection
bash scripts/agent_verify.sh --risk      # + risk controls audit
```

## When to Run What

| Change Type | Validation Required |
|---|---|
| Any code change | Standard verify (lint, type-check, tests) |
| Performance claim | Standard + overfitting check |
| Risk/execution change | Standard + risk audit |
| New dataset | Data audit (provenance, coverage, gaps) |
| Strategy optimization | Walk-forward + overfitting check |
| Instruction/skill edit | Instruction hygiene (stale references) |
| Config change | Pillar integrity + architecture consistency |
| New feature in router | Verify feature fires from BOTH evaluation paths |

## Temporal Hygiene Checklist (Before Any Backtest)

- [ ] What granularity am I testing? (1m/5m/15m/30m/1h/1d)
- [ ] What signals does this strategy use?
- [ ] What is each signal's native update frequency?
- [ ] Is information loss <50% for ALL signals?
- [ ] Do daily/8h signals have lag parameter? (prevent look-ahead)
- [ ] Are cooldowns scaled to granularity?
- [ ] Is data coverage >95% for this granularity?
- [ ] Have I tested on MULTIPLE windows? (prevent window overfitting)

### Red Flags (Blocking)

| Situation | Information Loss | Action |
|-----------|-----------------|--------|
| L2 signals at 1h | 99.97% | **STOP** |
| Daily features at 5m without lag | Look-ahead bias | **STOP** |
| 1.5 year window when 5 years available | Weak validation | **WARN** |
| Only one granularity tested | Window overfitting risk | **WARN** |

## Acceptance Criteria (All Must Pass)

1. All `ruff` lint checks pass
2. All `mypy` type checks pass
3. All `pytest` tests pass
4. No stale instruction references (hygiene check)
5. Temporal hygiene checks pass (frequency alignment validated)

## Automated Hooks

14 post-edit hooks auto-fire to catch problems before they compound:

| Hook | Trigger | Runtime |
|------|---------|---------|
| `post-code-edit` | Any `src/**/*.py` change | ~15s |
| `post-config-change` | Any `config/*.json` change | ~5-20s |
| `post-router-refactor` | Router module changes | ~30-90s |
| `time-window-hygiene` | Before any backtest | ~5-15s |
| `pillar-lockstep` | Deploy or verify | ~1s |
| `post-research-audit` | Hypothesis updates | ~5s |
| `post-script-create` | New script created | ~2s |
| `post-test-failure` | pytest failure | ~10s |
| `file-size-gate` | File size check | ~2s |
| `post-deployment` | After VPS deploy | ~10s |

Hooks are consolidated in a single task runner script for maintainability. No hook is optional — skipping hooks is a process violation.

## Overfitting Detection

Three automated safeguards against the #1 failure mode in quantitative trading:

1. **Probability of Backtest Overfitting (PBO)**: Combinatorial symmetric cross-validation — measures the probability that the "best" strategy is actually the result of noise optimization
2. **Deflated Sharpe Ratio**: Adjusts for multiple testing and non-normal returns — the more strategies you test, the higher the bar for significance
3. **IS→OOS Decay Tracking**: Automatic flagging when in-sample to out-of-sample performance degrades beyond thresholds — the gap between what you optimized and what you'd actually get
