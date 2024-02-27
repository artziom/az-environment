FROM php:8.3-fpm

# Set php.ini file
RUN mv "$PHP_INI_DIR/php.ini-development" "$PHP_INI_DIR/php.ini"

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

# Install xdebug
RUN pecl install xdebug \
	&& docker-php-ext-enable xdebug \
    && echo "xdebug.client_host=host.docker.internal" >> $PHP_INI_DIR/php.ini \
    && echo "xdebug.mode=develop,debug" >> $PHP_INI_DIR/php.ini \
    && echo "xdebug.start_with_request=trigger" >> $PHP_INI_DIR/php.ini