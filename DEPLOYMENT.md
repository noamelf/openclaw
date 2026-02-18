# OpenClaw Docker Deployment - Local Setup

This document describes the local Docker deployment of OpenClaw on Noam's Mac.
It is intended for agentic coding assistants operating on this setup.

## Architecture Overview

```
                     http://openclaw.localhost
                              |
                     [Caddy :80] (reverse proxy)
                              |
                   [openclaw-gateway :18789]
                     /        |        \
              Anthropic   Telegram   WhatsApp
              Claude API   Bot        Baileys
                              |
                    [CoreDNS dns-allowlist]
                    (domain allowlist firewall)
```

All services run in Docker Compose on a custom bridge network (`172.30.0.0/24`).
The gateway container can **only** resolve explicitly allowed domains via CoreDNS.

## Services

| Service            | Image                    | Purpose                    | Port                                 |
| ------------------ | ------------------------ | -------------------------- | ------------------------------------ |
| `openclaw-gateway` | `openclaw:local`         | Main AI gateway (Node.js)  | `127.0.0.1:18789`, `127.0.0.1:18790` |
| `dns-allowlist`    | `coredns/coredns:1.11.1` | DNS-based domain allowlist | `172.30.0.10:53` (internal only)     |
| `caddy`            | `caddy:2-alpine`         | Reverse proxy for web UI   | `127.0.0.1:80`                       |
| `openclaw-cli`     | `openclaw:local`         | One-shot CLI container     | None (exits after command)           |

## Channels & Integrations

- **Web UI**: `http://openclaw.localhost` (device auth disabled for Docker)
- **Telegram**: `@noamelf_bot` (token in `openclaw.json`, user ID `2063253135`)
- **WhatsApp**: `+972545551680` via Baileys (session in `~/.openclaw/credentials/`)
- **Gmail/Calendar/Tasks**: via `gog` (gogcli v0.11.0) for `noam.elf@gmail.com`
- **LLM**: Anthropic Claude Opus 4.6 (`ANTHROPIC_API_KEY` in `.env`)

## File Locations

### Repo (Docker host)

| File                        | Purpose                                              |
| --------------------------- | ---------------------------------------------------- |
| `.env`                      | Docker Compose env vars (secrets, paths, ports)      |
| `docker-compose.yml`        | Service definitions, networking, resource limits     |
| `Dockerfile`                | Image build (node:22-bookworm + pnpm + gog)          |
| `Caddyfile`                 | Caddy reverse proxy config (`:80` -> gateway)        |
| `network-lockdown/Corefile` | CoreDNS domain allowlist                             |
| `docker-setup.sh`           | Initial setup script (builds image, runs onboarding) |

### Config (mounted into container at `/home/node/.openclaw`)

| Host Path                                    | Purpose                                       |
| -------------------------------------------- | --------------------------------------------- |
| `~/.openclaw/openclaw.json`                  | Main config (gateway mode, channels, plugins) |
| `~/.openclaw/credentials/`                   | Channel session data (WhatsApp, etc.)         |
| `~/.openclaw/devices/paired.json`            | Paired devices + operator bearer token        |
| `~/.openclaw/config/gogcli/credentials.json` | Google OAuth client credentials               |
| `~/.openclaw/config/gogcli/keyring/`         | Encrypted gog OAuth tokens                    |
| `~/.openclaw/workspace/`                     | Agent workspace directory                     |

## Common Commands

### Stack management

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# Rebuild image after code changes
docker build -t openclaw:local -f Dockerfile .
docker compose up -d --force-recreate openclaw-gateway

# View logs
docker compose logs -f openclaw-gateway
docker compose logs -f dns-allowlist
docker compose logs -f caddy
```

### Gateway operations (exec into running container)

```bash
# Run CLI commands inside the gateway container
docker compose exec openclaw-gateway node dist/index.js <command>

# Check health
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"

# Channel status
docker compose exec openclaw-gateway node dist/index.js channels status
```

### Google Workspace (gog)

```bash
# Send email
docker compose exec openclaw-gateway gog gmail send --to "recipient@example.com" --subject "Subject" --body "Body"

# List calendar events
docker compose exec openclaw-gateway gog calendar events

