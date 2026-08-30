# LUCIFER Plugin Catalogue

> **300+ active plugins** · Auto-discovered at startup · No restart required to add a new plugin

LUCIFER uses a WordPress-style plugin architecture. Every feature is a plugin: exchanges, AI agents, data sources, indicators, strategies, risk rules, and dashboard widgets. Adding a plugin means dropping files into `src/plugins/<category>/<name>/`:

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
| `coinspot` / `coinspot_exchange` | CoinSpot | Crypto (AUD) | V2 REST API · Market + limit orders · Live trading active |
| `swyftx` / `swyftx_exchange` | Swyftx | Crypto (AUD) | JWT auth · Full order management · Live trading active |
| `bybit` / `bybit_exchange` | Bybit | Crypto (USDT) | V5 API · Decommissioned — paper only |
| `ig` / `ig_exchange` | IG Markets | CFDs / Commodities | REST API · Tiered holdings · Paper mode active |
| `paper_exchange` | Paper Trading | All asset classes | Simulates all exchanges with real market data |
| `binance_exchange` | Binance | Crypto | Stub — not yet enabled |
| `kraken_exchange` | Kraken | Crypto | Stub — not yet enabled |
| `ib_exchange` | Interactive Brokers | Equities / Futures | Stub — not yet enabled |

---

## AI Brain Plugins

The AI Brain workspace — debates, console, metrics, and analysis tools.

| Plugin | Purpose |
|---|---|
| `brain_console` | **Brain Console** — live terminal feed of every AI Brain decision, exit debate, executor order, and log line. Colour-coded by action. Tabs: All / Decisions / Log / Queue. |
| `brain_metrics` | **Brain Metrics** — equity curve, win rate by agent, regime distribution, debate duration histogram, and A/B prompt performance over time. |
| `debate_viewer` | **Debate Viewer** — full-screen live terminal UI streaming all 8 agent verdicts and reasoning as they complete. Signal history sidebar (last 50 debates), debate replay, live progress counter (X / 8 agents), mobile-responsive collapsible history drawer. |
| `brain_conviction_trend` | **Conviction Trend** — charts the Brain's confidence score over time across all instruments, highlighting when conviction is rising or falling across the watchlist. |
| `correlation_guard` | **Correlation Guard** — monitors Pearson correlation between all open positions; flags pairs with r > 0.85 (dangerously correlated). Blocks new buys into already-correlated coins. |
| `post_mortem` | **Post Mortem** — deep-dive analysis of closed trades: entry context vs actual outcome, which agents called it correctly, what the ReasoningBank learned from it. |

### AI Brain Architecture

The AI Brain is the platform's autonomous trading intelligence. All inference runs locally on Ollama — `qwen3:8b` for 6 real-time debate agents, `qwen2.5:14b` for the Risk Manager and nightly reflection. A dedicated **Shared Debate Daemon** process handles all debate execution — bots never call the debate engine directly.

**Data flow:**
```
Market data + watchlist + ReasoningBank
        ↓
  MTF Gate (1h / 4h / 6h alignment check)
        ↓  pass (≥ 2/3 bullish)
  Bot writes request → data/debate_queue.json
        ↓
  Debate Daemon (debate_service.py) picks up within 5s
  Deduplicates by instrument (15-min cooldown)
        ↓
  8-Agent Debate Panel (sequential)
  ┌─ Technical Analyst   (qwen3:8b)
  ├─ Chart Analyst       (qwen3:8b)  ← EMA stack, SuperTrend, VWAP, momentum
  ├─ Sentiment Analyst   (qwen3:8b)
  ├─ Fundamental Analyst (qwen3:8b)  ← portfolio, cash, exposure
  ├─ Bull Researcher     (qwen3:8b)
  ├─ Bear Researcher     (qwen3:8b)  ← COT signal, spread cost
  ├─ Risk Manager        (qwen2.5:14b) ← ReasoningBank, COT, financing, ATR sizing
  └─ Portfolio Manager   (qwen3:8b)  ← calculates size, writes order
        ↓
  Result written → data/debate_results/{instrument}.json
        ↓
  Bot polls result file → executes on fresh BUY
        ↓
  BrainExecutor thread → place_buy() → exchange
```

