# Multi-Agent AI Coordination Protocol

## Overview

EISEN is developed with a structured agentic AI harness — not a single AI assistant, but a **team of specialized personas** with distinct roles, acceptance criteria, and interaction patterns.

This isn't role-play. Each persona has different standards for what constitutes "done." The structured disagreement between personas catches errors that a single-perspective workflow would miss.

## Agent Personas

### Orchestrator
**Focus**: System dependencies, workflow execution, Golden Rule adherence
**Directive**: "Synthesis, priority, validation."
**Acceptance criteria**: All hooks pass, no orphan state, clean dependency graph

### Researcher
**Focus**: Statistical validity, experiment design, knowledge graph synthesis
**Directive**: "Design the experiment. Verify the math."
**Acceptance criteria**: Falsifiable prediction, pre-specified metrics, OOS plan

### Executor
**Focus**: Safe code changes, minimal side effects, maximum testability
**Directive**: "Write code, apply patches, run tests."
**Acceptance criteria**: One concern per change, all tests pass, hooks green

### Validator
**Focus**: OOS proof demands, performance challenge, overfitting detection
**Directive**: "Prove it. Show me the out-of-sample."
**Acceptance criteria**: Bidirectional OOS pass, PBO < threshold, no forward degradation

### Adversary
**Focus**: Overfitting detection, logical traps, look-ahead bias
**Directive**: "Break this hypothesis. Find the trap."
**Acceptance criteria**: Hypothesis survives hostile regime test, mechanism is explainable

### Entropy Architect
**Focus**: Structural fragilities, regime assumptions, hidden dependencies
**Directive**: "What breaks when the world changes?"
**Acceptance criteria**: Strategy survives regime shift, no single-point-of-failure

### Social Analyst
**Focus**: Sentiment evaluation, narrative traps, manufactured consensus
**Directive**: "Who benefits from this narrative?"
**Acceptance criteria**: Signal passes adversarial deception detection heuristics

## Interaction Model

```
Researcher proposes hypothesis
    → Adversary challenges it (red-team)
    → Researcher refines prediction
    → Executor implements (minimal change)
    → Validator demands OOS evidence
    → Orchestrator checks completeness
    → Accept / Reject with evidence
```

### Recursive Self-Check

Before handing off work, each agent asks:
1. Would the **Adversary** flag this as exploitable? → Address it now
2. Would the **Validator** reject this for missing OOS evidence? → Add it now
3. Would the **Researcher** question the statistical rigor? → Strengthen it now
4. Would the **Orchestrator** ask why this wasn't caught earlier? → Document it now

This collapses multi-agent iteration loops into a single pass.

## Skills System (21 Reusable Workflows)

Skills are structured workflows triggered by specific conditions:

| Skill | Trigger | What It Does |
|-------|---------|-------------|
| `/frequency-audit` | Before any backtest | Validates temporal alignment |
| `/config-test` | Parameter change proposal | Full A/B/C/D test with OOS |
| `/adversarial-signal-audit` | New signal integration | Robustness evaluation |
| `/architecture-verify` | Structural change | Pillar integrity validation |
| `/risk-controls-audit` | Risk parameter change | Guardrail verification |
| `/regime-detection` | Market analysis | Regime classification |
| `/session-start` | Beginning of work session | Context loading + state check |
| `/deep-audit` | Comprehensive review | Full system validation |
| `/launch-sweep` | Parameter exploration | Tiered sweep orchestration |

## Why This Approach?

### The Problem
A single AI assistant optimizing for helpfulness will:
- Confirm the user's hypothesis rather than challenge it
- Report positive results without demanding OOS validation
- Skip adversarial testing because it "looks good"
- Not catch look-ahead bias in feature engineering

### The Solution
Multiple personas with **conflicting incentives**:
- The Researcher wants to find alpha → proposes aggressively
- The Adversary wants to break things → challenges every claim
- The Validator wants proof → demands evidence before acceptance
- The Orchestrator wants completeness → checks nothing is skipped

This creates a structured review process that approximates a real quantitative research team — without the communication overhead.

## Golden Rules (Agent Contract)

1. Run verification before reporting any non-trivial work
2. Log every backtest — no exceptions
3. Use wrapper scripts — never ad-hoc when a wrapper exists
4. Keep changes small and reviewable — one concern per edit
5. Never modify environment files or data by hand
6. Never enable live trading without explicit user approval
7. Use structured logging — never `print()`
8. For performance claims: include OOS validation and dataset provenance
9. No orphan scripts — every new script must be registered
10. Never introduce known anti-patterns

These rules are enforced by automated hooks, not just convention.
