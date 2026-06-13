# ARTHA — Phase 1 Implementation Tasks
## The ONLY phase currently in scope

**Phase 1 success metric:**
`rl_agent.online_finetune()` has been called at least once with real IBKR paper trade outcomes.
All 296 existing tests still pass.

Complete these tasks IN ORDER. Do not start Task N+1 until Task N passes its check.

---

## TASK 1: Project Setup (Day 1)

**What to do:**
1. Create `artha/` package with `__init__.py`
2. Create `artha/models/base.py` — BaseAgentOutput (file already exists in repo)
3. Create `artha/models/trading_signals.py` — TradingSignal, TimingSignal, RiskAssessment, SentimentSignal, RegimeSignal (file already exists)
4. Create `artha/models/investment_models.py` — FundamentalScorecard, ThesisDocument, etc. (file already exists)
5. Install uv: `pip install uv`
6. Create `pyproject.toml` with Ruff + Black + mypy config
7. Set up `.env.dev` (mock LLM, paper IBKR) and `.env.prod` (real LLM, live IBKR)
8. Install pre-commit hooks: `pre-commit install`

**Check:** `python -c "from artha.models.trading_signals import TradingSignal; print('ok')`

---

## TASK 2: LiteLLM Multi-Provider Setup (Day 1–2)

**What to do:**
1. `pip install litellm openai`
2. Set up `.env.dev` with OPENROUTER_API_KEY and GOOGLE_AI_KEY
3. Create `artha/config/settings.py` using Pydantic BaseSettings
4. Test LiteLLM routing: call artha-lite (Gemini Flash) and artha-super (DeepSeek R1)
5. Verify fallback: simulate rate limit, confirm rotation to fallback model
6. Test response_format=json_object forces valid JSON from both models

**Check:** Both `artha-lite` and `artha-super` return valid JSON to a simple test prompt.

---

## TASK 3: Alpha Vantage MCP Setup (Day 2)

**What to do:**
1. Install: `uvx marketdata-mcp-server YOUR_ALPHA_VANTAGE_API_KEY`
2. Test: call TIME_SERIES_DAILY for RELIANCE.NS
3. Test: call RSI for RELIANCE.NS
4. Test: call COMPANY_OVERVIEW for RELIANCE.NS
5. Wire into CrewAI as MCP tool (see existing mcp_data_layer.py for reference pattern)
6. Test fallback: simulate Alpha Vantage timeout → fallback to yfinance

**Check:** TechnicalAgent can get RSI for RELIANCE.NS via Alpha Vantage MCP.

---

## TASK 4: Agent Intent Map + Intent Router (Day 3)

**What to do:**
1. File `artha/config/agent_intent_map.yaml` already exists — review it
2. Create `artha/flows/intent_router.py`:
   ```python
   def get_agents_for_intent(intent: str) -> dict:
       map = load_yaml("artha/config/agent_intent_map.yaml")
       return map["intents"][intent]
   ```
3. Unit test: assert "technical_analysis" intent returns max 3 agents
4. Unit test: assert "general_advice" returns RegimeClassifier + MacroCycleAgent only
5. Unit test: assert NO intent returns > 5 agents
6. Wire into existing AdvisoryFlow's intent classification step

**Check:** A "what is the RSI of TCS?" query triggers ONLY TechnicalAgent + TimingAgent. Not all 12 agents.

---

## TASK 5: Restructure TechnicalAgent Output (Day 3–4)

**What to do:**
1. Modify `agents/enhanced_technical_agent.py`:
   - Change return type from string to `TradingSignal`
   - Update prompt to request JSON matching TradingSignal schema
   - Add: `result = TradingSignal.model_validate_json(raw_output)`
   - Add retry once on ValidationError
   - Add: return `TradingSignal.get_degraded_default()` after failed retry
2. Create contract test fixture: `artha/tests/contracts/fixtures/technical_agent.json`
3. Write contract test: mock LLM, assert TradingSignal returns
4. Write contract failure test: mock invalid JSON, assert degraded default returns

**Check:** Existing unit tests still pass. New contract test passes.

---

## TASK 6: Restructure RiskAgent Output (Day 4)

**What to do:**
1. RiskAgent MUST NOT call an LLM — it is pure Python math
2. Move all calculation logic to pure functions
3. Return `RiskAssessment` Pydantic model directly (no LLM involved)
4. Verify: no `llm.complete()` or `agent.run()` calls in RiskAgent
5. Write unit tests for the pure math functions

**Check:** RiskAgent returns RiskAssessment without any LLM call.

---

## TASK 7: Restructure TimingAgent + SentimentAgent (Day 4–5)

Same pattern as Task 5. Return `TimingSignal` and `SentimentSignal` respectively.
Both should use artha-lite (Gemini Flash) model.

**Check:** All three restructured agents return typed Pydantic models.

---

## TASK 8: Build IBKRFeedbackBridge — THE CRITICAL TASK (Day 5–8)

**This is the most important file in Phase 1.**

Create `artha/rl/feedback_bridge.py`:

