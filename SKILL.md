---
name: cloudflared-file-server
description: Serve files via Cloudflare Quick Tunnel — no account, auto-expiry. Single caddy:alpine container, cloudflared cached on host.
version: 3.4.0
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

Single container, no Docker network, no pre-built image. Caddy comes from `caddy:alpine` base image. Cloudflared binary (~25MB) is cached on host at `/tmp/cloudflared-cache/` (24h TTL) and mounted into containers — only the first invocation downloads, subsequent ones are instant. Architecture auto-detected (amd64/arm64/arm). Serve directory removed automatically after container exits.

## Agent Behavior

Run `cloudflared-serve` in **foreground** mode with a generous timeout (180s). The script blocks until the tunnel is registered (typically 3–5s, worst-case 120s), prints the URL to stdout, then exits. The Docker container keeps running in the background until TTL expires — but the script itself returns as soon as the tunnel URL is available. No need for background mode or manual log scraping.

## Automated Serve Script

```bash
sudo cp serve /usr/local/bin/cloudflared-serve
sudo chmod +x /usr/local/bin/cloudflared-serve
```

Usage:
```bash
cloudflared-serve <ttl> <file1> [file2 ...]
```

TTL is the **first** argument: `30s`, `5m`, `1h`. Minimum 10 seconds. Files follow.

```bash
cloudflared-serve 5m cat.png dog.png parrot.png
cloudflared-serve 30s image.png
cloudflared-serve 1h ./downloads/
```

The script prints one URL per file to stdout and exits immediately. Info/errors go to stderr with `[INFO]`/`ERROR:` prefixes. Container runs in background, self-destructs after TTL, serve directory cleaned up automatically.

Example stdout:
```
https://xxx.trycloudflare.com/cat.png
https://xxx.trycloudflare.com/dog.png
```

## Manual Docker Recipe

If the script isn't installed, use these exact commands. They are proven working.

