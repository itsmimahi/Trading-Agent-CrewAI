# ARTHA — Personal Finance Advisor
## AI Code Assistant Context File (CLAUDE.md)

> **Read this entire file before writing any code.**
> This file is the single source of truth for every architectural decision in this project.
> All decisions here were made deliberately. Do not override them without explicit instruction.

---

## What Is Artha?

Artha (अर्थ) is an AI-powered personal finance advisor built on CrewAI 1.14.5.
It evolved from the `Trading-CrewAI-Agent` codebase.

It combines:
- **Trading Book** — RL-primary (PPO/SAC). Short horizon. Regime-aware position sizing.
- **Investment Book** — LLM-primary. Long horizon. Thesis-driven portfolio with quarterly integrity checks.
- **Shared Intelligence Layer** — RegimeClassifier, MacroCycleAgent, GlobalSignalMonitor.

Accounts: IBKR (global, paper → live) + Zerodha/IndMoney (India, NSE/BSE).

---

## THE MOST IMPORTANT RULE

**Every agent returns a typed Pydantic model. No agent returns a raw string to the orchestrator.**
Natural language is only generated at the final human-facing presentation layer, after all computation is complete.

If you write an agent that returns a string, you have broken the architecture.

---

## The 12 Non-Negotiable Engineering Decisions

### 1. Agent Selection — NEVER use LLM to decide which agents run

Use `artha/config/agent_intent_map.yaml` exclusively.
A static YAML lookup decides which agents run for each intent.
Maximum agents per query: **5**.
Never spin up all agents for any query.

```python
# CORRECT
agent_map = load_yaml("agent_intent_map.yaml")
agents_to_run = agent_map[intent]["tier1"]  # rule-based lookup

# WRONG — never do this
agents_to_run = llm.decide("which agents should handle this query?")
```

### 2. Output Contracts — Pydantic validation on every agent

Every agent output validates against a Pydantic model.
On ValidationError: retry once with stricter prompt.
If retry fails: return `Model.get_degraded_default()` — never crash.
Every model has `data_quality: Literal["good", "degraded"] = "good"`.

```python
# CORRECT
result = TradingSignal.model_validate_json(raw_llm_output)

# WRONG — never do this
result = {"symbol": raw.split("Symbol:")[1], ...}  # string parsing
```

### 3. LLM Routing — Free models only, 3-tier

- **Super (Reasoning)**: `openrouter/deepseek/deepseek-r1` — ThesisAgent, MoatAgent, ManagementAgent
- **Lite (Fast)**: `gemini/gemini-2.0-flash-exp` — TechnicalAgent, FundamentalsAgent, SentimentAgent
- **No LLM**: Pure Python — RegimeClassifier, RiskAgent (math only), IntentClassifier (rule-based)

See `artha/config/llm_config.yaml` for the complete LiteLLM config.

### 4. Error Handling — Fallback chain, never hard fail

```python
DATA_FALLBACK_CHAIN = {
    "price":        ["alpha_vantage_mcp", "yfinance_mcp", "cache"],
    "technicals":   ["alpha_vantage_mcp", "yfinance_mcp"],
    "fundamentals": ["financial_datasets_mcp", "alpha_vantage_mcp"],
    "news":         ["alpha_vantage_mcp", "finbrain_mcp"],
    "ibkr":         ["ezib_mcp", "ib_insync_direct"],
}
```

A partial analysis with `data_quality="degraded"` is always better than a failed query.

### 5. RL Safety — Eval gate before every policy promotion

After every `online_finetune()` call:
1. Run new policy against 20 holdout scenarios (5 per regime)
2. New policy must achieve >= 95% of old policy's Sharpe
3. If it fails: keep old policy, alert user, increment rejection counter
4. After 3 consecutive rejections: pause online learning entirely

Policy files: `artha/rl/policies/trading_book/v{n}_{date}.zip`
Active policy pointer: `artha/rl/policies/trading_book/active_policy.txt`

### 6. Strict Tool-to-Agent Mapping

Each agent is initialised with ONLY the tools it legitimately needs:

