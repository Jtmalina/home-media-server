# Home Media Server — CLAUDE.md

## What this repo is

Docker Compose setup for a self-hosted home media server running on a
Windows 10/11 machine. Designed to be pulled and run on the server machine
with minimal manual steps.

## Machine context

- **OS:** Windows 10/11 (dual-boot with Ubuntu, Windows is default boot)
- **Docker:** Docker Desktop with WSL2
- **VPN:** ProtonVPN via WireGuard (routed through Gluetun container)
- **Remote access:** Tailscale MagicDNS — hostname `desktop-mcopf27.tail2717ef.ts.net`
- **Media drives:** D:\ drive
  - Movies: `D:\Home Media\Movies`
  - TV: `D:\Home Media\TV`
  - Downloads: `D:\Home Media\Downloads`
  - Per-service config: `D:\Home Media\config\<servicename>`

## Services

| Service | Port | Purpose |
|---|---|---|
| Radarr | 7878 | Movie management — used by OmniList |
| Sonarr | 8989 | TV show management — used by OmniList |
| Prowlarr | 9696 | Indexer aggregator for Radarr + Sonarr |
| FlareSolverr | 8191 | Cloudflare bypass proxy (used by Prowlarr) |
| qBittorrent | 8080 | Torrent client — internal only, behind VPN |
| Gluetun | — | VPN container (ProtonVPN WireGuard) |
| Tdarr | 8265 | Automated transcoding for Plex compatibility |
| n8n | 5678 | Workflow automation — media pipeline monitoring |
| Pi-hole | 8088 | Network-wide DNS ad blocking (web UI; DNS on 53) |

qBittorrent uses `network_mode: service:gluetun` — all its traffic exits
through the VPN. Radarr/Sonarr talk to qBittorrent at host `gluetun:8080`
on the internal Docker network.

**Plex** runs natively on Windows (not in Docker), reachable from containers
at `host.docker.internal:32400`. **windows_exporter** runs as a native Windows
service (not in Docker) — see "System metrics" below.

## What's missing / still needs to be done

### 1. WireGuard credentials (REQUIRED before first run)
Copy `.env.example` to `.env` and fill in all 5 WireGuard values.
Get them from: https://account.proton.me/u/0/vpn/WireGuard

```
WIREGUARD_PRIVATE_KEY=
WIREGUARD_ADDRESSES=
VPN_ENDPOINT_IP=
VPN_ENDPOINT_PORT=
WIREGUARD_PUBLIC_KEY=
```

### 2. First-time app wiring (after `docker compose up -d`)
Once containers are running, connect the apps to each other:

