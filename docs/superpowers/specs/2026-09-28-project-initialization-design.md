# Lets Discuss — Project Initialization Design

Status: agreed with the product owner on 2026-09-28 (grilling session).
Source requirements: `project_requirements_document.html` (PRD v2.0).

## Scope of this step

Create the projects, the Docker environment and the required packages so the API
and the web app can be browsed and tested. **No domain code**: no models, roles,
policies, migrations or features. The only code is a health endpoint and a status
page that prove the stack is wired together.

## Repository layout

Three git repositories, `git init -b main` only. **No commits** (the owner reviews
and commits).

```
lets-discuss/                 ← infra repo; .gitignore excludes the two app folders
├── docker-compose.yml
├── docker/                   php/ (Dockerfile, ini), nginx/conf.d/
├── docs/superpowers/         specs + plans
├── project_requirements_document.html
├── lets-discuss-api/         ← Laravel 13 repo
└── lets-discuss-web/         ← Vite + React + TypeScript repo
```

## Runtime (everything in Docker, custom compose, not Sail)

| Service     | Image / build                        | Purpose                                  | Host access |
|-------------|--------------------------------------|------------------------------------------|-------------|
| `api`       | `docker/php` (PHP 8.4-fpm-alpine)     | php-fpm for Laravel                      | via nginx   |
| `queue`     | same image                           | `queue:work` (broadcasts are queued)     | —           |
| `scheduler` | same image                           | `schedule:work`                          | —           |
| `reverb`    | same image                           | `reverb:start` WebSocket server          | via nginx   |
| `web`       | `node:24-alpine`                     | Vite dev server with HMR                 | via nginx   |
| `nginx`     | `nginx` alpine                       | vhosts for web + api (+ Reverb `/app`)   | :80         |
| `postgres`  | `postgres:18-alpine`                 | primary database                         | :5432       |
| `redis`     | `redis:8-alpine`                     | queue, cache, sessions                   | :6379       |
| `minio`     | `minio/minio` (+ `minio-init`)       | S3-compatible media storage, console     | :9000, :9001|
| `mailpit`   | `axllent/mailpit`                    | mail catcher                             | :8025       |
| `adminer`   | `adminer`                            | Postgres GUI                             | :8081       |

PHP image extensions: pdo_pgsql, pgsql, redis, gd, intl, zip, bcmath, pcntl, opcache,
exif, xdebug (off unless `XDEBUG_MODE` is set).

Hostnames (`*.localhost` resolves to loopback in Chrome/Edge/Firefox and curl):
- `http://lets-discuss.localhost` → React SPA
- `http://api.lets-discuss.localhost` → Laravel (`/api`, `/sanctum`, `/broadcasting`, `/swagger`, Reverb `/app`)

## Backend (lets-discuss-api)

- Laravel 13, API-only usage (`install:api`), PostgreSQL, Redis (phpredis) for queue/cache/session.
- Auth: Sanctum **SPA cookie** authentication; stateful domain `lets-discuss.localhost`,
  session cookie domain `.lets-discuss.localhost`, credentialed CORS for the SPA origin only.
- Real-time: Laravel Reverb (`install:broadcasting --reverb`).
- Media: spatie/laravel-medialibrary on the `s3` disk backed by MinIO (league/flysystem-aws-s3-v3).
- Content format: Markdown, rendered with league/commonmark (safe mode, later).
- Captcha: Cloudflare Turnstile, verified via the Laravel HTTP client (config only now; test keys).
- API docs: wotzebra/laravel-swagger-ui serving a hand-written, spec-first `resources/swagger/openapi.yaml`.
- Quality: PHPUnit, Pint, Larastan level 6.
- Deferred decision: RBAC model (enum + policies vs spatie/laravel-permission).

## Frontend (lets-discuss-web)

- Vite + React + TypeScript, npm.
- UI: Ant Design v5 components + Tailwind v4 utilities for layout (antd CSS-in-JS in an `@layer`).
- Data/state: TanStack Query (server state), Zustand (UI/auth state), React Router, axios
  (`withCredentials`, `withXSRFToken`).
- Real-time: laravel-echo + pusher-js against Reverb.
- Content/captcha: react-markdown, @marsidev/react-turnstile.
- Tests: Vitest + React Testing Library, Playwright E2E.
- Quality: ESLint (Vite default) + Prettier (+ Tailwind plugin).

## Definition of done

1. `docker compose up -d` starts every service; all health checks pass.
2. `http://lets-discuss.localhost` shows API health (`ok`) and Reverb state (`Connected`).
3. `http://api.lets-discuss.localhost/api/health` returns `{"status":"ok",...}`; `/swagger` renders the spec.
4. Adminer (:8081), Mailpit (:8025) and MinIO console (:9001) open in the browser.
5. Backend tests, Pint, Larastan; frontend tests, lint, build; Playwright E2E all pass.
