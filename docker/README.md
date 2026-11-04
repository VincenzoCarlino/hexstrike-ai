# HexStrike AI in Docker

This guide takes you from zero to an AI agent (Codex, Claude Code, or any MCP client) that runs
security tools against applications on your own machine. It works with Docker Desktop, OrbStack,
Rancher Desktop and Docker Engine on Linux, on macOS, Windows and Linux.

The image is Kali Linux with the HexStrike API server and almost every tool it integrates:
123 of the 127 checked by `/health` (`outguess` is no longer packaged by Kali).

## How it fits together

```
 your machine (the "host")
 ┌───────────────────────────────────────────────────────────────────────────────┐
 │                                                                               │
 │  Codex / Claude Code ──stdio──► docker exec -i hexstrike … hexstrike_mcp.py   │
 │                                          │                                    │
 │                         ┌────────────────┼──── container "hexstrike" ───────┐ │
 │                         │                ▼                                  │ │
 │                         │   HexStrike API :8888 ──► nmap, nuclei, ffuf, …   │ │
 │                         │                                 │                 │ │
 │                         │   /targets   (read-only)  ◄── HEXSTRIKE_TARGETS   │ │
 │                         │   /workspace (read/write) ◄── HEXSTRIKE_WORKSPACE │ │
 │                         └─────────────────────────────────┼─────────────────┘ │
 │                                                           │                   │
 │   your app on localhost:<port>  ◄── host.docker.internal:<port>               │
 └───────────────────────────────────────────────────────────────────────────────┘
```

- The **MCP bridge** (`hexstrike_mcp.py`) is what your AI client talks to. It runs inside the
  container, started by `docker exec`, so you do not need Python on the host.
- The **API server** runs the tools. It is published on `127.0.0.1:8888` only.
- **Inside the container, `localhost` is the container itself.** Your app running on the host's
  `localhost:3000` is `host.docker.internal:3000` from the tools' point of view.
- **`/targets`** is a host folder the tools can read (source code, binaries, captures), and
  **`/workspace`** is a host folder where they can write output.

## 1. Prerequisites

- A Docker runtime with Docker Compose v2 (`docker compose version` must work):

  | Runtime | Notes |
  |---|---|
  | Docker Desktop (macOS, Windows) | Nothing special. |
  | OrbStack (macOS) | Nothing special. |
  | Rancher Desktop (macOS, Windows) | In *Preferences → Container Engine* select **dockerd (moby)**, not containerd. Step 3 needs one extra setting. |
  | Docker Engine (Linux) | See the Linux note in step 3. |

- About 30 GB of free disk space. The image is about 7 GB; the build needs more while it runs.
- `git`.
- **Apple Silicon:** the image is `linux/amd64` (several tools ship only x86_64 binaries) and runs
  under emulation. Enable *Use Rosetta for x86_64/amd64 emulation* in Docker Desktop or Rancher
  Desktop (OrbStack uses Rosetta by default). Expect the build and some tools to be slower.

## 2. Get the code

```bash
git clone https://github.com/0x4m4/hexstrike-ai.git
cd hexstrike-ai
```

Line endings are handled by `.gitattributes`, so this works on Windows too.

## 3. Configure `.env`

Copy the example and edit it:

```bash
cp .env.example .env            # Windows PowerShell: Copy-Item .env.example .env
```

| Variable | What it is | Example |
|---|---|---|
| `HEXSTRIKE_TARGETS` | Host folder mounted **read-only** at `/targets`. Put the code you want to scan here. Mount a parent folder to give access to several repositories (`/targets/repo-a`, `/targets/repo-b`). | macOS/Linux: `/Users/me/Projects`<br>Windows: `C:/Users/me/Projects` |
| `HEXSTRIKE_WORKSPACE` | Host folder mounted **read/write** at `/workspace`, for reports and raw tool output. One folder per engagement keeps things tidy. | `C:/Users/me/hexstrike/workspace/my-app-audit` |
| `HEXSTRIKE_HOST_IP` | The address `host.docker.internal` resolves to inside the container. Leave it unset unless the table below says otherwise. | `192.168.127.254` |
| `TZ` | Time zone for timestamps in the container. | `Europe/Rome` |

