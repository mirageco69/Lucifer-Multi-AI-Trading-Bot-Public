# LUCIFER Plugin Catalogue

> **280+ active plugins** · Auto-discovered at startup · No restart required to add a new plugin · 10 specialised AI agents

LUCIFER uses a WordPress-style plugin architecture. Every feature is a plugin: exchanges, AI agents, data sources, indicators, strategies, risk rules, and dashboard widgets. Adding a plugin means dropping three files into `src/plugins/<category>/<name>/`:

```
plugin.json   — metadata (name, label, category, widget slot)
api.py        — Flask blueprint (backend routes)
widget.jsx    — React component (dashboard widget)
```

The platform discovers and loads them automatically. Disabling a plugin in the Plugin Manager removes it from the UI and API without touching any other code.

---

## Exchange Plugins

These plugins connect LUCIFER to trading exchanges. Each manages its own authentication, order placement, lot tracking, and P&L.

| Plugin | Exchange | Asset Class | Notes |
|---|---|---|---|
| `coinspot` | CoinSpot | Crypto (AUD) | V2 REST API · Market + limit orders · Live trading active |
| `swyftx` | Swyftx | Crypto (AUD) | JWT auth · Full order management · Live trading active |
| `bybit` | Bybit | Crypto (USDT) | V5 API · Spot trading · Paper/demo only |
| `ig` / `ig_exchange` | IG Markets | CFDs / Commodities | REST API · Tiered holdings · Paper mode (IG_PAPER=1) |
| `paper_exchange` | Paper Trading | All asset classes | Simulates all exchanges with real market data |
| `binance_exchange` | Binance | Crypto | Stub — not yet enabled |
| `kraken_exchange` | Kraken | Crypto | Stub — not yet enabled |
| `ib_exchange` | Interactive Brokers | Equities / Futures | Stub — not yet enabled |

---

## AI Plugins

### Core AI & Brain

| Plugin | Purpose |
|---|---|
| `ollama` | Core Ollama integration — routes all LLM calls to local Ollama models. Serialised FIFO queue (`_OllamaQueue`) ensures one inference at a time. `keep_alive: -1` keeps models in VRAM permanently. No cloud API calls. |
| `local_ollama_advisor` | Pre-trade gate — validates every trade with structured LLM reasoning before the Brain was introduced. Superseded by the Brain for buy decisions. |
| `claude_advisor` | Cloud AI gate — **disabled** (zero cloud API policy; local Ollama only) |
| `ensemble` | Ensemble voting — combines multiple strategy signals into a weighted decision |
| `ai_tuner` | **AI Tuner** — fine-tune all AI agent parameters from the dashboard: model selection per agent, confidence thresholds, agent weights (Technical 30%, Sentiment 25%, Risk 25%, Vision 20%), timeouts, position sizing scale, and sell signal thresholds. Route: `/ai-tuner` |
| `ai_chat` / `ai_chat_assistant` | **AI Chat** — persistent conversation history, Ollama streaming, live trading context (portfolio, positions, Brain state), model selector, quick prompts. Configurable log/trade/status context injection. Route: `/chat` |
| `llm_prompts` | **LLM Prompt Editor** — edit all AI system prompts from the dashboard without touching Python files. Changes take effect immediately, no restart. |
| `ab_testing` | **A/B Prompt Testing** — Thompson-sampling prompt experimentation; Brain autonomously creates, tests, promotes, and retires agent prompt variants nightly. Variants stored in `data/ab_prompts.json` only — never `llm_prompts.json`. |

### AI Brain Workspace

