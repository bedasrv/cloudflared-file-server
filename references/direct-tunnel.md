# Direct Cloudflared Tunnel (No Docker)

When you need to expose a running HTTP service (not static files), use cloudflared directly — no Docker container overhead.

## Usage

```bash
# Tunnel a local HTTP service
# Run from the skill directory
CLOUDFLARED_CACHE_DIR="$(pwd)/.cloudflared-cache"
mkdir -p "$CLOUDFLARED_CACHE_DIR"
CLOUDFLARED_CACHED="${CLOUDFLARED_CACHE_DIR}/cloudflared-latest-$(uname -m)"

# Download if not cached
if [ ! -x "$CLOUDFLARED_CACHED" ]; then
    arch=$(uname -m)
    case "$arch" in x86_64) arch="amd64";; aarch64) arch="arm64";; armv7l) arch="arm";; esac
    curl -fsSL "https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-${arch}" -o "$CLOUDFLARED_CACHED"
    chmod +x "$CLOUDFLARED_CACHED"
fi

"$CLOUDFLARED_CACHED" tunnel --url http://localhost:<PORT> 2>&1 | tee /tmp/cf-tunnel-url.log

# URL appears in stderr output:
# INF |  https://<random>.trycloudflare.com
```

## When to Use

- Exposing mitmweb, Jupyter, web UIs, API servers
- Any running HTTP service that needs a public URL temporarily
- Faster than Docker-based serving (no container startup)

## Pitfalls

- **Output goes to stderr**: Use `2>&1 | tee` to capture the URL. Cloudflared doesn't print it to stdout.
- **Binary cached at `.cloudflared-cache/`**: Same cache as the Docker skill. Only download once.
- **No auto-cleanup**: Unlike the Docker skill, there's no TTL. Kill manually when done.
- **Only tunnels HTTP**: The `--url` flag tunnels a local HTTP server. For raw TCP, use `--hostname` with a named tunnel (requires Cloudflare account).
