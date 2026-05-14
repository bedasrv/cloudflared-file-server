# Docker Setup Reference

The canonical approach. One nginx container serves files, one cloudflared container exposes them, one docker:cli container handles auto-shutdown. No host-level process juggling, no lock files, no DNS interference.

## Architecture

```
Files on host                  Docker bridge network (cf-serve)
     │                              │
     ▼                              ▼
nginx:alpine (:80)  ←────────  cloudflare/cloudflared  ──→  https://xxx.trycloudflare.com
     │                              │
     └── shared volume mount ───────┘         docker:cli (auto-kill timer, --rm)
```

## Quick Start

```bash
# One-time: pull images
docker pull nginx:alpine cloudflare/cloudflared:latest docker:cli

# Serve files
SERVE_DIR=/tmp/cloudflare-serve
mkdir -p "$SERVE_DIR"
# Hard links — zero extra disk space. Falls back to cp if cross-device.
ln /path/to/files/* "$SERVE_DIR/" 2>/dev/null || cp /path/to/files/* "$SERVE_DIR/"

docker network create cf-serve
docker run -d --name cf-nginx --network cf-serve -v "$SERVE_DIR:/usr/share/nginx/html:ro" nginx:alpine
docker run -d --name cf-tunnel --network cf-serve cloudflare/cloudflared tunnel --url http://cf-nginx:80
docker run -d --rm --name cf-timer -v /var/run/docker.sock:/var/run/docker.sock docker:cli \
  sh -c "sleep 300 && docker rm -f cf-tunnel cf-nginx && docker network rm cf-serve"

# Get URL
sleep 5
docker logs cf-tunnel 2>&1 | grep -oP 'https://[-a-z0-9]+\.trycloudflare\.com'
```

## Adding Files While Running

```bash
ln newfile.png /tmp/cloudflare-serve/ 2>/dev/null || cp newfile.png /tmp/cloudflare-serve/
# Immediately available at https://xxx.trycloudflare.com/newfile.png
```

No restart needed. nginx serves from the mounted volume live.

## Multi-Tunnel (rarely needed)

If you genuinely need separate tunnel URLs for different file sets:

```bash
# Second serve directory
mkdir -p /tmp/other-files
cp files/* /tmp/other-files/

# Second nginx
docker run -d --name cf-nginx2 --network cf-serve \
  -v /tmp/other-files:/usr/share/nginx/html:ro \
  nginx:alpine

# Second tunnel (different container name, same network)
docker run -d --name cf-tunnel2 --network cf-serve \
  cloudflare/cloudflared tunnel --url http://cf-nginx2:80

# Second timer
docker run -d --rm --name cf-timer2 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  docker:cli \
  sh -c "sleep 300 && docker rm -f cf-tunnel2 cf-nginx2"
```

Each stack is fully isolated — no conflicts, no shared state.

## Tear Down

```bash
# Kill timer first (--rm auto-removes it, so just rm the siblings)
docker rm -f cf-timer cf-tunnel cf-nginx 2>/dev/null
docker network rm cf-serve 2>/dev/null
# Files in $SERVE_DIR unaffected (host volume, not container data)
```

## Why Docker

The host-level approach (Python http.server + cloudflared binary) has multiple failure modes:

| Problem | Docker fix |
|---------|-----------|
| Cloudflare enforces one tunnel per process | Each tunnel is a container — unlimited concurrent tunnels |
| SIGHUP kills background processes | Containers are process-group isolated |
| Lock file conflicts between instances | Container names, not lock files |
| dnscrypt-proxy blocks trycloudflare.com DNS | Container uses its own DNS |
| Orphaned processes after script exit | `docker rm -f` is surgical |
| Port collisions (random 30000-60000) | Internal Docker network, no host ports exposed |