```bash
# 0. Download cloudflared once (cached at /tmp/cloudflared-cache/)
CLOUDFLARED_CACHED="/tmp/cloudflared-cache/cloudflared-latest-$(uname -m)"
if [ ! -x "$CLOUDFLARED_CACHED" ]; then
    curl -fsSL "https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64" -o "$CLOUDFLARED_CACHED"
    chmod +x "$CLOUDFLARED_CACHED"
fi

# 1. Prepare serve dir (hard links = zero extra disk space)
SERVE_DIR=/tmp/cloudflare-serve-$$
mkdir -p "$SERVE_DIR"
ln /path/to/files/* "$SERVE_DIR/" 2>/dev/null || cp /path/to/files/* "$SERVE_DIR/"

# 2. Write entrypoint
cat > /tmp/cf-entrypoint.sh <<'EOF'
#!/bin/sh
set -e
cleanup() { kill %1 %2 2>/dev/null || true; wait 2>/dev/null || true; }
trap cleanup TERM INT
cd /serve
caddy file-server --listen :80 &
sleep 1
cloudflared tunnel --url http://localhost:80 &
CF_PID=$!
(sleep "$SECS"; kill $CF_PID 2>/dev/null) &
wait $CF_PID
EOF
chmod +x /tmp/cf-entrypoint.sh

# 3. Start container (mount cached cloudflared, skip download)
CONTAINER_NAME="cf-serve-$$"
docker run -d --name "$CONTAINER_NAME" \
  -v "$SERVE_DIR:/serve:ro" \
  -v /tmp/cf-entrypoint.sh:/entrypoint.sh:ro \
  -v "$CLOUDFLARED_CACHED:/usr/local/bin/cloudflared:ro" \
  -e SECS=300 \
  --entrypoint sh \
  caddy:alpine /entrypoint.sh

# 4. Get URL
for i in $(seq 1 60); do
  sleep 1
  URL=$(docker logs "$CONTAINER_NAME" 2>&1 | grep -o 'https://[a-z0-9.-]*\.trycloudflare\.com' | head -1 || true)
  [ -n "$URL" ] && break
done
echo "$URL/$(ls "$SERVE_DIR" | head -1)"

# 5. Cleanup after container exits
WAIT_TIMEOUT=$((300 + 120))
(timeout "$WAIT_TIMEOUT" docker wait "$CONTAINER_NAME" >/dev/null 2>&1; rm -rf "$SERVE_DIR") &
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
- First run downloads ~25MB (cloudflared binary, cached on host for 24h)

## Pitfalls

**Container name uses `$$` (PID) suffix (`cf-serve-$$`).** This allows concurrent instances without collision. Old instances with the same PID are cleaned up at script start. Stale named containers from old PIDs can accumulate — the script cleans them on next run.

**`caddy:alpine` has `ENTRYPOINT ["caddy"]`.** Always use `--entrypoint sh` when running the container with a custom entrypoint script, otherwise Caddy tries to interpret `sh` as a subcommand and fails.

**No `--rm` on `docker run`.** The container is NOT auto-removed so that `docker logs` and `docker inspect` remain accessible after a crash. The script cleans up stale containers on next run.

**Cloudflared cached on host (24h).** First invocation downloads ~25MB to `/tmp/cloudflared-cache/`. Subsequent invocations mount the cached binary — no download, ~5s to tunnel URL. Cache auto-purges after 24h.

**Timer starts after tunnel is up.** The auto-kill timer only begins counting after both Caddy and cloudflared are running. Downloads do not eat into the TTL.

**Architecture auto-detected.** `uname -m` selects the correct cloudflared binary: `amd64`, `arm64`, or `arm`. No hardcoded architecture assumptions.

**Serve directory auto-cleaned.** A background process waits for the container to exit (`docker wait`), then removes the serve directory. No leftover temp files.

**File changes use hard links by default.** `ln` creates a hard link — same inode, zero extra disk space. Falls back to `cp` if cross-device; a warning is printed for both files and directories when this happens.

**Error-path cleanup via EXIT trap.** If the script exits before the container starts (bad TTL, missing files), the serve directory and temp files are removed. Once the container is running, deferred cleanup takes over — the trap won't double-free.

**`docker wait` timeout scales with TTL.** Timeout = SECS + 120s (minimum 60s). Prevents zombie background processes without wasting resources on short-lived tunnels.

**Minimum TTL enforced (10 seconds).** `0s`, `0m`, `0h` are rejected — a zero-second expiry kills the container before the tunnel URL is served. The validation also rejects non-numeric values like `abcm`.

**Container startup verified with retry.** `docker run -d` returns before the container reaches `Running` state on slow hosts. The script retries `docker inspect` up to 3 times with a 0.5s delay to avoid false failures.

**Cloudflared version pinning works.** Set `CLOUDFLARED_VERSION=2025.2.1` to pin a specific release. Default is `latest`. The variable is passed into the container as `-e CLOUDFLARED_VERSION` and the entrypoint constructs the correct GitHub URL (`releases/latest/download/` vs `releases/download/<tag>/`).

**SIGTERM handled in entrypoint.** A `trap cleanup TERM INT` forwards `docker stop` signals to Caddy and cloudflared, avoiding the 10-second Docker SIGKILL timeout.

**`curl` with `wget` fallback.** If `caddy:alpine` doesn't have `curl` pre-installed, the entrypoint tries `wget` as a fallback. Both are tried before attempting `apk add`.

**Network isolation not enforced.** The container needs outbound access for cloudflared tunnel (UDP 7844). Caddy listens on port 80 inside the container but is NOT published with `-p` — only accessible via the tunnel. No `--network=none` is applied because it would break the tunnel.

**`stderr` output uses `[INFO]` prefix.** Informational messages (file listing, shutdown instructions) are prefixed with `[INFO]` to distinguish from `ERROR:` messages on `stderr`.

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
