---
name: cloudflared-file-server
description: Serve files via Cloudflare Quick Tunnel — no account, auto-expiry. Single caddy:alpine container downloads cloudflared on the fly.
version: 3.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [file-sharing, docker, cloudflare, tunnel, networking, caddy]
    category: devops
    min_hermes_version: "2.0"
    dependencies:
      commands: [docker]
      images: [caddy:alpine]
---

# Cloudflared File Server

Serve files via temporary Cloudflare Quick Tunnel URLs. No Cloudflare account needed. Single `caddy:alpine` container — downloads `cloudflared` on first run, serves files, self-destructs after TTL. Caddy is pre-installed in the base image (no download needed).

## Trigger — When to Use This Skill

Load this skill whenever:
- User asks for a file or link to a file
- You generate or create a file the user needs to access
- A file exceeds Discord (25MB) or Telegram (2GB) limits
- You need to share binaries, images, logs, or any artifact

## Architecture

```
Files on host  ──ro mount──→  caddy:alpine container
                               ├── Caddy file-server :80 (pre-installed)
                               ├── cloudflared tunnel → localhost:80 (download once)
                               └── async sleep $TTL → kill cloudflared → container exits

                               https://xxx.trycloudflare.com
```

Single container, no Docker network, no pre-built image. Caddy comes from the `caddy:alpine` base image. Only cloudflared binary is downloaded fresh.

## Automated Serve Script

```bash
sudo cp scripts/serve /usr/local/bin/cloudflared-serve
sudo chmod +x /usr/local/bin/cloudflared-serve
```

Usage:
```bash
cloudflared-serve <ttl> <file1> [file2 ...]
```

TTL is the **first** argument: `30s`, `5m`, `1h`. Files follow.

```bash
cloudflared-serve 5m cat.png dog.png parrot.png
cloudflared-serve 30s image.png
cloudflared-serve 1h ./downloads/
```

The script prints the tunnel URL to stdout and exits immediately. Container runs in background, self-destructs after TTL.

## Manual Docker Recipe

```bash
# 0. Prepare files
SERVE_DIR=/tmp/cloudflare-serve
mkdir -p "$SERVE_DIR"
ln /path/to/files/* "$SERVE_DIR/" 2>/dev/null || cp /path/to/files/* "$SERVE_DIR/"

# 1. Start container (detached, caddy is pre-installed)
docker run -d --rm --name cf-serve \
  -v "$SERVE_DIR:/serve:ro" \
  -e TTL=300 \
  caddy:alpine \
  sh -c '
apk add --no-cache curl >/dev/null 2>&1
curl -fsSL --connect-timeout 10 --max-time 300 \
  "https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64" -o /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared
cd /serve
caddy file-server --listen :80 &
sleep 1
cloudflared tunnel --url http://localhost:80 &
CF_PID=$!
(sleep $TTL; kill $CF_PID 2>/dev/null) &
wait $CF_PID
'

# 2. Get URL
docker logs cf-serve 2>&1 | grep -oP 'https://[-a-z0-9]+\.trycloudflare\.com'
```

## Adding Files While Running

```bash
ln newfile.png /tmp/cloudflare-serve-<PID>/
# Immediately available at https://xxx.trycloudflare.com/newfile.png
```

## Verifying

```bash
curl -sI "https://TUNNEL.trycloudflare.com/filename"
```

## Tear Down

```bash
docker rm -f cf-serve
```

## Limitations

- 200 concurrent requests max (HTTP 429 when exceeded)
- No SSE, no WebSocket support
- No SLA — dev/testing only, not production
- Subject to Cloudflare Terms of Service
- First run downloads ~25MB (cloudflared binary only; caddy is pre-installed)
- Downloads happen per-invocation (no image caching)

## Pitfalls

**Container name (`cf-serve`) is hardcoded.** Tear down old instances first if any conflict.

**Downloads on every invocation.** Cloudflared (~25MB) is fetched fresh each run. Caddy is pre-installed in the base image. If you need faster spin-up, consider pre-building an image.

**File changes use hard links by default.** `ln` creates a hard link — same inode, zero extra disk space. Falls back to `cp` if cross-device.

**Local DNS may block trycloudflare.com.** The tunnel is still up — verify by checking `docker logs cf-serve` instead of curl.

**Firewall must allow outbound QUIC.** cloudflared connects to Cloudflare edge on UDP port 7844.

## Installation into Hermes

```bash
hermes skills install --force https://raw.githubusercontent.com/bedasrv/cloudflared-file-server/main/SKILL.md
hermes skills tap add bedasrv/cloudflared-file-server
```
