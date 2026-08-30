# LUCIFER — Internal Engineering Reference

> **Last Updated**: August 2026

This is the internal docs directory. The definitive public-facing feature reference is [`docs/public/README.md`](public/README.md).

For engineering guidance, patterns, gotchas, and service maps, see [`CLAUDE.md`](../CLAUDE.md) in the project root — it is the single source of truth for how to work in this codebase.

---

## Current Status (August 2026)

| Item | Detail |
|---|---|
| **Platform** | LUCIFER v2.6 |
| **Live trading** | CoinSpot — enabled 2026-08-17 |
| **Demo trading** | Swyftx demo bot + IG paper bot both running |
| **AI Brain** | 8-agent debate engine, sole buy authority |
| **Dashboard** | React 18 + Vite (dashboard-new/) on HTTPS port 5443 |
| **Plugin system** | 280+ auto-discovered plugins (Prometheus Phase 1–5 complete) |
| **Processes** | supervisor.ps1 manages all bots at Windows logon |

## Directory Map

```
docs/
  public/
    README.md    — public-facing feature reference (keep updated)
  README.md      — this file (internal pointer)

knowledge/
  roadmap_prometheus.md        — Prometheus v3.1 roadmap (Phases 0–7)
  prometheus_plugin_audit.md   — plugin inventory, route map, anomalies
```

## Key Links

- Public README: [`docs/public/README.md`](public/README.md)
- Engineering rules: [`CLAUDE.md`](../CLAUDE.md)
- Prometheus roadmap: [`knowledge/roadmap_prometheus.md`](../knowledge/roadmap_prometheus.md)
- Private repo: https://github.com/mirageco69/Lucifer-Multi-AI-Trading-Bot-Private
- Public docs repo: https://github.com/mirageco69/Lucifer-Multi-AI-Trading-Bot-Public
