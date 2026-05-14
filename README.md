# cloudflared-file-server

Serve files via temporary Cloudflare Quick Tunnel URLs — no account, auto-expiry, pure Docker.

```bash
cloudflared-serve cat.png dog.png parrot.png 5m
# → https://xxx.trycloudflare.com/cat.png
# → https://xxx.trycloudflare.com/dog.png
# → https://xxx.trycloudflare.com/parrot.png
```

Files are served until TTL expires, then containers self-destruct. No cleanup needed.

## Why

Sharing files from a terminal is annoying. Discord caps at 25MB. Telegram caps at 2GB. Email attachments are slow. Cloud storage needs accounts.

This gives you a public HTTPS URL for any file in under 10 seconds. No signup. No API key. No Cloudflare account. The URL auto-expires.

## Install

### As a standalone CLI

```bash
git clone https://github.com/bedasrv/cloudflared-file-server.git
cd cloudflared-file-server
sudo cp scripts/serve /usr/local/bin/cloudflared-serve
sudo chmod +x /usr/local/bin/cloudflared-serve
```

Requires: Docker (`docker` command in PATH), outbound UDP on port 7844.

### As a Hermes Agent skill

```bash
# Direct install (use --force if scanner blocks community skills)
hermes skills install --force https://raw.githubusercontent.com/bedasrv/cloudflared-file-server/main/SKILL.md

# Or add repo as skill source
hermes skills tap add bedasrv/cloudflared-file-server
```

Then verify: `hermes skills list | grep cloudflared`

The skill loads automatically when Hermes needs to share a file. It teaches Hermes the Docker-based approach — no host-level cloudflared, no `nohup` hacks.

## Usage

```bash
# Single file (default 5-minute TTL)
cloudflared-serve image.png

# Directory
cloudflared-serve ./downloads/ 10m

# Multiple files, custom TTL
cloudflared-serve cat.png dog.png parrot.png 2m

# TTL formats: 30s, 5m, 1h
```

The script:
1. Copies files into a temp serve directory
2. Starts nginx container (serves files)
3. Starts cloudflared container (creates tunnel)
4. Starts timer container (auto-shutdown after TTL)
5. Prints the tunnel URL
6. Exits immediately — containers run in background

Files are available at `https://<tunnel>.trycloudflare.com/<filename>`.

## How It Works

```
Files → nginx:alpine (:80) → cloudflare/cloudflared → https://xxx.trycloudflare.com
         └── Docker bridge network ──┘         docker:cli (auto-kill timer)
```

Three containers, one network. The timer container mounts the Docker socket and kills siblings after TTL — fully Docker-native, no host-level process management.

## Manual Docker (no script)

```bash
SERVE_DIR=/tmp/cloudflare-serve
mkdir -p "$SERVE_DIR"
cp /path/to/files/* "$SERVE_DIR/"

docker network create cf-serve
docker run -d --name cf-nginx --network cf-serve -v "$SERVE_DIR:/usr/share/nginx/html:ro" nginx:alpine
docker run -d --name cf-tunnel --network cf-serve cloudflare/cloudflared tunnel --url http://cf-nginx:80
docker run -d --rm --name cf-timer -v /var/run/docker.sock:/var/run/docker.sock docker:cli \
  sh -c "sleep 300 && docker rm -f cf-tunnel cf-nginx && docker network rm cf-serve"

sleep 5
docker logs cf-tunnel 2>&1 | grep -oP 'https://[a-z0-9-]+\.trycloudflare\.com'
```

See `references/docker-setup.md` for detailed docs.

## Tear Down

Manual: `docker rm -f cf-tunnel cf-nginx cf-timer && docker network rm cf-serve`

The timer auto-kills everything after TTL. `docker:cli` container self-destructs with `--rm`.

## Limitations

- 200 concurrent requests (HTTP 429 beyond that)
- No SSE, no WebSocket
- No SLA — dev/testing only
- Cloudflare ToS applies
- Symlinks don't cross Docker volume boundaries (use `cp`, not `ln -s`)

## License

MIT
