# Hermes Momentum Trading Agent Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task once the user confirms a target repository.

**Goal:** Build a paper-only Alpaca intraday momentum trading system where Hermes emits constrained TradePlan JSON and deterministic Python code owns data, validation, risk, execution, and logging.

**Architecture:** Start with an offline, test-driven core library for schemas, strategy validation, and fixtures. Add a Hermes skill wrapper only after the deterministic core is proven. Integrate Alpaca paper trading last, behind strict post-validation and paper-only guards.

**Tech Stack:** Python 3.11+, Pydantic, pytest, Alpaca Python SDK, pandas/polars, SQLite or DuckDB for logs, Hermes skills.

---

## Phase 0: Decisions required before coding

- Choose target repo path. [Source: synthesis from [[hermes-momentum-trading-agent]], 2026-04-28]
- Choose MVP symbol universe: `SPY`/`QQQ`, a fixed large-cap list, or externally supplied symbols. [Source: synthesis from [[hermes-momentum-trading-agent]], 2026-04-28]
- Choose log storage: SQLite, DuckDB, CSV, or Parquet. [Source: synthesis from [[hermes-momentum-trading-agent]], 2026-04-28]
- Confirm Alpaca credentials are for paper trading only. [Source: synthesis from [[hermes-momentum-trading-agent]], 2026-04-28]

## Phase 1: Core schemas

### Task 1: Create project skeleton

**Objective:** Create a minimal Python package with tests.

**Files:**
- Create: `pyproject.toml`
- Create: `src/hermes_momentum_trader/__init__.py`
- Create: `tests/__init__.py`

**Verification:**
- Run `python -m pytest`.
- Expected: test discovery works, even if zero tests initially.

### Task 2: Define Pydantic models

**Objective:** Add strict models for `StrategySpec`, `MarketSnapshot`, and `TradePlan`.

**Files:**
- Create: `src/hermes_momentum_trader/models.py`
- Create: `tests/test_models.py`

**Required behavior:**
- Invalid `decision` values fail validation.
- Missing required market fields fail validation.
- `NO_TRADE` supports a rationale.
- `TRADE` requires entry, stop, targets, size, and symbol.

**Verification:**
- Run `python -m pytest tests/test_models.py -v`.

## Phase 2: Deterministic strategy engine

### Task 3: Add breakout strategy evaluator

**Objective:** Implement deterministic evaluation for `IntradayMomentumBreakout`.

**Files:**
- Create: `src/hermes_momentum_trader/strategies.py`
- Create: `tests/test_breakout_strategy.py`

**Required behavior:**
- Accept only snapshots inside time window.
- Reject symbols outside instrument filters.
- Require price above VWAP and rising EMAs when trend context requires it.
- Require 5-minute close above range high and volume confirmation.
- Compute entry, stop, R-based target, measured-move target, and max size.
- Reject if target ordering is invalid.

**Verification:**
- Run `python -m pytest tests/test_breakout_strategy.py -v`.

### Task 4: Add first pullback strategy evaluator

**Objective:** Implement deterministic evaluation for `FirstMomentumPullback`.

**Files:**
- Modify: `src/hermes_momentum_trader/strategies.py`
- Create: `tests/test_pullback_strategy.py`

**Required behavior:**
- Require impulse move and volume thresholds.
- Require orderly first pullback into EMA9/VWAP context.
- Reject if pullback is too deep.
- Require reclaim above EMA9 and volume confirmation.
- Compute stop at pullback low and targets from prior high / R multiple.

**Status:** Completed in `hermes-momentum-trader` commit `0913bb19815e0833a51f8cbbf544fc42947bd597` with deterministic `evaluate_first_momentum_pullback`, six pullback strategy tests, and a full suite result of `18 passed in 0.04s`. [Source: local repo verification, 2026-04-28]

**Verification:**
- Run `python -m pytest tests/test_pullback_strategy.py -v`.
- Full-suite verification completed with `. .venv/bin/activate && python -m pytest`. [Source: local repo verification, 2026-04-28]

## Phase 3: Hermes skill integration

### Task 5: Create `trading/momentum-trader` skill

**Objective:** Add a Hermes skill that evaluates one payload and returns exactly one TradePlan JSON object.

**Files:**
- Create: `~/.hermes/skills/trading/momentum-trader/SKILL.md`
- Create: `skills/trading/momentum-trader/SKILL.md`
- Create: `skills/trading/momentum-trader/examples/valid_breakout.json`
- Create: `skills/trading/momentum-trader/examples/no_trade_ambiguous.json`
- Create: `tests/test_momentum_trader_skill.py`

**Required behavior:**
- Never place orders.
- Never invent strategy parameters.
- Always output JSON only.
- Default to `NO_TRADE` on ambiguity.
- Preserve paper-only/no-live-execution boundary.

**Status:** Completed in `hermes-momentum-trader` commit `d6e8f3c27e350bb931f319e36de411f1fcd907c5`; the skill was also installed locally as `momentum-trader` under `~/.hermes/skills/trading/momentum-trader`. [Source: local repo and skill verification, 2026-04-28]

