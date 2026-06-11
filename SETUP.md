# Home Media Server Setup

Radarr · Sonarr · Prowlarr · FlareSolverr · qBittorrent (via ProtonVPN)

---

## Prerequisites

- [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/) installed and running
- WSL2 enabled (Docker Desktop does this automatically)

---

## Step 1 — Fill in your WireGuard credentials

1. Go to [https://account.proton.me/u/0/vpn/WireGuard](https://account.proton.me/u/0/vpn/WireGuard)
2. Create a new WireGuard config (pick any server)
3. Download or copy the config — it looks like this:

```ini
[Interface]
PrivateKey = abc123...
Address = 10.2.0.2/32

[Peer]
PublicKey = xyz789...
Endpoint = 185.159.157.5:51820
```

4. Open `docker-compose.yml` and fill in the `gluetun` environment block:

| Compose variable | Config field |
|---|---|
| `WIREGUARD_PRIVATE_KEY` | `[Interface] PrivateKey` |
| `WIREGUARD_ADDRESSES` | `[Interface] Address` |
| `VPN_ENDPOINT_IP` | `[Peer] Endpoint` (IP part only, no port) |
| `VPN_ENDPOINT_PORT` | `[Peer] Endpoint` (port, usually `51820`) |
| `WIREGUARD_PUBLIC_KEY` | `[Peer] PublicKey` |

---

## Step 2 — Set your timezone

Replace `America/New_York` in all four service blocks with your timezone.
Full list: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones

---

## Step 3 — Start everything

Open PowerShell in this folder and run:

```powershell
docker compose up -d
```

Check everything started:

```powershell
docker compose ps
```

All services should show `running`. Check gluetun specifically — if the VPN fails to connect the qBittorrent container won't start.

---

## Step 4 — First-time configuration

### qBittorrent
- Open: `http://localhost:8080`
- Default login: `admin` / `adminadmin`
- Change the password immediately in Settings → Web UI
- Set default download path to `/downloads`

### Prowlarr
- Open: `http://localhost:9696`
- Add your indexers
- Go to Settings → Apps → Add Radarr and Sonarr:
  - Radarr URL: `http://radarr:7878`
  - Sonarr URL: `http://sonarr:8989`
  - Get API keys from each app: Settings → General → API Key

### FlareSolverr
- No UI — add it in Prowlarr as a proxy
- Prowlarr → Settings → Indexers → Add Proxy → FlareSolverr
  - URL: `http://flaresolverr:8191`

### Radarr
- Open: `http://localhost:7878`
- Settings → Download Clients → Add qBittorrent:
  - Host: `gluetun` (qBittorrent shares gluetun's network)
  - Port: `8080`
- Settings → Media Management → Add Root Folder → `/movies`

### Sonarr
- Open: `http://localhost:8989`
- Settings → Download Clients → Add qBittorrent:
  - Host: `gluetun`
  - Port: `8080`
- Settings → Media Management → Add Root Folder → `/tv`

---

## Step 5 — Expose via Tailscale

All services bind to the host machine's ports, so they're reachable on your
Tailscale hostname automatically once the machine is on the Tailscale network:

| Service | URL |
|---|---|
| Radarr | `http://<tailscale-hostname>:7878` |
| Sonarr | `http://<tailscale-hostname>:8989` |
| Prowlarr | `http://<tailscale-hostname>:9696` |
| qBittorrent | internal only — not exposed |

Update these URLs in your OmniList settings (gear icon → Radarr/Sonarr).

---

## Useful commands

```powershell
# Stop everything
docker compose down

# Restart a single service
docker compose restart radarr

# View logs
docker compose logs -f gluetun    # check VPN status
docker compose logs -f radarr

# Update all images
docker compose pull
docker compose up -d
```

---

## Auto-start with Windows

Docker Desktop starts automatically with Windows by default. All containers
with `restart: unless-stopped` will come back up automatically after a reboot.