**Key files:**

| File | Purpose |
|---|---|
| `services/debate_service.py` | Shared Debate Daemon — sole process that runs all debates; reads queue, deduplicates, writes results |
| `services/ai_brain_loop.py` | Main autonomous loop — gates, cooldown, queue submission, result polling |
| `services/ai_decision_engine.py` | 8-agent debate pipeline |
| `services/ai_context_builder.py` | Builds full market context for each debate (exchange-specific portfolio) |
| `services/chart_analyst.py` | Chart Analyst (Agent 1b) — EMA stack, SuperTrend, VWAP, Bollinger, trend lines |
| `services/exit_debate.py` | 2-agent exit debate engine (replaces fixed stop-loss) |
| `services/reasoning_bank.py` | Local RAG memory — add / retrieve / update learned patterns |
| `services/decision_ledger.py` | Full audit trail of every Brain decision and outcome |
| `services/reflection_engine.py` | 2am nightly reflection — extracts patterns into ReasoningBank |
| `services/watchlist_mtf.py` | Multi-timeframe signals from watchlist price array (no external API) |
| `services/watchlist_discovery.py` | Hourly paper evaluation of watchlist candidates |
| `data/debate_queue.json` | Bots write debate requests here |
| `data/debate_results/` | Daemon writes one result file per instrument |
| `data/brain_orders.json` | Live buy queue — executor reads and places orders |
| `data/brain_orders_ig.json` | Live buy queue for IG Markets |
| `data/reasoning_bank.json` | Accumulated learned patterns |
| `data/decision_ledger.json` | Full decision history with outcomes |

---

## AI Plugins

General AI and LLM integration plugins.

| Plugin | Purpose |
|---|---|
| `ollama` | Core Ollama integration — routes all LLM calls to local Ollama models. No cloud API calls are ever made. |
| `local_ollama_advisor` | Pre-trade gate — validates every trade with structured LLM reasoning. Superseded by the Brain for buy decisions. |
| `claude_advisor` | Cloud AI gate — **disabled** (zero cloud API policy; local Ollama only) |
| `ensemble` / `ai_ensemble` | Ensemble voting — combines multiple strategy signals into a weighted decision |
| `ai_chat` / `ai_chat_assistant` | **AI Chat** — persistent conversation history, Ollama streaming, live trading context (portfolio, positions, Brain state), model selector, quick prompts, mobile sidebar |
| `llm_prompts` | **LLM Prompt Editor** — edit all AI system prompts from the dashboard without touching Python files. Changes take effect immediately, no restart. |
| `ab_testing` | **A/B Testing** — autonomous prompt variant testing using Thompson sampling. Brain creates, tests, promotes, and retires its own agent prompts nightly. |
| `ai_tuner` | **AI Tuner** — fine-tune all AI agent parameters: model selection, confidence thresholds, agent weights, timeouts. |

---

## Mission Control Plugins

Widgets and tools for the Mission Control workspace — the platform's primary operational dashboard.

