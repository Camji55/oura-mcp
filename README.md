# Oura Ring MCP Server

A self-hosted [MCP](https://modelcontextprotocol.io) server that exposes your
Oura Ring data — daily activity, readiness, sleep, workouts, heart rate,
stress, SpO2, sessions, and tags — to Claude (claude.ai connectors or Claude
Code) over HTTP.

The entire deployment is a single `docker-compose.yml`: the Python server is
embedded inline via Compose's `configs.content`, so there is no Dockerfile and
no separate source file to manage. Read-only by design — the Oura v2 API only
exposes reads, so the server can't modify anything on your account.

## Tools

| Tool | Description |
|---|---|
| `get_daily_activity` | Steps, calories, MET minutes by intensity, sedentary/resting time, activity score |
| `get_daily_readiness` | Readiness score, temperature deviation from baseline, contributor scores (HRV balance, resting HR, etc.) |
| `get_daily_sleep` | Daily sleep scores and contributors |
| `get_sleep_periods` | Detailed sleep periods: bedtimes, stage durations, efficiency, avg HR/HRV, lowest HR |
| `get_workouts` | Logged workouts with type, intensity, calories, and start/end times |
| `get_activity_summary` | Compact multi-day summary with per-day rows and period averages |
| `get_heart_rate` | Per-day HR summaries from the intraday timeseries: min/avg/max bpm and averages by source |
| `get_daily_stress` | Time in high-stress and high-recovery zones, plus Oura's day classification |
| `get_daily_spo2` | Nightly average SpO2 and breathing disturbance index |
| `get_sessions` | Meditation, breathing, nap, and relaxation sessions with type, mood, and times |
| `get_tags` | User-entered tags and notes (illness, travel, alcohol, custom) |

Date-range tools default to the last 7 days when called without arguments.

## Requirements

- Docker with Compose v2.23+ (inline `configs.content` support)
- An Oura account with a [personal access token](https://cloud.ouraring.com/personal-access-tokens)

## Quick start

1. Clone the repo and create your `.env`:

   ```bash
   cp .env.example .env
   ```

2. Edit `.env`:
   - `OURA_ACCESS_TOKEN` — your personal access token from
     [cloud.ouraring.com/personal-access-tokens](https://cloud.ouraring.com/personal-access-tokens)
   - `MCP_AUTH_TOKEN` — a long random secret that gates access to the server.
     Generate one:

     ```bash
     openssl rand -hex 32
     ```

3. Start it:

   ```bash
   docker compose up -d
   ```

   First start takes ~30s while pip installs dependencies (cached in a volume
   afterwards). The server listens on `127.0.0.1:8000`; override with
   `OURA_MCP_PORT` / `OURA_MCP_BIND` in `.env`.

4. Check health:

   ```bash
   curl http://localhost:8000/health
   ```

   `{"status": "ok"}` means the server is up. To also verify your Oura token
   works, authenticate the same endpoint:

   ```bash
   source .env && curl -H "Authorization: Bearer $MCP_AUTH_TOKEN" http://localhost:8000/health
   ```

   `{"status": "ok", "oura_api": true}` means the server can reach the Oura
   API with your token.

## Connecting Claude

**claude.ai (custom connector):** add a connector with the URL

```
https://<your-host>/mcp/<MCP_AUTH_TOKEN>
```

**Claude Code:**

```bash
claude mcp add --transport http oura https://<your-host>/mcp --header "Authorization: Bearer <MCP_AUTH_TOKEN>"
```

## Configuration

All configuration is via environment variables, loaded from `.env` by Docker
Compose:

| Variable | Required | Description |
|---|---|---|
| `OURA_ACCESS_TOKEN` | yes | Oura personal access token |
| `MCP_AUTH_TOKEN` | yes | Secret gating all `/mcp` requests (path segment or Bearer header) |
| `OURA_TIMEOUT` | no | Oura API request timeout in seconds (default `30`) |
| `OURA_MCP_PORT` | no | Host port the server is published on (default `8000`) |
| `OURA_MCP_BIND` | no | Host interface to bind (default `127.0.0.1`; set `0.0.0.0` to expose beyond this machine) |

Compose fails fast with a clear error if either required variable is missing.

## Security notes

- **Run behind HTTPS.** The auth token travels in the URL path (for claude.ai
  connectors) or a header, so put the server behind a TLS-terminating reverse
  proxy (Caddy, nginx, Cloudflare Tunnel, Tailscale) before exposing it to the
  internet. Path-based tokens can also end up in proxy access logs — treat
  those logs as sensitive.
- The published port binds `127.0.0.1` by default, so nothing is exposed
  beyond the machine unless you deliberately set `OURA_MCP_BIND=0.0.0.0`.
- There is no unauthenticated mode: the server refuses to start if
  `MCP_AUTH_TOKEN` is unset.
- `/health` answers anonymous callers with process liveness only
  (`{"status": "ok"}`). The Oura connectivity detail — which would reveal
  whether your token is currently valid — requires the auth token, and the
  upstream check behind it is cached for 60 seconds so it can't be used to
  burn your Oura API quota.
- Auth token comparison is constant-time (`hmac.compare_digest`), so response
  timing leaks nothing about the token.
- Dependencies are pinned to exact versions in an embedded lockfile
  (`oura_requirements` in the compose file), so every container start installs
  the same audited package set. To upgrade: bump the pins, recreate, re-audit.
- Keep `.env` out of version control (already covered by `.gitignore`). If a
  token leaks, revoke it at cloud.ouraring.com and generate a new
  `MCP_AUTH_TOKEN`.
- The container runs with `no-new-privileges`, a 256 MB memory limit, and
  log rotation out of the box.

## How it works

`docker-compose.yml` starts a stock `python:3.12-slim` container, installs
[FastMCP](https://github.com/jlowin/fastmcp), httpx, and uvicorn from the
embedded lockfile at boot, and runs the embedded `oura_server.py` (both
injected via Compose `configs`). Auth is a
small Starlette middleware that accepts either `POST /mcp/<token>` (claude.ai)
or `POST /mcp` with a Bearer header (Claude Code), and the Oura client
transparently follows `next_token` pagination.

## License

[MIT](LICENSE)
