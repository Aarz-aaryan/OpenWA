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
- [x] Bring up `docker compose up -d` (openwa-api + docker-proxy)
- [x] Aaryan scans QR via http://localhost:2785 (or Tailscale IP)
- [x] Generate dedicated session-scoped API key for Aarz
- [x] Wire Aarz → OpenWA REST (send text/media)
- [x] End-to-end test: Aarz → Aaryan WhatsApp → confirmed received ✅
- [ ] ~~Wire webhook OpenWA → Hermes (incoming messages)~~ — CANCELLED per Aaryan, channel is one-way outbound only

## What's Done
- Repo forked + cloned (2026-07-18)
- `.env` configured (SQLite default, ENGINE_TYPE=whatsapp-web.js, headless)
- Docker stack running (openwa-api + docker-proxy, both healthy)
- WhatsApp session `aarz` authenticated by Aaryan (phone +923****9201, pushName "Aaryan Tahir", status `ready`)
- First end-to-end test send → HTTP 201, messageId `true_207928443289777@lid_3EB02058C4AD9AD583A_out` ✅ confirmed received by Aaryan
- `openwa_send.py` helper resolves session by name (or auto-picks first ready) and calls correct endpoint `POST /api/sessions/{id}/messages/send-text`
- 2026-07-20 — discovered whatsapp-web.js requires OGG/Opus for PTT voice notes (WAV fails with Puppeteer `t: t` error). Helper now auto-transcodes via ffmpeg.
- 2026-07-20 — voice note end-to-end test → HTTP 201, messageId `true_207928443289777@lid_3EB03370E1F182DAC5D327_out` ✅
- 2026-07-20 — wired morning briefing cron (`1e2da068fa94`, `0 11 * * *`) to also send to WhatsApp via new STEP 7 in the prompt. Helper script location: `~/.hermes/profiles/aarz/scripts/openwa_send.py`. Backup of pre-edit prompt: `cron/prompt-backups/1e2da068fa94.PRE-WHATSAPP-PATCH.txt`.

## What's Left
- **NOT building inbound webhook** — Aaryan confirmed (2026-07-18) this channel is one-way outbound only: Aarz → Aaryan. He will NOT reply via WhatsApp to Aarz. Inbound messages are received by the WhatsApp session and dropped (no webhook registered).
- **First live cron run with WhatsApp** — tomorrow (2026-07-21) at 11am EDT, the morning briefing cron will fire and attempt the WhatsApp delivery. Aaryan should check WhatsApp then. If voice/text fails, will surface as warning in the Discord message.
- (Optional, on request) Wire MCP mount at `/mcp` so any agy/Claude/Codex session can use WhatsApp as native tools
- (Optional, on request) Move stack from Hermes host to r-server if Hermes needs cleanup
- (Optional, on request) Add more crons to WhatsApp notify when Aaryan gives more rules

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
