# Local AI Containers

Docker-based local AI stack using LM Studio as the inference backend.

## Services

- Open WebUI
- Hermes Agent
- SearXNG

All persistent runtime data is stored in Docker named volumes.

## Architecture

```text
LM Studio (macOS)
       │
       │ host.docker.internal:1234
       │
Docker ├── Open WebUI
       ├── Hermes
       └── SearXNG
            │
            └── Internet

Shared Docker network:
local-ai-network
```

## Requirements

- macOS
- Docker Desktop
- LM Studio
- Tailscale (optional)

## Initial Setup

Create the shared Docker network once:

```bash
docker network create local-ai-network
```

Start SearXNG:

```bash
cd searxng
docker compose up -d
```

Start Open WebUI:

```bash
cd ../open-webui
docker compose up -d
```

Initial Hermes setup:

```bash
cd ../hermes
docker compose run --rm hermes setup
docker compose up -d
```

## LM Studio

Run the LM Studio API server on port `1234`.

Docker containers connect using:

```text
http://host.docker.internal:1234/v1
```

## SearXNG

Open WebUI:

```text
http://searxng:8080/search?q=<query>
```

Hermes:

```text
http://searxng:8080
```

## Update

Run inside each service directory:

```bash
docker compose pull && docker compose up -d
```

For SearXNG:

```bash
docker compose build --pull
docker compose up -d
```

## Network

Check connected containers:

```bash
docker network inspect local-ai-network
```

Expected containers:

```text
open-webui
hermes
searxng
```

## Remote Access with Tailscale

Services remain bound to `127.0.0.1`.

Open WebUI:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale serve --bg --https=443 localhost:3000
```

SearXNG Dashboard:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale serve --bg --https=8443 localhost:8888
```

Check:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale serve status
```

## Security

Do not commit API keys, tokens, credentials, runtime data, or Docker volume exports.