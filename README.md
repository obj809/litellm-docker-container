# LiteLLM Docker Container

Run the [LiteLLM](https://docs.litellm.ai/) proxy locally with Docker Compose.
LiteLLM gives you a single OpenAI-compatible endpoint in front of many model
providers (Anthropic, OpenAI, Ollama, etc.), plus an admin UI, virtual API
keys, and request logging backed by Postgres.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) with Compose v2

## Quick start

```bash
# 1. Create your env file and fill in keys
cp .env.example .env
#    - set LITELLM_MASTER_KEY (e.g. `openssl rand -hex 24`, prefix with sk-)
#    - set LITELLM_SALT_KEY (`openssl rand -hex 32`) — set once, never change it
#    - add at least one provider API key (see .env.example)

# 2. Start the proxy + database
docker compose up -d

# 3. Check it's healthy
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

## Common commands

```bash
docker compose logs -f litellm   # tail logs
docker compose down              # stop (keeps the database volume)
docker compose down -v           # stop and delete all data
docker compose pull              # update to the latest image
```

## Files

| File                 | Purpose                                              |
| -------------------- | ---------------------------------------------------- |
| `docker-compose.yml` | LiteLLM proxy + Postgres services                    |
| `config.yaml`        | Model definitions and proxy settings                 |
| `.env.example`       | Template for secrets (copy to `.env`, not committed) |
