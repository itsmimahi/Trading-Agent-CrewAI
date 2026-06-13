# ARTHA — Migration Status

Status key:
- ✅ KEEP AS-IS — Do not modify. Used exactly as is.
- 🔄 EXTEND — Add to, do not rewrite. Existing tests must keep passing.
- 🔃 REPLACE — Build new in artha/ then remove old once tested.
- 🆕 NEW — Build from scratch. Does not exist yet.
- ⛔ REMOVE — Delete only after replacement is tested.

---

## Core RL Layer — DO NOT TOUCH

| File | Status | Notes |
|------|--------|-------|
| `RL_Agent/risk_metrics.py` | ✅ KEEP | IBKRFeedbackBridge imports risk_aware_reward from here |
| `RL_Agent/rl_agent_v2.py` | ✅ KEEP | Core PPO/SAC policy. feedback_bridge calls online_finetune() from here |
| `RL_Agent/portfolio_trading_env.py` | ✅ KEEP | Gymnasium env. Used by both RL instances |
| `RL_Agent/regime_classifier.py` | ✅ KEEP | RegimeClassifier. Promoted to shared layer but not modified |
| `RL_Agent/__init__.py` | ✅ KEEP | |
| `scripts/test_rl_v2.py` | ✅ KEEP | 68 tests must keep passing |

---

## Advisory Flow — Extend, Don't Rewrite

| File | Status | Notes |
|------|--------|-------|
| `main_advisory_agent/advisory_flow.py` | 🔄 EXTEND | ArthaAdvisoryFlow in artha/flows/ extends this |
| `main_advisory_agent/memory_store.py` | 🔄 EXTEND | LanceDB schema extended with new scopes |
| `main_advisory_agent/advisory_memory_hooks.py` | 🔄 EXTEND | New hooks for thesis, megatrend, user profile scopes |
| `main_advisory_agent/routing.py` | 🔄 EXTEND | Add agent_intent_map.yaml lookup. Keep existing market routing logic. |
| `main_advisory_agent/conversation_manager.py` | ✅ KEEP | |
| `main_advisory_agent/flow_persistence.py` | ✅ KEEP | SQLiteFlowPersistence used by both books |
| `main_advisory_agent/approval_provider.py` | 🔄 EXTEND | Default approval=required (not auto-approve) |
| `scripts/test_advisory_flow_topology.py` | ✅ KEEP | 228 tests must keep passing |

---

## Agents — Restructure Output, Keep Logic

| File | Status | Notes |
|------|--------|-------|
| `agents/enhanced_technical_agent.py` | 🔄 EXTEND | Return TradingSignal Pydantic model. Phase 1. |
| `agents/enhanced_timing_agent.py` | 🔄 EXTEND | Return TimingSignal Pydantic model. Phase 1. |
| `agents/enhanced_risk_agent.py` | 🔄 EXTEND | Return RiskAssessment. Remove LLM — pure Python math. Phase 1. |
| `agents/enhanced_sentiment_agent.py` | 🔄 EXTEND | Return SentimentSignal Pydantic model. Phase 1. |
| `agents/enhanced_research_agent.py` | 🔄 EXTEND | Return FundamentalScorecard. Phase 2. |
| `agents/enhanced_macro_agent.py` | 🔄 EXTEND | Return MacroSignal. Phase 3. |
| `agents/enhanced_dividends_yield_optimizer.py` | ✅ KEEP | Use as DividendAgent. |
| `agents/enhanced_portfolio_manager.py` | 🔄 EXTEND | Phase 4 PortfolioOptimizer. |
| `agents/enhanced_trade_strategy_planner.py` | 🔄 EXTEND | Merge into TradingFlow later |
| `agents/orchestrator_agent.py` | ✅ KEEP | A2A registry |
| `agents/base_agent.py` | ✅ KEEP | New agents inherit from EnhancedBaseAgent |
| `agents/enhanced_base_agent.py` | ✅ KEEP | Memory recall + write. New agents inherit from this. |

---

## Data Tools — Replace with MCP

| File | Status | Notes |
|------|--------|-------|
| `main_advisory_agent/mcp_data_layer.py` | 🔃 REPLACE | Replace custom tool classes with Alpha Vantage MCP + Financial Datasets MCP. Phase 1. |
| `StockQuoteTool` (in mcp_data_layer.py) | ⛔ REMOVE | After Alpha Vantage MCP is wired and tested |
| `FundamentalsTool` | ⛔ REMOVE | After Financial Datasets MCP is wired and tested |
| `NewsTool` | ⛔ REMOVE | After Alpha Vantage MCP NEWS_SENTIMENT is tested |
| `TechnicalIndicatorTool` | ⛔ REMOVE | After Alpha Vantage MCP technical indicators are tested |
| `EconomicDataTool` | ⛔ REMOVE | After Alpha Vantage MCP economic indicators are tested |
| `MarketDataTool` | ⛔ REMOVE | After yfinance MCP is tested |

---

## API Layer — Replace Flask with FastAPI

| File | Status | Notes |
|------|--------|-------|
| `main_advisory_agent/main.py` (Flask) | 🔃 REPLACE | Replace with artha/api/main.py (FastAPI). Phase 1. |
| `artha/api/main.py` | 🆕 NEW | FastAPI app with /health endpoint |

---

## New Files to Create (Phase 1)

| File | Status | Notes |
|------|--------|-------|
| `artha/__init__.py` | 🆕 NEW | |
| `artha/config/settings.py` | 🆕 NEW | Pydantic BaseSettings |
| `artha/config/agent_intent_map.yaml` | 🆕 NEW | Already created |
| `artha/config/llm_config.yaml` | 🆕 NEW | Already created |
| `artha/models/base.py` | 🆕 NEW | Already created |
| `artha/models/trading_signals.py` | 🆕 NEW | Already created |
| `artha/models/investment_models.py` | 🆕 NEW | Already created |
| `artha/rl/feedback_bridge.py` | 🆕 NEW | THE critical file. Build in Task 8. |
| `artha/rl/eval_gate.py` | 🆕 NEW | RL policy safety. Build in Task 9. |
| `artha/flows/intent_router.py` | 🆕 NEW | Reads agent_intent_map.yaml |
| `artha/tests/contracts/` | 🆕 NEW | Contract test fixtures per agent |
| `pyproject.toml` | 🆕 NEW | uv + Ruff + Black + mypy |
| `.env.dev` | 🆕 NEW | Mock LLM, paper IBKR |
| `.env.prod` | 🆕 NEW | Real LLM, live IBKR |
| `CHANGELOG.md` | 🆕 NEW | Track all notable changes |

---

## Do Not Touch (Existing Working Infrastructure)

| Path | Reason |
|------|--------|
| `checkpoints/` | Flow state checkpoints — do not modify format |
| `data/memory/` | LanceDB data — extend schema, never drop tables |
| `idempotency_cache.db` | Order deduplication — keep as-is |
| `performance_data.db` | Portfolio performance history — keep as-is |
| `hf_models/` | Local HuggingFace embeddings — keep as-is |
| `docker-compose.yml` | Extend for new services, don't break existing |
| `.env.example` | Keep as template reference |
