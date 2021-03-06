FROM debian:jessie
MAINTAINER MOHSEN@IPROPERTY
ENV DEBIAN_FRONTEND noninteractive

#Installing Nginx
ENV NGINX_VERSION 1.10.1-1~jessie

RUN apt-key adv --keyserver hkp://pgp.mit.edu:80 --recv-keys 573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62 \
 && echo "deb http://nginx.org/packages/debian/ jessie nginx" >> /etc/apt/sources.list \
 && apt-get update \
 && apt-get install --no-install-recommends --no-install-suggests -y \
    ca-certificates \
    nginx=${NGINX_VERSION} \
    nginx-module-xslt \
    nginx-module-geoip \
    nginx-module-image-filter \
    nginx-module-perl \
    nginx-module-njs \
    gettext-base \
 && rm -rf /var/lib/apt/lists/*


# forward request and error logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
 && ln -sf /dev/stderr /var/log/nginx/error.log

RUN sed -i "s/nginx;/root;/g" /etc/nginx/nginx.conf
RUN sed -i "s/worker_processes 1/worker_processes 2/g" /etc/nginx/nginx.conf


#Installing PHP

COPY php/docker-php-ext-* /usr/local/bin/
COPY docker-php-source /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-php-*

ENV PHPIZE_DEPS \
    autoconf \
    file \
    g++ \
    gcc \
    libc-dev \
    make \
    pkg-config \
    re2c

RUN apt-get update && apt-get install -y \
    $PHPIZE_DEPS \
    ca-certificates \
    curl \
    libedit2 \
    libsqlite3-0 \
    libxml2 \
    xz-utils \
    --no-install-recommends && rm -r /var/lib/apt/lists/*

ENV PHP_INI_DIR /usr/local/etc/php
RUN mkdir -p $PHP_INI_DIR/conf.d
ENV PHP_EXTRA_CONFIGURE_ARGS --enable-fpm --with-fpm-user=root --with-fpm-group=root
ENV PHP_VERSION 5.6.25
ENV PHP_FILENAME php-5.6.25.tar.xz
ENV PHP_SHA256 7535cd6e20040ccec4594cc386c6f15c3f2c88f24163294a31068cf7dfe7f644
ENV GPG_KEYS 6E4F6AB321FDC07F2C332E3AC2BF0BC433CFC8B3

RUN set -xe \
 && cd /usr/src \
 && curl -fSL "http://php.net/get/$PHP_FILENAME/from/this/mirror" -o php.tar.xz \
 && echo "$PHP_SHA256 *php.tar.xz" | sha256sum -c - \
 && curl -fSL "http://php.net/get/$PHP_FILENAME.asc/from/this/mirror" -o php.tar.xz.asc \
 && export GNUPGHOME="$(mktemp -d)" \
 && for key in $GPG_KEYS; do \
    gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
    done \
 && gpg --batch --verify php.tar.xz.asc php.tar.xz \
 && rm -r "$GNUPGHOME"

RUN set -xe \
 && buildDeps=" \
    $PHP_EXTRA_BUILD_DEPS \
    libcurl4-openssl-dev \
    libedit-dev \
    libsqlite3-dev \
    libssl-dev \
    libxml2-dev \
    " \
 && apt-get update && apt-get install -y $buildDeps --no-install-recommends && rm -rf /var/lib/apt/lists/* \
 && docker-php-source extract \
 && cd /usr/src/php \
 && ./configure \
    --with-config-file-path="$PHP_INI_DIR" \
    --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
    --disable-cgi \
    --enable-ftp \
    --enable-mbstring \
    --enable-mysqlnd \
    --enable-json \ 
    --with-curl \
    --with-libedit \
    --with-openssl \
    --with-zlib \
    $PHP_EXTRA_CONFIGURE_ARGS \
 && make -j"$(nproc)" \
 && make install \
 && { find /usr/local/bin /usr/local/sbin -type f -executable -exec strip --strip-all '{}' + || true; } \
 && make clean \
 && docker-php-source delete \
 && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false $buildDeps

WORKDIR /var/www/html

RUN set -ex \
 && cd /usr/local/etc \
 && if [ -d php-fpm.d ]; then \
     # for some reason, upstream's php-fpm.conf.default has "include=NONE/etc/php-fpm.d/*.conf"
     sed 's!=NONE/!=!g' php-fpm.conf.default | tee php-fpm.conf > /dev/null; \
     cp php-fpm.d/www.conf.default php-fpm.d/www.conf; \
    else \
     # PHP 5.x don't use "include=" by default, so we'll create our own simple config that mimics PHP 7+ for consistency
     mkdir php-fpm.d; \
     cp php-fpm.conf.default php-fpm.d/www.conf; \
     { \
             echo '[global]'; \
             echo 'include=etc/php-fpm.d/*.conf'; \
     } | tee php-fpm.conf; \
    fi \
 && { \
    echo '[global]'; \
    echo 'error_log = /proc/self/fd/2'; \
    echo; \
    echo '[www]'; \
    echo '; if we send this to /proc/self/fd/1, it never appears'; \
    echo 'access.log = /proc/self/fd/2'; \
    echo; \
    echo 'clear_env = no'; \
    echo; \
    echo '; Ensure worker stdout and stderr are sent to the main error log.'; \
    echo 'catch_workers_output = yes'; \
    } | tee php-fpm.d/docker.conf \
 && { \
    echo '[global]'; \
    echo 'daemonize = no'; \
    echo; \
    echo '[www]'; \
    echo 'listen = [::]:9000'; \
    } | tee php-fpm.d/default-docker.conf
