az-environment
==============

Generic Docker + go-task base for Symfony (PHP) + Next.js (Node) + Postgres projects.
Provides the taskfile includes, the compose files and the Dockerfiles; the project repository
keeps everything project-specific.

Installation
------------

1. Clone repository
2. Change branch to ```project-initializer```
```shell
git checkout project-initializer
```
2. Create and/or copy required directories & files with command
```shell
task env:init
```
2. Modify `.env.local` file
3. Build docker containers and create a new Symfony project with command
```shell
task env:new-project
```

Environments
------------

`ENV` in the project's `.env.local` selects the environment. It is read through go-task's `dotenv`,
which **wins over shell environment variables** — `ENV=prod task ...` does nothing, edit `.env.local`.

`task docker:compose:init` generates three files from that value:

| Generated (gitignored) | Source |
|---|---|
| `compose.yaml` | `docker/compose/compose.{ENV}.yaml` |
| `compose.override.yaml` | the project's `compose.override.{ENV}.yaml`, if present |
| `.env.compose` | `docker/env/.env.compose.{ENV}` |

`compose.project.yaml` is copied from `docker/compose/compose.project.yaml` **only when the project
does not already provide one** — projects own their extra services (reverse proxy, workers, …).
The list of `-f` flags is built once in `Taskfile.yml` (`COMPOSE_FILES`), so a project without an
override still works.

### dev

Code is bind-mounted from the host, `php` runs `php:8.4-fpm` with the Symfony CLI and xdebug,
`node` runs `next dev`, Postgres binds `.data/postgres` and publishes its port.
`task app:start` brings up the containers and both development servers.

### prod

Images carry the code — nothing is bind-mounted:

- `dockerfile/php/prod` — multi-stage on `dunglas/frankenphp:php8.4-alpine`: `base` (extensions) →
  `vendor` (`composer install --no-dev`, classmap-authoritative autoloader, `cache:warmup`,
  `assets:install`) → `runtime` (no composer, no xdebug, opcache with `validate_timestamps=0`).
  The entrypoint waits for the database, generates the JWT keypair if missing and runs migrations,
  but only when the container's first argument starts with `-` — so workers sharing the image
  (`php bin/console messenger:consume …`) do not re-run migrations. Serves HTTP on `:8000` as
  `www-data`, with a healthcheck.
- `dockerfile/node/prod` — `deps` (`npm ci`) → `builder` (`next build`, needs `NEXT_PUBLIC_*`
  as build args) → `runtime` (`.next/standalone` plus `.next/static` and `public`, `node server.js`
  as user `node` on `:3000`, healthcheck). Requires `output: "standalone"` in `next.config.ts`.
- Postgres uses a named volume, publishes no port and has a `pg_isready` healthcheck that project
  services can wait on with `depends_on: condition: service_healthy`.

Build context for both images is the application directory, so the Dockerfiles are passed by
absolute path and any extra files (entrypoint, php.ini) are inlined as heredocs.

### Secrets

`docker/env/.env.compose.prod` ships with `CHANGE_ME` placeholders.
`task env:secrets:generate` fills `APP_SECRET`, `SESSION_SECRET`, `JWT_PASSPHRASE` and
`POSTGRES_PASSWORD` with `openssl rand -hex 24`, is idempotent (existing values are kept),
`chmod 600`s the file and fails if any other placeholder is left. The `secrets-configured`
precondition blocks `docker:compose:up` while `.env.compose` still contains `CHANGE_ME`.

Compose interpolates variables inside `env_file` contents, so `DATABASE_URL` and the public URLs
are assembled from `${POSTGRES_*}` and `${*_SUBDOMAIN}.${DOMAIN_NAME}` instead of being duplicated.
Build args are resolved earlier and cannot read `env_file` — pass those from the project's override.

Tasks
-----

| Task | Notes |
|---|---|
| `app:start` / `app:stop` | delegate to a per-`ENV` variant |
| `app:install` | dev only — in prod dependencies live in the image |
| `app:deploy` | prod only: `env:build` → start Postgres → `db:dump` → `up -d` → `app:smoke` |
| `app:smoke` | waits until `php` and `node` report `healthy` |
| `app:migrate`, `app:console -- <cmd>`, `app:logs -- <service>`, `app:messenger:failed` | both environments |
| `db:dump`, `db:list`, `db:restore -- <file>` | `pg_dump`/`psql` through `docker compose exec`, gzipped into `.data/backups`; retention via `BACKUP_KEEP` (default 7, override through the environment) |

`app:deploy` dumps the database **before** `up -d`, because in prod the migrations run from the
`php` container entrypoint.