| Plugin | Route | Purpose |
|---|---|---|
| `brain_console` (ai_brain) | `/ai-brain` | **Brain Console** — real-time terminal feed of AI Brain decisions, debates, ReasoningBank entries, and reflection logs. Colour-coded by action. Tabs: All / Decisions / Log / Queue. |
| `debate_viewer` | `/debate-viewer` | **Debate Viewer** — full-screen live terminal streaming all 8 agent verdicts and reasoning as a debate unfolds. History sidebar with last 50 debates. Debate replay. Mobile-responsive. |
| `brain_metrics` | `/brain-metrics` | **Brain Metrics** — equity curve, win rate trend, regime breakdown, gate block analysis, and agent performance stats for the AI Brain. |
| `brain_conviction_trend` | `/conviction-trend` | **Conviction Trend** — Brain vote split over time; flags markets where agents disagree (low conviction = caution flag). |
| `correlation_guard` | `/correlation-guard` | **Correlation Guard** — portfolio correlation checker; flags undiversified holdings where positions move together (Pearson r > 0.85). |
| `post_mortem` | `/post-mortem` | **Post-Mortem** — AI analysis of individual closed trades; writes reasoning patterns to the ReasoningBank on confirmed entries. |
| `ai_activity` | `/ai-activity` | **AI Activity** — chronological feed of all AI actions across all agents. |
| `ai_decisions` | `/ai-decisions` | **AI Decisions** — filterable table of all Brain decisions with verdict, confidence, and outcome tracking. |

### Autonomous Agents

| Plugin | Purpose |
|---|---|
| `exit_agent` | **Agent 10 — Exit Agent** — autonomous AI position manager evaluating all open crypto positions every 30 min. TRAIL (updates trail level), SCALE_OUT (partial sell), and CUT (full exit) verdicts written to `data/exit_orders.json` and executed by the bot executor thread. Cooldowns persisted across restarts in `data/exit_agent_last_eval.json`. History (last 200 evaluations) in `data/exit_agent_history.json`. |
| `cb_watchdog` | **Agent 9 — CB Watchdog** — triggered on every circuit breaker level change. Classifies drawdown cause as REAL_DRAWDOWN / CORRELATED_CRASH / STRATEGY_FAILURE / DATA_ARTIFACT / RECOVERABLE_DIP. Reports verdict, confidence, and remediation action via Telegram. Runs on `qwen3:8b` (MODEL_FAST) to complete within the Brain tick cycle. |

### Mission Control AI Widgets

| Plugin | Purpose |
|---|---|
| `ai_brain_mc` | **AI Brain Status Card** — Mission Control widget showing last Brain decision, open positions, MTF gate result, and top opportunity score. |
| `ai_briefing` | **AI Briefing** — AI-generated market briefing derived from recent decisions and recommendations. |
| `brain_console` (mc) | **Brain Console Widget** — Mission Control embedded terminal feed of Brain decisions. |
| `paper_comparison` | **Paper Bot Comparison** — side-by-side Legacy (rules-only) vs AI Brain paper bot: total value, realised/unrealised P&L, cash, win rate, trade count, and holdings. Context-aware verdict text. |

### AI Brain Architecture

The AI Brain is the platform's autonomous trading intelligence. All inference runs locally on Ollama — `qwen3:8b` for the 6 real-time debate agents, `qwen2.5:14b` for the Risk Manager and nightly reflection. Agent 9 (CB Watchdog) and Agent 10 (Exit Agent) run on `qwen3:8b` and are supervised independently of the main debate pipeline.

**Data flow:**
```
Market data + watchlist + ReasoningBank + IG intelligence layer
        ↓
  MTF Gate (1h / 4h / 6h alignment check — ≥ 2/3 bullish required)
        ↓  pass
  8-Agent Debate Panel (sequential via Debate Daemon)
    1. Technical Analyst    5. Bull Researcher
    2. Chart Analyst        6. Bear Researcher
    3. Sentiment Analyst    7. Risk Manager (veto, qwen2.5:14b)
    4. Fundamental Analyst  8. Portfolio Manager
        ↓  approved
  brain_orders.json queue
        ↓
  BrainExecutor thread → place_buy() → exchange
```

**Key files:**