| Plugin | Purpose |
|---|---|
| `executive` | **Executive Summary** — top-level portfolio snapshot: total value, daily P&L, open positions count, cash available, and circuit breaker status. |
| `ai_brain_mc` | **AI Brain Widget** — Mission Control status card showing last decision, open positions, MTF gate result, and opportunity score. |
| `brain_console` | **Brain Console** (MC slot) — live decision feed on Mission Control. |
| `paper_comparison` | **Paper Bot Comparison** — side-by-side Legacy (rules-only) vs AI Brain paper bot: total value, realised/unrealised P&L, cash, win rate, trades, and holdings. |
| `process_control` | **Process Control** — restart any bot, the dashboard, or all services remotely from Mission Control. Writes `.restart` files to `data/restart_requests/` — supervisor executes within 60s. No RDP required. |
| `nathans_test_trading` | **Nathan's Test Trading** — side-by-side IG demo and Swyftx demo account panels: balance, total/realised/unrealised P&L, open positions with sizing bars, recent trades, win rate. Auto-refreshes every 30s. |
| `exec_health` | **System Health** — bot process list (all supervised processes with PIDs), exchange connection status, database health, circuit breaker state, risk mode, system resources (CPU/RAM/disk). Reset Peak button for circuit breaker. |
| `cot_signals` | **COT Signals** — CFTC Commitments of Traders crowding signals for all tracked commodities, metals, and CME crypto futures. Separate sections for CME Crypto and Commodities & Metals. |
| `exchange_coinspot_live` | **CoinSpot Live** — live exchange account widget: balance, open orders, recent fills. |
| `exchange_ig_live` | **IG Markets Live** — live IG account widget: open positions, margin used, account balance. |
| `paper_trading` | **Paper Trading** — paper portfolio overview: cash, holdings, unrealised P&L, paper trade history. |
| `paper_settings` | **Paper Settings** — configure paper trading parameters per exchange: DCA amounts, position sizes, risk limits. |
| `paper_pnl` | **Paper P&L** — detailed P&L breakdown for paper portfolios across all exchanges. |
| `paper_analytics` | **Paper Analytics** — win rate, average gain/loss, Sharpe, drawdown for paper trading sessions. |
| `moonbag_tracker` | **Moonbag Tracker** — tracks small residual holdings (moonbags) left after partial sells; monitors their unrealised P&L over time. |
| `maintenance_mode` | **Maintenance Mode** — global pause switch that halts all new buys across all bots with a single toggle and a reason message. |
| `ai_activity` | **AI Activity** — recent AI Brain events feed: last debate, last decision, executor status. |
| `ai_decisions` | **AI Decisions** — ledger of all Brain decisions with filters by coin, verdict, date, and confidence. |
| `departments` | **Departments** — bot department health overview (if department manager is active). |
| `dept_health` | **Department Health** — per-department status indicators. |
| `open_trades` | **Open Trades** — all currently open positions across all exchanges with entry price, current price, P&L, and stop distance. |
| `portfolio_mc` | **Portfolio** — portfolio composition pie chart, allocation percentages, concentration risk. |
| `markets_mc` | **Markets** — quick market snapshot on Mission Control: top movers, Fear & Greed, breadth indicator. |
| `research_agent` | **Research Agent Status** — last research run, current project, research queue length. |
| `resource_monitor` | **Resource Monitor** — CPU, RAM, disk usage bars with historical trend. |
| `rss_news` | **News Feed** — latest crypto and macro news headlines on Mission Control. |
| `tracked_coins` | **Tracked Coins** — list of all coins currently being monitored by the bot with their current signal state. |
| `stop_loss_exposure` | **Stop Loss Exposure** — total AUD at risk across all open positions if all stop-losses triggered simultaneously. |
| `alerts_mc` | **Alerts** — active priority interrupt alerts and system warnings on Mission Control. |
| `bot_status` | **Bot Status** — live status indicator for each running bot process. |
| `exchange_bybit` | Bybit exchange summary widget. |
| `exchange_coinspot` | CoinSpot exchange summary widget. |
| `exchange_ig` | IG Markets exchange summary widget. |
| `exchange_swyftx` | Swyftx exchange summary widget. |
| `exec_security` | **Security** — active sessions, recent auth events, failed login count. |
| `ai_briefing` | **AI Briefing** — daily AI-generated market briefing injected at session start. |

---

## Research Plugins