# List task lists
docker compose exec openclaw-gateway gog tasks lists
```

### DNS allowlist management

To allow a new domain, add a server block to `network-lockdown/Corefile`:

```
newdomain.example.com {
    log
    forward . 8.8.8.8 8.8.4.4
}
```

Then restart CoreDNS:

```bash
docker compose restart dns-allowlist
```

### Testing DNS from gateway

```bash
# Test allowed domain resolves
docker compose exec openclaw-gateway node -e "require('dns').resolve('api.anthropic.com', (e,a) => console.log(e||a))"

# Test blocked domain is refused
docker compose exec openclaw-gateway node -e "require('dns').resolve('evil.com', (e,a) => console.log(e||a))"
```

## Security Hardening

| Measure                     | Details                                                                                   |
| --------------------------- | ----------------------------------------------------------------------------------------- |
| **LAN-only ports**          | All Docker ports bound to `127.0.0.1`                                                     |
| **DNS allowlist**           | CoreDNS at `172.30.0.10` blocks all non-allowlisted domains                               |
| **Non-root container**      | Gateway runs as `node` user (uid 1000)                                                    |
| **Resource limits**         | 2GB RAM, 2 CPUs on gateway                                                                |
| **Strong keyring password** | 64-char hex for gog token encryption                                                      |
| **File permissions**        | `~/.openclaw` is `700`, `.env` is `600`                                                   |
| **Auto-restart**            | `restart: unless-stopped` + Docker Desktop login item                                     |
| **Device auth disabled**    | `dangerouslyDisableDeviceAuth: true` (required for Docker, mitigated by LAN-only binding) |

### Allowed domains (DNS allowlist)

| Domain                        | Service               |
| ----------------------------- | --------------------- |
| `api.anthropic.com`           | Claude API            |
| `api.telegram.org`            | Telegram Bot API      |
| `web.whatsapp.com`            | WhatsApp WebSocket    |
| `whatsapp.net` (+ subdomains) | WhatsApp media CDN    |
| `whatsapp.com` (+ subdomains) | WhatsApp calls        |
| `gmail.googleapis.com`        | Gmail API             |
| `www.googleapis.com`          | Google APIs           |
| `oauth2.googleapis.com`       | Google OAuth          |
| `tasks.googleapis.com`        | Google Tasks API      |
| `calendar.googleapis.com`     | Google Calendar API   |
| `accounts.google.com`         | Google account auth   |
| `raw.githubusercontent.com`   | Baileys version check |

## Environment Variables (`.env`)

| Variable                 | Purpose                                     |
| ------------------------ | ------------------------------------------- |
| `OPENCLAW_CONFIG_DIR`    | Host path to `~/.openclaw`                  |
| `OPENCLAW_WORKSPACE_DIR` | Host path to agent workspace                |
| `OPENCLAW_GATEWAY_TOKEN` | Auth token for gateway API                  |
| `OPENCLAW_GATEWAY_BIND`  | Network bind mode (`lan`)                   |
| `ANTHROPIC_API_KEY`      | Anthropic API key for Claude                |
| `GOG_KEYRING_PASSWORD`   | Encryption key for gog OAuth tokens         |
| `GOG_ACCOUNT`            | Google account email (`noam.elf@gmail.com`) |

## Troubleshooting

- **Gateway crash loop**: Check `docker compose logs openclaw-gateway`. Ensure `gateway.mode=local` in `openclaw.json`.
- **Web UI 308 redirect**: Caddyfile must use `:80` not `openclaw.localhost` (prevents auto-HTTPS).
- **WhatsApp not connecting**: Verify `plugins.entries.whatsapp.enabled: true` in `openclaw.json`. WhatsApp is a bundled plugin disabled by default.
- **WebSocket refused**: Ensure `gateway.controlUi.allowedOrigins` includes `http://openclaw.localhost`.
- **gog auth expired**: Re-auth on host Mac, `gog auth tokens export`, copy to container, `gog auth tokens import`.
- **DNS issues**: `docker compose logs dns-allowlist` shows all queries. Each CoreDNS server block covers the domain + all subdomains.
- **dnsmasq does NOT work** in Docker Desktop for Mac. CoreDNS is the only viable option.
- **CLI binary not in PATH**: Use `node dist/index.js` or `node /app/openclaw.mjs` inside containers.
