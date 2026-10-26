# Lets Discuss — Project Initialization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up `lets-discuss-api` (Laravel 13) and `lets-discuss-web` (Vite + React + TS) inside a root Docker Compose environment, with every agreed package installed, so both can be browsed and tested end to end.

**Architecture:** A root infra repo owns `docker-compose.yml`, the PHP image and Nginx vhosts. One PHP 8.4 image runs four containers (php-fpm, queue, scheduler, Reverb); a Node 24 container runs the Vite dev server; Nginx publishes `lets-discuss.localhost` (SPA) and `api.lets-discuss.localhost` (Laravel + Reverb WebSocket) on port 80. Postgres, Redis, MinIO, Mailpit and Adminer are supporting services. The only application code is `GET /api/health` and an SPA status page that proves API + WebSocket wiring.

**Tech Stack:** Docker Compose, PHP 8.4-fpm-alpine, Laravel 13, Sanctum, Reverb, PostgreSQL 18, Redis 8, MinIO, Mailpit, Adminer, Nginx, Node 24, Vite, React, TypeScript, Ant Design 5, Tailwind CSS 4, TanStack Query, Zustand, React Router, axios, laravel-echo, Vitest, React Testing Library, Playwright, PHPUnit, Pint, Larastan.

**Spec:** `docs/superpowers/specs/2026-09-28-project-initialization-design.md`

## Global Constraints

- **Shell:** run every command in Git Bash from the root `lets-discuss/` folder unless the step says otherwise, with `MSYS_NO_PATHCONV=1` exported in the same invocation (Git Bash otherwise rewrites container paths such as `/var/www/html` into Windows paths).
- **No host PHP/Composer/npm for project work:** all `composer`, `php`, `artisan` and `npm` commands run inside containers (`docker compose exec|run ...`). Host tools used: `docker`, `git`, `curl`.
- **Git:** `git init -b main` in the root, `lets-discuss-api/` and `lets-discuss-web/`. **Do not commit, do not add remotes.** The owner commits.
- **Versions:** PHP 8.4, Laravel `^13.0`, PostgreSQL 18, Redis 8, Node 24, antd `^5`, Tailwind 4.
- **Hostnames:** SPA `http://lets-discuss.localhost`, API `http://api.lets-discuss.localhost`. Host ports: 80 (nginx), 5432, 6379, 8025 (Mailpit), 8081 (Adminer), 9000/9001 (MinIO API/console).
- **Scope:** no models, roles, policies, migrations or features beyond `/api/health` and the status page. Domain packages are installed and configured only.
- **Dev credentials** (local only): Postgres `lets_discuss` / `lets_discuss` / `secret`; MinIO `minio` / `minio-secret`, bucket `lets-discuss`; Turnstile test keys `1x00000000000000000000AA` (site) and `1x0000000000000000000000000000000AA` (secret).

## Review Focus

1. **Cookie scope across subdomains** — the SPA on `lets-discuss.localhost` must be able to read `XSRF-TOKEN` issued by `api.lets-discuss.localhost`; the cookie must carry domain `.lets-discuss.localhost`. Pinned by `SpaAuthenticationTest::test_csrf_cookie_is_issued_for_the_shared_parent_domain` (Task 4).
2. **Credentialed CORS** — the SPA origin gets `Access-Control-Allow-Origin: http://lets-discuss.localhost` + `Allow-Credentials: true`; any other origin gets no CORS grant. Pinned by the two CORS tests in Task 4.
3. **A dependency is down** — if Postgres or Redis is unreachable, `/api/health` answers `503` naming the failed service instead of a `500`; if the API itself is unreachable the SPA shows `unreachable` instead of hanging or crashing. Pinned by `HealthCheckTest` (Task 3) and `ApiHealthCard.test.tsx` (Task 7).
4. **WebSocket unavailable** — when Reverb cannot be reached the status page shows `Unavailable`/`Disconnected` and follows later transitions back to `Connected`. Pinned by `ReverbStatusCard.test.tsx` (Task 7).
5. **Windows bind-mount file watching** — editing a file on the Windows host must trigger Vite HMR inside the container (requires polling). Pinned by the HMR verification step in Task 6.

---

## File Structure

```
lets-discuss/
├── .gitignore                                  # ignores the two app folders + .env
├── README.md                                   # URLs, commands (Task 8)
├── docker-compose.yml                          # grows task by task
├── docker/
│   ├── php/Dockerfile                          # PHP 8.4 image shared by 4 services
│   ├── php/conf.d/lets-discuss.ini             # upload/memory limits
│   ├── php/conf.d/xdebug.ini                   # xdebug client settings (mode via env)
│   └── nginx/conf.d/
│       ├── 00-upgrade-map.conf                 # WebSocket Connection header map
│       ├── api.conf                            # api.lets-discuss.localhost → php-fpm + reverb
│       └── web.conf                            # lets-discuss.localhost → vite
├── lets-discuss-api/                           # Laravel 13 (own repo)
│   ├── app/Http/Controllers/HealthController.php
│   ├── routes/api.php                          # GET /api/health
│   ├── bootstrap/app.php                       # statefulApi()
│   ├── config/cors.php, config/reverb.php, config/services.php, config/swagger-ui.php, config/media-library.php
│   ├── resources/swagger/openapi.yaml
│   ├── phpstan.neon
│   └── tests/Feature/{HealthCheckTest,SpaAuthenticationTest,SwaggerUiTest}.php
└── lets-discuss-web/                           # Vite React TS (own repo)
    ├── .env, .env.example, .prettierrc, .prettierignore
    ├── vite.config.ts                          # dev server, tailwind, vitest
    ├── playwright.config.ts, e2e/status.spec.ts
    └── src/
        ├── main.tsx, App.tsx, index.css, env.d.ts
        ├── lib/http.ts                         # axios instance for Sanctum
        ├── lib/echo.ts                         # lazy Echo/Reverb singleton
        ├── test/setup.ts
        └── features/status/
            ├── api.ts                          # HealthResponse, fetchHealth, useApiHealth
            ├── ApiHealthCard.tsx (+ .test.tsx)
            ├── useConnectionState.ts           # ConnectionLike, useConnectionState
            ├── ReverbStatusCard.tsx (+ .test.tsx)
            └── StatusPage.tsx
```

---

### Task 1: Root repo and supporting services

**Files:**
- Create: `.gitignore`, `docker-compose.yml`

**Interfaces:**
- Produces: compose services `postgres`, `redis`, `minio`, `minio-init`, `mailpit`, `adminer`; named volumes `postgres-data`, `redis-data`, `minio-data`. Later tasks append services to the same `docker-compose.yml`.

- [ ] **Step 1: Initialise the root repo**

```bash
git init -b main
```

- [ ] **Step 2: Create `.gitignore`**

```gitignore
# The two applications are separate repositories
/lets-discuss-api/
/lets-discuss-web/

.env
```

- [ ] **Step 3: Create `docker-compose.yml` with the supporting services**

```yaml
name: lets-discuss

services:
  postgres:
    image: postgres:18-alpine
    environment:
      POSTGRES_DB: lets_discuss
      POSTGRES_USER: lets_discuss
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      # Postgres 18 images keep PGDATA under /var/lib/postgresql/18/docker
      - postgres-data:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U lets_discuss -d lets_discuss"]
      interval: 5s
      timeout: 5s
      retries: 10
    restart: unless-stopped

  redis:
    image: redis:8-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 10
    restart: unless-stopped

  minio:
    image: minio/minio:RELEASE.2025-04-22T22-12-26Z
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: minio-secret
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio-data:/data
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 5s
      timeout: 5s
      retries: 10
    restart: unless-stopped

  minio-init:
    image: minio/mc:latest
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      /bin/sh -c "
      mc alias set local http://minio:9000 minio minio-secret &&
      mc mb --ignore-existing local/lets-discuss &&
      mc anonymous set download local/lets-discuss
      "
    restart: "no"

  mailpit:
    image: axllent/mailpit:latest
    ports:
      - "8025:8025"
    restart: unless-stopped

  adminer:
    image: adminer:latest
    environment:
      ADMINER_DEFAULT_SERVER: postgres
    ports:
      - "8081:8080"
    depends_on:
      - postgres
    restart: unless-stopped

volumes:
  postgres-data:
  redis-data:
  minio-data:
```

Note: the MinIO tag is pinned to a release that still ships the full web console (MinIO stopped publishing community images later in 2025). If `docker pull` reports the tag as missing, switch to `minio/minio:latest` and record that in the task report.

- [ ] **Step 4: Start the services**

