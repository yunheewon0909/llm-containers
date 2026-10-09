# Crawl4AI

Self-hosted page extraction alongside the existing SearXNG service.
Uses the official ARM64-compatible `unclecode/crawl4ai:0.9.4` image and the
existing `local-ai-network`. Plain Markdown extraction needs no LLM API key.

| Client | SearXNG | Crawl4AI |
| --- | --- | --- |
| macOS host | `http://127.0.0.1:8888` | `http://127.0.0.1:11235` |
| Container on `local-ai-network` | `http://searxng:8080` | `http://crawl4ai:11235` |

## Setup

Run from this directory. Create a token only if `.env` does not already exist:

```bash
test -f .env || (umask 077; printf 'CRAWL4AI_API_TOKEN=%s\n' "$(openssl rand -hex 32)" > .env)
chmod 600 .env
docker network inspect local-ai-network >/dev/null
docker compose config --quiet
docker compose pull
docker compose up -d --wait --wait-timeout 120
```

The shared network is created by the existing stack. If setting up a new
machine, create it once with `docker network create local-ai-network`.

Compose refuses to start with a missing or empty token. An exported shell
variable overrides `.env`; avoid an old `CRAWL4AI_API_TOKEN` in your shell.
The repository's existing `.gitignore` excludes the real `.env`.

## Check and fetch a page

`GET /health` does not require authentication. Other API requests use a Bearer
token. The commands below read the token without printing it:

```bash
curl --fail --silent --show-error http://127.0.0.1:11235/health

set -a
. ./.env
set +a

curl --fail --silent --show-error http://127.0.0.1:11235/crawl \
  -H "Authorization: Bearer $CRAWL4AI_API_TOKEN" \
  -H 'Content-Type: application/json' \
  --data '{"urls":["https://example.com"],"crawler_config":{"type":"CrawlerRunConfig","params":{"stream":false,"cache_mode":"bypass","check_robots_txt":true}}}'
```

For each item in `results`, check `success` and `status_code`, then use
`markdown.raw_markdown`. A successful HTTP response alone does not establish
that the page was fetched successfully. Retain the source `url` with the text.

The interactive UI is at `http://127.0.0.1:11235/playground`; set the same token
in its token bar. Do not print the token in logs or commit it.

## Search then fetch

SearXNG finds URLs; the caller selects URLs and sends them to Crawl4AI. Adding
this service alone does not change Open WebUI or Hermes's existing fetch tool.

```bash
curl --fail --silent --show-error --get http://127.0.0.1:8888/search \
  --data-urlencode 'q=Crawl4AI documentation' \
  --data-urlencode 'format=json'
```

Start with up to three distinct HTTP(S) URLs from SearXNG's `results[].url`.
Send them to `/crawl` using the request format above. Skip failed fetches and
pass successful Markdown, titles, and source URLs to the agent. Fetch pages
only when the snippets do not answer the question. Upstream sites can still
limit requests or deny access.

For an agent running directly on macOS, use the host URLs in the table. For
Open WebUI or another container, use the container URLs and provide the token
to that client. Client-specific tool integration is a separate step.

## Operations

```bash
docker compose ps
docker compose logs --tail=100 crawl4ai
docker compose stop
docker compose start --wait
```

To recreate only this service, use `docker compose up -d --wait`. After editing
the version in `compose.yaml`, use `docker compose pull` and
`docker compose up -d --wait` to update it. Never run these commands from
another service's directory by mistake.

The container has a 4 GiB memory limit, 512 PID limit, and 1 GiB private shared
memory, following the upstream deployment defaults. Runtime caches, Redis
jobs, and screenshot/PDF artifacts use temporary filesystems and are lost
when the container stops. Save any required results in the caller.

## References

- [Official self-hosting documentation](https://docs.crawl4ai.com/core/self-hosting/)
- [0.9.4 release](https://github.com/unclecode/crawl4ai/releases/tag/v0.9.4)
- [Upstream 0.9.4 Compose configuration](https://github.com/unclecode/crawl4ai/blob/v0.9.4/docker-compose.yml)

The self-hosting guide still includes older version examples; this setup pins
the verified 0.9.4 release instead of using `latest`.


## Verification (2026-10-09)

Compose validation, empty-token rejection, ARM64 execution, health, missing
and invalid token rejection (HTTP 401), page extraction, and shared-network
reachability all passed. A SearXNG GitHub-engine search returned 30 results;
fetching the official Crawl4AI repository returned 47,119 Markdown characters.

The default search engines returned no results during verification: Brave and
Google CSE were rate-limited; DuckDuckGo and Startpage reported CAPTCHA.
The GitHub engine was selected per request, without changing SearXNG settings:

```bash
curl --fail --silent --show-error --get http://127.0.0.1:8888/search \
  --data-urlencode 'q=crawl4ai' \
  --data-urlencode 'engines=github' \
  --data-urlencode 'format=json'
```

This engine is useful for repository searches. General web searches still
need a responsive general-purpose search engine. Open WebUI and Hermes tool
configuration has not yet been changed.
