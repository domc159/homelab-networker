# Network Stack Installer

Single-script installer for a self-hosted network stack on Ubuntu Server 25.x (minimal install). Run it once on a fresh machine and all services start automatically.

## What gets installed

| Service | Purpose | Port |
|---|---|---|
| NPMplus | Reverse proxy with SSL (Nginx Proxy Manager Plus) | 80, 443, 8070 (admin) |
| Technitium DNS | Full-featured DNS server with web UI | 53, 8071 (web) |
| Uptime Kuma | Service monitoring & uptime dashboard | 8072 |
| WireGuard | VPN server, 10 client configs auto-generated | 51820/udp |
| Wireshark | Browser-accessible packet capture | 3000 |
| Homarr | Service dashboard / homepage | 8074 |
| Authentik | Identity provider / SSO (+ PostgreSQL + Redis) | 9000 (HTTP), 9443 (HTTPS) |

## Requirements

- Ubuntu Server 25.x (minimal)
- Root access
- Internet connection

No other preparation is needed. The script installs Docker, configures the kernel, frees port 53, creates all directories, generates configs and passwords, and starts all containers.

## Usage

```bash
git clone https://github.com/youruser/network-stack.git
cd network-stack
sudo bash install.sh
```

At the end of the run the script prints a summary with all URLs and generated passwords.

## Directory layout

```
/home/networker/
    docker-compose.yml
    .env                  # generated passwords, edit before first run if needed
    link-wg-certs.sh      # helper to symlink WireGuard client configs after first boot

/home/networkder_data/
    npmplus/
    dns/
    uptimekuma/
    wireguard/
        config/           # peer1-peer10 generated here on first WireGuard start
        certs/            # symlinks created by link-wg-certs.sh
    wireshark/
    homarr/
    authentik/
        media/
        templates/
        geoip/
    postgresql/
    redis/
```

## Post-install steps

**Required:**

Edit `/home/networker/.env` and set:
- `DOMAIN` -- your public domain or hostname
- `INTERNAL_SUBNET` -- your home LAN subnet (e.g. `192.168.1.0/24`)

Forward UDP port `51820` on your home router to the server LAN IP for WireGuard to work externally.

**WireGuard client configs:**

After the first `docker compose up`, run:

```bash
sudo bash /home/networker/link-wg-certs.sh
```

Client `.conf` files and QR code PNGs will be available under `/home/networkder_data/wireguard/certs/peer1` through `peer10`.

**Wireshark (HTTPS required):**

Wireshark runs with `network_mode: host` and listens on port `3000` over plain HTTP. Because the Selkies frontend requires a secure context, access it through NPMplus as a reverse proxy with SSL termination and WebSocket support enabled.

In the NPMplus admin UI (`http://<server-ip>:8070`):
1. Add a Proxy Host pointing to `http://<server-ip>:3000`
2. Enable WebSocket support
3. Assign an SSL certificate and enable Force SSL

**Authentik -- first-time setup:**

Authentik requires a one-time setup wizard before it is usable. After all containers are running, open:

```
http://<server-ip>:9000/if/flow/initial-setup/
```

Create an admin account and complete the wizard. Authentik will be available at `http://<server-ip>:9000` (or via NPMplus with SSL).

To protect other services (NPMplus, DNS UI, Homarr, ...) with Authentik SSO, add a Proxy Provider in Authentik and point the NPMplus Proxy Host for each service at the Authentik outpost instead of directly at the service.

Generated secrets (`AUTHENTIK_SECRET_KEY` and `PG_PASS`) are stored in `.env`. Do not regenerate them after first boot -- doing so will break the database connection and all stored sessions.

## Managing the stack

```bash
cd /home/networker

# View running containers
docker compose ps

# View logs for a specific service
docker compose logs -f wireguard

# Restart a single service
docker compose restart npmplus

# Stop everything
docker compose down

# Pull updated images and restart
docker compose pull && docker compose up -d
```

## Notes

- `/home/networker/.env` contains plaintext passwords. Do not commit it to version control. It is created with mode `600`.
- `systemd-resolved` stub listener is disabled during install so Technitium DNS can bind port 53. System DNS resolution continues to work via `systemd-resolved` using upstream servers `1.1.1.1` and `8.8.8.8`.
- For WireGuard clients to reach your full home LAN, add a static route on your home router: `10.13.13.0/24` via the server LAN IP.
- Technitium DNS default admin credentials are set from `.env` at first boot. Change them via the web UI after logging in.
- Authentik depends on PostgreSQL and Redis. Both containers use health checks so Authentik only starts when the database is ready. On slow hardware the first startup may take 1-2 minutes.
- PostgreSQL data is persisted in `/home/networkder_data/postgresql/`. Do not delete this directory after first boot.
- `AUTHENTIK_SECRET_KEY` in `.env` is used to sign sessions and tokens. Changing it after first boot will invalidate all active sessions.