Run: `docker compose up -d postgres redis minio minio-init mailpit adminer`
Expected: all images pull; `docker compose ps` shows `postgres`, `redis`, `minio` as `healthy`, and `mailpit` and `adminer` as `running`.

- [ ] **Step 5: Verify each service**

```bash
docker compose exec postgres psql -U lets_discuss -d lets_discuss -tAc "select version();"
docker compose exec redis redis-cli ping
docker compose logs minio-init
for p in 8025 8081 9001; do printf "%s " $p; curl -s -o /dev/null -w "%{http_code}\n" http://localhost:$p; done
```
Expected: `PostgreSQL 18.x ...`; `PONG`; minio-init log contains `Bucket created successfully` (or `already exists`) and `Access permission ... set to download`; each port prints `200`.

- [ ] **Step 6: Review (no commit)**

Run: `git status --short`
Expected: `.gitignore`, `docker-compose.yml` and `docs/` are untracked; `project_requirements_document.html` and `user_stories.html` also appear. Leave everything uncommitted.

---

### Task 2: PHP image and Laravel 13 scaffold

**Files:**
- Create: `docker/php/Dockerfile`, `docker/php/conf.d/lets-discuss.ini`, `docker/php/conf.d/xdebug.ini`
- Modify: `docker-compose.yml` (add `x-php` anchor and `api` service)
- Create (generated): `lets-discuss-api/**` via `composer create-project`
- Modify: `lets-discuss-api/.env`, `lets-discuss-api/.env.example`, `lets-discuss-api/phpunit.xml`

**Interfaces:**
- Consumes: `postgres`, `redis`, `mailpit` services (Task 1).
- Produces: image `lets-discuss/php:8.4`; YAML anchor `x-php` (`&php`) reused by `queue`, `scheduler` and `reverb` in later tasks; service `api` (php-fpm on port 9000 inside the network); Laravel app at `/var/www/html`.

- [ ] **Step 1: Create `docker/php/Dockerfile`**

```dockerfile
FROM php:8.4-fpm-alpine

COPY --from=mlocati/php-extension-installer:latest /usr/bin/install-php-extensions /usr/local/bin/
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

RUN apk add --no-cache git unzip \
    && install-php-extensions pdo_pgsql pgsql redis gd intl zip bcmath pcntl opcache exif xdebug

COPY conf.d/ /usr/local/etc/php/conf.d/

# Xdebug is installed but idle; enable per run with XDEBUG_MODE=debug,coverage
ENV XDEBUG_MODE=off \
    COMPOSER_HOME=/tmp/composer

WORKDIR /var/www/html
```

- [ ] **Step 2: Create `docker/php/conf.d/lets-discuss.ini`**

```ini
; Posts carry up to 4 images; nginx client_max_body_size is 25m to match
upload_max_filesize = 20M
post_max_size = 25M
memory_limit = 512M
```

- [ ] **Step 3: Create `docker/php/conf.d/xdebug.ini`**

```ini
xdebug.client_host = host.docker.internal
xdebug.start_with_request = yes
```

- [ ] **Step 4: Add the PHP anchor and `api` service to `docker-compose.yml`**

Insert directly under `name: lets-discuss`:

```yaml
x-php: &php
  build:
    context: ./docker/php
  image: lets-discuss/php:8.4
  working_dir: /var/www/html
  volumes:
    - ./lets-discuss-api:/var/www/html
  environment:
    XDEBUG_MODE: ${XDEBUG_MODE:-off}
  extra_hosts:
    - "host.docker.internal:host-gateway"
  depends_on:
    postgres:
      condition: service_healthy
    redis:
      condition: service_healthy
  restart: unless-stopped
```

Add under `services:` (before `postgres`):

```yaml
  api:
    <<: *php
```

- [ ] **Step 5: Build the image and verify extensions and limits**

```bash
export MSYS_NO_PATHCONV=1
docker compose build api
docker compose run --rm --no-deps api php -m | grep -Ei "^(pdo_pgsql|pgsql|redis|gd|intl|zip|bcmath|pcntl|exif|xdebug|zend opcache)$"
docker compose run --rm --no-deps api php -i | grep -E "^(upload_max_filesize|post_max_size|memory_limit)"
docker compose run --rm --no-deps api php -v
```
Expected: all 11 extensions listed; `upload_max_filesize => 20M`, `post_max_size => 25M`, `memory_limit => 512M`; `PHP 8.4.x`.

- [ ] **Step 6: Scaffold Laravel 13 inside the container**