**qBittorrent** (http://localhost:8080)
- Default login: admin / adminadmin — change immediately
- Set download path to `/downloads`

**Prowlarr** (http://localhost:9696)
- Add indexers
- Settings → Apps → Add Radarr: URL `http://radarr:7878`, API key from Radarr → Settings → General
- Settings → Apps → Add Sonarr: URL `http://sonarr:8989`, API key from Sonarr → Settings → General
- Settings → Indexers → Add FlareSolverr proxy: URL `http://flaresolverr:8191`

**Radarr** (http://localhost:7878)
- Settings → Download Clients → Add qBittorrent: host `gluetun`, port `8080`
- Settings → Media Management → Root Folder → `/movies`

**Sonarr** (http://localhost:8989)
- Settings → Download Clients → Add qBittorrent: host `gluetun`, port `8080`
- Settings → Media Management → Root Folder → `/tv`

### 3. Tdarr first-run setup
After `docker compose up -d`, open http://localhost:8265:

1. **Nodes** tab — confirm `MainNode` shows as connected and enabled
2. **Libraries** tab → Add Library:
   - Movies: source path `/media/movies`, transcode cache `/temp`
   - TV: source path `/media/tv`, transcode cache `/temp`
3. **Flows** tab → Community → search "Plex" → import the **"Plex Compatible"** flow
4. Assign the flow to each library and enable scanning

Tdarr will scan your library, flag incompatible files, and transcode them in the background.
The `D:\Home Media\transcode_cache` folder is used as scratch space — it can grow large during
transcoding but clears automatically when done.

### 4. OmniList settings update
After confirming services are reachable, update the Servarr config in OmniList
(gear icon → settings) with the Tailscale Funnel URLs:
- Radarr: `https://desktop-mcopf27.tail2717ef.ts.net`
- Sonarr: `https://desktop-mcopf27.tail2717ef.ts.net:8443`

Note: use `https://` — Tailscale Funnel provides TLS termination.

### 5. Sonarr Tailscale port
Sonarr is exposed via Tailscale Funnel on port 8443 (Funnel only supports 443, 8443, 10000).
Funnel proxies 8443 → localhost:8989. This is already configured and working.

### 6. Auto-start verification
Docker Desktop starts with Windows and `restart: unless-stopped` handles
container restarts. After a reboot, verify all containers come back up:
```powershell
docker compose ps
```

## System metrics (windows_exporter)

The n8n monitoring workflows (RAM/temp/disk alerts + daily summary) pull host
metrics from **windows_exporter**, which runs as a **native Windows service**
(not in Docker). n8n scrapes it at `http://host.docker.internal:9182/metrics`.

Install (one-time, admin PowerShell):
```powershell
# Download the MSI (or grab latest from github.com/prometheus-community/windows_exporter/releases)
msiexec /i "C:\Temp\windows_exporter.msi" /quiet `
  ENABLED_COLLECTORS="cpu,memory,logical_disk,os,thermalzone" LISTEN_PORT="9182"
```

**Gotcha:** the collector for per-volume free space is `logical_disk`, **not**
`disk`. If you use `disk`, no disk metrics appear and the workflows show D: as
"Unavailable". Also, re-running the MSI caches the original `ENABLED_COLLECTORS`
and ignores changes — to fix the collectors after install, edit the service
directly (admin):
```powershell
sc.exe config windows_exporter binPath= "C:\PROGRA~1\windows_exporter\windows_exporter.exe --log.file eventlog --collectors.enabled cpu,memory,logical_disk,os,thermalzone"
Restart-Service windows_exporter
```

Verify it's serving the expected metrics:
```powershell
(Invoke-RestMethod "http://localhost:9182/metrics") -split "`n" |
  Select-String 'windows_logical_disk_free_bytes\{volume="D:"\}'
```

Metrics consumed by the workflows:
- `windows_os_physical_memory_free_bytes` / `windows_os_visible_memory_bytes` — RAM %
- `windows_logical_disk_free_bytes{volume="D:"}` — D: free space
- `windows_thermalzone_temperature_celsius{...}` — temps

## n8n monitoring workflows

Importable JSON lives in `n8n-workflows/`. After importing, link the Gmail SMTP
credential to the email nodes and toggle each workflow **Active**.
- `1-service-health-alerts.json` — every 5 min; emails on any service down,
  low disk, or RAM/temp spikes sustained across 2 consecutive checks.
- `2-daily-summary.json` — 8am daily status email (service status, download
  queues, system health).

Email uses Gmail SMTP (`smtp.gmail.com:465`, SSL) with a Google App Password.

## Key commands

```powershell
# Start all services
docker compose up -d

# Stop all services
docker compose down

# View logs (useful for debugging VPN connection)
docker compose logs -f gluetun
docker compose logs -f radarr

# Restart a single service
docker compose restart sonarr

# Pull latest images and restart
docker compose pull && docker compose up -d
```

## Files

- `docker-compose.yml` — service definitions, reads credentials from `.env`
- `.env.example` — template for credentials, copy to `.env` and fill in
- `.env` — gitignored, never commit this
- `SETUP.md` — step-by-step first-run guide