| File | Purpose |
|---|---|
| `services/ai_brain_loop.py` | Main autonomous loop — gates, cooldown, execution |
| `services/ai_decision_engine.py` | 8-agent debate pipeline — `run_debate()` |
| `services/debate_service.py` | Standalone Debate Daemon — sole process calling `run_debate()`; reads `data/debate_queue.json` |
| `services/ai_context_builder.py` | Builds full market + IG intelligence context for each debate |
| `services/exit_debate.py` | 2-agent exit debate engine (replaces fixed stop-loss) |
| `services/exit_agent.py` | Agent 10 — autonomous exit manager |
| `services/cb_watchdog_agent.py` | Agent 9 — CB Watchdog diagnosis |
| `services/auto_circuit_breaker.py` | 3-level graduated drawdown protection singleton |
| `services/reasoning_bank.py` | Local RAG memory — add / retrieve / update learned patterns |
| `services/decision_ledger.py` | Full audit trail of every Brain decision and outcome |
| `services/reflection_engine.py` | 2am nightly reflection — extracts patterns into ReasoningBank |
| `services/watchlist_mtf.py` | Multi-timeframe signals from watchlist price array (no external API) |
| `services/watchlist_discovery.py` | Hourly paper evaluation of watchlist candidates |
| `data/brain_orders.json` | Live buy queue — Brain writes, executor reads |
| `data/brain_orders_ig.json` | Live buy queue for IG Markets |
| `data/debate_queue.json` | Debate request queue — bots write, Debate Daemon reads |
| `data/debate_results/` | Per-instrument debate results (15-min TTL) |
| `data/exit_debates.json` | Open position exit debate state (floor, hold count) |
| `data/reasoning_bank.json` | Accumulated learned patterns |
| `data/decision_ledger.json` | Full decision history with outcomes |
| `data/chat_conversations.json` | AI Chat conversation history |
| `data/exit_orders.json` | Pending/executed exit orders from Agent 10 (SCALE_OUT / CUT) |
| `data/exit_agent_history.json` | Last 200 exit evaluations — all verdicts including HOLD |
| `data/exit_trail_levels.json` | Active trailing stop levels per coin (TRAIL verdicts) |
| `data/exit_agent_last_eval.json` | Last evaluation timestamps per position — persists cooldowns across restarts |
| `data/auto_cb_state.json` | Circuit breaker peak, level, and drawdown — reset to `{}` if stale |
| `data/auto_cb_force_peak.flag` | Write a float here to inject a test CB peak without restarting the bot |

---

## Mission Control Plugins

Mission Control is the main operational dashboard. Each widget is a separate plugin, placed into named slots on the MC page.

### Overview

| Plugin | Route | Purpose |
|---|---|---|
| `executive` | `/executive` | **Mission Control** — top-level overview page combining all MC widgets into a single command view |
| `exec_health` | `/exec-health` | **System Health** — CPU temperature (LHM), RAM, disk, bot process status, WHEA error counter, thermal protection status |
| `exec_security` | `/exec-security` | **Security** — active sessions, auth events, failed login attempts, 2FA status |
| `dept_health` | — | **Department Health** — compact grid of all department plugin statuses |
| `departments` | — | **Departments** — full department breakdown with plugin health by workspace |
| `maintenance_mode` | `/maintenance` | **Maintenance Mode** — pause all new buy entries for N minutes with a countdown; auto-resumes when the timer expires |

### Trading & Positions

| Plugin | Purpose |
|---|---|
| `open_trades` | **Open Trades** — live open positions across all exchanges with entry price, current P&L, and stop-loss distance |
| `stop_loss_exposure` | **Stop-Loss Exposure** — total capital at risk across all open positions if every stop-loss triggers simultaneously |
| `tracked_coins` | **Tracked Coins** — live prices and RSI for all actively tracked coins |
| `bot_status` | **Bot Status** — real-time bot running status, mode (live/paper/demo), and exchange |
| `exchange_coinspot` | **CoinSpot Panel** — CoinSpot paper portfolio: cash, holdings, P&L |
| `exchange_coinspot_live` | **CoinSpot Live Panel** — live CoinSpot account balance and positions |
| `exchange_swyftx` | **Swyftx Panel** — Swyftx paper portfolio |
| `exchange_bybit` | **Bybit Panel** — Bybit paper portfolio |
| `exchange_ig` | **IG Markets Panel** — IG paper CFD portfolio: cash, open positions, overnight financing |
| `exchange_ig_live` | **IG Live Panel** — live IG account positions and balance |
| `portfolio_mc` | **Portfolio Summary** — aggregated paper portfolio totals across all exchanges |
| `markets_mc` | **Markets** — live market prices with Fear & Greed index and trending coins |
| `rss_news` | **RSS Crypto News** — latest crypto headlines with sentiment scoring and coin matching |
| `alerts_mc` | **Alerts** — active alert feed from the alert engine |
| `resource_monitor` | **Resource Monitor** — CPU, RAM, GPU, and system resource usage |
| `research_agent` | **Research Agent Status** — AI research agent progress and current task |