**Verification:**
- Skill contract test verifies the safety phrases and examples. [Source: local repo verification, 2026-04-28]
- `skill_view("momentum-trader")` loads the installed skill successfully. [Source: local skill verification, 2026-04-28]

### Task 6: Add LLM output validator

**Objective:** Parse Hermes output, enforce schema, and reject malformed or unsafe plans.

**Files:**
- Create: `src/hermes_momentum_trader/hermes_client.py`
- Create: `tests/test_hermes_client.py`

**Required behavior:**
- Invalid JSON becomes `NO_TRADE`.
- Unsupported strategy names become `NO_TRADE`.
- Oversized positions become `NO_TRADE`.
- Mismatched symbol becomes `NO_TRADE`.

**Status:** Completed in `hermes-momentum-trader` commit `d6e8f3c27e350bb931f319e36de411f1fcd907c5` with `validate_hermes_trade_plan_output`, strict JSON parsing, TradePlan schema enforcement, strategy allow-listing, expected-symbol enforcement, and max-size rejection. [Source: local repo verification, 2026-04-28]

**Verification:**
- Run `python -m pytest tests/test_hermes_client.py -v`.
- Full-suite verification completed with `. .venv/bin/activate && python -m pytest`; result: `26 passed in 0.05s`. [Source: local repo verification, 2026-04-28]

## Phase 4: Alpaca paper integration

### Task 7: Add Alpaca data adapter

**Objective:** Fetch recent bars/quotes and build `MarketSnapshot` inputs.

**Files:**
- Create: `src/hermes_momentum_trader/alpaca_data.py`
- Create: `tests/test_alpaca_data.py`

**Required behavior:**
- No live orders.
- Paper credentials only.
- Stale data forces no-trade eligibility.
- Wide spread forces no-trade eligibility.

**Verification:**
- Run mocked tests with no real credentials.

### Task 8: Add paper execution adapter

**Objective:** Submit Alpaca paper orders only after deterministic post-validation passes.

**Files:**
- Create: `src/hermes_momentum_trader/execution.py`
- Create: `tests/test_execution.py`

**Required behavior:**
- Reject live trading configuration.
- Recompute risk per share and max size.
- Enforce max trades per day, max open positions, max symbol exposure, and daily loss stop.
- Dry-run mode is default.

**Verification:**
- Run mocked execution tests; no network calls in unit tests.

## Phase 5: Logging and analysis

### Task 9: Add decision/trade logging

**Objective:** Store snapshots, strategy decisions, rejected trades, accepted paper orders, fills, exits, and P&L.

**Files:**
- Create: `src/hermes_momentum_trader/logging_store.py`
- Create: `tests/test_logging_store.py`

**Required behavior:**
- Every decision is logged, including `NO_TRADE`.
- Logs include model/skill version, strategy spec hash, and snapshot hash.

**Verification:**
- Run tests against a temporary SQLite/DuckDB database.

### Task 10: Create `trading/data-analyst` skill

**Objective:** Analyze trade logs and propose conservative parameter suggestions without trading.

**Files:**
- Create: `~/.hermes/skills/trading/data-analyst/SKILL.md`
- Create: `reports/trading/README.md`

**Required behavior:**
- Compute expectancy, win rate, average R, max drawdown, and performance by strategy/symbol/time window.
- Mark low-sample findings as low confidence.
- Never call execution code.

**Verification:**
- Run against fixture trade logs and compare expected metrics.

## Phase 6: Operating modes

### Task 11: Add shadow mode CLI

**Objective:** Run the system during market hours without placing orders.

**Files:**
- Create: `src/hermes_momentum_trader/cli.py`
- Create: `tests/test_cli.py`

**Required behavior:**
- Fetch snapshots.
- Evaluate strategies.
- Log decisions.
- Never submit orders.

**Verification:**
- Run `hermes-momentum-trader shadow --symbols SPY QQQ --dry-run`.

### Task 12: Add paper mode CLI

**Objective:** Enable paper-order submission only when explicitly requested.

**Files:**
- Modify: `src/hermes_momentum_trader/cli.py`
- Create: `tests/test_paper_mode.py`

**Required behavior:**
- Requires `--paper` flag.
- Refuses if Alpaca client is not paper.
- Uses all circuit breakers.

**Verification:**
- Run in mocked mode first; then run one controlled paper test after user confirms credentials.

## Acceptance criteria

- All core strategy decisions are reproducible from fixture JSON. [Source: synthesis from [[hermes-momentum-trading-agent]], 2026-04-28]
- Hermes never places orders directly. [Source: synthesis from [[hermes-momentum-trading-agent]], 2026-04-28]
- Python independently validates every TradePlan before paper execution. [Source: synthesis from [[hermes-momentum-trading-agent]], 2026-04-28]
- Every decision is logged. [Source: synthesis from [[hermes-momentum-trading-agent]], 2026-04-28]
- Paper mode cannot run against live Alpaca credentials. [Source: synthesis from [[hermes-momentum-trading-agent]], 2026-04-28]
- The data analyst can generate performance reports from logs. [Source: synthesis from [[hermes-momentum-trading-agent]], 2026-04-28]
