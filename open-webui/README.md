# Open WebUI

## Requirements

- Docker Desktop
- LM Studio
- LM Studio API server running on port `1234`

## Start

```bash
docker compose up -d
```

Open:

```text
http://localhost:3000
```

## Connect to LM Studio

Add an OpenAI-compatible connection in Open WebUI:

```text
http://host.docker.internal:1234/v1
```

If LM Studio authentication is enabled, enter the API token in Open WebUI.

## Status

```bash
docker compose ps
```

## Logs

```bash
docker compose logs -f
```

## Restart

```bash
docker compose restart
```

## Stop

```bash
docker compose down
```

## Update

```bash
docker compose pull && docker compose up -d
```

## Persistent Data

Open WebUI data is stored in the Docker named volume:

```text
open-webui-data
```

Inspect:

```bash
docker volume inspect open-webui-data
```

## Reset

> This deletes all Open WebUI data.

```bash
docker compose down
docker volume rm open-webui-data
docker compose up -d
```

Do not use `docker compose down -v` for normal shutdowns or updates.