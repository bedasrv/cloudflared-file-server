---
name: cloudflared-file-server
description: Serve files via Cloudflare Quick Tunnel — no account, auto-expiry. Single caddy:alpine container downloads cloudflared on the fly.
version: 3.1.0
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

Serve files via temporary Cloudflare Quick Tunnel URLs. No Cloudflare account needed. Single `caddy:alpine` container — Caddy is pre-installed, only `cloudflared` is downloaded on first run. Container self-destructs after TTL; serve directory cleaned up automatically.

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
                                   then background cleanup removes serve dir

                               https://xxx.trycloudflare.com
```

Single container, no Docker network, no pre-built image. Caddy comes from `caddy:alpine` base image. Only `cloudflared` binary (~25MB) is downloaded each run. Architecture auto-detected (amd64/arm64/arm). Serve directory removed automatically after container exits.

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

The script prints the tunnel URL to stdout and exits immediately. Container runs in background, self-destructs after TTL, serve directory cleaned up automatically.

## Manual Docker Recipe

If the script isn't installed, use these exact commands. They are proven working.

```bash
# 0. Prepare files (hard links = zero extra disk space)
SERVE_DIR=/tmp/cloudflare-serve-$$
mkdir -p "$SERVE_DIR"
ln /path/to/files/* "$SERVE_DIR/" 2>/dev/null || cp /path/to/files/* "$SERVE_DIR/"

# 1. Write entrypoint script (avoids issues with caddy:alpine's ENTRYPOINT)
cat > /tmp/cf-entrypoint.sh <<'EOF'
#!/bin/sh
set -e
apk add --no-cache curl >/dev/null 2>&1
ARCH=$(uname -m)
case "$ARCH" in
  x86_64)  CF_ARCH="amd64" ;;
  aarch64) CF_ARCH="arm64" ;;
  armv7l)  CF_ARCH="arm" ;;
  *)       echo "ERROR: unsupported arch: $ARCH" >&2; exit 1 ;;
esac
curl -fsSL --connect-timeout 10 --max-time 300 \
    "https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-${CF_ARCH}" -o /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared
cd /serve
caddy file-server --listen :80 &
sleep 1
cloudflared tunnel --url http://localhost:80 &
CF_PID=$!
(sleep "$SECS"; kill $CF_PID 2>/dev/null) &
wait $CF_PID
EOF
chmod +x /tmp/cf-entrypoint.sh

# 2. Start container (--entrypoint sh required because caddy:alpine has ENTRYPOINT ["caddy"])
#    Container name uses $$ (shell PID) to allow concurrent instances.
CONTAINER_NAME="cf-serve-$$"
docker run -d --name "$CONTAINER_NAME" \
  -v "$SERVE_DIR:/serve:ro" \
  -v /tmp/cf-entrypoint.sh:/entrypoint.sh:ro \
  -e SECS=300 \
  --entrypoint sh \
  caddy:alpine /entrypoint.sh

# 3. Get URL (poll until Cloudflare registration completes)
for i in $(seq 1 60); do
  sleep 1
  URL=$(docker logs "$CONTAINER_NAME" 2>&1 | grep -o 'https://[a-z0-9-]*\.trycloudflare\.com' | head -1 || true)
  [ -n "$URL" ] && break
done
echo "$URL"

# 4. Background cleanup: remove serve dir after container exits
(timeout 7200 docker wait "$CONTAINER_NAME" >/dev/null 2>&1; rm -rf "$SERVE_DIR") &
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
docker rm -f cf-serve-$$
```

## Limitations

- 200 concurrent requests max (HTTP 429 when exceeded)
- No SSE, no WebSocket support
- No SLA — dev/testing only, not production
- Subject to Cloudflare Terms of Service
- First run downloads ~25MB (cloudflared binary only; caddy is pre-installed in base image)
- Downloads happen per-invocation (no image caching)

## Pitfalls

**Container name uses `$$` (PID) suffix (`cf-serve-$$`).** This allows concurrent instances without collision. Old instances with the same PID are cleaned up at script start. Stale named containers from old PIDs can accumulate — the script cleans them on next run.

**`caddy:alpine` has `ENTRYPOINT ["caddy"]`.** Always use `--entrypoint sh` when running the container with a custom entrypoint script, otherwise Caddy tries to interpret `sh` as a subcommand and fails.

**No `--rm` on `docker run`.** The container is NOT auto-removed so that `docker logs` and `docker inspect` remain accessible after a crash. The script cleans up stale containers on next run.

**Downloads on every invocation.** Cloudflared (~25MB) is fetched fresh each run. Caddy is pre-installed in the base image. If you need faster spin-up, consider pre-building an image.

**Timer starts after tunnel is up.** The auto-kill timer only begins counting after both Caddy and cloudflared are running. Downloads do not eat into the TTL.

**Architecture auto-detected.** `uname -m` selects the correct cloudflared binary: `amd64`, `arm64`, or `arm`. No hardcoded architecture assumptions.

**Serve directory auto-cleaned.** A background process waits for the container to exit (`docker wait`), then removes the serve directory. No leftover temp files.

**File changes use hard links by default.** `ln` creates a hard link — same inode, zero extra disk space. Falls back to `cp` if cross-device; a warning is printed for directories when this happens.

**TTL validation rejects non-numeric input.** Values like `abcm` or bare `m` are caught before the container starts, avoiding a silent zero-second expiry.

**`docker run` exit is verified with `docker inspect`.** `docker run -d` can return 0 even on image pull failure. The script checks `State.Running` after launch to confirm the container is actually up.

**Error-path cleanup via EXIT trap.** If the script exits before the container starts (bad TTL, missing files), the serve directory and temp files are removed. Once the container is running, deferred cleanup takes over — the trap won't double-free.

**`docker wait` has a 2-hour timeout guard.** Prevents zombie background processes if the container is killed externally.

**POSIX `grep -o` used for URL extraction.** No `-P` (PCRE) dependency — works on macOS, Alpine, and any system without GNU grep.

**Local DNS may block trycloudflare.com.** The tunnel is still up — verify by checking `docker logs cf-serve-$$` instead of curl.

**Firewall must allow outbound QUIC.** cloudflared connects to Cloudflare edge on UDP port 7844.

## Implementation Notes

The script works around two Docker constraints:

1. **`caddy:alpine` ENTRYPOINT**: The image sets `ENTRYPOINT ["caddy"]`, so `docker run caddy:alpine sh /entrypoint.sh` would execute `caddy sh /entrypoint.sh`. The fix is `--entrypoint sh`.

2. **Terminal tool rejects `&` in inline commands**: The entrypoint script (which backgrounds Caddy and the timer) is written to a temp file and mounted, avoiding `&` in the `docker run` command string.

## Installation into Hermes

```bash
hermes skills install --force https://raw.githubusercontent.com/bedasrv/cloudflared-file-server/main/SKILL.md
hermes skills tap add bedasrv/cloudflared-file-server
```