```python
class IBKRFeedbackBridge:
    """
    Connects IBKR paper trade outcomes back to RL online_finetune().
    This closes the learning loop — without it, the RL agent never improves
    from real paper trading.
    """

    def record_entry(self, trade_id, symbol, entry_price, entry_obs_vector, action_taken):
        """Call this immediately after every approved trade execution.
        Stores in SQLite feedback buffer."""
        ...

    def check_and_process_outcomes(self, review_after_days: int = 5):
        """Scheduled: runs daily. Finds trades older than review_after_days.
        Fetches actual P&L from IBKR. Computes reward. Adds to experience buffer."""
        ...

    def _compute_reward(self, entry_price, current_price, drawdown, var_breach) -> float:
        """Uses risk_aware_reward() from RL_Agent/risk_metrics.py"""
        from RL_Agent.risk_metrics import risk_aware_reward
        ...

    def _maybe_finetune(self):
        """If experience buffer >= 64 samples: call online_finetune() + run eval gate."""
        if len(self.experience_buffer) >= 64:
            self._run_eval_gate_and_promote()

    def _run_eval_gate_and_promote(self):
        """New policy must beat old policy on holdout scenarios before promotion."""
        ...
```

**SQLite schema for feedback buffer:**
```sql
CREATE TABLE trade_feedback (
    trade_id TEXT PRIMARY KEY,
    symbol TEXT NOT NULL,
    entry_price REAL,
    entry_timestamp TEXT,   -- UTC ISO format
    entry_obs BLOB,         -- pickled numpy array
    action_taken BLOB,      -- pickled numpy array
    review_after_days INTEGER DEFAULT 5,
    reviewed_at TEXT,       -- NULL until reviewed
    exit_price REAL,        -- NULL until reviewed
    actual_return REAL,     -- NULL until reviewed
    reward REAL,            -- NULL until computed
    processed BOOLEAN DEFAULT FALSE
);
```

**Check:** After placing 2 paper trades and waiting (or mocking the date), `rl_agent.online_finetune()` is called with a real experience tuple.

---

## TASK 9: RL Eval Gate (Day 8–9)

Create `artha/rl/eval_gate.py`:
- Holdout scenario set: 20 fixed historical scenarios (5 per regime)
- Store in `artha/rl/holdout_scenarios/` as pickle files
- After every finetune: run new policy vs old policy on all 20 scenarios
- If new Sharpe >= 95% of old Sharpe: promote (update active_policy.txt)
- If not: reject, alert, increment rejection counter
- If 3 consecutive rejections: pause online learning + alert user

**Check:** Intentionally degrade a policy, confirm eval gate rejects it.

---

## TASK 10: FastAPI Setup + Health Endpoint (Day 9–10)

Replace Flask with FastAPI in `artha/api/main.py`:
```python
@app.get("/health")
async def health():
    return {
        "status": "ok",
        "ibkr_connected": ibkr_service.is_connected(),
        "rl_policy_version": read_active_policy_version(),
        "thesis_checks_overdue": count_overdue_thesis_checks(),
        "last_finetune_date": get_last_finetune_date(),
    }
```

**Check:** `curl localhost:8000/health` returns valid JSON with all fields.

---

## TASK 11: Integration Validation (Day 10)

Run the full flow end-to-end in paper trading mode:
1. Send a "technical analysis of RELIANCE.NS" query
2. Verify: only TechnicalAgent + TimingAgent run (check logs)
3. Verify: both return TradingSignal Pydantic models
4. Verify: RLAgentV2 receives the observation vector
5. Place a paper trade via IBKR
6. Verify: IBKRFeedbackBridge records the entry
7. (Mock) advance time by 5 days
8. Verify: feedback bridge fetches P&L, computes reward
9. Run 296 existing tests: all must pass

**PHASE 1 COMPLETE when:** Steps 1–9 all succeed.

---

## What NOT to build in Phase 1

- ThesisAgent — Phase 2
- MutualFundAgent — Phase 2
- Portfolio Genesis — Phase 4
- Thematic agents — Phase 4
- TaxAgent — Phase 4
- WealthDashboard — Phase 4
- BondAgent — Phase 3
- MoatAgent — Phase 3
- ManagementAgent — Phase 5

If anyone (including an AI assistant) suggests building these before Phase 1 is complete, say no.

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `CLAUDE.md` | AI assistant context — read first |
| `artha/config/agent_intent_map.yaml` | Which agents run for which intent |
| `artha/config/llm_config.yaml` | LiteLLM multi-provider routing |
| `artha/models/trading_signals.py` | Trading agent Pydantic models |
| `artha/models/investment_models.py` | Investment agent Pydantic models |
| `artha/rl/feedback_bridge.py` | THE critical file — build in Task 8 |
| `artha/rl/eval_gate.py` | RL policy safety gate |
| `RL_Agent/risk_metrics.py` | DO NOT MODIFY — use as-is |
| `RL_Agent/rl_agent_v2.py` | DO NOT MODIFY — use as-is |
| `DECISIONS.md` | Log of all architectural decisions |
| `MIGRATION.md` | What to keep vs change per file |
