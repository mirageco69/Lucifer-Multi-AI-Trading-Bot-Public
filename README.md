# LUCIFER — The AI Multi-Trading Platform

> **Version**: v2.2 &nbsp;|&nbsp; **Updated**: August 2026 &nbsp;|&nbsp; **Status**: Live Trading

LUCIFER is a self-hosted, plugin-based automated trading platform for cryptocurrency and CFD/commodity markets. It runs 24/7 on a dedicated Windows machine, trades across multiple exchanges simultaneously, and manages risk through a layered AI gate, strategy engine, and research pipeline.

---

## What It Does

| Capability | Detail |
|---|---|
| **Multi-exchange trading** | CoinSpot, Swyftx, Bybit (crypto) · IG Markets (CFDs/commodities) |
| **AI trading brain** | 7-agent autonomous brain — sole buy authority; intelligent exit debates consult ReasoningBank, MTF, regime, breadth before cutting a loss |
| **Strategy engine** | 139+ strategies across crypto and commodity asset classes |
| **Research pipeline** | Autonomous AI research agent, backtester, statistical tools, strategy certification |
| **Risk management** | Intelligent exit debates, trailing stops, DCA, drawdown limits, regime detection — no mechanical fixed stop-losses |
| **Dashboard** | HTTPS web dashboard — Mission Control, Charts, Research, Strategies, Settings |
| **Process Control** | Restart any bot, the dashboard, or all services remotely from Mission Control — no RDP required |
| **Priority interrupt** | 3-tier interrupt agent monitors macro events, exchange health, and price moves; injects alerts directly into Brain debates |
| **Plugin system** | WordPress-style auto-discovery — 280+ active plugins, no restart to add new ones |
| **Telegram alerts** | Trade signals, daily P&L reports, AI decisions, breadth alerts |
| **Paper trading** | Full dry-run mode — real data, no real money |
| **IG Markets holdings** | Tiered commodity positions tracked separately with full P&L |

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
│  264+ plugins auto-discovered from src/plugins/**              │
│  Each plugin: plugin.json + api.py + widget.jsx                │
├────────────────────────────────────────────────────────────────┤
│              Exchanges                │   AI / LLM              │
│  CoinSpot · Swyftx · Bybit · IG      │  Ollama (local)         │
│  Paper exchange (dry-run)             │  Claude (cloud)         │
└────────────────────────────────────────────────────────────────┘
```

---

## Key Features

### Multi-Exchange Support
Trade the same strategy across CoinSpot (crypto, AUD), Swyftx (crypto, AUD), Bybit (crypto, USDT), and IG Markets (CFDs/commodities) simultaneously. Each exchange has its own plugin with independent lot tracking, P&L, and order management.

### 7-Agent Autonomous AI Brain

LUCIFER's AI Brain is a fully autonomous trading intelligence built entirely on local LLMs — no cloud API calls, no subscriptions, no data leaving your machine. It runs 24/7 on [Ollama](https://ollama.com), debating every potential trade through a structured 7-agent panel before a single order is placed.

#### The Brain is the sole buy authority

When the Brain is enabled, the bot's own signal-triggered buys are completely blocked. Every buy decision flows through the debate pipeline and is placed by a dedicated executor thread — not the strategy engine. This eliminates conflicting buy paths and ensures every open position has a full reasoning trail behind it.

#### The 7-Agent Debate Panel

Each trade candidate triggers a parallel debate across six specialist agents, with a final Risk Manager adjudicating:

| Agent | Model | Role |
|---|---|---|
| **Technical Analyst** | qwen3:8b | RSI, MACD, EMA, Bollinger, multi-timeframe alignment |
| **Sentiment Analyst** | qwen3:8b | Fear & Greed index, crypto news sentiment, macro events |
| **Fundamental Analyst** | qwen3:8b | Portfolio exposure, open positions, available cash, correlation |
| **Bull Researcher** | qwen3:8b | Builds the strongest possible case FOR entering the trade |
| **Bear Researcher** | qwen3:8b | Builds the strongest possible case AGAINST entering the trade |
| **Risk Manager** | qwen2.5:14b | Reads all 5 agent reports + ReasoningBank history. Final veto — cannot be overruled |
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
- Scans all 276+ plugins for missing files, import errors, and blueprint compliance gaps
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
| **AI** | Ollama (local LLMs only) · qwen3:8b · qwen2.5:14b · mistral-nemo:12b · serialised inference queue |
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
| **Storage** | 50 GB free | 1 TB SSD (data accumulates over time) |
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

### In Progress
- [ ] First external tester onboarding — setup wizard, status page, empty states, install logging

### Future
- [ ] Mobile app (Android — PWA wrapper + push notifications)

---

## Licence

Private — all rights reserved. This software is not open source.

Contact: lucifersdevteam@gmail.com
