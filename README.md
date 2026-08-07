# Local AI Containers

Docker Compose configurations for running local AI services with **LM Studio** as the inference backend.

All application data is stored in Docker named volumes. No application data or credentials are bind-mounted to the host.

## Services

### Open WebUI

Web interface for local LLMs.

```text
open-webui/
├── compose.yaml
└── README.md
```

See [`open-webui/README.md`](./open-webui/README.md) for setup and usage.

### Hermes Agent

Local AI agent powered by LM Studio.

```text
hermes/
├── compose.yaml
└── README.md
```

See [`hermes/README.md`](./hermes/README.md) for setup and usage.

## Requirements

- macOS
- Docker Desktop
- LM Studio

LM Studio should run on the host with its API server enabled on port `1234`.

## Quick Start

Open WebUI:

```bash
cd open-webui
docker compose up -d
```

Hermes:

```bash
cd hermes
docker compose run --rm hermes setup
docker compose up -d
```

## Update

Run inside each service directory:

```bash
docker compose pull && docker compose up -d
```

## Security

Do not commit API keys, tokens, credentials, runtime data, or Docker volume exports to this repository.