| Agent | Allowed Tools |
|-------|--------------|
| TechnicalAgent | TIME_SERIES, RSI, MACD, BBANDS, ATR, STOCH, VWAP, EMA, SMA |
| FundamentalsAgent | INCOME_STATEMENT, BALANCE_SHEET, CASH_FLOW, EARNINGS, COMPANY_OVERVIEW |
| ManagementAgent | EARNINGS_CALL_TRANSCRIPT, INSIDER_TRANSACTIONS, INSTITUTIONAL_HOLDINGS |
| SentimentAgent | NEWS_SENTIMENT, TOP_GAINERS_LOSERS |
| MacroCycleAgent | REAL_GDP, CPI, FEDERAL_FUNDS_RATE, TREASURY_YIELD, UNEMPLOYMENT, WTI, BRENT |
| RiskAgent | Portfolio tools, GLOBAL_QUOTE only. NO LLM — pure Python math. |
| MoatAgent | EARNINGS_CALL_TRANSCRIPT, COMPANY_OVERVIEW, web_fetch for annual reports |

**Never inject all tools into all agents.**

### 7. Memory Scopes — Write to the correct LanceDB scope

```
/investment/thesis/{symbol}     — ThesisAgent only
/trading/executions             — IBKRFeedbackBridge only
/market/{code}/analysis         — market_analysis_step only
/shared/macro                   — MacroCycleAgent only
/shared/megatrends              — MegatrendResearchAgent only
/user/goals                     — GoalDiscoveryAgent only
/user/profile                   — ApprovalGate only (approve/reject patterns)
/portfolio/holdings             — ExecutionAgents only
```

Recall scoring: `score = 0.50 × semantic + 0.30 × recency + 0.20 × regime_match`

### 8. Prices — Always use ADJUSTED prices for RL training

```python
# CORRECT for RL training
endpoint = "TIME_SERIES_DAILY_ADJUSTED"

# WRONG — splits and dividends will corrupt RL observations
endpoint = "TIME_SERIES_DAILY"
```

For live order placement: use actual market price. Never mix adjusted and unadjusted in the same calculation.

### 9. Timestamps — Always store in UTC

```python
# CORRECT
from datetime import datetime, timezone
ts = datetime.now(timezone.utc)

# WRONG
ts = datetime.now()  # naive datetime — timezone bugs guaranteed
```

Use `zoneinfo` (Python 3.9+) for market hours checks. Never manual offset arithmetic.

### 10. Testing Philosophy

- Unit tests: pure Python (risk_metrics, regime_classifier, Pydantic models) — fully deterministic
- Contract tests: mock LLM with pre-written JSON fixture, test Pydantic validation + routing
- Contract failure tests: mock LLM returns invalid JSON, assert degraded default returned
- E2E test: paper trading IS the system test

Test fixtures location: `artha/tests/contracts/fixtures/{agent_name}.json`

### 11. Prompt Management — Versioned config files

Prompts are YAML files in `artha/config/prompts/{agent_name}/v{n}.yaml`.
Never hardcode prompts as Python strings.
To change a prompt: create v{n+1}.yaml, update agent config pointer, test.

### 12. Build Order — Phase 1 first, always

**Phase 1 is the only phase currently in scope.**
Do not build Phase 2 features until Phase 1 success metric is met.

Phase 1 success metric: `rl_agent.online_finetune()` has been called at least once with real IBKR paper trade outcomes. All 296 existing tests still pass.

See `PHASE1_TASKS.md` for the ordered task list.

---

## What to Keep vs What to Change

### KEEP UNCHANGED (do not refactor these)
- `RL_Agent/` — entire directory. Risk metrics, regime classifier, portfolio env, RLAgentV2. These are production-quality.
- `main_advisory_agent/memory_store.py` — keep, schema will be extended
- `main_advisory_agent/advisory_memory_hooks.py` — keep, extend with new scopes
- `main_advisory_agent/advisory_flow.py` — keep as base, ArthaAdvisoryFlow extends this
- `integration/event_bus.py` — keep but bypass in single-process deployment (direct fn calls instead)
- `scripts/test_rl_v2.py` — keep, 68 tests must continue passing
- `scripts/test_advisory_flow_topology.py` — keep, 228 tests must continue passing
- All existing `.zip` checkpoint files in any checkpoints/ directory

### EXTEND (add to, don't rewrite)
- `main_advisory_agent/advisory_flow.py` → `artha/flows/artha_advisory_flow.py` extends it
- `agents/enhanced_base_agent.py` → new agents inherit from this
- `config/` → add new YAML files alongside existing .env approach

### REPLACE (build new, remove old once tested)
- Custom tool classes (StockQuoteTool, FundamentalsTool, NewsTool, etc.) → replace with MCP calls
- Flask app in `main_advisory_agent/main.py` → replace with FastAPI in `artha/api/main.py`
- Direct ib_insync calls → replace with EZIB-MCP in Phase 2

