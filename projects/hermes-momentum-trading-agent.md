---
type: project
status: exploring
created: 2026-04-28
tags:
  - hermes
  - trading
  - alpaca
  - paper-trading
  - llm-agents
source: /Users/sebastien/Downloads/hermes-momentum-trader.md
---

# Hermes Momentum Trading Agent

## One-line brief

Build a paper-trading intraday momentum system where Hermes evaluates structured market snapshots against strict strategy specifications and emits TradePlan JSON, while Python/Alpaca owns all market data, risk validation, and execution. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

## Goal

Use Hermes Agent as the orchestration and reasoning layer for a momentum-based day trading experiment on Alpaca paper trading. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

The system should test rule-based intraday momentum setups such as breakout, first pullback, MA/VWAP pullback, tight consolidation burst, and relative-strength continuation. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

## Non-negotiable safety boundary

This is a paper-trading research project, not live trading. Hermes should never directly place orders or modify broker state. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

Risk management and order execution belong in deterministic Python code using Alpaca, not in the LLM. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

If any snapshot is ambiguous or missing required data, Hermes should return `NO_TRADE`. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

## Proposed architecture

1. Python market-data loop fetches Alpaca bars, quotes, account state, and positions. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
2. Python computes deterministic indicators: VWAP, 9 EMA, 20 EMA, RVOL, opening range, impulse leg, spread bps, daily context, and session context. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
3. Python builds a `MarketSnapshot` JSON payload. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
4. Hermes receives `account_equity`, one or more `StrategySpec` objects, and one `MarketSnapshot`. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
5. Hermes returns exactly one `TradePlan` JSON object: either `TRADE` or `NO_TRADE`. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
6. Python validates the TradePlan independently and submits paper orders only if all constraints pass. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
7. Python logs every snapshot, decision, order, fill, and outcome for analysis. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
8. A separate Trading Data Analyst skill analyzes historical logs and suggests parameter changes, but never trades. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

## Core data contracts

### StrategySpec

A `StrategySpec` defines allowed strategy behavior: name, description, direction, time window, instrument filters, setup conditions, entry rules, stop rules, targets, position sizing, and invalidations. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

Initial strategies:
- `IntradayMomentumBreakout` for breakouts above an opening or intraday range with volume confirmation. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
- `FirstMomentumPullback` for first pullbacks to 9/20 EMA or VWAP after a strong momentum leg. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

### MarketSnapshot

A `MarketSnapshot` should include symbol, timestamp, exchange timezone, price, bid/ask, spread bps, VWAP, EMA9, EMA20, opening range, impulse leg, RVOL, volume bar, average volume, daily context, and session info. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

### TradePlan

A `TradePlan` should include decision, strategy name, side, symbol, entry, stop, targets, size, and rationale. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

For `NO_TRADE`, Hermes should still explain the evaluated strategy and rejection reason in `rationale`. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

## Hermes skill design

### `trading/momentum-trader`

Purpose: evaluate one snapshot against predefined strategy specs and output exactly one TradePlan JSON object. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

Hard constraints:
- Use only provided strategies. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
- Do not invent strategies or parameters. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
- Do not change risk limits or time windows. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
- Do not trade if invalidation conditions are present. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
- Output only JSON. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

### `trading/data-analyst`

Purpose: analyze historical trades and price data, compute performance metrics, and propose conservative parameter suggestions without executing trades. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

Suggested metrics: total P&L in R, win rate, average win, average loss, expectancy, max drawdown, performance by strategy, symbol, time of day, and volatility regime. [Source: /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

## Key implementation risks

1. LLM nondeterminism: the trading decision should be constrained by JSON schemas, low temperature, strict prompts, and deterministic post-validation. [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
2. Strategy ambiguity: every rule needs a computable predicate; vague phrases like “strong trend” should become numerical fields in `MarketSnapshot` or `StrategySpec`. [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
3. Risk leakage: Python must recompute entry, stop, risk-per-share, max size, buying-power limits, position limits, and daily loss limits before any order. [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
4. Data quality: stale bars, wide spreads, partial candles, premarket/main-session confusion, and missing quotes should force `NO_TRADE`. [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
5. Overfitting: the data analyst should propose parameter changes only after enough sample size and should mark low-confidence results explicitly. [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
6. Compliance and safety: keep the system paper-only until there is a long verified track record and separate human approval. [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

## Open questions

- Which universe should the MVP trade: `SPY`/`QQQ`, large-cap watchlist, gapper scanner, or manually supplied symbols? [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
- Should Hermes be called once per symbol per bar, only when deterministic prefilters trigger, or only for shortlisted candidates? [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
- What Alpaca data plan and bar frequency will be used? [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
- What is the account-equity assumption for paper sizing? [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
- What hard account-level circuit breakers should Python enforce: max trades/day, max open positions, max daily loss, max symbol exposure, no-trade time near close? [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
- Where should logs live: SQLite, DuckDB, CSV, or Parquet? [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
- Should strategy specs live in Hermes skills, project config files, or both? [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

## Recommended MVP path

1. Build schema models for `StrategySpec`, `MarketSnapshot`, and `TradePlan`. [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
2. Implement deterministic strategy validators in Python first, before involving Hermes. [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
3. Create the Hermes `trading/momentum-trader` skill and test it against fixture snapshots. [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
4. Add Alpaca paper data ingestion and indicator computation. [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
5. Add the execution adapter with strict post-validation and paper-only enforcement. [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
6. Log every decision and build the `trading/data-analyst` reporting loop. [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]
7. Run shadow mode first, then paper orders, then evaluate. [Source: synthesis from /Users/sebastien/Downloads/hermes-momentum-trader.md, 2026-04-28]

## Related notes

- [[gbrain-live-sync-verification]]

## Raw source

Original research file: `/Users/sebastien/Downloads/hermes-momentum-trader.md`. [Source: User-provided local file, 2026-04-28]
