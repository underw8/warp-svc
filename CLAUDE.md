# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

warp-svc packages the official Cloudflare WARP Linux client into a Docker image that exposes it as a SOCKS5 proxy on `0.0.0.0:1080`. The upstream `warp-svc`/`warp-cli` binaries only bind to localhost, so this repo's job is entirely about the plumbing around them: process supervision, port forwarding, registration, health checking, and log rotation. There is no application source code — everything is shell scripts, a Dockerfile, and supervisor/logrotate config.

## Architecture

Four processes run under supervisord (`configs/supervisord.conf`), each with `autorestart=true`:

- **warp-svc** (`scripts/warp.sh`) — starts the actual `warp-svc` daemon, then drives `warp-cli` to: kill any stale `warp-svc` process, register a new WARP account (retries up to 5 times), pin the internal proxy to port 40000, set proxy mode, disable DNS logging, apply `FAMILIES_MODE`, apply `WARP_LICENSE` if set, and connect. Once `warp-cli status` reports connected, it starts the `healthcheck` program via `supervisorctl` (healthcheck is `autostart=false` — it only runs once WARP is actually up).
- **socat** — forwards `tcp-listen:1080` (the container's public port) to `tcp:localhost:40000` (warp-cli's internal proxy port). This forwarding is the actual fix for the localhost-only binding limitation.
- **healthcheck** (`scripts/healthcheck.sh`) — every 60s, curls `https://www.cloudflare.com/cdn-cgi/trace/` through the local proxy on port 40000 and checks for `warp=on`. If the check fails, it runs `supervisorctl restart warp-svc` to force re-registration/reconnection.
- **logrotate** (`scripts/logrotate.sh`) — every 60s, runs `logrotate /etc/logrotate.conf` against `/var/lib/cloudflare-warp/*.txt` (`configs/logrotate.conf`: 10M size, 5 rotations, compressed, copytruncate).

All scripts trap `SIGTERM`/`SIGINT` for graceful shutdown, since supervisord forwards signals to child processes on container stop.

State (`/var/lib/cloudflare-warp`) is a Docker volume — it holds the WARP registration, so it must be persisted on the host or the container re-registers as a brand new device each time it restarts. Since a WARP+ license only supports 4 devices, losing this volume burns a device slot.

## Environment variables

- `WARP_LICENSE` — WARP+ license key, applied via `warp-cli registration license` if non-empty.
- `FAMILIES_MODE` — one of `off`, `malware`, `full`; applied via `warp-cli dns families`.

## Making changes

- Changes to startup/registration logic go in `scripts/warp.sh`; changes to process lifecycle/logging go in `configs/supervisord.conf`.
- There's no test suite or linter in this repo — validate changes by building the image and running it (see below).
- The image is multi-arch (`linux/amd64,linux/arm64`); avoid adding logic that only works on one architecture.

### Build and run locally

```bash
docker build -t warp-svc .
docker run -d --name=warp -p 127.0.0.1:1080:1080 -v "$(pwd)/warp:/var/lib/cloudflare-warp" warp-svc
```

Verify the proxy is working:

```bash
curl -x socks5h://127.0.0.1:1080 -sL https://cloudflare.com/cdn-cgi/trace | grep warp
# expect: warp=on
```

Check WARP connection status directly:

```bash
docker exec warp warp-cli --accept-tos status
```

### CI/release

`.github/workflows/build.yml` builds and pushes the multi-arch image to `ghcr.io/${{ github.repository }}` whenever a tag matching `v*` is pushed.