```bash
export MSYS_NO_PATHCONV=1
mkdir -p lets-discuss-api
docker compose run --rm --no-deps api composer create-project laravel/laravel:^13.0 . --prefer-dist
```
Expected: ends with `Application key set successfully` (the skeleton's post-create scripts also migrate its default SQLite database; that is fine and will be replaced in the next step).

- [ ] **Step 7: Initialise the API repo**

```bash
git -C lets-discuss-api init -b main
```

- [ ] **Step 8: Point `.env` at the Docker services**

In `lets-discuss-api/.env` set these keys (uncomment any that the skeleton ships commented out; leave all other keys as generated):

```dotenv
APP_NAME="Lets Discuss"
APP_URL=http://api.lets-discuss.localhost
FRONTEND_URL=http://lets-discuss.localhost

DB_CONNECTION=pgsql
DB_HOST=postgres
DB_PORT=5432
DB_DATABASE=lets_discuss
DB_USERNAME=lets_discuss
DB_PASSWORD=secret

SESSION_DRIVER=redis
CACHE_STORE=redis
QUEUE_CONNECTION=redis

REDIS_CLIENT=phpredis
REDIS_HOST=redis
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=smtp
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_FROM_ADDRESS="no-reply@lets-discuss.localhost"
MAIL_FROM_NAME="${APP_NAME}"
```

Apply the same keys to `lets-discuss-api/.env.example` (keep `APP_KEY=` empty there). Delete the skeleton's `lets-discuss-api/database/database.sqlite` because it is no longer used.

- [ ] **Step 9: Keep tests on in-memory SQLite**

Open `lets-discuss-api/phpunit.xml` and make sure these `<env>` lines exist inside `<php>`, uncommented:

```xml
<env name="DB_CONNECTION" value="sqlite"/>
<env name="DB_DATABASE" value=":memory:"/>
```

- [ ] **Step 10: Start `api` and migrate against Postgres**

```bash
docker compose up -d api
docker compose exec api php artisan migrate --force
docker compose exec api php artisan about --only=environment,drivers
```
Expected: migrations run on `pgsql`; `about` shows Laravel `13.x`, PHP `8.4.x`, Database `pgsql`, Cache/Queue/Session `redis`, Mail `smtp`.

- [ ] **Step 11: Verify mail reaches Mailpit**

```bash
docker compose exec api php artisan tinker --execute='Mail::raw("ping", fn ($m) => $m->to("dev@lets-discuss.localhost")->subject("Mailpit check"));'
curl -s http://localhost:8025/api/v1/messages | grep -o "Mailpit check"
```
Expected: `Mailpit check`.

- [ ] **Step 12: Run the skeleton test suite**

Run: `docker compose exec api php artisan test`
Expected: the skeleton's example tests PASS.

- [ ] **Step 13: Review (no commit)**

Run: `git status --short && git -C lets-discuss-api status --short | head -20`
Expected: root shows new `docker/`; the API repo shows the scaffold as untracked, and `.env` is ignored by the skeleton's `.gitignore`.

---

### Task 3: Nginx API vhost, worker containers and `/api/health`

**Files:**
- Create: `docker/nginx/conf.d/00-upgrade-map.conf`, `docker/nginx/conf.d/api.conf`
- Modify: `docker-compose.yml` (add `queue`, `scheduler`, `nginx`)
- Create: `lets-discuss-api/app/Http/Controllers/HealthController.php`
- Create: `lets-discuss-api/routes/api.php` (via `install:api`)
- Test: `lets-discuss-api/tests/Feature/HealthCheckTest.php`

**Interfaces:**
- Consumes: `x-php` anchor, `api` service (Task 2).
- Produces: `GET /api/health` → `200 {"status":"ok","services":{"database":"up","redis":"up"}}` or `503 {"status":"degraded","services":{...}}` where each service is `"up"|"down"`. Nginx service `nginx` on host port 80 with the `$connection_upgrade` map available to every vhost.

- [ ] **Step 1: Install API routing and Sanctum**

```bash
docker compose exec api php artisan install:api --without-migration-prompt --no-interaction
docker compose exec api php artisan migrate --force
```
Expected: `routes/api.php` exists; Sanctum is in `composer.json`; the `personal_access_tokens` migration runs.

- [ ] **Step 2: Write the failing test** — `lets-discuss-api/tests/Feature/HealthCheckTest.php`

```php
<?php

namespace Tests\Feature;

use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Redis;
use Tests\TestCase;

class HealthCheckTest extends TestCase
{
    public function test_reports_ok_when_all_services_are_up(): void
    {
        $this->getJson('/api/health')
            ->assertOk()
            ->assertExactJson([
                'status' => 'ok',
                'services' => ['database' => 'up', 'redis' => 'up'],
            ]);
    }

    public function test_reports_degraded_with_503_when_the_database_is_down(): void
    {
        config(['database.connections.sqlite.database' => '/nonexistent/lets-discuss.sqlite']);
        DB::purge('sqlite');

        $this->getJson('/api/health')
            ->assertStatus(503)
            ->assertJsonPath('status', 'degraded')
            ->assertJsonPath('services.database', 'down')
            ->assertJsonPath('services.redis', 'up');
    }

    public function test_reports_degraded_with_503_when_redis_is_down(): void
    {
        config([
            'database.redis.default.host' => '127.0.0.1',
            'database.redis.default.port' => 1,
        ]);
        Redis::purge('default');

        $this->getJson('/api/health')
            ->assertStatus(503)
            ->assertJsonPath('status', 'degraded')
            ->assertJsonPath('services.database', 'up')
            ->assertJsonPath('services.redis', 'down');
    }
}
```

- [ ] **Step 3: Run it to verify it fails**

Run: `docker compose exec api php artisan test --filter=HealthCheckTest`
Expected: 3 FAIL with `Expected response status code [200] but received 404` (and 404 instead of 503).

- [ ] **Step 4: Implement the controller** — `lets-discuss-api/app/Http/Controllers/HealthController.php`

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Redis;
use Throwable;

class HealthController extends Controller
{
    public function __invoke(): JsonResponse
    {
        $services = [
            'database' => $this->probe(fn () => DB::connection()->getPdo()),
            'redis' => $this->probe(fn () => Redis::connection()->ping()),
        ];

        $healthy = ! in_array('down', $services, true);

        return response()->json([
            'status' => $healthy ? 'ok' : 'degraded',
            'services' => $services,
        ], $healthy ? 200 : 503);
    }

    /**
     * @param  callable(): mixed  $check
     * @return 'up'|'down'
     */
    private function probe(callable $check): string
    {
        try {
            $check();

            return 'up';
        } catch (Throwable) {
            return 'down';
        }
    }
}
```

- [ ] **Step 5: Register the route** — replace the body of `lets-discuss-api/routes/api.php` with:

```php
<?php

use App\Http\Controllers\HealthController;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

Route::get('/health', HealthController::class)->name('health');

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

(The `/user` route is what `install:api` generates; keep it.)

- [ ] **Step 6: Run the test to verify it passes**

Run: `docker compose exec api php artisan test --filter=HealthCheckTest`
Expected: 3 PASS.

- [ ] **Step 7: Create `docker/nginx/conf.d/00-upgrade-map.conf`**

```nginx
# Shared by every vhost that proxies WebSockets (Reverb, Vite HMR)
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

- [ ] **Step 8: Create `docker/nginx/conf.d/api.conf`**

```nginx
server {
    listen 80;
    server_name api.lets-discuss.localhost;

    root /var/www/html/public;
    index index.php;
    charset utf-8;

    # Up to 4 images per post; PHP post_max_size is 25M to match
    client_max_body_size 25m;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ ^/index\.php(/|$) {
        fastcgi_pass api:9000;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;
        fastcgi_hide_header X-Powered-By;
    }

    location ~ \.php$ {
        return 404;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

- [ ] **Step 9: Add `queue`, `scheduler` and `nginx` to `docker-compose.yml`**

Under `services:`:

```yaml
  queue:
    <<: *php
    command: php artisan queue:work --tries=3 --timeout=90

  scheduler:
    <<: *php
    command: php artisan schedule:work

  nginx:
    image: nginx:1.29-alpine
    ports:
      - "80:80"
    volumes:
      - ./docker/nginx/conf.d:/etc/nginx/conf.d:ro
      - ./lets-discuss-api:/var/www/html:ro
    depends_on:
      - api
    restart: unless-stopped
```

- [ ] **Step 10: Start and verify through the browser-facing hostname**

```bash
docker compose up -d queue scheduler nginx
docker compose exec nginx nginx -t
curl -s -i http://api.lets-discuss.localhost/api/health
docker compose ps queue scheduler
```
Expected: `syntax is ok`; `HTTP/1.1 200 OK` with body `{"status":"ok","services":{"database":"up","redis":"up"}}`; queue and scheduler `running` (check `docker compose logs queue` shows `Processing jobs from the [default] queue`).

- [ ] **Step 11: Verify the degraded path against real services**

```bash
docker compose stop redis
curl -s -o /dev/null -w "%{http_code}\n" http://api.lets-discuss.localhost/api/health
docker compose start redis
```
Expected: `503`, then Redis healthy again.

- [ ] **Step 12: Review (no commit)**

Run: `docker compose exec api php artisan test`
Expected: all tests PASS. Leave changes uncommitted.

---

### Task 4: Sanctum SPA cookies, CORS and Reverb

**Files:**
- Modify: `lets-discuss-api/bootstrap/app.php`, `lets-discuss-api/.env`, `lets-discuss-api/.env.example`
- Create: `lets-discuss-api/config/cors.php` (via `config:publish cors`)
- Create (generated): `lets-discuss-api/config/broadcasting.php`, `config/reverb.php`, `routes/channels.php` (via `install:broadcasting`)
- Modify: `lets-discuss-api/config/reverb.php` (allowed origins)
- Modify: `docker/nginx/conf.d/api.conf` (add `/app/` WebSocket proxy), `docker-compose.yml` (add `reverb`)
- Test: `lets-discuss-api/tests/Feature/SpaAuthenticationTest.php`

**Interfaces:**
- Consumes: `/api/health` route (Task 3), nginx `$connection_upgrade` map (Task 3).
- Produces: `GET /sanctum/csrf-cookie` → 204 + `XSRF-TOKEN` cookie scoped to `.lets-discuss.localhost`; credentialed CORS for `http://lets-discuss.localhost` on `api/*`, `sanctum/csrf-cookie`, `broadcasting/auth`; Reverb WebSocket at `ws://api.lets-discuss.localhost/app/{REVERB_APP_KEY}`. `REVERB_APP_KEY` value in `lets-discuss-api/.env` is copied to the web app in Task 6.

- [ ] **Step 1: Write the failing tests** — `lets-discuss-api/tests/Feature/SpaAuthenticationTest.php`

```php
<?php

namespace Tests\Feature;

use Tests\TestCase;

class SpaAuthenticationTest extends TestCase
{
    private const SPA_ORIGIN = 'http://lets-discuss.localhost';

    public function test_csrf_cookie_is_issued_for_the_shared_parent_domain(): void
    {
        $response = $this->withHeaders([
            'Origin' => self::SPA_ORIGIN,
            'Referer' => self::SPA_ORIGIN.'/',
        ])->get('/sanctum/csrf-cookie');

        $response->assertNoContent();

        $cookie = $response->getCookie('XSRF-TOKEN', decrypt: false);
        $this->assertNotNull($cookie, 'XSRF-TOKEN cookie was not set');
        $this->assertStringEndsWith('lets-discuss.localhost', (string) $cookie->getDomain());
    }

    public function test_spa_origin_may_make_credentialed_cross_origin_requests(): void
    {
        $this->withHeaders([
            'Origin' => self::SPA_ORIGIN,
            'Access-Control-Request-Method' => 'GET',
        ])->options('/api/health')
            ->assertNoContent()
            ->assertHeader('Access-Control-Allow-Origin', self::SPA_ORIGIN)
            ->assertHeader('Access-Control-Allow-Credentials', 'true');
    }

    public function test_unknown_origins_are_not_granted_cors_access(): void
    {
        $this->withHeaders([
            'Origin' => 'http://evil.example',
            'Access-Control-Request-Method' => 'GET',
        ])->options('/api/health')
            ->assertHeaderMissing('Access-Control-Allow-Origin');
    }
}
```

- [ ] **Step 2: Run them to verify they fail**

Run: `docker compose exec api php artisan test --filter=SpaAuthenticationTest`
Expected: FAIL. The cookie domain is empty; `Access-Control-Allow-Origin` is `*` rather than the SPA origin; the unknown origin receives `*`.

- [ ] **Step 3: Enable stateful API requests** — in `lets-discuss-api/bootstrap/app.php`, make `withMiddleware` read:

```php
    ->withMiddleware(function (Middleware $middleware): void {
        $middleware->statefulApi();
    })
```

- [ ] **Step 4: Publish and configure CORS**

Run: `docker compose exec api php artisan config:publish cors`

Replace `lets-discuss-api/config/cors.php` with:

```php
<?php

return [

    'paths' => ['api/*', 'sanctum/csrf-cookie', 'broadcasting/auth'],

    'allowed_methods' => ['*'],

    'allowed_origins' => [env('FRONTEND_URL', 'http://lets-discuss.localhost')],

    'allowed_origins_patterns' => [],

    'allowed_headers' => ['*'],

    'exposed_headers' => [],

    'max_age' => 0,

    // Sanctum SPA auth sends the session + XSRF cookies cross-subdomain
    'supports_credentials' => true,

];
```

- [ ] **Step 5: Configure session and Sanctum domains** — add to `lets-discuss-api/.env` and `.env.example`:

```dotenv
SANCTUM_STATEFUL_DOMAINS=lets-discuss.localhost
SESSION_DOMAIN=.lets-discuss.localhost
SESSION_SECURE_COOKIE=false
SESSION_SAME_SITE=lax
```

- [ ] **Step 6: Run the tests to verify they pass**

Run: `docker compose exec api php artisan config:clear && docker compose exec api php artisan test --filter=SpaAuthenticationTest`
Expected: 3 PASS.

- [ ] **Step 7: Install broadcasting with Reverb**

```bash
docker compose exec api php artisan install:broadcasting --reverb --without-node --no-interaction
```
Expected: `laravel/reverb` added to `composer.json`; `config/broadcasting.php`, `config/reverb.php` and `routes/channels.php` created; `REVERB_APP_ID`, `REVERB_APP_KEY` and `REVERB_APP_SECRET` written to `.env`.

- [ ] **Step 8: Point Laravel at the `reverb` container** — in `lets-discuss-api/.env` set (keep the generated `REVERB_APP_*` values):

```dotenv
BROADCAST_CONNECTION=reverb

# Laravel -> Reverb (inside the Docker network)
REVERB_HOST=reverb
REVERB_PORT=8080
REVERB_SCHEME=http

# Reverb server bind
REVERB_SERVER_HOST=0.0.0.0
REVERB_SERVER_PORT=8080
REVERB_ALLOWED_ORIGINS=lets-discuss.localhost
```

Mirror these keys into `.env.example` with empty `REVERB_APP_ID=`, `REVERB_APP_KEY=` and `REVERB_APP_SECRET=`.

- [ ] **Step 9: Restrict Reverb origins** — in `lets-discuss-api/config/reverb.php`, inside `apps.apps[0]`, replace the `allowed_origins` line with:

```php
                'allowed_origins' => explode(',', env('REVERB_ALLOWED_ORIGINS', 'lets-discuss.localhost')),
```

- [ ] **Step 10: Proxy the WebSocket through Nginx** — in `docker/nginx/conf.d/api.conf`, add this block above `location / {`:

```nginx
    # Reverb (Pusher protocol): ws://api.lets-discuss.localhost/app/{key}
    location /app/ {
        proxy_pass http://reverb:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 60s;
    }
```

- [ ] **Step 11: Add the `reverb` service** — in `docker-compose.yml` under `services:`:

```yaml
  reverb:
    <<: *php
    command: php artisan reverb:start --host=0.0.0.0 --port=8080
```

Also change the `nginx` service's `depends_on` to:

```yaml
    depends_on:
      - api
      - reverb
```

- [ ] **Step 12: Restart and verify the WebSocket handshake through Nginx**

```bash
docker compose up -d reverb
docker compose restart api queue scheduler nginx
KEY=$(grep '^REVERB_APP_KEY=' lets-discuss-api/.env | cut -d= -f2 | tr -d '"')
curl -s -i -N --max-time 3 \
  -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: SGVsbG8sIHdvcmxkIQ==" \
  -H "Origin: http://lets-discuss.localhost" \
  "http://api.lets-discuss.localhost/app/$KEY?protocol=7&client=js&version=8.4.0" | head -1
```
Expected: `HTTP/1.1 101 Switching Protocols`. Then check `docker compose logs reverb` shows `Starting server on 0.0.0.0:8080`.

- [ ] **Step 13: Verify the cookie over real HTTP**

Run: `curl -s -i -H "Origin: http://lets-discuss.localhost" -H "Referer: http://lets-discuss.localhost/" http://api.lets-discuss.localhost/sanctum/csrf-cookie | grep -iE "^(HTTP|access-control-allow|set-cookie)"`
Expected: `204`; `Access-Control-Allow-Origin: http://lets-discuss.localhost`; `Access-Control-Allow-Credentials: true`; `Set-Cookie: XSRF-TOKEN=...; domain=.lets-discuss.localhost` and `lets_discuss_session=...; domain=.lets-discuss.localhost; httponly`.

- [ ] **Step 14: Review (no commit)**

Run: `docker compose exec api php artisan test`
Expected: all PASS.

---

### Task 5: API domain packages, Swagger UI and code quality

**Files:**
- Modify: `lets-discuss-api/composer.json` (via composer)
- Create: `lets-discuss-api/config/media-library.php`, `config/swagger-ui.php`, `app/Providers/SwaggerUiServiceProvider.php` (published)
- Modify: `lets-discuss-api/config/services.php`, `.env`, `.env.example`
- Create: `lets-discuss-api/resources/swagger/openapi.yaml`, `lets-discuss-api/phpstan.neon`
- Test: `lets-discuss-api/tests/Feature/SwaggerUiTest.php`

**Interfaces:**
- Consumes: `/api/health` and `/sanctum/csrf-cookie` contracts (Tasks 3–4), MinIO bucket `lets-discuss` (Task 1).
- Produces: `s3` disk working against MinIO; `config('services.turnstile.site_key')` and `config('services.turnstile.secret_key')`; Swagger UI at `http://api.lets-discuss.localhost/swagger` (local env only); `vendor/bin/phpstan analyse` and `vendor/bin/pint --test` clean.

- [ ] **Step 1: Install the domain packages**

```bash
docker compose exec api composer require spatie/laravel-medialibrary "league/flysystem-aws-s3-v3:^3.0" league/commonmark
docker compose exec api php artisan vendor:publish --provider="Spatie\MediaLibrary\MediaLibraryServiceProvider" --tag="medialibrary-config"
```
Expected: packages install without conflicts; `config/media-library.php` exists. Do **not** publish or run the media migration (that belongs to the Media feature).

- [ ] **Step 2: Configure MinIO and Turnstile** — add to `lets-discuss-api/.env` and `.env.example`:

```dotenv
FILESYSTEM_DISK=local
MEDIA_DISK=s3

AWS_ACCESS_KEY_ID=minio
AWS_SECRET_ACCESS_KEY=minio-secret
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=lets-discuss
AWS_ENDPOINT=http://minio:9000
AWS_URL=http://localhost:9000/lets-discuss
AWS_USE_PATH_STYLE_ENDPOINT=true

# Cloudflare test keys: always pass (https://developers.cloudflare.com/turnstile/troubleshooting/testing/)
TURNSTILE_SITE_KEY=1x00000000000000000000AA
TURNSTILE_SECRET_KEY=1x0000000000000000000000000000000AA
```

In `lets-discuss-api/config/services.php`, add inside the returned array:

```php
    'turnstile' => [
        'site_key' => env('TURNSTILE_SITE_KEY'),
        'secret_key' => env('TURNSTILE_SECRET_KEY'),
    ],
```

- [ ] **Step 3: Verify the `s3` disk round-trips through MinIO**

```bash
docker compose exec api php artisan config:clear
docker compose exec api php artisan tinker --execute='Storage::disk("s3")->put("healthcheck.txt", "ok"); echo Storage::disk("s3")->get("healthcheck.txt");'
curl -s http://localhost:9000/lets-discuss/healthcheck.txt; echo
docker compose exec api php artisan tinker --execute='Storage::disk("s3")->delete("healthcheck.txt");'
```
Expected: `ok` from tinker, and `ok` from the public URL.

- [ ] **Step 4: Write the failing Swagger test** — `lets-discuss-api/tests/Feature/SwaggerUiTest.php`

```php
<?php

namespace Tests\Feature;

use Tests\TestCase;

class SwaggerUiTest extends TestCase
{
    public function test_swagger_ui_is_served_in_the_local_environment(): void
    {
        $this->app->detectEnvironment(fn () => 'local');

        $this->get('/swagger')->assertOk();
    }
}
```

- [ ] **Step 5: Run it to verify it fails**

Run: `docker compose exec api php artisan test --filter=SwaggerUiTest`
Expected: FAIL, `Expected response status code [200] but received 404`.

- [ ] **Step 6: Install Swagger UI**

```bash
docker compose exec api composer require wotzebra/laravel-swagger-ui
docker compose exec api php artisan swagger-ui:install
```
Expected: `config/swagger-ui.php` and `app/Providers/SwaggerUiServiceProvider.php` are created, and the provider is registered in `bootstrap/providers.php`.

- [ ] **Step 7: Point the config at the spec** — in `lets-discuss-api/config/swagger-ui.php`, in the first entry of `files`, change only these keys and leave the rest as published:

```php
            'versions' => [
                'v1' => resource_path('swagger/openapi.yaml'),
            ],
            'server_url' => env('APP_URL'),
```

- [ ] **Step 8: Write the spec** — `lets-discuss-api/resources/swagger/openapi.yaml`

```yaml
openapi: 3.1.0
info:
  title: Lets Discuss API
  version: 0.1.0
  description: >
    Spec-first contract for the Lets Discuss platform. The SPA authenticates with
    Sanctum session cookies: call GET /sanctum/csrf-cookie first, then send the
    XSRF-TOKEN cookie value back in the X-XSRF-TOKEN header on state-changing requests.
servers:
  - url: http://api.lets-discuss.localhost
tags:
  - name: System
    description: Operational endpoints
  - name: Auth
    description: Session and CSRF handling
paths:
  /api/health:
    get:
      tags: [System]
      operationId: getHealth
      summary: Report whether the API and its dependencies are reachable
      security: []
      responses:
        '200':
          description: All dependencies are up
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Health'
              example:
                status: ok
                services: { database: up, redis: up }
        '503':
          description: At least one dependency is down
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Health'
              example:
                status: degraded
                services: { database: up, redis: down }
  /sanctum/csrf-cookie:
    get:
      tags: [Auth]
      operationId: initCsrf
      summary: Issue the XSRF-TOKEN and session cookies
      security: []
      responses:
        '204':
          description: Cookies set on the .lets-discuss.localhost domain
          headers:
            Set-Cookie:
              schema:
                type: string
              description: XSRF-TOKEN and lets_discuss_session cookies
components:
  securitySchemes:
    sessionCookie:
      type: apiKey
      in: cookie
      name: lets_discuss_session
    xsrfToken:
      type: apiKey
      in: header
      name: X-XSRF-TOKEN
  schemas:
    Health:
      type: object
      required: [status, services]
      properties:
        status:
          type: string
          enum: [ok, degraded]
        services:
          type: object
          additionalProperties:
            type: string
            enum: [up, down]
security:
  - sessionCookie: []
    xsrfToken: []
```

- [ ] **Step 9: Run the test to verify it passes**

Run: `docker compose exec api php artisan test --filter=SwaggerUiTest`
Expected: PASS.

- [ ] **Step 10: Verify in the browser**

Open `http://api.lets-discuss.localhost/swagger`.
Expected: Swagger UI lists `GET /api/health` and `GET /sanctum/csrf-cookie`, and "Try it out" on `/api/health` returns `200`. If the UI shows a parse error for YAML, convert the file to `openapi.json` (same content), update `versions` to match, and note it in the task report.

- [ ] **Step 11: Add Larastan** — `docker compose exec api composer require --dev larastan/larastan`, then create `lets-discuss-api/phpstan.neon`:

```neon
includes:
    - vendor/larastan/larastan/extension.neon

parameters:
    level: 6
    paths:
        - app/
        - config/
        - database/
        - routes/
```

- [ ] **Step 12: Run all quality gates**

```bash
docker compose exec api vendor/bin/pint --test
docker compose exec api vendor/bin/phpstan analyse --memory-limit=1G --no-progress
docker compose exec api php artisan test
```
Expected: Pint reports no issues (if it does, run `vendor/bin/pint` and re-run `--test`); PHPStan `[OK] No errors`; all tests PASS.

- [ ] **Step 13: Review (no commit)**

Run: `git -C lets-discuss-api status --short`
Expected: new configs, spec, provider, phpstan.neon and tests appear as untracked.

---

### Task 6: Web scaffold in Docker (Vite, packages, Tailwind + antd, lint, Vitest)

**Files:**
- Create (generated): `lets-discuss-web/**` via `create-vite` (react-ts template)
- Modify: `lets-discuss-web/vite.config.ts`, `src/index.css`, `src/main.tsx`, `src/App.tsx`, `eslint.config.js`, `package.json`, `.gitignore`
- Create: `lets-discuss-web/.env`, `.env.example`, `.prettierrc`, `.prettierignore`, `src/env.d.ts`, `src/test/setup.ts`, `src/lib/http.ts`, `src/lib/echo.ts`
- Delete: `lets-discuss-web/src/App.css`, `src/assets/react.svg`, `public/vite.svg`
- Create: `docker/nginx/conf.d/web.conf`
- Modify: `docker-compose.yml` (add `web`, volume `web-node-modules`, nginx `depends_on`)

**Interfaces:**
- Consumes: `REVERB_APP_KEY` from `lets-discuss-api/.env` (Task 4), API at `http://api.lets-discuss.localhost`.
- Produces: `http` (axios instance, `import { http } from '../../lib/http'`); `getEcho(): Echo<'reverb'>` (`src/lib/echo.ts`); `ImportMetaEnv` keys `VITE_API_URL`, `VITE_REVERB_APP_KEY`, `VITE_REVERB_HOST`, `VITE_REVERB_PORT`, `VITE_REVERB_SCHEME`, `VITE_TURNSTILE_SITE_KEY`; npm scripts `dev`, `build`, `lint`, `test`, `test:watch`, `e2e`, `format`, `format:check`; service `web` with `node_modules` in named volume `web-node-modules`.

- [ ] **Step 1: Scaffold with a throwaway Node container**

```bash
export MSYS_NO_PATHCONV=1
docker run --rm -v "$(pwd -W):/work" -w /work node:24-alpine \
  npm create vite@latest lets-discuss-web -- --template react-ts --no-interactive
git -C lets-discuss-web init -b main
```
Expected: `lets-discuss-web/` holds the react-ts template and no `node_modules`. If `create-vite` still prompts, answer **No** to "install and start now". (`pwd -W` gives the Windows path, which Docker Desktop needs.)

- [ ] **Step 2: Configure the dev server and Vitest** — replace `lets-discuss-web/vite.config.ts`:

```ts
import tailwindcss from '@tailwindcss/vite'
import react from '@vitejs/plugin-react'
import { defineConfig } from 'vitest/config'

export default defineConfig({
  plugins: [react(), tailwindcss()],
  server: {
    host: '0.0.0.0',
    port: 5173,
    strictPort: true,
    allowedHosts: ['lets-discuss.localhost'],
    // Browser reaches Vite through nginx on :80
    hmr: { host: 'lets-discuss.localhost', clientPort: 80 },
    // Windows bind mounts do not deliver inotify events into the container
    watch: { usePolling: true, interval: 300 },
  },
  test: {
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
    include: ['src/**/*.test.{ts,tsx}'],
  },
})
```

- [ ] **Step 3: Create env files** — `lets-discuss-web/.env.example`:

```dotenv
VITE_API_URL=http://api.lets-discuss.localhost
VITE_REVERB_APP_KEY=
VITE_REVERB_HOST=api.lets-discuss.localhost
VITE_REVERB_PORT=80
VITE_REVERB_SCHEME=http
# Cloudflare test key: always passes
VITE_TURNSTILE_SITE_KEY=1x00000000000000000000AA
```

Copy it to `lets-discuss-web/.env` and set `VITE_REVERB_APP_KEY` to the `REVERB_APP_KEY` value from `lets-discuss-api/.env`:

```bash
cp lets-discuss-web/.env.example lets-discuss-web/.env
KEY=$(grep '^REVERB_APP_KEY=' lets-discuss-api/.env | cut -d= -f2 | tr -d '"')
sed -i "s/^VITE_REVERB_APP_KEY=.*/VITE_REVERB_APP_KEY=$KEY/" lets-discuss-web/.env
printf '\n# Local environment\n.env\n' >> lets-discuss-web/.gitignore
```

- [ ] **Step 4: Type the env** — create `lets-discuss-web/src/env.d.ts`:

```ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string
  readonly VITE_REVERB_APP_KEY: string
  readonly VITE_REVERB_HOST: string
  readonly VITE_REVERB_PORT: string
  readonly VITE_REVERB_SCHEME: 'http' | 'https'
  readonly VITE_TURNSTILE_SITE_KEY: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

If the template generated `src/vite-env.d.ts`, delete it (this file replaces it).

- [ ] **Step 5: Add `web` and its vhost** — create `docker/nginx/conf.d/web.conf`:

```nginx
server {
    listen 80;
    server_name lets-discuss.localhost;

    location / {
        proxy_pass http://web:5173;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        # Vite HMR runs over a WebSocket on the same origin
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }
}
```

In `docker-compose.yml`, under `services:`:

```yaml
  web:
    image: node:24-alpine
    working_dir: /app
    command: sh -c "npm install && npm run dev"
    volumes:
      - ./lets-discuss-web:/app
      # Linux-native deps stay in the container; the host never sees them
      - web-node-modules:/app/node_modules
    restart: unless-stopped
```

Add `web-node-modules:` under the top-level `volumes:`, and change the nginx `depends_on` to `[api, reverb, web]` (block style, as before).

- [ ] **Step 6: Start `web` and install the application packages**

```bash
docker compose up -d web
docker compose logs -f web   # wait for "VITE vX ready", then Ctrl+C
docker compose exec web npm install \
  antd@^5 @ant-design/cssinjs@^1 @ant-design/v5-patch-for-react-19 \
  react-router @tanstack/react-query zustand axios laravel-echo pusher-js \
  react-markdown @marsidev/react-turnstile
docker compose exec web npm install -D \
  tailwindcss @tailwindcss/vite \
  vitest jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event \
  @playwright/test prettier prettier-plugin-tailwindcss eslint-config-prettier
```
Expected: both installs finish without `ERESOLVE` errors. If `@ant-design/v5-patch-for-react-19` fails to resolve against the installed React major, record the React version and stop to ask.

- [ ] **Step 7: Tailwind + antd layer order** — replace `lets-discuss-web/src/index.css`:

```css
/* antd's CSS-in-JS sits between Tailwind's reset and its utilities */
@layer theme, base, antd, components, utilities;

@import 'tailwindcss';
```

Delete `src/App.css`, `src/assets/react.svg` and `public/vite.svg`. In `index.html`, remove the `<link rel="icon" ...vite.svg>` line and set `<title>Lets Discuss</title>`.

- [ ] **Step 8: Shared HTTP client** — create `lets-discuss-web/src/lib/http.ts`:

```ts
import axios from 'axios'

// Sanctum SPA auth: the session + XSRF cookies are scoped to .lets-discuss.localhost,
// and the API lives on a sibling subdomain, so credentials and the XSRF header must be sent explicitly.
export const http = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  withCredentials: true,
  withXSRFToken: true,
  headers: { Accept: 'application/json' },
})
```

- [ ] **Step 9: Echo client** — create `lets-discuss-web/src/lib/echo.ts`:

```ts
import Echo from 'laravel-echo'
import Pusher from 'pusher-js'

declare global {
  interface Window {
    Pusher: typeof Pusher
  }
}

window.Pusher = Pusher

let echo: Echo<'reverb'> | undefined

export function getEcho(): Echo<'reverb'> {
  echo ??= new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: Number(import.meta.env.VITE_REVERB_PORT),
    wssPort: Number(import.meta.env.VITE_REVERB_PORT),
    forceTLS: import.meta.env.VITE_REVERB_SCHEME === 'https',
    enabledTransports: ['ws', 'wss'],
  })
  return echo
}
```

- [ ] **Step 10: App shell** — replace `lets-discuss-web/src/main.tsx`:

```tsx
import '@ant-design/v5-patch-for-react-19'
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'
import './index.css'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