### NEW (build from scratch)
- `artha/` package (entire new package)
- `artha/models/` — all Pydantic output models
- `artha/rl/feedback_bridge.py` — IBKRFeedbackBridge (build first in Phase 1)
- `artha/config/agent_intent_map.yaml` — intent to agent routing
- `artha/config/llm_config.yaml` — LiteLLM multi-provider config

---

## Package Structure

```
artha/
  __init__.py
  config/
    settings.py               # Pydantic BaseSettings
    llm_config.yaml           # LiteLLM config
    agent_intent_map.yaml     # THE MAP — intent → agents
    prompts/
      technical_agent/v1.yaml
      thesis_agent/v1.yaml
  models/                     # ALL Pydantic output models
    trading_signals.py
    investment_models.py
    shared_models.py
    base.py                   # BaseAgentOutput
  agents/
    trading/
    investment/
    shared/
    genesis/
    thematic/
  flows/
    artha_advisory_flow.py
    trading_flow.py
    investment_flow.py
  skills/                     # 12 skill modules
  mcp/
    client.py
    fallback.py
    cache.py
  rl/
    policies/
    feedback_bridge.py        # BUILD FIRST
    eval_gate.py
  api/
    main.py                   # FastAPI
  tests/
    unit/
    contracts/
      fixtures/
    integration/
```

---

## Capital Allocation Rules (System-Enforced)

- Trading Book: maximum 10% of total portfolio
- Investment Book: minimum 60% of total portfolio
- Liquid Reserve: minimum 15%
- Max single position (Trading): 5% of Trading Book
- Max single holding (Investment): 20% of Investment Book
- Stop-loss on investments: thesis-based ONLY, not price-based
- HIGH_VOL regime: Trading reduces size 50%, Investment pauses new entries

---

## Critical Anti-Patterns (Never Do These)

```python
# 1. NEVER run all agents by default
crew = build_crew(agents=all_12_agents)  # WRONG

# 2. NEVER return text from an agent that feeds another agent
return "The RSI is 65, suggesting overbought conditions..."  # WRONG
return TradingSignal(rsi=65, action="SELL", confidence=0.8)  # CORRECT

# 3. NEVER use unadjusted prices for RL training
data = yf.download(symbol, auto_adjust=False)  # WRONG for RL

# 4. NEVER store a naive datetime
created_at = datetime.now()  # WRONG
created_at = datetime.now(timezone.utc)  # CORRECT

# 5. NEVER silently swallow agent failures
try:
    result = agent.run(prompt)
except Exception:
    pass  # WRONG — silent failure corrupts downstream

# 6. NEVER mix Trading Book and Investment Book capital
if trading_needs_more_capital:
    transfer_from_investment_book()  # WRONG — capital separation is inviolable

# 7. NEVER place orders on illiquid stocks
# Always check: avg_daily_volume > 1Cr INR AND order_size < 2% of avg_daily_volume

# 8. NEVER skip the approval gate
order.execute_direct()  # WRONG — every material decision needs human approval
```

---

## Pre-Flight Checklist Before Any Investment (Portfolio Genesis Gate)

Before deploying any capital, verify ALL of the following are true:
1. Emergency fund >= 6 months of expenses (liquid form)
2. Term insurance in place (minimum 10-15× annual income)
3. Health insurance in place (minimum ₹10L cover)
4. Zero high-interest debt (credit card, personal loans > 12%)

If any of these is false: BLOCK investment. Advise what to fix first.
This is not configurable. It is not optional.

---

## Current Build Status

See `PHASE1_TASKS.md` for the current task list.
See `DECISIONS.md` for a log of all architectural decisions with dates.
See `MIGRATION.md` for file-by-file migration status.

**The single most important task right now:**
Build `artha/rl/feedback_bridge.py` — the IBKRFeedbackBridge.
Without it, the RL agent learns nothing from real paper trades.

---

## Questions for the Human Developer

If you (the AI assistant) are unsure about any of the following, ask before proceeding:
- Which phase is currently in scope?
- Which agents should handle a specific intent? (check agent_intent_map.yaml)
- Which LLM tier should a new agent use?
- Is a particular data field used for RL training or display only? (affects adjusted vs unadjusted)
- Should an output go to LanceDB, SQLite, or neither?

---

*Last updated: June 2026 | v2.0 | Mahesh Vishnu Wanve*
