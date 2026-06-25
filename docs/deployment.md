# Deployment

Images are built locally and pushed to the server. The server runs containers via Docker Compose — no source code or git repo is needed on the server.

## Server

| Parameter | Value |
|-----------|-------|
| Host | `185.138.186.243` |
| Port | `2222` |
| User | `user` |
| Key | `~/.ssh/compmaster_host` |
| Compose dir | `/srv/mind/` |

Connect: `ssh -i ~/.ssh/compmaster_host -p 2222 user@185.138.186.243`

## Dev environment

| Service | URL | Internal port |
|---------|-----|---------------|
| mind_api | https://dev-api.mind-awake.life | 3001 |
| mind_web | https://dev.mind-awake.life | 8001 |
| PostgreSQL | — | 5434 (host) |
| gRPC | grpc-dev.mind-awake.life | 50052 |

nginx on the server proxies all HTTPS traffic; containers are not exposed directly.

## Deploy script

`deploy-dev.sh` at the repo root handles the full cycle for a given service:

```bash
# Deploy API
./deploy-dev.sh mind_api

# Deploy web
./deploy-dev.sh mind_web
```

What it does per run:
1. Builds the Docker image locally (`docker compose build`)
2. Streams the image to the server via `docker save | gzip | docker load` — no intermediate file
3. Copies `docker-compose.dev.yml` and env files to `/srv/mind/`
4. Restarts the service on the server with `docker compose up -d`

## Prerequisites

- Docker Desktop running locally
- SSH key `~/.ssh/compmaster_host` with access to the server
- `.env.dev` at repo root (`VITE_API_BASE_URL`)
- `mind_api/.env.dev` with all API + DB configuration

## Env files

| File | Used by | Scope |
|------|---------|-------|
| `.env.dev` | root compose (build args) | `VITE_API_BASE_URL` for web build |
| `mind_api/.env.dev` | postgres + mind_api containers | DB credentials, JWT, ports, gRPC, CORS |

`mind_api/.env.dev` is the single source of truth for the API and database config. The root compose reads it directly via `env_file: ./mind_api/.env.dev`.

## observe-js

Both projects reference `observe-js` via git tag:
```
"observe-js": "git+https://github.com/mind-systems/observe-js.git#v0.2.0"
```

`git` is installed in both Dockerfiles so npm can clone it during `npm ci`.

## Ports in use (host)

| Port | Service |
|------|---------|
| 3001 | mind_api dev |
| 5434 | PostgreSQL dev |
| 50052 | gRPC dev |
| 8001 | mind_web dev |