Replace `lets-discuss-web/src/App.tsx` (placeholder page; Task 7 swaps in the status page):

```tsx
import { StyleProvider } from '@ant-design/cssinjs'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { App as AntApp, ConfigProvider } from 'antd'
import { createBrowserRouter, RouterProvider } from 'react-router'

const queryClient = new QueryClient()

const router = createBrowserRouter([
  { path: '/', element: <main className="p-6 text-2xl font-semibold">Lets Discuss</main> },
])

export default function App() {
  return (
    <StyleProvider layer>
      <ConfigProvider>
        <AntApp>
          <QueryClientProvider client={queryClient}>
            <RouterProvider router={router} />
          </QueryClientProvider>
        </AntApp>
      </ConfigProvider>
    </StyleProvider>
  )
}
```

- [ ] **Step 11: Test setup** — create `lets-discuss-web/src/test/setup.ts`:

```ts
import '@testing-library/jest-dom/vitest'
import { cleanup } from '@testing-library/react'
import { afterEach } from 'vitest'

afterEach(() => cleanup())

// antd's responsive helpers call matchMedia, which jsdom does not implement
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: (query: string) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: () => {},
    removeListener: () => {},
    addEventListener: () => {},
    removeEventListener: () => {},
    dispatchEvent: () => false,
  }),
})
```

