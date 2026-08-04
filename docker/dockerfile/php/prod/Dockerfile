# syntax=docker/dockerfile:1

FROM dunglas/frankenphp:php8.4-alpine AS base

ENV APP_ENV=prod \
    APP_DEBUG=0 \
    SERVER_NAME=:8000 \
    SERVER_ROOT=public/

RUN install-php-extensions apcu intl opcache pdo_pgsql zip

FROM base AS vendor

COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer

COPY composer.json composer.lock symfony.lock ./
RUN composer install --no-dev --no-scripts --no-autoloader --prefer-dist --no-progress --no-interaction

COPY . .
RUN composer dump-autoload --no-dev --classmap-authoritative \
    && php bin/console cache:warmup \
    && php bin/console assets:install public --no-interaction

FROM base AS runtime

RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"

COPY <<'INI' $PHP_INI_DIR/conf.d/app-prod.ini
expose_php = Off
memory_limit = 256M
realpath_cache_size = 4096K
realpath_cache_ttl = 600

opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 0
INI

COPY <<'SH' /usr/local/bin/app-entrypoint
#!/bin/sh
set -e

wait_for_database() {
    attempt=1
    until php bin/console dbal:run-sql "SELECT 1" >/dev/null 2>&1; do
        if [ "$attempt" -ge 30 ]; then
            echo "Database is not reachable, giving up." >&2
            return 1
        fi
        attempt=$((attempt + 1))
        sleep 2
    done
}

if [ "${1#-}" != "$1" ]; then
    wait_for_database
    php bin/console lexik:jwt:generate-keypair --skip-if-exists
    chmod 600 config/jwt/private.pem
    php bin/console doctrine:migrations:migrate --no-interaction --allow-no-migration
fi

exec docker-php-entrypoint "$@"
SH

RUN chmod +x /usr/local/bin/app-entrypoint

COPY --from=vendor --chown=www-data:www-data /app /app

RUN mkdir -p var/cache var/log config/jwt \
    && chown -R www-data:www-data var config/jwt /data /config

USER www-data

HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3 \
    CMD php -r 'exit(@fsockopen("127.0.0.1", 8000) ? 0 : 1);'

ENTRYPOINT ["app-entrypoint"]
CMD ["--config", "/etc/frankenphp/Caddyfile", "--adapter", "caddyfile"]