### Paper Trading

| Plugin | Route | Purpose |
|---|---|---|
| `paper_trading` | `/paper-trading` | **Paper Trading Live View** — real-time paper portfolio with open positions and recent fills |
| `paper_settings` | `/paper-settings` | **Paper Settings** — per-exchange paper account configuration (starting balance, DCA amounts) |
| `paper_analytics` | `/paper-analytics` | **Paper Analytics** — performance metrics for paper accounts: win rate, Sharpe, drawdown, trade frequency |
| `paper_comparison` | `/paper-comparison` | **Bot Comparison** — Legacy (rules-only) vs AI Brain paper bot side-by-side |
| `paper_pnl` | `/paper-pnl` | **P&L Analytics** — equity curve, realised/unrealised breakdown, daily P&L chart |
| `nathans_test_trading` | `/nathans-test-trading` | **Nathan's Test Trading** — side-by-side IG demo and Swyftx demo performance; each panel shows balance, P&L, open positions, recent trades, win rate |
| `moonbag_tracker` | `/moonbags` | **Moonbag Tracker** — compares every sold coin against its current live price. Shows missed gains (sold too early) and avoided losses (sold at the right time), with total AUD impact. Reads from `swyftx_demo_trades.json`. |
| `process_control` | `/process-control` | **Process Control** — restart any bot, the dashboard, or all services from the dashboard. Writes to `data/restart_requests/`; supervisor picks up within 60s. |

---

## Trading Workspace Plugins

| Plugin | Route | Purpose |
|---|---|---|
| `chart` | `/chart` | **Chart** — interactive OHLC price chart with indicator overlays (EMA, Bollinger, SuperTrend, VWAP) |
| `holdings` | `/holdings` | **Holdings** — complete open positions across all exchanges with FIFO P&L, cost basis, and current value |
| `markets` | `/markets` | **Markets** — live price feed for all tracked instruments with sortable columns |
| `watchlist` | `/watchlist` | **Watchlist** — 72-coin opportunity watchlist with signal scores, RSI, MTF alignment, and discovery status |
| `trade_journal` | `/journal` | **Trade Journal** — searchable history of all closed trades with P&L, holding time, and strategy attribution |
| `ig_markets` | `/ig` | **IG Markets** — full IG CFD trading interface: open positions, spread monitor, ATR sizer, overnight financing tracker |
| `accountancy` | `/accountancy` | **Accountancy** — ledger-level breakdown of all transactions; reconciles exchange fills against local records |

---

## Research Workspace Plugins

| Plugin | Route | Purpose |
|---|---|---|
| `research_centre` | `/research-centre` | **Research Centre** — AI research agent control panel: launch research cycles, view findings, promote to ReasoningBank |
| `research_lab` | `/research` | **Research Lab** — strategy backtester with configurable date ranges and per-strategy P&L breakdown |
| `market_breadth` | `/market-breadth` | **Market Breadth** — percentage of tracked coins above EMA-21; fires Telegram alerts at configurable strong/weak thresholds |
| `hurst_exponent` | — | **Hurst Exponent** — regime detector: H > 0.5 = trending, H < 0.5 = mean-reverting, H ≈ 0.5 = random walk |
| `monte_carlo` | — | **Monte Carlo** — drawdown simulation: shuffles trade P&L 1000× to stress-test worst-case loss sequences |
| `rule_significance` | — | **Rule Significance Testing** — compares RSI/MACD signal win rate against random entry baseline to prove statistical edge |
| `strategy_replay` | — | **Strategy Replay** — replay historical price data through any strategy to see signal-by-signal decision logic |
| `ai_chat` (research) | `/chat` | **AI Chat** — full-screen AI conversation with live platform context |

---

## Performance Workspace Plugins