- [ ] **Step 12: Prettier + ESLint** — create `lets-discuss-web/.prettierrc`:

```json
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

Create `lets-discuss-web/.prettierignore`:

```
dist
node_modules
playwright-report
test-results
package-lock.json
```

In `lets-discuss-web/eslint.config.js`, add `import eslintConfigPrettier from 'eslint-config-prettier'` at the top, and append `eslintConfigPrettier` as the **last** element passed to the exported config (`defineConfig([...])` / `tseslint.config(...)`, whichever the template uses). Also add `'playwright-report', 'test-results'` to the global ignores next to `'dist'`.

- [ ] **Step 13: npm scripts** — in `lets-discuss-web/package.json` `scripts`, keep `dev`, `build`, `lint` and `preview`, and add:

```json
    "test": "vitest run",
    "test:watch": "vitest",
    "e2e": "playwright test",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
```

- [ ] **Step 14: Verify build, lint and format**

```bash
docker compose exec web npm run format
docker compose exec web npm run lint
docker compose exec web npm run format:check
docker compose exec web npm run build
```
Expected: lint reports no problems, format check passes, and `tsc -b && vite build` succeeds.

- [ ] **Step 15: Verify through Nginx**

```bash
docker compose up -d nginx
docker compose restart nginx
curl -s http://lets-discuss.localhost | grep -o "<title>Lets Discuss</title>"
```
Expected: `<title>Lets Discuss</title>`. Open `http://lets-discuss.localhost` in the browser. It should show "Lets Discuss", with no console errors and `[vite] connected.` in the console.

