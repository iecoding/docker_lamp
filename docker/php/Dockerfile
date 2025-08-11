FROM php:8.2-apache



# System dependencies
# Reemplazar fuentes de paquetes por HTTPS
RUN echo "deb https://deb.debian.org/debian bookworm main" > /etc/apt/sources.list && \
    echo "deb https://security.debian.org/debian-security bookworm-security main" >> /etc/apt/sources.list && \
    echo "deb https://deb.debian.org/debian bookworm-updates main" >> /etc/apt/sources.list && \
    apt-get update && \
    apt-get install -y --fix-missing \
    libfreetype6-dev libjpeg62-turbo-dev libpng-dev libwebp-dev libxpm-dev \
    libonig-dev libxml2-dev libzip-dev libicu-dev libmcrypt-dev libmagickwand-dev \
    libssl-dev libz-dev libxslt1-dev unzip curl git default-mysql-client iputils-ping \
    && apt-get clean && rm -rf /var/lib/apt/lists/*
    
RUN docker-php-ext-configure gd --with-jpeg --with-webp --with-xpm \
    && docker-php-ext-install -j$(nproc) \
        bcmath exif fileinfo gd iconv intl mbstring mysqli opcache \
        pdo pdo_mysql zip

# Instala extensiones adicionales vía PECL
RUN pecl install imagick mcrypt \
    && docker-php-ext-enable imagick mcrypt \
    && rm -rf /tmp/pear

# Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/bin/
RUN install-php-extensions xdebug

RUN docker-php-ext-install -j$(nproc) dom xml simplexml

COPY ./docker/apache/apache.conf /etc/apache2/sites-available/apache.conf
RUN a2ensite apache.conf && a2dissite 000-default.conf
RUN a2enmod rewrite ssl

# PHP Configurations and cert
COPY ./docker/php/php.ini /usr/local/etc/php/
COPY ./docker/apache/certs/server.crt /etc/apache2/ssl/server.crt
COPY ./docker/apache/certs/server.key /etc/apache2/ssl/server.key

# Application code
COPY ./app /var/www/html
RUN git config --global --add safe.directory /var/www/html
WORKDIR /var/www/html

# Permissions
RUN chown -R www-data:www-data /var/www/html && \
    find /var/www/html -type d -exec chmod 755 {} \; && \
    find /var/www/html -type f -exec chmod 644 {} \;

# Entrypoint
RUN printf '#!/bin/bash\n\
set -e\n\
set -x\n\
echo "Waiting a few seconds before verifying database connection..."\n\
sleep 10\n\
echo "Waiting for database to be ready..."\n\
MAX_TRIES=30\n\
TRIES=0\n\
until mysqladmin ping -h db -u webserver_user -webserver_password --silent >/dev/null 2>&1; do\n\
  if [ "$TRIES" -ge "$MAX_TRIES" ]; then\n\
    echo "❌ Cannot connect to database after many attempts. Aborting."\n\
    exit 1\n\
  fi\n\
  echo "⏳ Attempt  $((TRIES+1)) of $MAX_TRIES: Database is not available yet..."\n\
  TRIES=$((TRIES+1))\n\
  sleep 3\n\
done\n\
echo "✅ Database available."\n\
echo "🎉 Everything is ready. Running Apache..."\n\
exec apache2-foreground\n' > /usr/local/bin/entrypoint.sh && \
chmod +x /usr/local/bin/entrypoint.sh