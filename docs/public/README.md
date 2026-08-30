# LUCIFER — The AI Multi-Trading Platform

> **Version**: v2.6 &nbsp;|&nbsp; **Updated**: August 2026 &nbsp;|&nbsp; **Status**: Live Trading

LUCIFER is a self-hosted, plugin-based automated trading platform for cryptocurrency and CFD/commodity markets. It runs 24/7 on a dedicated Windows machine, trades across multiple exchanges simultaneously, and manages risk through a layered AI gate, strategy engine, and research pipeline.

---

## What It Does

| Capability | Detail |
|---|---|
| **Multi-exchange trading** | CoinSpot, Swyftx, Bybit (crypto) · IG Markets (CFDs/commodities) |
| **AI trading brain** | 8-agent autonomous brain — sole buy authority; intelligent exit debates consult ReasoningBank, MTF, regime, breadth before cutting a loss |
| **Strategy engine** | 139+ strategies across crypto and commodity asset classes |
| **Research pipeline** | Autonomous AI research agent, backtester, statistical tools, strategy certification |
| **Risk management** | Intelligent exit debates, trailing stops, DCA, drawdown limits, regime detection — no mechanical fixed stop-losses |
| **IG Markets intelligence** | COT positioning data, overnight financing tracking, bid/offer spread monitoring, ATR-based position sizing — all injected into every IG Brain debate |
| **Swyftx demo bot** | Isolated demo account running the same Brain/strategy pipeline as live — separate data store, side-by-side P&L vs IG demo in Nathan's Test Trading dashboard |
| **Dashboard** | HTTPS web dashboard — Mission Control, Charts, Research, Strategies, Settings |
| **Process Control** | Restart any bot, the dashboard, or all services remotely from Mission Control — no RDP required |
| **Priority interrupt** | 3-tier interrupt agent monitors macro events, exchange health, and price moves; injects alerts directly into Brain debates |
| **Plugin system** | WordPress-style auto-discovery — 280+ active plugins, no restart to add new ones |
| **Telegram alerts** | Trade signals, daily P&L reports, AI decisions, breadth alerts |
| **Paper trading** | Full dry-run mode — real data, no real money |
| **IG Markets holdings** | Tiered commodity positions tracked separately with full P&L, overnight financing, and live spread data |

---

## Screenshots

### AI Brain Console
Live terminal feed of AI Brain decisions, debates, and executor activity. Colour-coded by action type — green for BUY, yellow for HOLD, red for EXIT, purple for DEBATE.