- [ ] **Step 16: Verify HMR across the Windows bind mount (Review Focus 5)**

With the page open, change the placeholder text in `src/App.tsx` from `Lets Discuss` to `Lets Discuss (HMR)` on the host and save.
Expected: the browser updates within about 2 seconds without a full reload. Revert the text.

- [ ] **Step 17: Review (no commit)**

Run: `git -C lets-discuss-web status --short`
Expected: scaffold and config files are untracked; `.env` and `node_modules` do not appear.

---

### Task 7: Status page (API health + Reverb connection)

**Files:**
- Create: `lets-discuss-web/src/features/status/api.ts`
- Create: `lets-discuss-web/src/features/status/ApiHealthCard.tsx`
- Test: `lets-discuss-web/src/features/status/ApiHealthCard.test.tsx`
- Create: `lets-discuss-web/src/features/status/useConnectionState.ts`
- Create: `lets-discuss-web/src/features/status/ReverbStatusCard.tsx`
- Test: `lets-discuss-web/src/features/status/ReverbStatusCard.test.tsx`
- Create: `lets-discuss-web/src/features/status/StatusPage.tsx`
- Modify: `lets-discuss-web/src/App.tsx`

**Interfaces:**
- Consumes: `http` (Task 6), `getEcho()` (Task 6), `GET /api/health` contract (Task 3).
- Produces: `HealthResponse`, `fetchHealth()` and `useApiHealth()` in `api.ts`; `ConnectionLike` and `useConnectionState(connection): string`; `<ApiHealthCard />` renders `data-testid="api-status"` with the text `checking` | `ok` | `degraded` | `unreachable`, and `data-testid="service-{name}"` rows; `<ReverbStatusCard connection />` renders `data-testid="reverb-state"` with the text `Connected` | `Connecting` | `Unavailable` | `Failed` | `Disconnected` | `Initializing`. Task 8's E2E test relies on these test ids and texts.

- [ ] **Step 1: Write the failing API card tests** — `src/features/status/ApiHealthCard.test.tsx`

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { render, screen } from '@testing-library/react'
import type { ReactElement } from 'react'
import { beforeEach, describe, expect, it, type Mock, vi } from 'vitest'
import { http } from '../../lib/http'
import { ApiHealthCard } from './ApiHealthCard'

