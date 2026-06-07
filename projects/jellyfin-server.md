# Jellyfin Home Media Server

## Project Goals

- **Cost** — minimize hardware and running costs
- **Longevity** — reduce unnecessary wear on hardware
- **Accessibility** — accessible remotely at any time, by anyone in the family

## Why Jellyfin

Jellyfin is a self-hosted, open source media server. The connection model is direct client-to-server — you own and control the entire infrastructure.

Plex was ruled out because of its architecture. Plex functions more like a node than a true home server. Client connections are routed through Plex's relay servers, which requires your server to already be online and authenticated with Plex before any client can reach it. This fundamentally breaks Wake-on-LAN — if the server is off, Plex will never let a client attempt a connection, because port 443 is never hit on your end. You are dependent on Plex's infrastructure being online.

Jellyfin is direct. If the server is reachable, it works. No third party involved.

## Usage Pattern & Power Design

My family doesn't use the server until after dinner — roughly 6PM. Most are asleep by midnight. That means worst case, the server is actually needed for around **6 hours a day**, and many days not at all.

Running a full server 24/7 for a 6 hour/day use case is:
- A waste of energy — 75% of runtime is idle
- Unnecessary wear on HDDs, RAM, and CPU
- Not in line with the cost and longevity goals of this project

The solution was **Wake-on-LAN (WoL)**. The server sleeps when not in use and is woken on demand. Access is fully on-demand — no fixed schedule.

## Architecture

### The Problem with WoL + Remote Access

WoL requires sending a magic packet to the server's local network. This works fine from inside the house, but remotely requires something always-on sitting on the local network to handle it.

### The Solution — Raspberry Pi

A Raspberry Pi stays on 24/7. It draws roughly 3-5W, making it cheap to run continuously. It serves two purposes:

1. **Reverse proxy + automatic WoL** — intercepts incoming requests, wakes the server, and forwards traffic
2. **VPN endpoint** — runs Wireguard for secure remote admin access when needed

These are separate concerns. The reverse proxy handles normal family usage automatically. The VPN is for maintenance and fixing issues remotely.

### Automatic Wake — How It Works

A custom script (`wake-server.sh`) runs on the Pi and uses `tcpdump` to watch port 443 for incoming traffic. When a connection is detected, it fires a WoL magic packet to the server. A 30-second cooldown prevents duplicate packets on subsequent requests.

```bash
# Core logic
PACKET=$(sudo timeout 5 tcpdump -i eth0 -nn port $LISTEN_PORT -c 1 2>/dev/null)
if [[ -n "$PACKET" && $((CURRENT_TIME - LAST_WAKE)) -ge $COOLDOWN_SECONDS ]]; then
    wakeonlan "$TARGET_MAC"
fi
```

**Caddy** acts as the reverse proxy, handling TLS certificates automatically and forwarding HTTPS traffic to the Jellyfin server.

The server is put into **sleep** rather than fully shut down, so it resumes in seconds rather than going through a full boot.

### End-to-End Flow

1. Browser hits `yourname.duckdns.org`
2. `wake-server.sh` detects traffic on port 443 via tcpdump and sends WoL magic packet
3. Server wakes from sleep
4. Caddy receives the HTTPS request, handles TLS, and forwards it to Jellyfin
5. Jellyfin responds

No manual intervention needed. The server wakes itself on demand.

### Network

- **DuckDNS** — dynamic DNS, keeps the domain pointed at the home's public IP
- **Caddy** — reverse proxy on the Pi, handles TLS automatically
- **Wireguard** — VPN on the Pi for remote admin access
- **HTTPS only** — HTTP is blocked at the firewall

## What I Would Do Differently

**Not Debian minimal.**

Debian minimal was a pain to initialize. Adding users, ensuring stability, and configuring everything from scratch was more work than it was worth for a home server. More importantly, the Debian minimal installer does not follow the standard partition layout. Instead of the typical 3-partition setup, it created a separate partition for each major directory. This caused two separate issues:

- The `/var` partition filled up, requiring a boot USB to resize it
- A stale fstab entry for a removed drive caused the server to hang on boot, requiring emergency mode to fix (see [troubleshooting](https://github.com/daxkharris/troubleshooting/blob/main/issues/fstab-missing-drive-hang.md))

A standard Debian install or Ubuntu Server would have been a smoother starting point.

## Future Improvements

- Usage tracking — log how many times the server wakes per day and total uptime to validate the power savings
- Improve Sleep/Wake scripts
