FROM php:8.3-fpm

# Set php.ini file
RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"

# Install composer
COPY --from=composer /usr/bin/composer /usr/bin/composer

# Install Symfony CLI
RUN curl -1sLf 'https://dl.cloudsmith.io/public/symfony/stable/setup.deb.sh' | bash \
    && apt update && apt install -y \
    symfony-cli \
    libicu-dev libpq-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install APCU, OPCACHE, INTL, PDO
RUN pecl install apcu \
    && docker-php-ext-install opcache intl pdo_pgsql \
    && docker-php-ext-enable apcu