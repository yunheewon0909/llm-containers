# Hermes Agent

## Requirements

- Docker Desktop
- LM Studio
- LM Studio API server running on port `1234`

## Initial Setup

```bash
docker compose run --rm hermes setup
```

When using LM Studio, set the Base URL to:

```text
http://host.docker.internal:1234/v1
```

If LM Studio authentication is enabled, enter the API token during setup.

## Start

```bash
docker compose up -d
```

## Gateway

```text
http://localhost:8642
```

## CLI

```bash
docker exec -it hermes hermes
```

## Shared Docker Network

The following Docker network must exist:

```bash
docker network create local-ai-network
```

Hermes, Open WebUI, and SearXNG use:

```text
local-ai-network
```

## Web Search with SearXNG

Configure Hermes Web Search to use SearXNG.

URL:

```text
http://searxng:8080
```

Test connectivity:

```bash
docker exec hermes curl -s \
  "http://searxng:8080/search?q=test&format=json"
```

## Terminal Backend

When Hermes itself runs inside Docker, select:

```text
Local
```

This executes terminal commands inside the Hermes container.

Do not select the Docker terminal backend unless nested Docker sandbox execution is intentionally configured.

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

## Run Setup Again

```bash
docker compose run --rm hermes setup
```

## Update

```bash
docker compose pull && docker compose up -d
```

## Persistent Data

Hermes data is stored in the Docker named volume:

```text
hermes-data
```

Inspect:

```bash
docker volume inspect hermes-data
```

## Reset

> This deletes all Hermes configuration, credentials, sessions, memories, and other persistent data.

```bash
docker compose down
docker volume rm hermes-data
docker compose run --rm hermes setup
docker compose up -d
```

Do not use `docker compose down -v` for normal shutdowns or updates.