---
name: cloudflared-file-server
description: Serve files via Cloudflare Quick Tunnel — no account, auto-expiry. Spin up temporary trycloudflare.com URLs using Docker containers.
version: 2.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [file-sharing, docker, cloudflare, tunnel, networking]
    category: devops
    min_hermes_version: "2.0"
    dependencies:
      commands: [docker]
      images: [nginx:alpine, cloudflare/cloudflared:latest, docker:cli]
---

# Cloudflared File Server

Serve files via temporary Cloudflare Quick Tunnel URLs. No Cloudflare account needed. Uses Docker containers for isolation — no host-level process juggling, no lock files, no DNS interference.

## Trigger — When to Use This Skill

Load this skill whenever:
- User asks for a file or link to a file
- You generate or create a file the user needs to access
- A file exceeds Discord (25MB) or Telegram (2GB) limits
- You need to share binaries, images, logs, or any artifact

## Architecture

```
Files on host                  Docker bridge network
     │                              │
     ▼                              ▼
nginx:alpine (:80)  ←────────  cloudflare/cloudflared  ──→  https://xxx.trycloudflare.com
     │                              │
     └── shared volume mount ───────┘         docker:cli (auto-kill timer)
```

Three containers, one network, zero host-level processes. Timer container uses mounted Docker socket to kill siblings after TTL — fully Docker-native, no `nohup`, no `disown`, no shell backgrounding.

## Automated Serve Script

The repo includes `scripts/serve` — a single command that does everything. Install it:

```bash
# Copy to PATH
sudo cp scripts/serve /usr/local/bin/cloudflared-serve
sudo chmod +x /usr/local/bin/cloudflared-serve
```

Usage:
```bash
cloudflared-serve /path/to/file.png
cloudflared-serve /path/to/dir/ 5m
cloudflared-serve file1.png file2.iso file3.pdf 10m
```

First argument is a file or directory. Second (optional) is TTL: `30s`, `5m`, `1h` (default: `5m`).

The script prints the tunnel URL to stdout and exits immediately. Files are served until TTL expires.

## Manual Docker Recipe

If the script isn't installed, use these exact commands. They are proven working.

```bash
# 0. Prepare files (hard links = zero extra disk space)
SERVE_DIR=/tmp/cloudflare-serve
mkdir -p "$SERVE_DIR"
ln /path/to/files/* "$SERVE_DIR/" 2>/dev/null || cp /path/to/files/* "$SERVE_DIR/"

# 1. Pull images (one-time)
docker pull nginx:alpine cloudflare/cloudflared:latest docker:cli

# 2. Create network
docker network create cf-serve

# 3. Start nginx
docker run -d --name cf-nginx --network cf-serve \
  -v "$SERVE_DIR:/usr/share/nginx/html:ro" \
  nginx:alpine

# 4. Start cloudflared tunnel
docker run -d --name cf-tunnel --network cf-serve \
  cloudflare/cloudflared tunnel --url http://cf-nginx:80

# 5. Auto-shutdown timer (pure Docker, no host-level nohup)
# Adjust sleep value for desired TTL (300 = 5 minutes)
docker run -d --rm --name cf-timer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  docker:cli \
  sh -c "sleep 300 && docker rm -f cf-tunnel cf-nginx && docker network rm cf-serve"

# 6. Get URL (wait ~5s for Cloudflare registration)
sleep 5
docker logs cf-tunnel 2>&1 | grep -oP 'https://[-a-z0-9]+\.trycloudflare\.com'
```

## Adding Files While Running

```bash
ln newfile.png /tmp/cloudflare-serve/ 2>/dev/null || cp newfile.png /tmp/cloudflare-serve/
# Immediately available at https://xxx.trycloudflare.com/newfile.png
```

No restart needed. nginx serves from the mounted volume live.

## Verifying

```bash
# Check all files return 200
curl -sI "https://TUNNEL.trycloudflare.com/filename"
```

If local DNS blocks `*.trycloudflare.com` (dnscrypt-proxy does), the URL still works from the user's device — the containers are alive, the tunnel is up.

## Tear Down

```bash
# Manual (if no timer)
docker rm -f cf-tunnel cf-nginx
docker network rm cf-serve

# Or if timer is running (timer container self-destructs with --rm)
docker rm -f cf-timer cf-tunnel cf-nginx
docker network rm cf-serve
```

## Limitations

- 200 concurrent requests max (HTTP 429 when exceeded)
- No SSE, no WebSocket support
- No SLA — dev/testing only, not production
- Subject to Cloudflare Terms of Service
- File changes use hard links by default — zero disk overhead. Falls back to copy if cross-device.

## Pitfalls

**Never use host-level cloudflared.** Cloudflare enforces one tunnel per process. Background-process management causes SIGHUP kills even with `nohup`/`disown -h`. Always use Docker containers.

**Container names (`cf-nginx`, `cf-tunnel`, `cf-timer`, `cf-serve`) are hardcoded.** If containers/networks with these names already exist, `docker run` / `docker network create` will fail. Tear down old instances first.

**The timer container needs Docker socket access.** This is intentional — the timer uses `docker:cli` image to kill sibling containers. Without socket mount, the timer can't clean up.

**Local DNS may block trycloudflare.com.** dnscrypt-proxy2, Pi-hole, and similar resolvers may refuse `*.trycloudflare.com` subdomains. The tunnel is still up — verify by checking container logs or processes instead of curl.

**Firewall must allow outbound QUIC.** cloudflared connects to Cloudflare edge on UDP port 7844. If outbound UDP is blocked, the tunnel won't establish.

**Use hard links, not copies.** `ln` creates a hard link — same inode, zero extra disk space. The script does this automatically with fallback to `cp` if cross-device. Large files or many files are no problem.

## Installation into Hermes

From a Hermes session, install this skill with:

```bash
hermes skills install https://raw.githubusercontent.com/bedasrv/cloudflared-file-server/main/SKILL.md
```

Or add the repo as a skill source:

```bash
hermes skills tap add bedasrv/cloudflared-file-server
```

Then verify:

```bash
hermes skills list | grep cloudflared
```
