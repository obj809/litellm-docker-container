# LiteLLM Docker Container

Run the [LiteLLM](https://docs.litellm.ai/) proxy locally with Docker Compose.
LiteLLM gives you a single OpenAI-compatible endpoint in front of many model
providers (Anthropic, OpenAI, Ollama, etc.), plus an admin UI, virtual API
keys, and request logging backed by Postgres.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) with Compose v2
- The external `webnet` network (see [Docker network](#docker-network))

## Quick start

```bash
# 1. Create your env file and fill in keys
cp .env.example .env
#    - set LITELLM_MASTER_KEY (e.g. `openssl rand -hex 24`, prefix with sk-)
#    - set LITELLM_SALT_KEY (`openssl rand -hex 32`) — set once, never change it
#    - add at least one provider API key (see .env.example)

# 2. Create the shared network (one time)
docker network create webnet

# 3. Start the proxy + database
docker compose up -d

# 4. Check it's healthy
curl http://localhost:4000/health/liveliness
```

The proxy listens on **http://localhost:4000**. The admin UI is at
**http://localhost:4000/ui** (log in with `UI_USERNAME` / `UI_PASSWORD`).

## Making a request

It speaks the OpenAI API, so point any OpenAI client at it:

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemini-flash",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

List available models:

```bash
curl http://localhost:4000/v1/models -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

## Managing virtual keys

Instead of handing out the master key, issue **virtual keys** — scoped `sk-...`
tokens with their own model allow-list, budget, rate limits, and expiry. They're
stored in Postgres and managed interchangeably from the UI or the API.

**From the UI:** open [http://localhost:4000/ui](http://localhost:4000/ui) →
**Virtual Keys** → **+ Create New Key**, set the limits, and copy the key (shown
once).

**From the API:** authenticate with the master key (only admin keys can mint
keys) and POST to `/key/generate`:

```bash
curl http://localhost:4000/key/generate \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "models": ["gemini-flash"],
    "max_budget": 10,
    "budget_duration": "30d",
    "rpm_limit": 60,
    "duration": "30d",
    "key_alias": "limited-demo-key"
  }'
```

The response contains a `key` field — use it like the master key, but scoped:
requests for a model outside `models`, or that exceed the budget, are rejected.

| Field                            | Effect                                      |
| -------------------------------- | ------------------------------------------- |
| `models`                         | Allowed `model_name` aliases (empty = all)  |
| `max_budget` + `budget_duration` | Spend cap and reset window (e.g. `30d`)     |
| `rpm_limit` / `tpm_limit`        | Requests / tokens per minute                |
| `duration`                       | Key lifetime / expiry                       |
| `key_alias`                      | Human-readable label (shown in UI and logs) |

See the [key management docs](https://docs.litellm.ai/docs/proxy/virtual_keys)
for the full set of options.

## Configuring models

Edit [`config.yaml`](./config.yaml) to add or change models, then restart:

```bash
docker compose restart litellm
```

Each `model_name` is the alias clients request; `litellm_params.model` is the
real provider model. See the
[provider docs](https://docs.litellm.ai/docs/providers) for supported models.

## Docker network

The `litellm` service joins an external Docker network named **`webnet`**, so it
can share a network with other containers (e.g. a reverse proxy or app) you run
outside this Compose project. Because it's declared `external: true`, Compose
won't create it for you — it must exist before `docker compose up`:

```bash
docker network create webnet              # create it (one time)
docker network ls                         # confirm it's listed
docker network inspect webnet             # see attached containers + subnet
docker network connect webnet <container> # attach another running container
docker network disconnect webnet <container>
docker network rm webnet                  # remove (after `docker compose down`)
```

The Postgres `db` service stays off `webnet` on purpose — only the proxy is
reachable from it. To attach another container at start time, use
`docker run --network webnet ...`.

## Common commands

```bash
docker compose logs -f litellm   # tail logs
docker compose down              # stop (keeps the database volume)
docker compose down -v           # stop and delete all data
docker compose pull              # update to the latest image
```

## Files

| File                      | Purpose                                              |
| ------------------------- | ---------------------------------------------------- |
| `docker-compose.yml`      | Local dev: LiteLLM proxy + Postgres, port published  |
| `docker-compose.prod.yml` | Production: no published port, env-driven DB password |
| `config.yaml`             | Model definitions and proxy settings                 |
| `.env.example`            | Template for secrets (copy to `.env`, not committed) |