vi.mock('../../lib/http', () => ({ http: { get: vi.fn() } }))

// Loosely typed so tests can resolve partial axios responses ({ data } only)
const get = http.get as unknown as Mock

function renderWithQuery(ui: ReactElement) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } })
  return render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>)
}

describe('ApiHealthCard', () => {
  beforeEach(() => get.mockReset())

  it('shows ok and every service when the API is healthy', async () => {
    get.mockResolvedValue({
      data: { status: 'ok', services: { database: 'up', redis: 'up' } },
    })

    renderWithQuery(<ApiHealthCard />)

    expect(screen.getByTestId('api-status')).toHaveTextContent('checking')
    expect(await screen.findByText('ok')).toBeInTheDocument()
    expect(screen.getByTestId('service-database')).toHaveTextContent('database: up')
    expect(screen.getByTestId('service-redis')).toHaveTextContent('redis: up')
  })

  it('shows degraded and which service is down when the API answers 503', async () => {
    get.mockResolvedValue({
      data: { status: 'degraded', services: { database: 'up', redis: 'down' } },
    })

    renderWithQuery(<ApiHealthCard />)

    expect(await screen.findByText('degraded')).toBeInTheDocument()
    expect(screen.getByTestId('service-redis')).toHaveTextContent('redis: down')
  })

  it('shows unreachable when the request fails', async () => {
    get.mockRejectedValue(new Error('Network Error'))

    renderWithQuery(<ApiHealthCard />)

    expect(await screen.findByText('unreachable')).toBeInTheDocument()
    expect(screen.queryByTestId('service-database')).not.toBeInTheDocument()
  })
})
```

- [ ] **Step 2: Run them to verify they fail**

Run: `docker compose exec web npx vitest run src/features/status/ApiHealthCard.test.tsx`
Expected: FAIL with `Failed to resolve import "./ApiHealthCard"`.

- [ ] **Step 3: Implement the query** — `src/features/status/api.ts`

```ts
import { useQuery } from '@tanstack/react-query'
import { http } from '../../lib/http'

export type ServiceState = 'up' | 'down'

export interface HealthResponse {
  status: 'ok' | 'degraded'
  services: Record<string, ServiceState>
}

export async function fetchHealth(): Promise<HealthResponse> {
  const { data } = await http.get<HealthResponse>('/api/health', {
    // A 503 still carries the body naming the service that is down
    validateStatus: (status) => status === 200 || status === 503,
  })
  return data
}

export function useApiHealth() {
  return useQuery({ queryKey: ['health'], queryFn: fetchHealth, refetchInterval: 10_000 })
}
```

- [ ] **Step 4: Implement the card** — `src/features/status/ApiHealthCard.tsx`

```tsx
import { Card, Tag } from 'antd'
import { useApiHealth } from './api'

const COLORS: Record<string, string> = {
  checking: 'processing',
  ok: 'success',
  degraded: 'warning',
  unreachable: 'error',
}

export function ApiHealthCard() {
  const { data, isPending, isError } = useApiHealth()
  const status = isPending ? 'checking' : isError || !data ? 'unreachable' : data.status

  return (
    <Card title="Laravel API">
      <Tag color={COLORS[status]} data-testid="api-status">
        {status}
      </Tag>
      {data && !isError && (
        <ul className="mt-3 space-y-1">
          {Object.entries(data.services).map(([name, state]) => (
            <li key={name} data-testid={`service-${name}`}>
              {name}: {state}
            </li>
          ))}
        </ul>
      )}
    </Card>
  )
}
```

- [ ] **Step 5: Run them to verify they pass**

Run: `docker compose exec web npx vitest run src/features/status/ApiHealthCard.test.tsx`
Expected: 3 PASS.

- [ ] **Step 6: Write the failing Reverb card tests** — `src/features/status/ReverbStatusCard.test.tsx`

```tsx
import { act, render, screen } from '@testing-library/react'
import { describe, expect, it } from 'vitest'
import { ReverbStatusCard } from './ReverbStatusCard'
import type { ConnectionLike } from './useConnectionState'

function createFakeConnection(initial: string) {
  const listeners = new Set<() => void>()
  const connection: ConnectionLike = {
    state: initial,
    bind: (_event, listener) => void listeners.add(listener),
    unbind: (_event, listener) => void listeners.delete(listener),
  }
  const transition = (next: string) => {
    connection.state = next
    listeners.forEach((listener) => listener())
  }
  return { connection, transition, listeners }
}

describe('ReverbStatusCard', () => {
  it('shows the current state and follows transitions', () => {
    const { connection, transition } = createFakeConnection('connecting')

    render(<ReverbStatusCard connection={connection} />)
    expect(screen.getByTestId('reverb-state')).toHaveTextContent('Connecting')

    act(() => transition('connected'))
    expect(screen.getByTestId('reverb-state')).toHaveTextContent('Connected')

    act(() => transition('unavailable'))
    expect(screen.getByTestId('reverb-state')).toHaveTextContent('Unavailable')

    act(() => transition('connected'))
    expect(screen.getByTestId('reverb-state')).toHaveTextContent('Connected')
  })

  it('falls back to the raw state name for unknown states', () => {
    const { connection } = createFakeConnection('reticulating')

    render(<ReverbStatusCard connection={connection} />)

    expect(screen.getByTestId('reverb-state')).toHaveTextContent('reticulating')
  })

  it('unsubscribes from the connection on unmount', () => {
    const { connection, listeners } = createFakeConnection('connected')

    const { unmount } = render(<ReverbStatusCard connection={connection} />)
    expect(listeners.size).toBe(1)

    unmount()
    expect(listeners.size).toBe(0)
  })
})
```

- [ ] **Step 7: Run them to verify they fail**

Run: `docker compose exec web npx vitest run src/features/status/ReverbStatusCard.test.tsx`
Expected: FAIL with `Failed to resolve import "./ReverbStatusCard"`.

- [ ] **Step 8: Implement the hook** — `src/features/status/useConnectionState.ts`

```ts
import { useCallback, useSyncExternalStore } from 'react'

/** The slice of pusher-js's ConnectionManager this app depends on. */
export interface ConnectionLike {
  state: string
  bind(event: 'state_change', listener: () => void): void
  unbind(event: 'state_change', listener: () => void): void
}

export function useConnectionState(connection: ConnectionLike): string {
  const subscribe = useCallback(
    (notify: () => void) => {
      connection.bind('state_change', notify)
      return () => connection.unbind('state_change', notify)
    },
    [connection],
  )

  return useSyncExternalStore(subscribe, () => connection.state)
}
```

- [ ] **Step 9: Implement the card** — `src/features/status/ReverbStatusCard.tsx`

```tsx
import { Card, Tag } from 'antd'
import { type ConnectionLike, useConnectionState } from './useConnectionState'

const LABELS: Record<string, { text: string; color: string }> = {
  initialized: { text: 'Initializing', color: 'default' },
  connecting: { text: 'Connecting', color: 'processing' },
  connected: { text: 'Connected', color: 'success' },
  unavailable: { text: 'Unavailable', color: 'error' },
  failed: { text: 'Failed', color: 'error' },
  disconnected: { text: 'Disconnected', color: 'default' },
}

export function ReverbStatusCard({ connection }: { connection: ConnectionLike }) {
  const state = useConnectionState(connection)
  const label = LABELS[state] ?? { text: state, color: 'default' }

  return (
    <Card title="Reverb WebSocket">
      <Tag color={label.color} data-testid="reverb-state">
        {label.text}
      </Tag>
    </Card>
  )
}
```

- [ ] **Step 10: Run them to verify they pass**

Run: `docker compose exec web npx vitest run src/features/status/ReverbStatusCard.test.tsx`
Expected: 3 PASS.

- [ ] **Step 11: Compose the page** — `src/features/status/StatusPage.tsx`

```tsx
import { getEcho } from '../../lib/echo'
import { ApiHealthCard } from './ApiHealthCard'
import { ReverbStatusCard } from './ReverbStatusCard'
import type { ConnectionLike } from './useConnectionState'