![AI Brain Console](https://raw.githubusercontent.com/mirageco69/Lucifer-Multi-AI-Trading-Bot/main/Images/AI%20Brain%20Console%201.png)

### AI Brain Widget
Mission Control Brain status widget — last decision, open positions, MTF gate status, and opportunity score at a glance.

![AI Brain Widget](https://raw.githubusercontent.com/mirageco69/Lucifer-Multi-AI-Trading-Bot/main/Images/AI%20Brain%20Widget.png)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        LUCIFER Platform                         │
├────────────────┬────────────────┬──────────────────────────────┤
│   Trading      │   Research     │   Dashboard                   │
│   Engine       │   Pipeline     │   (React + Flask)             │
│                │                │                               │
│  bot.py        │  research/     │  dashboard-new/               │
│  services/     │  backtester/   │  src/plugins/*/widget.jsx     │
│  strategies/   │  optimize/     │  src/api/                     │
├────────────────┴────────────────┴──────────────────────────────┤
│                     Plugin System                               │
│  280+ plugins auto-discovered from src/plugins/**              │
│  Each plugin: plugin.json + api.py + widget.jsx                │
├────────────────────────────────────────────────────────────────┤
│              Exchanges                │   AI / LLM              │
│  CoinSpot · Swyftx · Bybit · IG      │  Ollama (local only)    │
│  Paper exchange (dry-run)             │  qwen3:8b · qwen2.5:14b │
└────────────────────────────────────────────────────────────────┘
```

---

## Key Features

### Multi-Exchange Support
Trade the same strategy across CoinSpot (crypto, AUD), Swyftx (crypto, AUD), Bybit (crypto, USDT), and IG Markets (CFDs/commodities) simultaneously. Each exchange has its own plugin with independent lot tracking, P&L, and order management.

### 8-Agent Autonomous AI Brain

LUCIFER's AI Brain is a fully autonomous trading intelligence built entirely on local LLMs — no cloud API calls, no subscriptions, no data leaving your machine. It runs 24/7 on [Ollama](https://ollama.com), debating every potential trade through a structured 8-agent panel before a single order is placed.

#### The Brain is the sole buy authority

When the Brain is enabled, the bot's own signal-triggered buys are completely blocked. Every buy decision flows through the debate pipeline and is placed by a dedicated executor thread — not the strategy engine. This eliminates conflicting buy paths and ensures every open position has a full reasoning trail behind it.

#### The 7-Agent Debate Panel

Each trade candidate triggers a sequential debate across seven specialist agents, with a final Risk Manager adjudicating:

| Agent | Model | Role |
|---|---|---|
| **Technical Analyst** | qwen3:8b | RSI, MACD, EMA, Bollinger, multi-timeframe alignment |
| **Chart Analyst** | qwen3:8b | EMA stack, SuperTrend, VWAP, Bollinger squeeze, trend lines, momentum — reads the chart structure |
| **Sentiment Analyst** | qwen3:8b | Fear & Greed index, crypto news sentiment, macro events |
| **Fundamental Analyst** | qwen3:8b | Portfolio exposure, open positions, available cash, correlation |
| **Bull Researcher** | qwen3:8b | Builds the strongest possible case FOR entering the trade |
| **Bear Researcher** | qwen3:8b | Builds the strongest possible case AGAINST entering the trade |
| **Risk Manager** | qwen2.5:14b | Reads all agent reports + ReasoningBank history. Final veto — cannot be overruled |
| **Portfolio Manager** | qwen3:8b | Calculates position size (ATR + Kelly criterion), writes the execution order |

The debate takes 3–8 minutes per coin. If Ollama is busy or slow, the Brain waits and retries — it never skips the debate to place a faster order.

#### Multi-Timeframe Gate

Before the debate even starts, the Brain checks multi-timeframe alignment across 1h, 4h, and 6h timeframes derived from the watchlist's live price array (no external API calls). A coin must be bullish on at least 2 of 3 timeframes to proceed. `WEAK_BULLISH (score +1/3)` means the gate is blocked — the Brain checks again on the next cycle rather than debating a weak setup.

#### Intelligent Exit Debates

Fixed stop-losses are replaced entirely by AI-driven exit debates. When an open position hits a loss threshold, the Brain runs a 2-agent debate before deciding whether to cut or hold:

- **Recovery Analyst** — argues for holding, checks ReasoningBank for similar past recoveries, reviews MTF signal, market regime, Fear & Greed, and market breadth
- **Risk Arbiter** — challenges the recovery case, makes the final call

**Loss-side thresholds:**

| Loss | Action |
|---|---|
| −3% | Exit debate — hold or cut |
| −6% | Exit debate — floor tightened after first HOLD |
| −8% | Exit debate — floor tightened after second HOLD |
| −10% | Hard exit — no debate, always cuts |
| Regime → BEAR | Hard exit — always exits immediately |
| 7 days held | Force close — no debate |

**Profit-side thresholds:**

| Profit | Action |
|---|---|
| +3%, +5%, +8% | Profit debate — hold or take profit |
| +12% | Hard exit — locks in gains, no debate |

Each HOLD tightens the floor. After 3 HOLDs the position is force-closed regardless. This prevents the Brain from holding a losing position indefinitely while still giving recoveries a chance.

#### The ReasoningBank — How the Brain Learns

Every night at 2am, the Brain runs a nightly reflection using `qwen2.5:14b` — its deepest reasoning model. It reviews all closed trades from the day and extracts structured patterns:

- What conditions, signals, and reasoning led to winning trades
- What was missed or misleading in losing trades
- Calibration checks — were high-confidence decisions actually more accurate?

These patterns are stored in a local RAG memory called the **ReasoningBank** (`data/reasoning_bank.json`). Every future debate injects the most relevant ReasoningBank entries into the Risk Manager's context, so the Brain's past experience directly influences its next decision.

The ReasoningBank also grows from three other sources:
- **Research Agent** — writes findings when it promotes a strategy to production
- **Strategy Optimizer** — writes the rationale when it switches a coin's strategy
- **Exit Debates** — records whether each hold-or-cut decision was correct after the position closes

#### Watchlist Discovery Pipeline

The Brain doesn't just trade the coins you've configured — it actively scouts for new opportunities. Every hour, it evaluates the top candidates from a 72-coin watchlist and runs a full debate on each one. BUY decisions from discovery go to a **paper portfolio only** (never the live queue). After every 3 paper wins for a coin, the Brain sends a Telegram alert (including win rate and total P&L) suggesting it be promoted to the live list.

#### IG Markets Brain

The same Brain architecture extends to IG Markets CFD trading. A dedicated `IG-Brain` thread cycles through all configured instruments every 30 minutes during market hours (Mon–Fri, 8am–10pm AEST). IG Brain orders queue to `brain_orders_ig.json` and are picked up by a separate `IG-BrainExecutor` thread. The Brain checks whether the IG session is open before debating any instrument.

#### Brain Console

The Brain Console (visible in Mission Control) is a live terminal feed showing every Brain decision in real time, colour-coded by action type:

- 🟢 **Green** — BUY decision
- 🟡 **Yellow** — HOLD (exit debate held, position kept)
- 🔴 **Red** — EXIT (position cut)
- 🟣 **Purple** — DEBATE (exit or profit debate in progress)
- ⬜ **Grey** — BLOCK (MTF gate or other pre-debate filter blocked the coin)

Four tabs: **All** (merged chronological feed) · **Decisions** (ledger) · **Log** (filtered bot log) · **Queue** (executor order status).

#### Debate Viewer — Live Agent Terminal

The Debate Viewer is a dedicated full-screen terminal UI that streams every agent's reasoning in real time as a debate unfolds. Instead of seeing only the final verdict, you watch each of the 8 agents arrive at their conclusion one by one — their reasoning lines, confidence score, and sentiment displayed in a monospace terminal with ASCII box-drawing characters.

**What you see during a live debate:**

```
▶ SIGNAL DETECTED
BTC $111,100  │  CRYPTO  │  25/08/2026 20:19:44 AEST
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
┌─ TECHNICAL ANALYST
│  MTF alignment shows bullish momentum across 4h/6h timeframes
│  RSI 41 — approaching oversold
│  Price below EMA-21 (111,590) — mixed signals
└─ ████████████░░░░░░░░  BULLISH 7/10

┌─ BEAR RESEARCHER
│  Paper strategy win rate only 14.8% — signals are not reliable
│  CoinSpot premium 2.5% erodes entry cost basis significantly
└─ ████░░░░░░░░░░░░░░░░  BEAR CASE 4/10

...

VERDICT: NO_TRADE  │  Final score: 3/10         all agents passed
```

The terminal header shows a live progress counter (`3 / 8 agents`, amber, blinking) so you always know where the debate is up to.

**Signal history sidebar** lists the last 50 completed debates with asset, verdict badge, score, and timestamp — click any entry to replay its full reasoning in the terminal. A **LIVE** button returns you to the live feed instantly.

**Mobile-responsive** — the history sidebar collapses into a top drawer on small screens, triggered by a ☰ HISTORY button. The terminal is full-width.

Full dark and light theme support via CSS custom properties.

### IG Markets Intelligence Layer

When the IG Brain evaluates a CFD or commodity trade, it goes far beyond price and technical signals. Four dedicated services feed into every IG debate, giving the AI a full picture of the real cost and market structure before a single order is placed.

#### CFTC Commitments of Traders (COT) Positioning

The Brain checks the CFTC Commitments of Traders report — released weekly by the CFTC and automatically cached — for every commodity and futures instrument it trades. COT data shows how large institutional money managers, producers, and swap dealers are positioned relative to the last 52 weeks.

| Signal | Percentile | What It Means |
|---|---|---|
| **CROWDED_LONG** | ≥ 0.80 (52w) | Speculative longs at a 52-week extreme — mean-reversion risk; the Bear Researcher flags this |
| **NEUTRAL** | 0.21 – 0.79 | Balanced positioning — no strong crowding signal |
| **CROWDED_SHORT** | ≤ 0.20 (52w) | Speculative shorts at a 52-week extreme — contrarian opportunity |

COT data is available for all commodity and index futures tracked in the CFTC disaggregated report, including crypto futures for Bitcoin and Ethereum (CME contracts). Equity index instruments such as DAX, DOW, and NASDAQ return no COT signal and the field is silently absent from debates. The dashboard includes a dedicated **COT Signals** widget on Mission Control with separate sections for CME Crypto Futures and Commodities & Metals.

The Bear Researcher and Risk Manager both receive the COT signal in their context. A CROWDED_LONG flag at 52-week extreme positioning is treated as a genuine mean-reversion risk factor — not a veto, but a reason to require stronger confirming evidence.

#### Overnight Financing Tracker

Every IG CFD position carries an overnight financing charge — the cost of holding a leveraged position open past the daily cut. LUCIFER computes this precisely for every open position and prospectively for any new trade being evaluated.

**Formula**: `daily_cost = position_value × (benchmark_rate + 2.5% IG admin) / 365`

The benchmark rate varies by instrument currency: SONIA for GBP instruments, SOFR for USD, ESTR for EUR, RBA Cash Rate for AUD. Rates are stored in `data/ig_overnight_rates.json` and applied nightly via an automated scheduler.

The Risk Manager sees the projected daily carry cost (in AUD) and the annual rate (%) for every position being considered. This prevents the Brain from holding positions where the overnight financing erodes profits faster than the trade can move in its favour.

- Daily financing accrues into `financing_accrued` on each open position
- Total financing paid is tracked in `paper_ig_portfolio.json`
- The Open Positions table on the dashboard shows rate, daily cost, and total accrued for every IG position

#### Bid/Offer Spread Monitor

IG Markets instruments carry a bid/offer spread — the round-trip cost of entering and exiting a trade. LUCIFER logs the live bid and offer price for every tracked instrument every 60 seconds, maintaining a rolling 200-observation history.

From this history the spread tracker computes:
- **Current spread** — latest observed bid/offer gap in points
- **Typical spread** — median of the rolling window (filters outliers from wide-market open)
- **Typical % cost** — round-trip spread as a percentage of price, so the AI can reason about break-even move size
- **Min/max spread** — range over the observation window

The Bear Researcher receives current spread and typical round-trip cost percentage directly in its prompt. A wide spread relative to the expected move is a legitimate bear argument. The Risk Manager sees spread friction as part of the system state block, ensuring it is always weighed against the expected edge.

Spread data is visible in the Open Positions table on the dashboard (Spread and Spread Cost columns, shown only for IG positions).

#### ATR-Based Position Sizer

The Brain no longer guesses position size — it computes it from actual price volatility using Average True Range.

**How it works:**
1. The price history for each instrument is maintained in `data/ig_price_history.json` — IG-unit prices sampled every ~3 minutes by the commodity price scheduler, up to 200 observations per instrument (~10 hours of data)
2. ATR is computed as the mean absolute consecutive price difference over the last 50 ticks — a short-term volatility measure in IG price points
3. The suggested stop distance is `max(1.5 × ATR, min_stop_distance)` — never below the IG dealing rules minimum
4. Position size is `floor(risk_aud / (stop_pts × lot_value_aud))`, clamped to the minimum lot size from `ig_dealing_rules.json`
5. The `within_budget` flag signals when the minimum lot size forces the actual risk above 1.25× the configured budget — meaning the instrument is too large for the account at current settings

The Brain's Risk Manager sees the full sizing breakdown in its system state:
- Suggested lot size
- Suggested stop in points (and the ATR that drove it)
- Actual risk in AUD at that size
- Risk as a percentage of the portfolio
- Whether the trade is within budget or over

The **IG Markets → Sizer** dashboard tab lets you see ATR sizing for every instrument in real time, with an adjustable risk % slider to model different position sizes before they are live.

The risk percentage is controlled by `ig.atr_risk_pct` in Settings (default 1% of portfolio per trade).

---

### Shared Debate Daemon

#### The Problem It Solves

LUCIFER runs multiple bots simultaneously — a live CoinSpot bot, a Swyftx demo bot, and an IG paper bot. Before this architecture change, each bot contained its own copy of the debate engine and ran its own independent 8-agent debate whenever it wanted to evaluate a coin. This meant that if both the CoinSpot bot and the Swyftx demo bot both decided to look at SOL at the same time, Ollama would be asked to run 16 agent evaluations back-to-back (8 per bot), loading the same models, processing the same market data, and arriving at the same answer — twice. On a machine with limited VRAM, this caused queuing delays of 15–30 minutes and frequent timeouts.

#### How It Works Now

All debate execution now runs in a single dedicated process — `services/debate_service.py` — supervised alongside the bots at startup. No bot ever calls the debate engine directly. Instead:

1. **A bot reaches its entry gate** — price signal, MTF alignment, and all pre-debate filters have passed
2. **The bot writes a request** to `data/debate_queue.json`: which instrument, which asset class, which exchange, and which account type
3. **The Debate Daemon picks it up** within 5 seconds, checks whether that instrument is still in its 15-minute cooldown window, and if not, runs the full 8-agent debate
4. **The result is written** to `data/debate_results/{instrument}.json` — the full debate record including every agent's verdict, confidence score, final decision, and the price at the time of the debate
5. **Each bot polls its result file** every cycle and, if the result is fresh (under 15 minutes old) and the verdict is BUY, the bot queues the order for its own exchange

#### What This Means in Practice

- **One debate per coin, shared across all bots** — if CoinSpot and Swyftx demo both queue SOL at the same time, one debate runs and both bots read the same result. Ollama is called once, not twice.
- **Deduplication by cooldown** — a coin that was just debated is locked out for 15 minutes. Duplicate requests from the same or different bots within that window are silently dropped.
- **Exchange-aware results** — the result file records which exchange the debate was run for, so each bot can confirm the context matches before acting. The Fundamental Analyst in the debate sees the correct portfolio (CoinSpot holdings vs Swyftx demo holdings) based on the requesting bot's exchange.
- **Decoupled failure modes** — if the Debate Daemon crashes, the bots continue running their strategy loops and simply find no fresh debate results. No bot crashes with it. The supervisor restarts the daemon automatically.
- **Visible in the Debate Viewer** — every debate the daemon runs appears in the Debate Viewer terminal UI in real time, regardless of which bot triggered it.

The daemon runs as a supervised `DebateService` process and logs to `logs/debate_service.log`.

---

### Strategy Engine (139+ Strategies)
Strategies are organised into two asset-class groups:

**Crypto Strategies** (RSI-based, momentum, mean reversion, ML, DCA variants, arbitrage, regime-aware)  
**Commodity Strategies** (trend following, carry/term structure, calendar spreads, relative value, seasonality, fundamentals)

Strategies can be individually enabled, backtested, hyperopt-tuned, and certified before deployment.

### Research Pipeline
- **Research Agent** — autonomous AI that analyses bot performance nightly (3am AEST), detects weaknesses, and generates research projects
- **Backtester** — per-strategy historical testing with configurable date ranges
- **Statistical Tools** — Hurst exponent, Monte Carlo simulation, rule significance testing
- **Strategy Certification** — formal pass/fail gate before any strategy reaches production
- **Strategy Lab** — 139-strategy bench for comparing performance across asset classes

### Plugin Doctor v2
A 3-LLM autonomous self-healing engine that runs every 5 minutes in the dashboard:
- Scans all 280+ plugins for missing files, import errors, and blueprint compliance gaps
- Diagnoses root cause via LLM, proposes and applies a fix
- Requests a targeted service restart via `data/restart_requests/`
- Protected files (bot.py, data/*.json, supervisor.ps1) are diagnosed but never auto-repaired

### Risk Management
- Per-trade stop-loss with configurable buffer
- Trailing stop-loss
- Fear-weighted DCA (buy more in Extreme Fear, less in Extreme Greed)
- Daily drawdown limits with automatic trading pause
- Position concentration limits
- Regime detection (trending / ranging / high-volatility)
- **Thermal protection** — CPU temperature monitoring with automatic load shedding and graceful shutdown (see below)

### Paper Trading
Full paper-trading mode using real live market data. Switches to dry-run at the ribbon button — no code changes needed. All AI decisions, P&L tracking, and logs still run in real time.

---

## Workspaces (Dashboard Navigation)

| Workspace | Purpose |
|---|---|
| **Mission Control** | Portfolio overview, open positions, recent trades, exchange accounts |
| **Markets** | Live price feeds, charts, market breadth, sentiment snapshot |
| **Strategies** | Trading engine controls, strategy lab, optimisation, backtesting |
| **Research** | Research centre, AI agent, statistical tools, certification |
| **AI Brain** | AI Brain console, AI Chat, reasoning bank, LLM prompt editor, brain metrics |
| **Risk** | Risk framework, trailing stops, DCA rules, regime detector |
| **Exchange** | Per-exchange order books, trade history, ledger sync |
| **Settings** | All platform settings — trading, risk, AI, automation, notifications |
| **Engineering** | Blueprint health, tech debt, event bus, migration progress |

---

## Technology Stack

| Layer | Technology |
|---|---|
| **Backend** | Python 3.11+ · Flask · APScheduler |
| **Frontend** | React 18 · Vite · CSS Variables (5 themes) |
| **Database** | JSON flat-file store (data/) · SQLite for research |
| **AI** | Ollama (local only) · qwen3:8b · qwen2.5:14b · mistral-nemo:12b · serialised inference queue |
| **Exchanges** | REST APIs — CoinSpot V2 · Swyftx JWT · Bybit V5 · IG Markets REST |
| **Hosting** | Self-hosted Windows · HTTPS on port 5443 · mkcert TLS |
| **Notifications** | Telegram Bot API |

---

## System Requirements

| Component | Minimum | Recommended |
|---|---|---|
| **OS** | Windows 10 64-bit | Windows 11 Pro 64-bit |
| **CPU** | Intel Core i5 (any gen) | Intel Core i7 / Ryzen 7 |
| **RAM** | 16 GB | **32 GB+** — recommended if running local LLMs |
| **Storage** | 250 GB free | 1 TB SSD (data accumulates over time) |
| **Python** | 3.10+ | 3.11+ |
| **Node.js** | 18+ | 20 LTS |
| **Ollama** | Required for AI features | Latest version |
| **Network** | Broadband | Broadband with static or DDNS address for remote access |

**Local LLM models used**: `qwen3:8b` (fast — all real-time agents), `qwen2.5:14b` (deep — Risk Manager + reflection), `mistral-nemo:12b` (Plugin Doctor reviewer)

> A dedicated always-on machine is strongly recommended. The bot runs 24/7 and should not share resources with a primary workstation.

---

## Thermal Protection

LUCIFER runs 24/7 on a dedicated machine and actively monitors CPU temperature through [LibreHardwareMonitor](https://github.com/LibreHardwareMonitor/LibreHardwareMonitor). If the machine overheats, the bot responds in stages — shedding load before the situation becomes dangerous, and shutting down cleanly rather than crashing mid-trade.

### Why this matters

In August 2026 the host machine experienced repeated BSOD crashes (IRQL_NOT_LESS_OR_EQUAL, WHEA CPU parity errors) over three days caused by a failing AIO cooler pump. Under full load the CPU was reaching 93–99°C — within 1–3°C of TjMAX. Windows was crashing mid-cycle, corrupting open state files and leaving the bot in an indeterminate state on restart.

The thermal protection system was built specifically to prevent this scenario from repeating. With a healthy cooler the machine now sustains 84–85°C under full 100% CPU stress, leaving a 15–17°C buffer to TjMAX.

### Temperature Thresholds

| CPU Temp | Status | What LUCIFER Does |
|---|---|---|
| < 80°C | ✅ **Normal** | Nothing — full operation |
| 80–87°C | 🟡 **Warm** | Log the reading. No action taken. |
| 88–94°C | 🟠 **High** | Send Telegram warning: *"CPU at XX°C — monitor closely"* |
| 95–97°C | 🔴 **Critical** | Pause all new buys · Pause Ollama AI Brain inference (biggest CPU consumer) · Send urgent Telegram alert |
| 98°C+ | 🚨 **Emergency** | Graceful shutdown — saves all state, sends final Telegram alert, exits cleanly before Windows can crash |

> **Sells, stop-losses, drawdown protection, and trailing stops remain active at all temperature levels.** Only new buy orders are throttled. The bot protects your capital even while protecting the hardware.

### How it works

CPU temperature is read via the LibreHardwareMonitor (LHM) web server running locally on `http://localhost:8085`. LHM is started automatically by the supervisor at login alongside the bots. If LHM is not running (e.g. first boot after a fresh install), the bot operates normally but logs a warning — thermal protection activates as soon as LHM comes online.

The System Health widget in Mission Control shows CPU and GPU temperature in real time, colour-coded by threshold, along with fan RPM and a count of critical Windows events in the last 24 hours (WHEA errors are the early warning sign of thermal throttling).

### WHEA Errors — Early Warning

Windows Hardware Error Architecture (WHEA) events appear in the System event log before a BSOD. The System Health widget surface these: if the 24-hour critical event count is non-zero, investigate CPU temperature under load before it escalates. In the August 2026 incident, WHEA events were visible for over a day before the first crash.

---

## Roadmap

### Completed
- [x] Multi-exchange trading (CoinSpot, Swyftx, Bybit, IG Markets)
- [x] 7-agent autonomous AI Brain — sole gatekeeper for all new opens
- [x] Self-learning ReasoningBank — grows from every closed trade
- [x] Nightly reflection engine (2am, qwen2.5:14b)
- [x] Position monitor — stop-loss / take-profit / max-hold exits
- [x] ATR-based + Kelly criterion position sizing
- [x] Multi-timeframe alignment gate (1h/4h/1d RSI+EMA)
- [x] Correlation gate (Pearson r > 0.85 blocks)
- [x] Regime-calibrated confidence thresholds
- [x] Agent performance tracker + debate weights
- [x] A/B testing framework for agent prompts (Thompson sampling)
- [x] Opportunity scanner (6-signal coin ranking every 5 min)
- [x] Brain metrics dashboard (equity curve, regime, agent stats, journal)
- [x] Daily Telegram digest (8am)
- [x] Smart alert manager (throttle, dedup, severity routing)
- [x] Plugin Doctor v2 (3-LLM autonomous self-healing)
- [x] 276+ strategy plugins with auto-discovery
- [x] Research centre with AI agent, backtester, statistical tools
- [x] Strategy certification pipeline
- [x] IG Markets tiered holdings
- [x] Fear-weighted DCA
- [x] LUCIFER rebrand
- [x] Mission Control movable/resizable widget cards
- [x] Live CoinSpot trading — enabled 2026-08-17
- [x] Brain as sole buy authority — bot signal buys blocked; Brain order queue + executor thread
- [x] Intelligent exit debates — 2-agent debate replaces fixed stop-loss (2026-08-18)
- [x] Watchlist MTF engine — yfinance removed; all timeframe signals from watchlist price array
- [x] Watchlist discovery pipeline — suggested coins paper-evaluated by Brain every hour (was 4h); Telegram alert at each 3-win milestone
- [x] IG Brain fully wired — IG-Brain + IG-BrainExecutor threads; brain_orders_ig.json queue
- [x] Research → Brain pipeline — research agent + optimizer write findings to ReasoningBank
- [x] Brain Console terminal widget — live decision feed on Mission Control
- [x] Telegram breadth alerts — fires when market turns extreme (strong/weak breadth threshold)
- [x] Macro calendar auto-populate — ForexFactory feed, refreshes every 6 hours
- [x] Commodity strategy engines — trend, carry, seasonal, relative value, calendar spreads
- [x] A/B prompt auto-bootstrap — Brain autonomously creates, tests, promotes, and retires its own agent prompts nightly (Thompson sampling)

- [x] Priority Interrupt Agent (3-tier) — polls macro events, ForexFactory, exchange health, price moves, circuit breaker, drawdown, and IG session boundary every 30s; injects urgency-coloured alerts directly into Brain debates; 10-min cooldown per trigger, 30-min context expiry (2026-08-23)
- [x] Paper Bot Comparison widget — side-by-side Legacy (rules-only) vs AI Brain paper bot PnL, win rate, cash, and holdings on Mission Control (2026-08-23)
- [x] Process Control widget — restart any bot, the dashboard, or all services from Mission Control with 3 buttons; no RDP required (2026-08-23)
- [x] DCA amount fix — paper bots now honour per-exchange `dca_amount_aud` from `paper_trading_settings.json` (AI bot $10k, legacy/swyftx/bybit $1k); previously ignored due to two conflicting code paths in bot.py (2026-08-23)
- [x] Ollama serialising queue — all inference calls queued single-file; eliminates GPU race timeouts (2026-08-19)
- [x] Ollama keep-alive — both models stay warm in VRAM/RAM permanently, no cold-load penalty between debates (2026-08-19)
- [x] Auto circuit breaker stale-peak protection — detects and discards leftover peak from decommissioned exchanges (2026-08-19)
- [x] Thermal protection — 5-stage CPU temp response (warn → alert → pause buys → shed Ollama → graceful shutdown); LibreHardwareMonitor integration; System Health widget with live temps, bot process status, WHEA event counter (2026-08-21)
- [x] Drawdown peak spike rejection — bad price ticks > 3× current peak are discarded, preventing false drawdown alerts (2026-08-21)
- [x] AI Chat — persistent conversation history, Ollama streaming, live trading context, model selector, quick prompts, mobile sidebar (2026-08-22)
- [x] LUCIFER startup branding — Telegram startup message updated to LUCIFER branding with exchange name (2026-08-22)
- [x] PWA / favicon — LUCIFER icon in browser tab and "Add to Home Screen" on mobile (2026-08-22)
- [x] Change watcher — now tracks dashboard-new/src and dashboard-new/public alongside Python source (2026-08-22)

- [x] Chart Analyst agent (Agent 1b) — 8th debate agent using EMA stack, SuperTrend, VWAP, Bollinger Bands, auto trend lines, and 5-bar momentum from live 1h candles; feeds chart structure verdict into Bull/Bear/Risk debate (2026-08-25)
- [x] Adaptive trailing stop — tiered trail percentages that activate at 12%, tighten at 20% and 30% gains, with a floor that never exits below 10% profit; implemented for crypto lot engine and IG server-side stop amendments (2026-08-25)
- [x] Paper bot $100k reset — all AI crypto paper bots reset to $100,000 with aggressive settings: 25% position size, $5,000 min buy, $25,000 max buy, 10% profit target, 5% stop loss (2026-08-25)
- [x] Debate Viewer — full-screen live terminal UI streaming all 8 agent verdicts and reasoning as they complete; signal history sidebar (last 50 debates); debate replay; live progress counter (X / 8 agents); mobile-responsive with collapsible history drawer; full dark/light theme (2026-08-25)

- [x] UTF-8 BOM fix — eliminated charmap decode errors across all settings readers in bot.py, ig_bot.py, notification_router, runtime_alerts, trade_alert_dispatcher, production_manager, and plugin APIs; breadth monitor encoding fixed (2026-08-26)
- [x] Commodity price fetcher wired — fixed dead import in commodity_price_fetcher.py; scheduler now writes data/commodity_prices.json every 5 min; IG paper P&L unrealised column now populates on dashboard (2026-08-26)
- [x] Debate Viewer history navigation — fixed bug where a new live debate would eject you from a historical replay; viewing past debates now persists across live debate cycles (2026-08-26)
- [x] Screen wake lock — sun button in dashboard footer now requests WakeLock API alongside pausing the idle logout timer; screen stays on while watching debates on mobile/tablet (2026-08-26)
- [x] AI agent prompt tuning — Bear Researcher, Adversarial Reviewer, and Risk Manager prompts rebalanced; agents now weigh opportunity cost alongside downside risk rather than defaulting to NO_TRADE on any uncertainty (2026-08-26)

- [x] Trade celebration — profitable sell triggers a money rain animation and ching-ching sound on the dashboard; global poller fires on any page; paper bot profitable sells also trigger (paper sell audit logging added); toggle in Settings > System; TEST button in Debate Viewer header (2026-08-27)
- [x] Paper Bot Comparison verdict — context-aware verdict text: "losing less than" when both bots negative, "ahead of" when both positive, "leading" when one positive / one negative (2026-08-27)
- [x] Paper AI tuner speed fix — switched from 3-model parallel LLM panel to MODEL_FAST (qwen3:8b) only; tuning runs complete in ~4 min instead of ~12 min (2026-08-27)

- [x] CFTC COT data integration — weekly Commitments of Traders report auto-fetched and cached (7-day TTL); 52-week percentile crowding signals for all CFTC-tracked commodities and metals; Bitcoin and Ethereum covered via CME crypto futures section of the same report; COT Signals widget on Mission Control with separate CME Crypto and Commodities & Metals sections; `cot_signal` injected into every IG debate context (2026-08-28)
- [x] Overnight financing tracker — precise daily carry cost computed per position using benchmark rates (SONIA/SOFR/ESTR/RBA) + 2.5% IG admin; applied nightly to paper IG portfolio; financing accrued and daily rate shown in Open Positions table; `financing` injected into every IG debate context so Risk Manager weighs carry cost against expected edge (2026-08-28)
- [x] Bid/offer spread monitor — rolling 200-observation spread log per instrument; typical spread, min/max, and round-trip % cost derived from live bid/offer prices sampled every 60s by paper IG monitor; Spread and Spread Cost columns in Open Positions table; `spread_stats` injected into IG debate context (2026-08-28)
- [x] ATR-based IG position sizer — tick ATR computed from rolling 200-obs price history in IG points; suggested lot size = risk_aud / (1.5×ATR × lot_value_aud) clamped to IG dealing rules minimums; `within_budget` flag when min lot exceeds risk budget; IG Markets → Sizer tab on dashboard with live sizing for all 17 tracked instruments; `atr_sizing` injected into every IG debate context; risk % configurable via `ig.atr_risk_pct` in Settings (2026-08-28)
- [x] AI Brain fully wired to IG intelligence layer — Bear Researcher now receives COT crowding signal and spread cost as explicit context lines; Risk Manager system state block now includes COT, overnight financing rate, spread friction, and ATR sizing recommendation for every IG debate; all four fields null-guarded so they are silently absent when data has not yet populated (2026-08-28)
- [x] Swyftx demo bot — isolated demo account (`https://api.demo.swyftx.com.au`) running the same Brain + strategy pipeline as live Swyftx; completely separate data store (`swyftx_demo_portfolio.json`, `swyftx_demo_trades.json`); 30-second background sync loop; `EXCHANGE_OVERRIDE=swyftx_demo` routes all Brain orders to the demo API without touching the live exchange; shadow provider (no Telegram announcement) (2026-08-28)
- [x] Nathan's Test Trading page — new Mission Control page showing IG demo + Swyftx demo accounts side by side; each panel shows balance, total/realised/unrealised P&L, open positions with position sizing bars, recent trades, win rate and trade count; auto-refreshes every 30 seconds; manual sync button for immediate Swyftx demo pull (2026-08-28)

- [x] Shared Debate Daemon — AI debate execution extracted from every individual bot into a single standalone `services/debate_service.py` process (2026-08-29)
- [x] Prometheus Phase 3 — dynamic nav: `useNav` hook reads nav blocks from `/api/registry/nav`; AppShell builds sidebar from API response; `workspaces.js` deleted and replaced by `shell-nav.js`; all 34 plugin nav blocks wired; no hardcoded route list anywhere in the shell (2026-08-29)
- [x] Prometheus Phase 4 — Vite dev server proxy: `DashboardDev` supervisor entry runs Vite on port 3001; Flask proxies all non-API requests to Vite in `VITE_DEV=1` mode; falls back to compiled `dist/` if Vite is down; enables hot-module-replacement during development without a full rebuild cycle (2026-08-29)
- [x] Prometheus Phase 5 — bot restart panel: `POST /api/registry/restart` writes `.restart` files to `data/restart_requests/`; Plugin Manager shows a `BotRestartPanel` that polls `GET /api/registry/restart` every 10s and displays pending restart queue; plugin install returns `requires_bot_restart` flag so the UI prompts automatically after installing a bot-side plugin (2026-08-29)
- [x] CoinGecko ID overrides — 31 ticker-to-slug mappings added to `candle_fetcher.py` covering all coins whose lowercase ticker does not match the CoinGecko slug (BCH, BNB, TRX, ETC, MANA, SAND, AXS, APE, COMP, APT, STRK, SEI, TIA, METIS, MNT, ENJ, CHZ, FET, RENDER, SHIB, WIF, INJ, ROSE, ZEC, EGLD, HBAR, STX, MINA and more); eliminates 404 errors on CoinGecko candle fetches for the Brain Chart Analyst (2026-08-29)
- [x] Weekend IG market hours — IG CFD grey-market instruments now open Saturday 07:00 AEST; `macro_calendar.py` emits a synthetic "IG Weekend Markets Open" event each Saturday at that time (21:00 UTC Friday); Saturday branch skips the maintenance window entirely; Sunday event renamed "IG Weekly Session Open" for the regular 24h instrument session (2026-08-29)
- [x] Exchange-specific portfolio context in AI debates — `build_crypto_context()` now branches on the `exchange` parameter: when `exchange == "swyftx_demo"` it reads `swyftx_demo_portfolio.json` (cash, holdings, current prices) instead of `holdings.json`; the context `exchange` field is dynamically set so Portfolio Manager routes orders to the correct exchange; previously every bot's debate used coinspot portfolio data and coinspot order routing regardless of which exchange was running (2026-08-30)
- [x] Exec Health PID monitor — SwyftxDemo process now appears in the System Health widget bot process list; `BOT_SUPERVISOR_NAME=SwyftxDemo` mapped to "Swyftx Demo" label; cross-Vite-root import bug in the widget fixed by inlining the API helper (2026-08-30)

### In Progress
- [ ] First external tester onboarding — setup wizard, status page, empty states, install logging

### Future
- [ ] Mobile app (Android — PWA wrapper + push notifications)

---

## Licence

Private — all rights reserved. This software is not open source.

Contact: lucifersdevteam@gmail.com