| Plugin | Route | Purpose |
|---|---|---|
| `analytics` | `/analytics` | **Analytics** — platform-wide performance: total return, Sharpe ratio, max drawdown, win rate, profit factor across all exchanges and strategies |
| `reports` | `/reports` | **Reports** — downloadable trade reports by date range, exchange, and strategy |
| `tax` | `/tax` | **Tax (ATO)** — Australian Tax Office-format capital gains report: CGT events, cost basis, proceeds, and net gain per financial year |
| `auto_weekly_report` | `/weekly-report` | **Weekly Report** — autonomous Sunday report generated by Ollama and sent via Telegram: week summary, top/worst performers, strategy health, next-week outlook |

---

## System & Engineering Plugins

| Plugin | Route | Purpose |
|---|---|---|
| `engineering` | — | **Engineering** — blueprint health dashboard: plugin compliance scores, tech debt tracker, event bus monitor, Prometheus migration progress |
| `logs` | — | **Bot Logs** — live tail of all bot log files with exchange and log-level filtering |
| `audit_log` | — | **Audit Log** — immutable audit trail of all significant system events: trades, config changes, auth events, risk triggers |
| `telegram_log` | — | **Telegram Log** — live log of every Telegram message sent by the bot, with filtering and duplicate detection |
| `plugin_doctor` | — | **Plugin Doctor** — 3-LLM autonomous self-healing engine: scans all plugins for missing files, import errors, and blueprint compliance gaps; diagnoses root cause; proposes and applies fixes; requests targeted restart via `data/restart_requests/`. Protected files (bot.py, data/*.json, supervisor.ps1) are diagnosed but never auto-repaired. |
| `bot_settings` | — | **Bot Settings** — core bot configuration: trading parameters, risk limits, AI settings, portfolio targets, and system flags |
| `certification` | — | **Sprint Certification** — paper trading certification runner; tracks compliance against defined performance criteria before promoting a strategy to production |
| `workspace_manager` | — | **Workspace Manager** — save, restore, and share named widget grid layouts across all dashboard pages |
| `webhooks` (system) | — | **Webhooks** — outbound webhook subscriptions: fire HTTP POST to external URLs on bot events (trade executed, CB triggered, etc.) |

---

## Plugins Workspace

| Plugin | Purpose |
|---|---|
| `plugin_registry` | **Plugin Registry** — full list of all installed plugins with enable/disable toggle, version, category, and health status. The Plugin Manager page uses this. |
| `ui_design` | **UI Design** — theme preview and CSS variable inspector for development use |
| `webhooks` (plugins_ws) | **Webhook Config** — webhook subscription management UI |

---

## Risk Workspace Plugins

| Plugin | Route | Purpose |
|---|---|---|
| `risk_monitor` | `/risk` | **Risk Monitor** — real-time risk dashboard: circuit breaker level, drawdown from peak, daily P&L limit status, position concentration, correlation warnings |
| `health` | `/health` | **Health** — platform health overview: all bots, services, data feeds, and AI inference queue status |
| `health_monitor` | — | **Health Monitor** — detailed process health: uptime, restart count, last heartbeat per supervised process |
| `alert_engine` | `/alerts` | **Alert Engine** — configurable alert rules: price thresholds, drawdown limits, regime changes, breadth extremes. Smart throttle and dedup prevent alert storms. |

### Risk Rule Plugins

| Plugin | Function |
|---|---|
| `core` | Core risk rules — exposure limits, daily drawdown, position count caps |
| `fear_dca` | Fear-weighted DCA — increases buy size in Extreme Fear, reduces in Extreme Greed |
| `dca` | Standard DCA rules — dollar-cost averaging on dips |
| `trailing_stop` | Trailing stop-loss with configurable trail percentage; adaptive tiers at +12%/+20%/+30% gains |
| `sell_decision` | Lot-based sell decision engine — FIFO P&L, take-profit targeting |

---

## Data Source Plugins

| Plugin | Data Provided |
|---|---|
| `fear_greed` | Crypto Fear & Greed Index (Alternative.me) |
| `crypto_news` | Latest crypto news aggregated from multiple feeds |
| `rss_crypto_news` | RSS-based crypto news parser with sentiment scoring and coin extraction |
| `macro_news` | Macroeconomic and geopolitical news (MACRO/ tagged) |
| `sentiment_snapshot` | Point-in-time market sentiment composite |
| `reddit_sentiment` | Reddit crypto subreddit sentiment scoring |
| `influencer_feed` | Crypto influencer post monitoring |
| `coingecko_intel` | CoinGecko market intelligence — trending coins, top gainers/losers (one bulk call per cycle, not per-coin) |
| `watchlist_mtf` | Watchlist-derived multi-timeframe signal — aggregates 15-min price array into 1h/4h/6h bars. Replaces yfinance. No external API calls. |
| `market_chart` | OHLC price data and chart feeds |
| `deribit_feed` | Deribit options data — IV, put/call ratio |
| `derivatives_feed` | Futures funding rates and open interest |
| `onchain_macro_feed` | On-chain macro indicators (whale movements, exchange flows) |
| `system_health_monitor` | CPU, RAM, disk, process health monitoring — feeds the System Health widget |
| `macro_calendar` | Economic event calendar with IG session markers, blackout indicators, and ForexFactory feed (refreshed every 6 hours) |
| `myfxbook_sentiment` | **Crowd Sentiment** — Myfxbook community outlook: retail crowd positioning across 187 symbols with contrarian signals |
| `interrupt_agent` | **Priority Interrupt Agent** — 3-tier background service polling macro events (ForexFactory), exchange health, price moves, circuit breaker state, drawdown, and IG session boundary every 30s. Writes active context to `data/interrupt_context.json`. Injected as a coloured urgency banner at the top of every Brain debate prompt. |

---

## Home Widgets

These lightweight plugins populate the home/landing page of the dashboard.

| Plugin | Purpose |
|---|---|
| `portfolio_summary` | **Portfolio Summary** — aggregate paper portfolio totals across all exchanges |
| `exchange_row` | **Exchange Row** — top-of-home row showing live connection and P&L status for each exchange |
| `alerts_feed` | **Alerts Feed** — recent audit log events and signal decisions shown as a live activity feed |
| `system_health` | **System Health** — compact health row: CPU, memory, disk, and bot status |

---

## Notification Plugins

| Plugin | Channel |
|---|---|
| `telegram` / `telegram_notification` | **Telegram** — trade alerts, daily P&L reports, AI decisions, CB alerts, market breadth extremes, weekly summary; configurable per-event toggles. Handles all inbound bot commands. |
| `push_notifications` | **Browser Push** — real-time Web Push API notifications to any subscribed browser (desktop or mobile). Configurable: trade executed, stop-loss triggered, drawdown limit, breadth alerts, daily summary, system health alerts. |
| `webhook_notification` | **Webhook** — POST trade events as JSON to any external URL. Discord-mode formats the payload as a Discord embed. Supports custom auth header. |

---

## Help Plugin

| Plugin | Purpose |
|---|---|
| `help_ask` | **Help AI Chat** — ask the local AI questions about your LUCIFER platform setup; answers with context from running bot state |

---

## Telemetry Plugin

| Plugin | Purpose |
|---|---|
| `github_telemetry` | **GitHub Telemetry** — daily 06:00 snapshot of changed source files and data pushed to the private GitHub repo for tester review |

---

## Indicator Plugins

These plugins compute technical indicators on price data and make them available to strategies.

| Plugin | Indicator |
|---|---|
| `rsi` | Relative Strength Index |
| `ema` | Exponential Moving Average (configurable periods) |
| `sma` | Simple Moving Average |
| `macd` | MACD line, signal line, histogram |
| `bollinger_bands` | Bollinger Bands with configurable stddev |
| `atr` | Average True Range |
| `adx` | Average Directional Index + DI+/DI- |
| `supertrend` | SuperTrend directional signal |
| `stochastic_rsi` | Stochastic RSI oscillator |
| `ichimoku` | Ichimoku Cloud (Tenkan, Kijun, Senkou A/B, Chikou) |
| `vwap` | Volume Weighted Average Price |
| `obv` | On-Balance Volume |
| `swing_failure_pattern` | SFP detection (liquidity sweep reversal) |
| `functional` | Utility indicator helpers |

---

## Strategy Plugins — Crypto (130+)

Strategies are grouped into logical families. All can be individually enabled, backtested, hyperopt-tuned, and certified before deployment. The `hyperopt` and `optimisation_lab` plugins run automated parameter search across any enabled strategy.

### Momentum & Trend Following
`adx_momentum_strategy` · `adx_trend_strategy` · `ema_crossover_strategy` · `ema_crossover_long_strategy` · `ema_crossover_medium_strategy` · `ema_scalp_strategy` · `donchian_strategy` · `donchian_turtle_strategy` · `breakout_strategy` · `atr_breakout_strategy` · `supertrend_strategy` · `supertrend_pullback_strategy` · `supertrend_momentum_scalper_strategy` · `elder_ray_strategy` · `momentum_strategy` · `momentum_rotation_strategy` · `trend_following_strategy` · `multi_tf_trend_strategy` · `trend_strength_regime_strategy` · `opening_range_breakout_strategy` · `session_open_momentum_strategy`

### Mean Reversion & Oscillators
`bollinger_strategy` · `bollinger_breakout_strategy` · `bollinger_scalp_strategy` · `bollinger_squeeze_strategy` · `keltner_bollinger_squeeze_strategy` · `keltner_reversion_strategy` · `rsi_strategy` · `rsi_divergence_strategy` · `rsi_momentum_strategy` · `rsi_scalp_strategy` · `stoch_rsi_strategy` · `stoch_rsi_macd_strategy` · `cci_strategy` · `awesome_oscillator_strategy` · `mfi_momentum_strategy` · `williams_r_strategy` · `tsi_strategy` · `rate_of_change_divergence_strategy` · `mean_reversion_strategy` · `z_score_reversion_strategy` · `swing_failure_pattern_strategy` · `swing_strategy` · `swing_ema_pullback_strategy` · `swing_channel_strategy` · `swing_support_resistance_strategy` · `swing_rsi_divergence_strategy`

### MACD Family
`macd_strategy` · `macd_histogram_strategy` · `macd_zero_line_strategy` · `swing_macd_weekly_strategy`

### VWAP Family
`vwap_strategy` · `vwap_rsi_strategy` · `vwap_scalp_strategy`

### Ichimoku & Chart Structure
`ichimoku_strategy` · `swing_ichimoku_strategy` · `parabolic_sar_strategy` · `pivot_point_strategy` · `fibonacci_retracement_strategy` · `market_structure_strategy`

### Grid Trading
`grid_strategy` · `grid_dynamic_strategy` · `grid_trailing_strategy`

### DCA & Position Building
`dca_strategy` · `dca_smart_strategy` · `dca_value_averaging_strategy` · `anti_martingale_strategy` · `martingale_limited_strategy` · `equal_weight_rebalance_strategy`

### Volatility & Regime
`volatility_regime_strategy` · `volatility_expansion_strategy` · `high_volatility_regime_strategy` · `low_volatility_accumulation_strategy` · `garch_volatility_strategy` · `regime_adaptive_strategy` · `regime_hmm_strategy` · `drawdown_regime_strategy` · `correlation_regime_strategy` · `momentum_regime_strategy` · `seasonal_regime_strategy` · `macro_regime_strategy`

### AI & Machine Learning
`ai_strategy` · `ensemble_voting_strategy` · `anomaly_detection_strategy` · `reinforcement_learning_strategy` · `random_forest_signal_strategy` · `lstm_prediction_strategy` · `xgboost_strategy` · `neural_network_strategy` · `genetic_algorithm_strategy` · `kalman_filter_strategy` · `fourier_cycle_strategy` · `entropy_strategy`

### Quantitative & Portfolio
`black_litterman_strategy` · `copula_strategy` · `mean_variance_strategy` · `min_volatility_strategy` · `max_sharpe_strategy` · `risk_parity_strategy` · `kelly_criterion_strategy` · `hurst_exponent_strategy` · `stat_arb_strategy` · `cointegration_strategy` · `pairs_trading_strategy`

### Arbitrage & Rotation
`exchange_arb_strategy` · `dex_arb_strategy` · `triangular_arb_strategy` · `latency_arb_strategy` · `futures_spot_arb_strategy` · `funding_rate_arb_strategy` · `dominance_rotation_strategy` · `sector_rotation_strategy` · `multi_asset_rotation_strategy`

### Smart Money Concepts (SMC) & Order Flow
`breaker_block_strategy` · `equal_highs_lows_strategy` · `order_block_strategy` · `liquidity_sweep_strategy` · `fair_value_gap_strategy` · `wyckoff_strategy` · `supply_demand_zone_strategy` · `institutional_candle_strategy` · `smart_money_divergence_strategy` · `order_flow_scalp_strategy` · `microstructure_scalp_strategy`

### OBV & Volume
`obv_momentum_strategy` · `swing_volume_breakout_strategy` · `social_volume_strategy` · `on_chain_strategy` · `whale_alert_strategy`

### Sentiment & Events
`fear_greed_strategy` · `funding_sentiment_strategy` · `sentiment_nlp_strategy` · `news_event_breakout_strategy` · `flash_crash_buy_strategy` · `liquidation_cascade_strategy`

### Scalping
`spread_scalp_strategy` · `tick_scalp_strategy` · `short_scalp_strategy`

### Tools & Research
`strategy_lab` · `optimisation_lab` · `hyperopt` · `rsi_backtest` · `research_alias_strategies` · `trading_engine`

---

## Strategy Plugins — Commodity / CFD (30+)

Purpose-built for IG Markets CFD trading across energies, metals, grains, and soft commodities.

### Trend Following
`cfd_trend_following_strategy` · `commodity_trend_engine` · `commodity_momentum_rotation_strategy` · `cfd_breakout_retest_strategy`

### Mean Reversion & Range
`cfd_mean_reversion_strategy` · `cfd_range_strategy` · `cfd_scalp_strategy`

### Carry & Term Structure
`cfd_carry_trade_strategy` · `commodity_carry_engine`

### Calendar Spreads
`commodity_calendar_spreads_engine`

### Relative Value
`cfd_correlation_strategy` · `commodity_relative_value_engine`

### Seasonality
`commodity_seasonal_engine`

### Fundamental / Macro
`cfd_news_straddle_strategy`

### Risk Management for CFDs
`cfd_hedging_strategy` · `cfd_gap_trading_strategy`

---

## Service Plugins

| Plugin | Function |
|---|---|
| `exchange_ledger_sync` | Syncs executed trades from exchange APIs into the local ledger |

---

## Plugin Doctor

The Plugin Doctor (`system/plugin_doctor`) is a built-in 3-LLM diagnostics and self-healing tool accessible from the Settings workspace. It runs every 5 minutes and:

- Scans all 280+ plugins for missing required files (`plugin.json`, `api.py`, `widget.jsx`)
- Detects category mismatches between `plugin.json` and folder location
- Checks blueprint compliance gaps (missing required fields, wrong URL prefix)
- Catches import errors surfaced at startup
- Diagnoses root cause via LLM, proposes and applies a fix
- Requests a targeted service restart via `data/restart_requests/`
- Reports plugin enabled vs disabled count by category

Warnings are colour-coded: **red** for errors that will prevent a plugin working, **amber** for compliance warnings, **green** for passing checks.

**Protected files** (bot.py, supervisor.ps1, data/*.json) are diagnosed but never auto-repaired — Plugin Doctor reports the issue and leaves the fix to the operator.

---

## Adding a New Plugin

1. Create `src/plugins/<category>/<plugin_name>/`
2. Add `plugin.json` (see template below)
3. Add `api.py` with a Flask Blueprint at `/api/<plugin_name>`
4. Add `widget.jsx` exporting a named component matching `widget.component`
5. Restart the dashboard (or hot-reload if in dev mode)

```json
{
  "name": "my_plugin",
  "label": "My Plugin",
  "description": "What this plugin does.",
  "version": "1.0.0",
  "author": "LUCIFER",
  "enabled": true,
  "certified": false,
  "category": "data_sources",
  "widget": {
    "component": "MyPluginWidget",
    "slot": "data-my-plugin",
    "file": "widget.jsx"
  }
}
```

The platform picks it up automatically — no registration required.
