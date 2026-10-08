# Dify deployment

Docker Compose deployment of [Dify](https://dify.ai) for the HomeAgents home
server. The files `dify.yml`, `nginx/`, `sandbox/` and `ssrf_proxy/` come from
upstream Dify (Apache-2.0, tag 1.17.1) and keep their upstream licence. The local
changes are the pinned `langgenius/dify-*` image versions, the loopback binding of
every port the default profile publishes, and the trimmed comments. Everything
else in this repository is MIT, see `LICENSE`.

## What runs

Compose profiles decide which backing stores start, and `COMPOSE_PROFILES` in
`.env` derives them from two settings:

```
COMPOSE_PROFILES=${VECTOR_STORE:-weaviate},${DB_TYPE:-postgresql},collaboration
```

With the shipped defaults (`VECTOR_STORE=weaviate`, `DB_TYPE=postgresql`) the
active services are:

| Service | Image | Published |
|---|---|---|
| `nginx` | `nginx:latest` | `127.0.0.1:8085` maps to container port `80` |
| `api` | `langgenius/dify-api:1.17.1` | none |
| `worker` | `langgenius/dify-api:1.17.1` | none |
| `web` | `langgenius/dify-web:1.17.1` | none |
| `db_postgres` | `postgres:15-alpine` | none |
| `weaviate` | `cr.weaviate.io/semitechnologies/weaviate:1.39.2` | none |
| `redis` | `redis:6-alpine` | none |
| `sandbox` | `langgenius/dify-sandbox:0.2.15` | none |
| `plugin_daemon` | `langgenius/dify-plugin-daemon:0.6.10-local` | none |
| `ssrf_proxy` | `ubuntu/squid:latest` | none |

Every other store in the file (mysql, qdrant, milvus, opensearch and so on) sits
behind its own profile and stays off unless you change `VECTOR_STORE` or
`DB_TYPE`. Those optional services keep the port mappings upstream ships for them,
which are not loopback bound; only the default profile is.

Dify keeps its own private networks and is not attached to the platform's
`ai-platform-net`. To use the platform's LiteLLM gateway as a model provider, add
an OpenAI-API-compatible provider in Dify with base URL
`http://host.docker.internal:4000/v1` and the `LITELLM_MASTER_KEY` from the
`homeagents-core` `.env`. Then pick the model names declared in
`core/config/litellm/config.yaml`. `host.docker.internal` is a Docker Desktop
feature: on a plain Linux host, either add an `extra_hosts` entry pointing at
`host-gateway` for the `api` and `worker` services, or publish the gateway on the
host's bridge address instead of its loopback.

## Data

Persistent data lives in the core repository, so Dify stays reproducible and all
platform data has a single home:

```
../homeagents-core/VOLUMES/dify/
  app/storage/     uploaded files and generated media
  db/data/         Postgres
  redis/data/      Redis
  weaviate/        vector store
  plugin_daemon/   installed plugins
  sandbox/         sandbox dependencies
  certbot/         issued certificates (when the certbot profile is used)
```

Docker creates missing directories on first start, so a fresh clone needs no
preparation as long as `homeagents-core` is checked out next to this repository.

## Configuration

This stack is self-contained. Compose reads `.env` from this directory for both
variable interpolation and the containers' environment. Do not pass the platform
`.env` with `--env-file`, because it would override Dify's own defaults.

Start by copying the example file:

```powershell
Copy-Item .env.example .env
```

The file is named `.env.example` here because that follows upstream Dify. The
other HomeAgents repositories ship an `.env-example` file instead.

Do not skip this step. Compose reads `.env` by default and `COMPOSE_PROFILES`
lives in that file. Without it the profile-gated services are skipped and only
the profile-less ones start.

The values in `.env.example` are upstream Dify defaults. For example
`DB_PASSWORD=difyai123456`, `REDIS_PASSWORD=difyai123456`, `PLUGIN_DAEMON_KEY`
and `WEAVIATE_API_KEY`. They ship publicly with Dify itself, so there is no
private secret in the file, but change them before this instance becomes
reachable by anything other than your own machine.

The upstream `envs/*.env` fragments declared in `dify.yml` are marked
`required: false`, so they are simply skipped here. Every value comes from this
repository's `.env`.

The host port lives in `.env`:

```
EXPOSE_NGINX_PORT=8085    # published on 127.0.0.1 only
```

## Run

```powershell
docker compose -f dify.yml up -d     # from this directory
docker compose -f dify.yml down
```

Or through the CLI: `homeagents dify run` and `homeagents dify stop`.

The first start opens the installer at <http://localhost:8085/install>, where you
create the admin account.

## Upgrading

The `langgenius/dify-*` images should move together (api, worker, web, sandbox,
plugin daemon). Read the upstream release notes for environment variable changes
before restarting the stack, because Dify adds `PLUGIN_*` and store settings
between minor versions.

## TLS

`nginx/ssl/.gitkeep` keeps an empty directory in git. Certificates and keys are
git-ignored. Generate them per deployment. The compose file ships a `certbot`
service behind the `certbot` profile for that purpose
(`NGINX_HTTPS_ENABLED=false` in the shipped `.env.example`).

## Model traffic

Dify builds the LLM apps and workflows; the models live elsewhere. An
OpenAI-compatible provider points at the LiteLLM gateway, so this stack never
holds a provider key of its own. Weaviate runs next to it as Dify's private
vector store for retrieval features, and those features can also call remote
embedding or model plugins, which bill per call.
