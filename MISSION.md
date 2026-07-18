# MISSION — OpenWA WhatsApp Gateway

## Mission
Self-hosted WhatsApp API gateway so Aarz (Hermes orchestrator) can send WhatsApp messages to Aaryan from any tool call, cron, or subagent — and receive replies via webhook.

## Why
- WhatsApp = Aaryan's primary reachable channel outside Discord/SSH.
- Currently Aaryan must be online at his keyboard for Aarz to ping. WA = async escape hatch for cron alerts, escalations, overnight results.
- n8n (Aaryan's automation hub) lists OpenWA as a community-integrated tool — fits his existing stack.

## Status
**Active.** Deployed 2026-07-18 on Hermes host (100.100.35.6), Docker Compose with SQLite + local storage. WhatsApp session NOT YET AUTHENTICATED — Aaryan must scan QR.

## Progress
- [x] Fork `rmyndharis/OpenWA` → `Aarz-aaryan/OpenWA`
- [x] Clone to `/home/Aarz/openwa`
- [x] Configure minimal `.env` (SQLite, headless Chromium, no Postgres/Redis/MinIO)
- [ ] Bring up `docker compose up -d` (openwa-api + docker-proxy)
- [ ] Aaryan scans QR via http://localhost:2785 (or Tailscale IP)
- [ ] Generate dedicated session-scoped API key for Aarz
- [ ] Wire Aarz → OpenWA REST (send text/media)
- [ ] Wire webhook from OpenWA → Hermes (incoming messages)
- [ ] End-to-end test: cron fires → Aaryan receives WhatsApp → Aaryan replies → Hermes logs it
- [ ] Optional: expose MCP at `/mcp` so any Claude/agy agent can use WA as native tools

## What's Done
- Repo forked + cloned (2026-07-18)
- `.env` configured (SQLite default, ENGINE_TYPE=whatsapp-web.js, headless)

## What's Left
- Docker Compose stack running
- QR auth for Aaryan's WhatsApp number
- Hermes-Aarz ↔ OpenWA bridge (Python helper + Aarz tool integration)
- Webhook receiver in Hermes for inbound replies
- E2E test with real message to Aaryan

## Architecture

```
┌─────────────┐   HTTP REST    ┌──────────────────┐   WS        ┌─────────────┐
│  Aarz/Hermes│ ─────────────▶ │ OpenWA (Docker)  │ ─────────▶ │ WhatsApp    │
│  (tools)    │                │ localhost:2785   │            │ servers     │
└─────────────┘                │ + Chromium (WWJS)│            └─────────────┘
                               │ + SQLite         │
                               └──────────┬───────┘
                                          │ webhook (HMAC)
                                          ▼
                               ┌──────────────────────┐
                               │ Hermes webhook       │
                               │ listener (Aarz)      │
                               └──────────────────────┘
```

## Stack
- OpenWA v0.8.17 (NestJS, whatsapp-web.js engine)
- Port: 2785 (bound 127.0.0.1 only by default — change `127.0.0.1:${API_PORT}` to `0.0.0.0` in docker-compose.yml to expose on LAN/Tailscale)
- DB: SQLite at `./data/openwa.sqlite`
- Sessions: `./data/sessions/`
- API auth: API key (header `X-API-Key: <key>`)
- MCP: optional, off by default (toggle `MCP_ENABLED=true`)

## Key Decisions

1. **Hermes host, not r-server** — local QR scan via http://localhost:2785 is one click. r-server would require ssh tunnel or Tailscale-bridge setup. Architecture is portable: move stack to r-server later if Hermes needs to be cleaner.
2. **SQLite minimal config** — Aaryan only needs 1 WhatsApp number for now. No Postgres/Redis/MinIO. Lighter, faster boot, ~150MB image vs ~600MB.
3. **whatsapp-web.js engine (default)** — better-maintained, Chromium-based, well-documented. Baileys reserved for cases needing headless-no-Chromium.
4. **127.0.0.1 binding by default** — API not exposed to LAN. If we need cross-host access (e.g. from r-server), change compose port mapping.
5. **Docker socket proxy sidecar** — required by OpenWA's hard security model; never mount `/var/run/docker.sock` directly into `openwa-api`.

## Issues / Blockers
- Disk at 90% on Hermes host (only 24GB free). OpenWA image build pulls Chromium (~170MB). Acceptable for now; revisit if other Docker work pushes us past 95%.
- WhatsApp may ban accounts using unofficial APIs. Mitigation: SIMULATE_TYPING=true (default), stay under rate limits (RATE_LIMIT_MAX=100/min).

## Key Files
- `docker-compose.yml` — service defs (api + docker-proxy + optional postgres/redis/minio profiles)
- `.env` — runtime config (SQLite, headless, ports)
- `Dockerfile` — multi-stage build with non-root openwa user
- `scripts/openwa.sh` — wrapper for common ops (start/stop/logs/backup/restore)
- `data/` — persistent volume (sessions, SQLite, media, plugins)

## Verification
```bash
# Container health
docker ps --filter name=openwa-api
docker logs --tail 50 openwa-api

# API reachable
curl -sS http://localhost:2785/api/health/ready

# Swagger docs
open http://localhost:2785/api/docs

# Dashboard (for QR scan)
open http://localhost:2785
```
