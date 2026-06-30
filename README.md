# Home Media Server

A self-hosted media automation stack run with Docker Compose on a Windows
machine (Docker Desktop + WSL2). It downloads, organizes, transcodes, and serves
movies and TV — with all torrent traffic forced through a VPN, network-wide ad
blocking, and an n8n-based monitoring/automation layer.

> **Companion frontend:** [OmniList](https://github.com/Jtmalina/omni-list)
> ([live app](https://the-omni-list.vercel.app)) is the app used to browse and
> request media; it talks to Radarr/Sonarr over Tailscale.

---

## Architecture at a glance

```
                       ┌─────────── Tailscale Funnel (public) ───────────┐
                       │   Radarr (443)        Sonarr (8443)             │
                       └────────────────────────┬────────────────────────┘
                                                 │
  OmniList ──────────────────────────────────────┘   (HTTPS + API key)

  Prowlarr ──pushes indexers──▶ Radarr / Sonarr ──grab──▶ qBittorrent
     │                                                       │
     └─ FlareSolverr (Cloudflare solve)         network_mode: service:gluetun
                                                             │
                                                  Gluetun (ProtonVPN WireGuard)
                                                             │
                                                          Internet

  Radarr/Sonarr ──import (hardlink/copy)──▶ media library ──▶ Plex (native)
  Tdarr ──▶ transcodes library for Plex compatibility
  Decluttarr ──▶ prunes stalled/slow downloads from the *arr queues
  n8n ──▶ health alerts, daily summary, CF indexer auto-toggle
  Pi-hole ──▶ network-wide DNS ad blocking
```

**Key design point:** qBittorrent uses `network_mode: service:gluetun`, so
**all** of its traffic exits through the VPN. If Gluetun can't connect,
qBittorrent won't start (fail-closed). Radarr/Sonarr reach it at `gluetun:8080`
on the internal Docker network.

---

## Services

| Service | Port(s) | Purpose |
|---|---|---|
| **Radarr** | 7878 | Movie management |
| **Sonarr** | 8989 | TV management |
| **Prowlarr** | 9696 | Indexer aggregator (feeds Radarr/Sonarr) |
| **FlareSolverr** | 8191 | Cloudflare challenge solver (proxy for Prowlarr) |
| **qBittorrent** | 8080 | Torrent client — internal only, behind the VPN |
| **Gluetun** | — | VPN container (ProtonVPN WireGuard) |
| **Tdarr** | 8265 / 8266 | Automated transcoding for Plex compatibility |
| **n8n** | 5678 | Workflow automation — monitoring & self-healing |
| **Pi-hole** | 53, 8088 | Network-wide DNS ad blocking (UI on 8088) |
| **Decluttarr** | — | Auto-removes stalled/slow/failed downloads |

### Runs natively on Windows (not in Docker)

| Component | Endpoint | Purpose |
|---|---|---|
| **Plex** | `host.docker.internal:32400` | Media server / playback |
| **windows_exporter** | `host.docker.internal:9182` | Host CPU/RAM/disk/temp metrics for n8n |
| **Tailscale** | — | Remote access; Funnel exposes Radarr/Sonarr publicly |

---

## Prerequisites

- [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/) (WSL2 backend)
- A ProtonVPN account with WireGuard config access
- (Optional) Tailscale for remote access, Plex for playback

---

## Quick start

```powershell
# 1. Clone
git clone https://github.com/Jtmalina/home-media-server.git
cd home-media-server

# 2. Create your env file and fill it in (see Environment variables below)
Copy-Item .env.example .env
notepad .env

# 3. (Recommended) install the secret-scanning pre-commit hook
cp scripts/git-hooks/pre-commit .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit

# 4. Bring the stack up
docker compose up -d
docker compose ps
```

Then wire the apps together — see **[SETUP.md](SETUP.md)** for the click-by-click
first-run guide (qBittorrent password & download path, Prowlarr → Radarr/Sonarr,
download clients, root folders, Tdarr libraries).

---

## Environment variables

All secrets live in `.env` (gitignored). Copy `.env.example` and fill in:

| Variable | Required | Description |
|---|---|---|
| `WIREGUARD_PRIVATE_KEY` | ✅ | ProtonVPN WireGuard `[Interface] PrivateKey` |
| `WIREGUARD_ADDRESSES` | ✅ | `[Interface] Address`, e.g. `10.2.0.2/32` |
| `VPN_ENDPOINT_IP` | ✅ | `[Peer] Endpoint` IP (no port) |
| `VPN_ENDPOINT_PORT` | ✅ | `[Peer] Endpoint` port (usually `51820`) |
| `WIREGUARD_PUBLIC_KEY` | ✅ | `[Peer] PublicKey` |
| `TZ` | | Timezone, e.g. `America/New_York` |
| `PUID` / `PGID` | | User/group IDs for file ownership (default `1000`) |
| `PIHOLE_PASSWORD` | | Pi-hole admin UI password |
| `RADARR_KEY` | | Radarr API key — used by Decluttarr + n8n |
| `SONARR_KEY` | | Sonarr API key — used by Decluttarr + n8n |
| `PROWLARR_KEY` | | Prowlarr API key — used by n8n |

Get WireGuard values from <https://account.proton.me/u/0/vpn/WireGuard> (use the
**GNU/Linux → Router** config type). Get each *arr key from its
**Settings → General → API Key**.

> **Never commit real keys.** The `scripts/git-hooks/pre-commit` hook blocks any
> commit containing a 32-char hex string (the *arr API-key shape). The n8n
> workflows reference keys via `{{ $env.RADARR_KEY }}` so secrets never live in
> tracked files.

---

## Automation & monitoring (n8n)

Importable workflow JSON lives in `n8n-workflows/`. After importing each one,
link the Gmail SMTP credential to the email nodes and toggle it **Active**.

| Workflow | Schedule | What it does |
|---|---|---|
| `1-service-health-alerts.json` | every 5 min | Emails if any service is down, disk is low, or RAM/temp spikes are sustained across 2 checks |
| `2-daily-summary.json` | 8am daily | Service status, download queues, system health (RAM/disk/temp), and indexer health |
| `3-cf-indexer-autotoggle.json` | every 6h | Tests indexers; toggles the FlareSolverr `cf` tag on failing ones and reverts if it doesn't help |

These read host metrics from **windows_exporter** and call the *arr/Prowlarr
APIs. n8n is configured with `NODE_FUNCTION_ALLOW_BUILTIN=http` (so Code nodes
can call APIs) and `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` (so `{{ $env.X }}`
resolves). Email uses Gmail SMTP (`smtp.gmail.com:465`, SSL) with a Google App
Password.

**Decluttarr** runs continuously (no UI) and prunes downloads that are stalled,
slow (< `MIN_SPEED`), stuck on metadata, or failed — then blocklists them so the
*arr grabs a working release. Tunable via env in `docker-compose.yml`
(`REMOVE_STALLED`, `REMOVE_SLOW`, `MIN_SPEED`, `MAX_STRIKES`, …).

---

## APIs used

| API | Base | Consumed by |
|---|---|---|
| Radarr | `http://radarr:7878/api/v3` | Prowlarr, Decluttarr, n8n, OmniList |
| Sonarr | `http://sonarr:8989/api/v3` | Prowlarr, Decluttarr, n8n, OmniList |
| Prowlarr | `http://prowlarr:9696/api/v1` | n8n (CF auto-toggle, indexer health) |
| qBittorrent | `http://gluetun:8080/api/v2` | n8n (health check) |
| Tdarr | `http://tdarr:8265` | n8n (health check) |
| Plex | `http://host.docker.internal:32400` | n8n (health check) |
| windows_exporter | `http://host.docker.internal:9182/metrics` | n8n (system metrics) |
| Gmail SMTP | `smtp.gmail.com:465` | n8n (email) |

All *arr/Prowlarr calls authenticate with the API key via the `X-Api-Key` header
or `?apikey=` query param.

---

## Remote access & frontend

### Tailscale Funnel (public exposure)

Radarr and Sonarr are intentionally exposed to the public internet via Tailscale
Funnel (which only supports ports 443, 8443, 10000):

| Service | Public URL |
|---|---|
| Radarr | `https://<your-host>.ts.net` (443 → 7878) |
| Sonarr | `https://<your-host>.ts.net:8443` (8443 → 8989) |

Funnel terminates TLS, so use `https://`. Everything else stays on the private
Tailscale network or localhost.

### Frontend — OmniList

[OmniList](https://github.com/Jtmalina/omni-list) — live at
**[the-omni-list.vercel.app](https://the-omni-list.vercel.app)** — is the
companion app used to browse the library and request new movies/shows. Point it
at the Radarr/Sonarr **Funnel URLs** above and paste each service's **API key**
into its settings. Because Funnel exposes these publicly, treat the API keys as
sensitive and rotate them if they ever leak.

---

## Maintenance

```powershell
docker compose up -d            # start / apply changes
docker compose ps               # status
docker compose logs -f gluetun  # debug VPN connection
docker compose restart sonarr   # restart one service
docker compose pull; docker compose up -d   # update all images
docker compose down             # stop everything
```

After a reboot, Docker Desktop auto-starts and `restart: unless-stopped` brings
every container back. Verify with `docker compose ps`.

---

## Repo layout

| Path | Purpose |
|---|---|
| `docker-compose.yml` | All service definitions; reads secrets from `.env` |
| `.env.example` | Template for `.env` (copy and fill in) |
| `SETUP.md` | Step-by-step first-run wiring guide |
| `CLAUDE.md` | Detailed operational notes (windows_exporter, gotchas) |
| `n8n-workflows/` | Importable n8n automation workflows |
| `scripts/git-hooks/pre-commit` | Secret-scanning commit guard |

---

## Security notes

- `.env` is gitignored; **never commit credentials**. The pre-commit hook
  enforces this.
- Radarr/Sonarr are publicly reachable via Funnel — keep their API keys secret
  and rotate them if exposed (regenerate in each app, update `.env`, restart
  `n8n`/`decluttarr`, update Prowlarr's app connections and OmniList).
- qBittorrent is never exposed publicly; it is reachable only through Gluetun.
