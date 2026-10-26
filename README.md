# Lets Discuss — local environment

Three repositories share this folder:

| Folder              | What it is                                      |
|---------------------|-------------------------------------------------|
| `./` (this repo)    | Docker Compose, PHP image, Nginx vhosts, docs    |
| `lets-discuss-api/` | Laravel 13 API (Sanctum, Reverb, PostgreSQL)     |
| `lets-discuss-web/` | Vite + React + TypeScript SPA (antd, Tailwind)   |

Requirements: `project_requirements_document.html`. Setup decisions: `docs/superpowers/specs/`.

## Start

```bash
docker compose up -d
```

## URLs

| URL                                          | Service                                    |
|----------------------------------------------|--------------------------------------------|
| http://lets-discuss.localhost                | React SPA (Vite dev server, HMR) — shows API + Reverb status |
| http://api.lets-discuss.localhost/api/health | API health                                 |
| http://api.lets-discuss.localhost/swagger    | Swagger UI (spec: `lets-discuss-api/resources/swagger/openapi.yaml`) |
| http://localhost:8081                        | Adminer: system PostgreSQL, server `postgres`, user `lets_discuss`, password `secret` |
| http://localhost:8025                        | Mailpit                                    |
| http://localhost:9001                        | MinIO console: `minio` / `minio-secret`, bucket `lets-discuss` |

Every host port is bound to `127.0.0.1` only, so nothing is reachable from your LAN. Postgres listens on `localhost:5432` and Redis on `localhost:6379` for desktop clients. Public media URLs are served from `http://localhost:9000/lets-discuss/...`.

`*.localhost` resolves to your machine in Chrome, Edge, Firefox and curl, so there's no need to edit a hosts file.

## Services

| Service     | Role |
|-------------|------|
| `api`       | php-fpm (PHP 8.4, image `lets-discuss/php:8.4` from `docker/php`) |
| `queue`     | `php artisan queue:work` (Redis) |
| `scheduler` | `php artisan schedule:work` |
| `reverb`    | `php artisan reverb:start`, proxied by Nginx at `ws://api.lets-discuss.localhost/app/{key}` |
| `web`       | Vite dev server; `node_modules` lives in the `web-node-modules` volume |
| `nginx`     | Both vhosts on :80 |
| `postgres`, `redis`, `minio` (+ `minio-init`), `mailpit`, `adminer` | Supporting services |
| `e2e`       | Playwright runner (profile `e2e`, run on demand) |

MinIO no longer publishes official images, so this setup uses `pgsty/minio` and `pgsty/mc`, which are maintained community builds of the same server and CLI.

## Everyday commands

In Git Bash, prefix commands with `MSYS_NO_PATHCONV=1` so container paths aren't rewritten.

```bash
docker compose exec api php artisan test                 # backend tests
docker compose exec api vendor/bin/pint --test           # PHP style
docker compose exec api vendor/bin/phpstan analyse --memory-limit=1G   # static analysis
docker compose exec web npm test                         # frontend unit tests (Vitest)
docker compose exec web npm run lint                     # oxlint
docker compose exec web npm run format:check             # Prettier
docker compose --profile e2e run --rm e2e                # Playwright E2E against the running stack
XDEBUG_MODE=debug docker compose up -d api               # step-debug PHP
```

Install packages inside the containers, not on the host:

```bash
docker compose exec api composer require vendor/package
docker compose exec web npm install package
```

After changing `lets-discuss-api/.env`, restart the long-running PHP processes so they pick it up: `docker compose restart api queue scheduler reverb`.

## First-time setup on a new machine

1. `docker compose build`
2. API env:
   ```bash
   cp lets-discuss-api/.env.example lets-discuss-api/.env
   docker compose run --rm --no-deps api composer install
   docker compose run --rm --no-deps api php artisan key:generate
   # Writes REVERB_APP_ID/KEY/SECRET. The log broadcaster lets the app boot before the keys exist.
   docker compose run --rm --no-deps -e BROADCAST_CONNECTION=log api php artisan reverb:install
   ```
3. Web env: `cp lets-discuss-web/.env.example lets-discuss-web/.env`, then set `VITE_REVERB_APP_KEY` to the API's `REVERB_APP_KEY`.
4. `docker compose up -d && docker compose exec api php artisan migrate`
