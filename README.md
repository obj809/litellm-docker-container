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
#    - add at least one provider key (ANTHROPIC_API_KEY / OPENAI_API_KEY)

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
