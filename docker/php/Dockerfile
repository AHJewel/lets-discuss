FROM php:8.4-fpm-alpine

COPY --from=mlocati/php-extension-installer:latest /usr/bin/install-php-extensions /usr/local/bin/
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

RUN apk add --no-cache git unzip \
    && install-php-extensions pdo_pgsql pgsql redis gd intl zip bcmath pcntl opcache exif yaml xdebug

COPY conf.d/ /usr/local/etc/php/conf.d/

# Align www-data with the host user so the bind-mounted storage/ is writable
# by every PHP container (api, queue, scheduler, reverb) and by the host.
ARG HOST_UID=1000
ARG HOST_GID=1000
RUN apk add --no-cache shadow \
    && groupmod -o -g ${HOST_GID} www-data \
    && usermod -o -u ${HOST_UID} -g ${HOST_GID} www-data \
    && mkdir -p /tmp/composer && chown www-data:www-data /tmp/composer

# Xdebug is installed but idle; enable per run with XDEBUG_MODE=debug,coverage
ENV XDEBUG_MODE=off \
    COMPOSER_HOME=/tmp/composer

WORKDIR /var/www/html