| Plugin | Purpose |
|---|---|
| `research_centre` | **Research Centre** — AI research agent control panel: run a research project, view queue, read findings, browse research history. |
| `research_lab` | **Research Lab** — free-form research environment: query the AI agent directly, explore strategy hypotheses. |
| `ai_chat` | **AI Chat** (Research slot) — same persistent chat interface as Mission Control, contextualised to research queries. |
| `hurst_exponent` | **Hurst Exponent** — measures mean-reversion vs trend-persistence of any coin or instrument over a configurable lookback. |
| `monte_carlo` | **Monte Carlo Simulation** — runs N simulations of portfolio outcomes given historical return distribution. |
| `rule_significance` | **Rule Significance** — statistical significance testing for strategy rules: are these signals actually predictive or random? |
| `market_breadth` | **Market Breadth** — percentage of tracked coins above EMA-20/50, advance/decline ratio, breadth momentum. Feeds Telegram breadth alerts. |
| `strategy_replay` | **Strategy Replay** — replays any strategy against historical data with a step-through UI to visualise each signal. |

---

## Trading Plugins

| Plugin | Purpose |
|---|---|
| `chart` | **Charts** — candlestick charts with configurable interval, RSI overlay, EMA lines, volume bars, and trade markers. |
| `markets` | **Markets** — live price feed for all tracked coins: current price, RSI, 24h change, volume. |
| `holdings` | **Holdings** — all open lots with FIFO P&L, entry date, stop-loss level, AI exit signal, and manual sell controls. |
| `watchlist` | **Watchlist** — 72-coin discovery watchlist with scores, paper debate status, paper win/loss record, and promotion status. |
| `trade_journal` | **Trade Journal** — full closed trade history across all exchanges with filters, CSV export, and P&L summary. |
| `accountancy` | **Accountancy** — tax-year P&L summaries, realised gains/losses, fee totals, export for tax reporting. |
| `ig_markets` | **IG Markets** — full IG CFD interface: open positions, dealing rules, ATR sizer tab, spread tracker, overnight financing, COT signals. |

---

## System / Engineering Plugins

