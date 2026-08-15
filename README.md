# cloudflared-file-server

Serve files over temporary Cloudflare Quick Tunnel URLs. No Cloudflare
account, no DNS, no domain — just a URL that expires.

Single `caddy:alpine` container: Caddy serves the files, cloudflared exposes
them at `https://xxx.trycloudflare.com`, and the container self-destructs
after a configurable TTL.

> This repository is also distributed as a Hermes agent skill. The full
> operational documentation lives in [SKILL.md](SKILL.md) — installation,
> usage, architecture, and pitfalls.

## Quick start

```sh
git clone https://github.com/bedasrv/cloudflared-file-server.git
sudo ln -s "$PWD/cloudflared-file-server/scripts/serve" /usr/local/bin/cloudflared-serve
```

```sh
cloudflared-serve 5m cat.png dog.png parrot.png
# https://xxx.trycloudflare.com/cat.png
# https://xxx.trycloudflare.com/dog.png
```

TTL is the first argument (`30s`, `5m`, `1h`, minimum 10s). Files follow.

## How it works

- Caddy `file-server` on :80 inside the container — nothing published to the
  host network.
- `cloudflared tunnel --url http://localhost:80` opens the Quick Tunnel.
- An async timer kills cloudflared after TTL; the container exits and the
  serve directory is cleaned up automatically.
- The cloudflared binary (~25MB) is cached host-side (24h) and mounted into
  the container, so only the first run downloads it. Architecture
  auto-detected (amd64/arm64/arm).
- Files are hard-linked into the serve directory — zero extra disk space.

## Limitations

- ~200 concurrent requests max (HTTP 429 beyond that)
- No SSE, no WebSocket support
- No SLA — dev/testing tool, not production
- Subject to Cloudflare Terms of Service

## License

MIT — see [LICENSE](LICENSE).
