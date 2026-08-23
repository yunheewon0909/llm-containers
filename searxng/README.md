# SearXNG

Private search backend for Open WebUI and Hermes.

## Requirements

- Docker Desktop
- `local-ai-network`

## Initial Setup

Create the shared network if it does not already exist:

```bash
docker network create local-ai-network
```

Build and start:

```bash
docker compose build --pull
docker compose up -d
```

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
docker compose build --pull
docker compose up -d
```

## Persistent Data

SearXNG runtime data is stored in the Docker named volume:

```text
searxng-cache
```

Inspect:

```bash
docker volume inspect searxng-cache
```

## Network and Local Access

SearXNG is published only on the host loopback interface:

```text
http://localhost:8888
```

It is available to containers on `local-ai-network` at:

```text
http://searxng:8080
```

Test from Open WebUI:

```bash
docker exec open-webui curl -s \
  "http://searxng:8080/search?q=test&format=json"
```

Test from Hermes:

```bash
docker exec hermes curl -s \
  "http://searxng:8080/search?q=test&format=json"
```

## Remote Access with Tailscale

Keep SearXNG bound to localhost and expose it privately to the tailnet with:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale \
  serve --bg --https=8443 localhost:8888
```

Check the assigned URL:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale serve status
```

For the complete port matrix, including Hermes Dashboard and VNC, see the
[repository Tailscale instructions](../README.md#remote-access-with-tailscale).

## Open WebUI

SearXNG Query URL:

```text
http://searxng:8080/search?q=<query>
```

## Hermes

SearXNG URL:

```text
http://searxng:8080
```

## Reset

> This deletes the SearXNG persistent cache.

```bash
docker compose down
docker volume rm searxng-cache
docker compose build --pull
docker compose up -d
```

Do not use `docker compose down -v` for normal shutdowns or updates.