On Windows, write paths with forward slashes (`C:/Users/...`). Create the workspace folder before
starting the container.

### `HEXSTRIKE_HOST_IP` per runtime

| Runtime | `HEXSTRIKE_HOST_IP` |
|---|---|
| Docker Desktop, OrbStack | Leave unset. `host.docker.internal` reaches services listening on the host's `127.0.0.1`. |
| Rancher Desktop | Required. Run `rdctl shell grep internal /etc/hosts` and use the address on the `host.docker.internal` (or `host.rancher-desktop.internal` / `host.lima.internal`) line. On Windows with the networking tunnel it is usually `192.168.127.254`. Without it, `host.docker.internal` points to the Docker bridge inside Rancher's VM and nothing on your machine is reachable. |
| Docker Engine (Linux) | Leave unset. `host.docker.internal` is the `docker0` bridge address (`172.17.0.1`), so the target app must listen on `0.0.0.0` or on that address: services bound only to `127.0.0.1` are **not** reachable. |

## 4. Build and start

```bash
docker compose up -d --build
```

The first build downloads several GB and takes a long time (an hour or more under emulation).
Nothing is pushed anywhere: the image stays local as `hexstrike-ai:local`.

Check that the server is up (wait a minute after the first start):

```bash
curl http://127.0.0.1:8888/health
```

Look for `"status": "healthy"` and `"total_tools_available": 123`. On Windows PowerShell:
`Invoke-RestMethod http://127.0.0.1:8888/health`.

After changing `.env`, run `docker compose up -d` again to recreate the container.

## 5. Check the container can see your app and your files

Replace `3000` with your app's port, and use `https://` if the app serves TLS:

```bash
docker exec hexstrike curl -sk -o /dev/null -w "%{http_code}\n" http://host.docker.internal:3000/
```

| Result | Meaning |
|---|---|
| `2xx` / `3xx` | Reachable. |
| `400` | The app rejects the `host.docker.internal` host name (IIS Express, ASP.NET Core host filtering, Django `ALLOWED_HOSTS`...). Retry with `-H "Host: localhost:3000"`; if that works, tell your agent to send that header on every call (the example prompt does). |
| `000` | Not reachable. Check the app is running, the scheme (`http` vs `https`), `HEXSTRIKE_HOST_IP` (Rancher Desktop), or the bind address (Linux). |

And the mounts:

```bash
docker exec hexstrike ls /targets
docker exec hexstrike touch /workspace/.ok
```

The listing must show your folder's content, and the `touch` must succeed (delete `.ok` afterwards).

## 6. Connect your AI client

Every client starts the MCP bridge inside the running container with this command:

```
docker exec -i hexstrike /opt/hexstrike/venv/bin/python /opt/hexstrike/hexstrike_mcp.py --server http://127.0.0.1:8888 --timeout 600
```

`--timeout 600` lets a single tool run for up to 10 minutes (the bridge default is 5). Keep the
client's own tool timeout above it. The container must be running before the client starts.

### Codex (CLI, IDE extension, ChatGPT desktop app)

Add this to `~/.codex/config.toml` (Windows: `%USERPROFILE%\.codex\config.toml`):

```toml
[mcp_servers.hexstrike]
command = "docker"
args = ["exec", "-i", "hexstrike", "/opt/hexstrike/venv/bin/python", "/opt/hexstrike/hexstrike_mcp.py", "--server", "http://127.0.0.1:8888", "--timeout", "600"]
startup_timeout_sec = 30
tool_timeout_sec = 900
```

Restart Codex (in the ChatGPT desktop app, quit it completely and reopen it). MCP servers are
available in local mode, not in cloud tasks. In the CLI, `/mcp` lists the connected servers.

### Claude Code

```bash
claude mcp add --scope user hexstrike -- docker exec -i hexstrike /opt/hexstrike/venv/bin/python /opt/hexstrike/hexstrike_mcp.py --server http://127.0.0.1:8888 --timeout 600
```

`--scope user` makes it available in every project; drop it to add it to the current project only.
Raise the tool timeout by adding this to `~/.claude/settings.json`:

```json
{ "env": { "MCP_TOOL_TIMEOUT": "900000" } }
```

Check with `claude mcp list`, or `/mcp` inside a session.