export function StatusPage() {
  const connection = getEcho().connector.pusher.connection as unknown as ConnectionLike

  return (
    <main className="mx-auto max-w-3xl p-6">
      <h1 className="mb-6 text-2xl font-semibold">Lets Discuss — environment status</h1>
      <div className="grid gap-4 md:grid-cols-2">
        <ApiHealthCard />
        <ReverbStatusCard connection={connection} />
      </div>
    </main>
  )
}
```

In `src/App.tsx`, add `import { StatusPage } from './features/status/StatusPage'` and replace the placeholder route with:

```tsx
const router = createBrowserRouter([{ path: '/', element: <StatusPage /> }])
```

- [ ] **Step 12: Run the full web gate**

```bash
docker compose exec web npm test
docker compose exec web npm run lint
docker compose exec web npm run format:check
docker compose exec web npm run build
```
Expected: 6 tests PASS; lint, format and build all clean.

- [ ] **Step 13: Verify in the browser**

Open `http://lets-discuss.localhost`.
Expected:
- "Laravel API" shows `ok` with `database: up` and `redis: up`.
- "Reverb WebSocket" shows `Connected`.
- DevTools → Network → the `/api/health` request goes to `api.lets-discuss.localhost` and gets a `200` without a CORS error.
- Then run `docker compose stop reverb`. The tag changes to `Unavailable` or `Connecting` within about 30 seconds. After `docker compose start reverb` it returns to `Connected`.

- [ ] **Step 14: Review (no commit)**

Run: `git -C lets-discuss-web status --short`

---

### Task 8: Playwright E2E, end-to-end verification and README

**Files:**
- Create: `lets-discuss-web/playwright.config.ts`, `lets-discuss-web/e2e/status.spec.ts`
- Modify: `lets-discuss-web/.gitignore`, `docker-compose.yml` (add the `e2e` service under a profile)
- Create: `README.md` (root)

**Interfaces:**
- Consumes: `data-testid="api-status"` with the text `ok`, and `data-testid="reverb-state"` with the text `Connected` (Task 7). The nginx vhosts come from Tasks 3, 4 and 6.
- Produces: `docker compose --profile e2e run --rm e2e` runs the browser suite against the live stack.

- [ ] **Step 1: Write the E2E test** — `lets-discuss-web/e2e/status.spec.ts`

```ts
import { expect, test } from '@playwright/test'

test('status page reports a healthy API and a live Reverb connection', async ({ page }) => {
  await page.goto('/')

  await expect(page.getByTestId('api-status')).toHaveText('ok')
  await expect(page.getByTestId('service-database')).toHaveText('database: up')
  await expect(page.getByTestId('service-redis')).toHaveText('redis: up')
  await expect(page.getByTestId('reverb-state')).toHaveText('Connected', { timeout: 15_000 })
})
```

- [ ] **Step 2: Configure Playwright** — `lets-discuss-web/playwright.config.ts`

```ts
import { defineConfig, devices } from '@playwright/test'

// Chromium maps every *.localhost name to 127.0.0.1 itself. Inside the e2e container
// that is the wrong machine, so E2E_HOST_RESOLVER_RULES sends those names to nginx instead.
const hostResolverRules = process.env.E2E_HOST_RESOLVER_RULES

export default defineConfig({
  testDir: './e2e',
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 1 : 0,
  reporter: [['list'], ['html', { open: 'never' }]],
  use: {
    baseURL: process.env.E2E_BASE_URL ?? 'http://lets-discuss.localhost',
    trace: 'retain-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        launchOptions: {
          args: hostResolverRules ? [`--host-resolver-rules=${hostResolverRules}`] : [],
        },
      },
    },
  ],
})
```

Append the Playwright output folders to `lets-discuss-web/.gitignore`:

```bash
printf '\n# Playwright\nplaywright-report/\ntest-results/\n' >> lets-discuss-web/.gitignore
```

- [ ] **Step 3: Add the `e2e` service, matching the installed Playwright version**

Run: `docker compose exec web npx playwright --version`. It prints `Version 1.X.Y`.

In `docker-compose.yml` under `services:`, replace `1.X.Y` below with that exact version:

```yaml
  e2e:
    image: mcr.microsoft.com/playwright:v1.X.Y-noble
    profiles: [e2e]
    working_dir: /app
    ipc: host
    environment:
      E2E_HOST_RESOLVER_RULES: "MAP lets-discuss.localhost nginx, MAP api.lets-discuss.localhost nginx"
    volumes:
      - ./lets-discuss-web:/app
      - web-node-modules:/app/node_modules
    depends_on:
      - nginx
    command: npx playwright test
```

- [ ] **Step 4: Run the E2E suite**

Run: `docker compose --profile e2e run --rm e2e`
Expected: `1 passed`. If it fails, open `lets-discuss-web/playwright-report/index.html` and check the trace before changing anything.

- [ ] **Step 5: Show that the E2E test catches a broken WebSocket**

```bash
docker compose stop reverb
docker compose --profile e2e run --rm e2e
docker compose start reverb
```
Expected: FAIL on `reverb-state`. The test fails when Reverb is down, so it isn't passing vacuously. With Reverb back up, re-run and expect `1 passed`.

- [ ] **Step 6: Write the root `README.md`**

````markdown
# Lets Discuss — local environment

Three repositories share this folder:

| Folder              | What it is                                      |
|---------------------|-------------------------------------------------|
| `./` (this repo)    | Docker Compose, PHP image, Nginx vhosts, docs    |
| `lets-discuss-api/` | Laravel 13 API (Sanctum, Reverb, PostgreSQL)     |
| `lets-discuss-web/` | Vite + React + TypeScript SPA (antd, Tailwind)   |

## Start

```bash
docker compose up -d
```

## URLs

| URL                                        | Service                                    |
|--------------------------------------------|--------------------------------------------|
| http://lets-discuss.localhost              | React SPA (Vite dev server, HMR)           |
| http://api.lets-discuss.localhost/api/health | API health                                 |
| http://api.lets-discuss.localhost/swagger  | Swagger UI (spec: `lets-discuss-api/resources/swagger/openapi.yaml`) |
| http://localhost:8081                      | Adminer — server `postgres`, user `lets_discuss`, password `secret` |
| http://localhost:8025                      | Mailpit                                    |
| http://localhost:9001                      | MinIO console — `minio` / `minio-secret`, bucket `lets-discuss` |

Postgres is on `localhost:5432` and Redis on `localhost:6379` for desktop clients.

## Everyday commands

In Git Bash, prefix commands with `MSYS_NO_PATHCONV=1` so container paths aren't rewritten.

```bash
docker compose exec api php artisan test                 # backend tests
docker compose exec api vendor/bin/pint --test           # PHP style
docker compose exec api vendor/bin/phpstan analyse       # static analysis
docker compose exec web npm test                         # frontend unit tests
docker compose exec web npm run lint                     # ESLint
docker compose --profile e2e run --rm e2e                # Playwright E2E
XDEBUG_MODE=debug docker compose up -d api               # step-debug PHP
```

## First-time setup on a new machine

1. `docker compose build`
2. `cp lets-discuss-api/.env.example lets-discuss-api/.env`, then run `docker compose run --rm api composer install` and `docker compose run --rm api php artisan key:generate`, and generate Reverb keys with `php artisan reverb:install`.
3. `cp lets-discuss-web/.env.example lets-discuss-web/.env` and set `VITE_REVERB_APP_KEY` to the API's `REVERB_APP_KEY`.
4. `docker compose up -d && docker compose exec api php artisan migrate`
````

- [ ] **Step 7: Full definition-of-done check**

```bash
export MSYS_NO_PATHCONV=1
docker compose down && docker compose up -d
docker compose ps
curl -s http://api.lets-discuss.localhost/api/health; echo
curl -s -o /dev/null -w "swagger %{http_code}\n" http://api.lets-discuss.localhost/swagger
for p in 8025 8081 9001; do printf "%s " $p; curl -s -o /dev/null -w "%{http_code}\n" http://localhost:$p; done
docker compose exec api php artisan test
docker compose exec api vendor/bin/pint --test
docker compose exec api vendor/bin/phpstan analyse --memory-limit=1G --no-progress
docker compose exec web npm test
docker compose exec web npm run lint
docker compose exec web npm run build
docker compose --profile e2e run --rm e2e
```
Expected:
- Every long-running service is `running` or `healthy`, and `minio-init` has exited 0.
- Health returns `{"status":"ok","services":{"database":"up","redis":"up"}}`.
- Swagger returns `200`, and ports 8025, 8081 and 9001 each return `200`.
- Every test suite, lint, static analysis, build and E2E run passes.

Open `http://lets-discuss.localhost` and confirm it shows `ok` and `Connected`.

- [ ] **Step 8: Final review (no commit)**

```bash
git status --short
git -C lets-discuss-api status --short | wc -l
git -C lets-discuss-web status --short | wc -l
```
Expected: all three repos have untracked or changed files and **no commits** (`git log` prints "does not have any commits yet"). Hand over to the owner to review and commit.