| Plugin | Purpose |
|---|---|
| `plugin_doctor` | **Plugin Doctor v2** — 3-LLM autonomous self-healing engine. Scans all plugins every 5 min for missing files, import errors, blueprint compliance. Diagnoses root cause, proposes and applies fixes. Protected files (bot.py, data/*.json, supervisor.ps1) are diagnosed but never auto-repaired. |
| `engineering` | **Engineering** — blueprint health, tech debt tracker, migration progress, event bus status. |
| `audit_log` | **Audit Log** — full log of every settings change, auth event, manual trade, and plugin install. |
| `logs` | **Logs** — live log viewer for all bot log files with search, filter by level, and tail mode. |
| `bot_settings` | **Bot Settings** — per-bot configuration editor: exchange, coins, risk limits, strategy selection. |
| `certification` | **Strategy Certification** — formal pass/fail gate for strategies before they reach production: backtest criteria, minimum win rate, max drawdown. |
| `workspace_manager` | **Workspace Manager** — manage dashboard workspace layouts, slot assignments, and widget visibility. |
| `telegram_log` | **Telegram Log** — history of all Telegram messages sent by the bot. |
| `webhooks` | **Webhooks** (system) — configure outbound webhooks for system events. |

---

## Risk Plugins

| Plugin | Purpose |
|---|---|
| `risk_monitor` | **Risk Monitor** — live risk dashboard: current drawdown, daily P&L limit status, position concentration, consecutive loss count. |
| `health` | **Risk Health** — risk framework health check: all risk rules loaded and active, thresholds within expected ranges. |
| `health_monitor` | **Health Monitor** — continuous background health checker; writes to `data/risk_state.json`. |
| `alert_engine` | **Alert Engine** — configurable rule-based alerts: price moves, P&L thresholds, drawdown levels, Brain inactivity. |

---

## Home / Portal Plugins

| Plugin | Purpose |
|---|---|
| `portfolio_summary` | Portfolio overview card on the home/login portal. |
| `exchange_row` | Per-exchange status row: connected, balance, last trade. |
| `alerts_feed` | Live alert feed on the home portal. |
| `system_health` | System health indicator on the home portal — CPU temp, bot status, circuit breaker state. |

---

## Performance Plugins

| Plugin | Purpose |
|---|---|
| `analytics` | **Analytics** — detailed performance analytics: Sharpe ratio, Sortino, max drawdown, win rate by strategy, by coin, by time of day. |
| `reports` | **Reports** — printable performance reports with CSV export and date-range filtering. |
| `tax` | **Tax** — ATO-compliant capital gains summaries: realised gains, cost basis, holding period classification. |
| `auto_weekly_report` | **Weekly Report** — auto-generated weekly P&L and performance summary delivered via Telegram every Monday morning. |

---

## Data Source Plugins

| Plugin | Data Provided |
|---|---|
| `fear_greed` | Crypto Fear & Greed Index (Alternative.me) |
| `crypto_news` | Latest crypto news aggregated from multiple feeds |
| `rss_crypto_news` | RSS-based crypto news parser |
| `macro_news` | Macroeconomic and geopolitical news |
| `sentiment_snapshot` | Point-in-time market sentiment composite |
| `reddit_sentiment` | Reddit crypto subreddit sentiment scoring |
| `influencer_feed` | Crypto influencer post monitoring |
| `coingecko_intel` | CoinGecko market intelligence — trending, top gainers/losers |
| `market_chart` | OHLC price data and chart feeds |
| `deribit_feed` | Deribit options data — IV, put/call ratio |
| `derivatives_feed` | Futures funding rates and open interest |
| `onchain_macro_feed` | On-chain macro indicators (whale movements, exchange flows) |
| `system_health_monitor` | CPU, RAM, disk, process health monitoring |
| `interrupt_agent` | **Priority Interrupt Agent** — 3-tier background service polling macro events (ForexFactory), exchange health, price moves, circuit breaker, drawdown, and IG session boundary every 30s. Injects urgency-coloured alerts into every Brain debate. |
| `macro_calendar` | **Macro Calendar** — ForexFactory economic calendar with auto-populate every 6 hours. Emits IG market open/close events including Saturday 07:00 AEST weekend session. |
| `myfxbook_sentiment` | Myfxbook crowd positioning data for forex/CFD instruments. |

---

## Indicator Plugins

| Plugin | Indicator |
|---|---|
| `rsi` | Relative Strength Index |
| `ema` | Exponential Moving Average |
| `sma` | Simple Moving Average |
| `macd` | MACD line, signal line, histogram |
| `bollinger_bands` | Bollinger Bands |
| `atr` | Average True Range |
| `adx` | Average Directional Index + DI+/DI- |
| `supertrend` | SuperTrend directional signal |
| `stochastic_rsi` | Stochastic RSI oscillator |
| `ichimoku` | Ichimoku Cloud |
| `vwap` | Volume Weighted Average Price |
| `obv` | On-Balance Volume |
| `swing_failure_pattern` | SFP detection — liquidity sweep reversal |
| `functional` | Utility indicator helpers |

---

## Risk Rule Plugins

| Plugin | Function |
|---|---|
| `core` | Core risk rules — exposure limits, daily drawdown, position count caps |
| `fear_dca` | Fear-weighted DCA — increases buy size in Extreme Fear, reduces in Extreme Greed |
| `dca` | Standard DCA — dollar-cost averaging on dips |
| `trailing_stop` | Adaptive trailing stop — tiered trail percentages activating at 12%, tightening at 20% and 30% gains |
| `sell_decision` | Lot-based sell decision engine — FIFO P&L, take-profit targeting |

---

## Notification Plugins

| Plugin | Channel |
|---|---|
| `telegram` / `telegram_notification` | Telegram Bot — trade alerts, daily reports, AI decisions, breadth alerts, 2FA codes |
| `webhook_notification` | Generic webhook — POST trade events to any endpoint |
| `push_notifications` | Browser push notifications (experimental) |

---

## Strategy Plugins — Crypto (155+)

Strategies are grouped into logical families. All can be individually enabled, backtested, hyperopt-tuned, and certified.

### Momentum & Trend Following
`adx_momentum_strategy` · `adx_trend_strategy` · `ema_crossover_strategy` · `ema_crossover_long_strategy` · `ema_crossover_medium_strategy` · `ema_scalp_strategy` · `donchian_strategy` · `donchian_turtle_strategy` · `breakout_strategy` · `atr_breakout_strategy` · `supertrend_strategy` · `supertrend_momentum_scalper_strategy` · `supertrend_pullback_strategy` · `elder_ray_strategy` · `momentum_strategy` · `momentum_rotation_strategy` · `momentum_regime_strategy` · `multi_tf_trend_strategy` · `trend_following_strategy` · `trend_strength_regime_strategy` · `session_open_momentum_strategy` · `opening_range_breakout_strategy`

### Mean Reversion & Oscillators
`bollinger_strategy` · `bollinger_breakout_strategy` · `bollinger_scalp_strategy` · `bollinger_squeeze_strategy` · `keltner_bollinger_squeeze_strategy` · `keltner_reversion_strategy` · `rsi_strategy` · `rsi_momentum_strategy` · `rsi_scalp_strategy` · `rsi_divergence_strategy` · `stoch_rsi_strategy` · `stoch_rsi_macd_strategy` · `cci_strategy` · `awesome_oscillator_strategy` · `williams_r_strategy` · `tsi_strategy` · `rate_of_change_divergence_strategy` · `mean_reversion_strategy` · `z_score_reversion_strategy` · `swing_failure_pattern_strategy` · `swing_strategy` · `swing_channel_strategy` · `swing_ema_pullback_strategy` · `swing_rsi_divergence_strategy` · `swing_support_resistance_strategy` · `swing_volume_breakout_strategy` · `swing_macd_weekly_strategy` · `swing_ichimoku_strategy`

### DCA & Position Building
`dca_strategy` · `dca_smart_strategy` · `dca_value_averaging_strategy` · `anti_martingale_strategy` · `martingale_limited_strategy` · `equal_weight_rebalance_strategy`

### AI & Machine Learning
`ai_strategy` · `ensemble_voting_strategy` · `anomaly_detection_strategy` · `neural_network_strategy` · `reinforcement_learning_strategy` · `lstm_prediction_strategy` · `random_forest_signal_strategy` · `xgboost_strategy` · `kalman_filter_strategy` · `genetic_algorithm_strategy` · `garch_volatility_strategy` · `fourier_cycle_strategy` · `black_litterman_strategy` · `copula_strategy` · `entropy_strategy`

### Regime-Aware
`regime_adaptive_strategy` · `regime_hmm_strategy` · `drawdown_regime_strategy` · `correlation_regime_strategy` · `volatility_regime_strategy` · `high_volatility_regime_strategy` · `low_volatility_accumulation_strategy` · `volatility_expansion_strategy` · `macro_regime_strategy` · `market_structure_strategy` · `seasonal_regime_strategy` · `momentum_regime_strategy`

### Arbitrage & Rotation
`exchange_arb_strategy` · `dex_arb_strategy` · `futures_spot_arb_strategy` · `funding_rate_arb_strategy` · `latency_arb_strategy` · `triangular_arb_strategy` · `stat_arb_strategy` · `pairs_trading_strategy` · `cointegration_strategy` · `dominance_rotation_strategy` · `sector_rotation_strategy` · `multi_asset_rotation_strategy` · `commodity_momentum_rotation_strategy`

### Advanced & SMC (Smart Money Concepts)
`breaker_block_strategy` · `equal_highs_lows_strategy` · `order_block_strategy` · `liquidity_sweep_strategy` · `liquidation_cascade_strategy` · `fair_value_gap_strategy` · `supply_demand_zone_strategy` · `wyckoff_strategy` · `institutional_candle_strategy` · `smart_money_divergence_strategy` · `market_structure_strategy` · `order_flow_scalp_strategy` · `microstructure_scalp_strategy`

### Statistical & Research
`hurst_exponent_strategy` · `monte_carlo_strategy` · `rule_significance_strategy` · `mean_variance_strategy` · `max_sharpe_strategy` · `min_volatility_strategy` · `risk_parity_strategy` · `kelly_criterion_strategy` · `black_litterman_strategy`

### Technical & Chart Patterns
`macd_strategy` · `macd_histogram_strategy` · `macd_zero_line_strategy` · `obv_momentum_strategy` · `mfi_momentum_strategy` · `parabolic_sar_strategy` · `pivot_point_strategy` · `fibonacci_retracement_strategy` · `ichimoku_strategy` · `vwap_strategy` · `vwap_rsi_strategy` · `vwap_scalp_strategy` · `sma_crossover_long_strategy` · `sma_crossover_short_strategy`

### Sentiment & Alternative Data
`fear_greed_strategy` · `funding_sentiment_strategy` · `sentiment_nlp_strategy` · `social_volume_strategy` · `whale_alert_strategy` · `on_chain_strategy` · `news_event_breakout_strategy`

### Grid & Scalping
`grid_strategy` · `grid_dynamic_strategy` · `grid_trailing_strategy` · `tick_scalp_strategy` · `spread_scalp_strategy` · `short_scalp_strategy` · `flash_crash_buy_strategy`

---

## Strategy Plugins — Commodity / CFD (30+)

Purpose-built for IG Markets CFD trading across energies, metals, grains, and soft commodities.

### Trend Following
`commodity_trend_engine` · `cfd_trend_following_strategy` · `commodity_momentum_rotation_strategy` · `cfd_breakout_retest_strategy`

### Mean Reversion & Range
`cfd_mean_reversion_strategy` · `cfd_range_strategy` · `cfd_scalp_strategy`

### Carry & Term Structure
`cfd_carry_trade_strategy` · `commodity_carry_engine` · `commodity_term_structure_strategy`

### Calendar Spreads
`commodity_calendar_spreads_engine` · `seasonal_roll_strategy`

### Relative Value
`cfd_correlation_strategy` · `commodity_relative_value_engine` · `inter_commodity_spread_strategy`

### Seasonality
`commodity_seasonal_engine` · `agricultural_seasonal_strategy` · `energy_seasonal_strategy`

### Fundamental / Macro
`commodity_fundamentals_strategy` · `supply_demand_strategy` · `cfd_news_straddle_strategy`

### Risk Management for CFDs
`cfd_hedging_strategy` · `cfd_gap_trading_strategy`

---

## Plugin Manager

The Plugin Manager (`/settings/plugins`) is the central hub for managing all installed plugins. Features:

- **Browse & search** — full catalogue view with category filter, search, and enabled/disabled toggle
- **Install from zip** — drop a plugin zip onto the upload area to install or overwrite an existing plugin; install endpoint returns a `requires_bot_restart` flag and the Bot Restart Panel appears automatically
- **Plugin Doctor** — inline diagnostics: scans all plugins and reports missing files, import errors, and compliance gaps in real time
- **Bot Restart Panel** — polls `GET /api/registry/restart` every 10s; shows which bots have a pending restart request and their queue status

---

## Plugin Doctor

The Plugin Doctor is a built-in autonomous self-healing engine running every 5 minutes:

- Scans all 300+ plugins for missing files, import errors, and blueprint compliance gaps
- Diagnoses root cause via LLM (`qwen3:8b` scanner · `qwen2.5:14b` deep reviewer · `mistral-nemo:12b` final reviewer)
- Proposes and applies fixes automatically
- Requests targeted service restart via `data/restart_requests/`
- **Protected files** (bot.py, ig_bot.py, data/*.json, supervisor.ps1) are diagnosed but never auto-repaired

---

## Adding a New Plugin

1. Create `src/plugins/<category>/<plugin_name>/`
2. Add `plugin.json` (see template below)
3. Add `api.py` with a Flask Blueprint at `/api/<plugin_name>`
4. Add `widget.jsx` exporting a named component matching `widget.component`
5. The platform picks it up automatically — no registration, no restart required

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
