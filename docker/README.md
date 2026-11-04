# HexStrike AI in Docker

A Kali Linux image with the HexStrike API server and almost every tool it integrates
(123 of the 127 checked by `/health`; `outguess` is no longer packaged by Kali).

## Quick start

```bash
cp .env.example .env        # set the host folders to mount (see below)
docker compose up -d --build
curl http://127.0.0.1:8888/health
```

The first build downloads several GB and takes a while; the image is about 7 GB.
It targets `linux/amd64`.

Then point your MCP client at the bridge as usual, for example:

```bash
python3 hexstrike_mcp.py --server http://127.0.0.1:8888
```

## What the compose file sets up

| Need | How |
|---|---|
| API reachable only from this machine | `ports: 127.0.0.1:8888:8888`. The server exposes `/api/command` without authentication, so do not publish it on all interfaces. |
| Reach services on the host's `localhost` | `extra_hosts: host.docker.internal:host-gateway`. Inside the container use `http://host.docker.internal:<port>` instead of `http://localhost:<port>`. |
| Read files from the host | `HEXSTRIKE_TARGETS` is mounted read-only at `/targets`. Give tools paths under `/targets/...`. |
| Keep reports and output | `HEXSTRIKE_WORKSPACE` is mounted read/write at `/workspace`. |
| Raw-socket scans (nmap `-sS`, masscan, tcpdump) | `cap_add: NET_RAW, NET_ADMIN`. The container is not privileged and does not use host networking. |
| Tool databases survive rebuilds | Named volumes for nuclei templates, trivy, wpscan, amass and Metasploit. |

`127.0.0.1` inside the container is the container itself, not the host.

## Image notes

- The API runs under gunicorn (`docker/config/entrypoint.sh`); tool databases (wpscan, trivy,
  nmap scripts, nuclei templates) are refreshed in the background at start-up
  (`docker/config/update-tools-databases.sh`).
- HexStrike's Python dependencies are installed in a Python 3.12 virtual environment:
  Kali's Python 3.14 cannot build the pinned `mitmproxy<11` dependency chain.
- `setuptools` is capped below 81 for pip installs because several pinned packages
  still import `pkg_resources`.
- `dalfox` is pinned to 2.x, which matches the CLI used by `hexstrike_server.py`.

## Common commands

```bash
docker compose logs -f          # server logs
docker compose down             # stop
docker compose up -d --build    # rebuild after changes
docker exec -it hexstrike bash  # shell in the container
```
