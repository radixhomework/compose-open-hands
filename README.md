# compose-open-hands

Self-hosted [OpenHands](https://github.com/OpenHands/openhands) (Agent Canvas) deployed with Docker Compose, using [GLM](https://open.bigmodel.cn/) (Zhipu AI) as the backend LLM.

## What this deploys

A single container (`ghcr.io/openhands/agent-canvas`) bundling the OpenHands web UI, the agent server, and the automation backend. The agent executes tasks inside its own sandboxed workspace, working on projects from a host-mounted folder, and talks to GLM through Zhipu's OpenAI-compatible API.

- Web UI: `http://<host>:8000/canvas`
- State (settings, LLM profiles, conversations): `./data/openhands` (host) → `/home/openhands/.openhands`
- Projects the agent can access: `./projects` (host) → `/projects`

## Prerequisites

- Docker Engine 24+ with the compose plugin (Docker Desktop on Windows/macOS)
- A GLM API key from [open.bigmodel.cn](https://open.bigmodel.cn/) (China) or [z.ai](https://z.ai/) (international)

## Quick start

```bash
cp .env.example .env
# Edit .env: set GLM_API_KEY (secrets are pre-generated if you use the provided one)
mkdir -p data/openhands projects
docker compose up -d
```

Then open `http://localhost:8000/canvas`.

## Configuration

Everything is driven by `.env` (see [.env.example](.env.example) for all options):

| Variable | Purpose |
|---|---|
| `GLM_API_KEY` | Your Zhipu/z.ai API key (**required**) |
| `GLM_BASE_URL` | GLM endpoint — must match your account type, see below |
| `GLM_MODEL` | Model name, e.g. `glm-4.6`, `glm-4.5-air` |
| `OPENHANDS_PORT` / `OPENHANDS_BIND` | Where the UI is exposed |
| `OPENHANDS_VERSION` | Image tag to pin |
| `PROJECTS_DIR` | Host folder the agent may read/write |
| `LOCAL_BACKEND_API_KEY` | API key protecting the server API |
| `OH_SECRET_KEY` | Encrypts stored settings/secrets at rest |

### GLM endpoint

The base URL depends on where your key was issued and which plan you use — the endpoints are not interchangeable:

| Account | Base URL |
|---|---|
| Pay-as-you-go, China ([open.bigmodel.cn](https://open.bigmodel.cn/)) | `https://open.bigmodel.cn/api/paas/v4` |
| GLM Coding plan, China | `https://open.bigmodel.cn/api/coding/paas/v4` |
| Pay-as-you-go, international ([z.ai](https://z.ai/)) | `https://api.z.ai/api/paas/v4` |
| GLM Coding plan, international | `https://api.z.ai/api/coding/paas/v4` |

Under the hood OpenHands resolves models through [LiteLLM](https://docs.litellm.ai/), so the compose file sets `LLM_MODEL=openai/<GLM_MODEL>` with `LLM_BASE_URL` pointing at GLM — the standard way to target an OpenAI-compatible API. These variables seed the bundled agent server; per-backend LLM settings can also be reviewed or changed later in the UI (**Settings → LLM**, base URL under **Advanced**).

### Letting the agent use Docker

By default the agent's sandbox has no working Docker daemon. If you want the agent to build containers and run `docker compose` itself, uncomment `privileged: true` in [docker-compose.yml](docker-compose.yml). This gives the container full kernel access — only do it on infrastructure you trust.

## Security notes

- The agent can execute arbitrary shell commands, read/write the mounted project folder, and reach the network. Mount only what you are willing to grant.
- `LOCAL_BACKEND_API_KEY` and `OH_SECRET_KEY` are pre-generated in `.env` (`openssl rand -base64 32`). Rotate them if `.env` ever leaks; note that rotating `OH_SECRET_KEY` invalidates stored secrets.
- Do not expose port 8000 raw to the internet. Put it behind a reverse proxy with TLS (and ideally IP allow-listing or VPN) and set `OPENHANDS_BIND` accordingly.

### Reverse proxy (Apache)

[apache/openhands.conf](apache/openhands.conf) fronts the stack with TLS, WebSocket tunneling and long agent-run timeouts. On the server:

```bash
sudo a2enmod proxy proxy_http proxy_wstunnel headers rewrite ssl
sudo cp apache/openhands.conf /etc/apache2/sites-available/openhands.conf
sudo a2ensite openhands && sudo systemctl reload apache2
```

Then in `.env` set `OH_WEB_URL=https://openhands.example.com` (your real hostname) and `OPENHANDS_BIND=127.0.0.1` so the backend is only reachable through Apache, and restart with `docker compose up -d`. Requires Apache ≥ 2.4.47 for transparent WebSocket proxying (`upgrade=websocket`).

Note that the agent's sandbox tool ports (VS Code server, workers) are opened by the container directly on the host with random ports — they bypass the proxy by default. On a trusted LAN, allow those ports through the firewall; for a fully-proxied setup, see the OpenHands Docker sandbox docs (`AGENT_SERVER_USE_HOST_NETWORK`, `OH_SANDBOX_CONTAINER_URL_PATTERN`).

## Operations

```bash
docker compose logs -f          # follow logs
docker compose pull && docker compose up -d   # upgrade (bump OPENHANDS_VERSION in .env)
docker compose down             # stop (state persists in ./data/openhands)
```

If you change `GLM_*` values in `.env`, restart with `docker compose up -d` to apply.

## Troubleshooting

- **Container exits immediately with a message about `LOCAL_BACKEND_API_KEY`/`GLM_API_KEY`** — a required variable is empty; fill it in `.env`.
- **LLM errors about authentication or 401** — wrong endpoint for your key type (see the table above) or an invalid key.
- **Agent can't see my code** — put the project under `PROJECTS_DIR` (default `./projects`); it appears at `/projects` in the sandbox.
- **Docker commands fail inside the agent** — enable `privileged: true` as described above.
