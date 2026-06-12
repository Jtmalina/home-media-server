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

qBittorrent uses `network_mode: service:gluetun` — all its traffic exits
through the VPN. Radarr/Sonarr talk to qBittorrent at host `gluetun:8080`
on the internal Docker network.

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
(gear icon → settings) with the Tailscale URLs:
- Radarr: `http://desktop-mcopf27.tail2717ef.ts.net:7878`
- Sonarr: `http://desktop-mcopf27.tail2717ef.ts.net:8989`

Note: use `http://` not `https://` — Tailscale encrypts the tunnel and these
services don't have TLS certs configured.

### 5. Sonarr Tailscale port
Sonarr previously needed to be funneled through port 8443 due to Tailscale
Funnel port restrictions. With Docker the service binds directly to the host
on port 8989, so it should be reachable on the Tailscale LAN (not Funnel)
at port 8989 without any port remapping. Verify this works after setup.

### 6. Auto-start verification
Docker Desktop starts with Windows and `restart: unless-stopped` handles
container restarts. After a reboot, verify all containers come back up:
```powershell
docker compose ps
```

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
