# Adversarial Reasoning Framework

## When This Applies

This framework applies to any work involving signals, sentiment, strategy parameters, risk thresholds, or position sizing — anything where market counterparties can adapt to or exploit behavior.

## The Pluribus Principle

> "Regardless of which hand Pluribus is actually holding, it will first calculate how it would act with every possible hand, being careful to balance its strategy across all the hands so as to remain unpredictable."

Every strategy decision is evaluated not just by expected outcome, but by **exploitability** — is this move robust once the other side starts adapting?

## Theory of Mind Checklist

Before integrating any signal:

1. **Who sees this?** — Is this signal public (social media), semi-private (derivatives data), or private (on-chain whale tracking)?
2. **What are they optimizing?** — Retail (FOMO/fear), whales (distribution/accumulation), market makers (spread capture), quant funds (alpha extraction)
3. **How does acting update their model?** — Does our trade contribute to a detectable pattern?
4. **Hidden state?** — What can they see that we can't? Whale wallets, OTC desk flow, unreported positions, coordinated group chats
5. **Pluribus test** — Would a balanced strategy (sometimes follow, sometimes fade the signal) outperform always-follow? If yes, the signal is exploitable

## Signal Classification

### Chess-like Signals (Low adversarial risk)
- Technical indicators from public OHLCV — edge comes from computation quality, not information advantage
- Volatility models (ATR, GARCH) — reactive, not predictive
- Calendar effects (halving cycle, day-of-week) — structural, not adaptive

### Poker-like Signals (High adversarial risk — require audit)
- Social sentiment (can be manufactured; who benefits from this narrative?)
- Influencer predictions (track record? survivorship bias? paid promotion?)
- Funding rate extremes (crowded trades attract liquidation hunters)
- Volume spikes (organic demand or wash trading / spoofing?)
- On-chain signals (whale movement — real or decoy wallet?)

## Deception Detection Heuristics

| Pattern | Likely Interpretation |
|---------|----------------------|
| Bullish sentiment + bullish flow | Genuine move |
| Bullish sentiment + flat/bearish flow | **Narrative trap** (distribution) |
| Bearish sentiment + bullish flow | Smart money accumulation |
| Bearish sentiment + bearish flow | Genuine sell-off |
| Extreme sentiment unanimity (>85%) | Crowded trade, alpha near zero |
| Social volume spike + no price move | Manufactured noise |
| Sudden influencer consensus | Coordinated campaign, not organic |

## Crowdedness Penalty

```
weight_adjusted = base_weight × (1 - crowdedness_penalty)
crowdedness_penalty = min(1.0, social_volume_zscore / 5.0)
```

When a signal becomes consensus, its edge approaches zero. Adjust fusion weights accordingly — or generate contrarian signal.

## Adversarial Anti-Patterns (Never Do These)

- Take social signals at face value without checking flow confirmation
- Assume constant alpha from any sentiment-derived signal
- Use static source credibility weights during volatile events
- Treat sentiment unanimity as confidence (it's the opposite)
- Ignore temporal clustering of social posts (coordination signature)
- Trust a signal more because it "sounds right" to the LLM

## Recursive Self-Check Protocol

Before handing off signal/strategy work to another agent:

1. Would the **Adversary** flag this as exploitable? → Address it now
2. Would the **Validator** reject this for missing OOS evidence? → Add it now
3. Would the **Researcher** question the statistical rigor? → Strengthen it now
4. Would the **Orchestrator** ask why this wasn't caught earlier? → Document it now

This collapses multi-agent iteration loops into a single pass — catching errors that would otherwise require 3-4 review cycles.

## Application to EISEN

This framework led to several concrete architectural decisions:

- **Social features disabled by default**: Until 6+ months of live data accumulates, social signals cannot be backtested. Using them without historical validation violates the OOS requirement.
- **Funding rate extremes treated as warning, not signal**: Crowded funding trades attract liquidation cascades — following them is playing against market makers.
- **No LLM-generated sentiment scores in production**: LLMs are susceptible to narrative plausibility. A bearish story that "sounds right" is exactly how distribution traps work.
- **Calendar effects used structurally, not predictively**: The EU Open (09:00–11:00 UTC) noise pattern is a market microstructure feature, not an exploitable signal.