### Other MCP clients (Claude Desktop, Cursor, VS Code...)

Most clients take a JSON entry like this:

```json
{
  "mcpServers": {
    "hexstrike": {
      "command": "docker",
      "args": ["exec", "-i", "hexstrike", "/opt/hexstrike/venv/bin/python", "/opt/hexstrike/hexstrike_mcp.py", "--server", "http://127.0.0.1:8888", "--timeout", "600"]
    }
  }
}
```

### Alternative: run the bridge on the host

If you prefer, run the bridge from a local Python environment instead of `docker exec`: create a
virtual environment, `pip install requests mcp`, and point the client at
`python hexstrike_mcp.py --server http://127.0.0.1:8888`. On Windows add the environment variable
`PYTHONUTF8=1`, or the bridge's log output fails with `UnicodeEncodeError`.

## 7. Prompt your agent

The agent only knows about the container layout, `host.docker.internal` and the `Host` header if
you tell it. [`example-agent-prompt.md`](example-agent-prompt.md) is a template for a local
security assessment: fill in the placeholders and paste it into your client. It includes a
pre-flight check the agent runs before scanning.

## Security notes

- The API's `/api/command` endpoint runs any command in the container **without authentication**.
  That is why it is published on `127.0.0.1` only: never change the port mapping to `8888:8888`.
  Any process on your machine can still call it.
- `host.docker.internal` reaches **every** port on your machine. The container cannot tell your app
  from your other local services: keep the agent in scope with the prompt.
- `/targets` is read-only, so tools cannot modify your code. `/workspace` may end up holding
  unredacted tool output (tokens, personal data): treat it accordingly.
- The container is not privileged and does not use host networking. It has `NET_RAW` and
  `NET_ADMIN` so nmap SYN scans, masscan and tcpdump work.
- Only scan systems you own or are authorized to test.

## Troubleshooting

| Problem | Fix |
|---|---|
| `timed out dialing Hyper-V socket` on the first `docker compose up` (Rancher Desktop, Windows) | Run `docker compose up -d` again. |
| The build seems stuck on "unpacking" or "exporting" | It is writing a 7 GB image; give it time. |
| `curl` from the container returns `000` | See step 5. On Rancher Desktop, check `HEXSTRIKE_HOST_IP`. |
| `curl` from the container returns `400` | Pass `-H "Host: localhost:<port>"` (step 5). |
| `ls /targets` is empty | `HEXSTRIKE_TARGETS` is wrong or the path doesn't exist. On Windows use forward slashes. Then `docker compose up -d`. |
| The client shows no hexstrike tools | The container must be running (`docker ps`). Run the `docker exec -i …` command from step 6 by hand: it should wait for input without errors (Ctrl+C to quit). Restart the client after editing its config. |
| `/opt/hexstrike/hexstrike_mcp.py: No such file` | Your image predates the bridge being included: `docker compose up -d --build`. |
| Long scans fail after 5 minutes | Raise `--timeout` in the bridge command and the client's tool timeout (step 6). |
| Re-running a scan returns the old result | HexStrike caches identical commands for one hour: use `execute_command` with `use_cache=false`, or call `clear_cache`. |

## Image notes

- The API runs under gunicorn (`docker/config/entrypoint.sh`); tool databases (wpscan, trivy,
  nmap scripts, nuclei templates) are refreshed in the background at start-up
  (`docker/config/update-tools-databases.sh`).
- Named volumes keep the nuclei templates and the trivy, wpscan, amass and Metasploit data across
  rebuilds.
- HexStrike's Python dependencies are installed in a Python 3.12 virtual environment
  (`/opt/hexstrike/venv`): Kali's Python 3.14 cannot build the pinned `mitmproxy<11` dependency chain.
- `setuptools` is capped below 81 for pip installs because several pinned packages
  still import `pkg_resources`.
- `dalfox` is pinned to 2.x, which matches the CLI used by `hexstrike_server.py`.
- Wordlists: `/usr/share/wordlists` and SecLists at `/usr/share/seclists`.

## Common commands

```bash
docker compose logs -f          # server logs
docker compose down             # stop
docker compose up -d            # start, or apply .env changes
docker compose up -d --build    # rebuild after pulling changes
docker exec -it hexstrike bash  # shell in the container
```
