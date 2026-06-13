# ARTHA — Architecture Decision Log

Every significant architectural decision is logged here with rationale.
Before overriding any decision, read its rationale.

---

| # | Decision | Chosen | Alternatives Considered | Rationale | Date |
|---|----------|--------|------------------------|-----------|------|
| 1 | Agent selection logic | Static YAML config (agent_intent_map.yaml) | LLM-based dynamic, confidence-threshold routing | Zero cost, zero hallucination risk, deterministic, testable. The root cause of the original 40-agent problem. | June 2026 |
| 2 | Agent output format | Pydantic BaseModel + retry once | Raw string, dict, best-effort parse | Downstream agents (especially RL) need typed inputs. Silent bad data is worse than visible error. | June 2026 |
| 3 | LLM provider | DeepSeek R1 (super) + Gemini Flash (lite) via OpenRouter/LiteLLM | Azure OpenAI GPT-4.1 | Free tier sufficient for reasoning tasks. Provider rotation handles rate limits. Fully configurable. | June 2026 |
| 4 | Error handling | Fallback chain per data type + partial result | Hard fail, retry only | Financial systems must degrade gracefully. A partial analysis is always better than a failed query. | June 2026 |
| 5 | RL policy promotion | Automatic eval gate (95% Sharpe on holdout set) | Shadow mode, human review, no gate | Prevents corrupted policy from affecting real recommendations immediately. Balance of safety and learning speed. | June 2026 |
| 6 | Tool assignment | Strict per-agent mapping | All tools to all agents | Reduces token cost, eliminates LLM confusion from irrelevant tool descriptions, makes agents testable. | June 2026 |
| 7 | Price data for RL | TIME_SERIES_DAILY_ADJUSTED (adjusted) | Raw OHLCV | Corporate actions (splits, dividends) corrupt RL observations if unadjusted prices used. | June 2026 |
| 8 | Timestamp storage | UTC everywhere, convert only for display | Local timezone, mixed | Timezone bugs are silent and catastrophic in financial systems. IST + ET + UTC mixing = disaster. | June 2026 |
| 9 | Testing approach | Contract tests (mocked LLM) + paper trading as E2E | LLM eval tests, golden dataset | LLM outputs are non-deterministic. Test the contracts around the LLM, not the LLM itself. | June 2026 |
| 10 | Prompt management | Versioned YAML templates in config/ | Python string constants, database | Prompts change frequently. Versioning enables rollback. Config files decouple prompts from code. | June 2026 |
| 11 | Process architecture | Single process, async schedulers | Multi-container, Celery | Personal system. Operational simplicity wins. Docker-compose restart handles crashes. | June 2026 |
| 12 | Migration strategy | Rename + extend in place | Clean rewrite, parallel branches | 296 tests and paper trading history are too valuable to risk. New code alongside old until proven. | June 2026 |
| 13 | Capital separation | Hard-enforced at execution time | Recommended only | Trading losses must never impair long-term investment capital. System enforces, does not just advise. | June 2026 |
| 14 | Investment exits | Thesis-based only, not price-based | Price-based stop-loss | A stock down 30% with intact thesis is an opportunity. A stock up 20% with broken thesis is an exit. | June 2026 |
| 15 | Web framework | FastAPI (async) | Flask (existing) | Async-native matches asyncio.gather() crews. Auto Swagger docs. Better for async RL feedback bridge. | June 2026 |
