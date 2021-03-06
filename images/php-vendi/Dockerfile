FROM alpine:3.6

LABEL maintainer="Chris Haas <chris@vendiadvertising.com>"

# persistent / runtime deps
ENV PHPIZE_DEPS \
        autoconf \
        dpkg-dev dpkg \
        file \
        g++ \
        gcc \
        libc-dev \
        make \
        pcre-dev \
        git \
        pkgconf \
        re2c

ENV PHP_INI_DIR /usr/local/etc/php

ENV PHP_EXTRA_CONFIGURE_ARGS --enable-fpm --with-fpm-user=www-data --with-fpm-group=www-data

ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2"
ENV PHP_CPPFLAGS="$PHP_CFLAGS"
ENV PHP_LDFLAGS="-Wl,-O1 -Wl,--hash-style=both -pie"

ENV GPG_KEYS A917B1ECDA84AEC2B568FED6F50ABC807BD5DCD0 528995BFEDFBA7191D46839EF9BA0ADA31CBD89E

ENV PHP_VERSION 7.1.7
ENV PHP_URL="https://secure.php.net/get/php-7.1.7.tar.xz/from/this/mirror" PHP_ASC_URL="https://secure.php.net/get/php-7.1.7.tar.xz.asc/from/this/mirror"
ENV PHP_SHA256="0d42089729be7b2bb0308cbe189c2782f9cb4b07078c8a235495be5874fff729" PHP_MD5=""

COPY docker-php-source /usr/local/bin/

COPY docker-php-ext-* docker-php-entrypoint /usr/local/bin/

RUN apk add --no-cache --virtual .persistent-deps \
        ca-certificates \
        curl \
        tar \
        xz \
    && set -x \
    && addgroup -g 82 -S www-data \
    && adduser -u 82 -D -S -G www-data www-data \
    && rm -rf /var/lib/apt/lists/* \
    && mkdir -p $PHP_INI_DIR/conf.d \
    && set -xe; \
    \
    apk add --no-cache --virtual .fetch-deps \
        gnupg \
        libressl \
        libressl-dev \
    ; \
    \
    mkdir -p /usr/src; \
    cd /usr/src; \
    \
    wget -O php.tar.xz "$PHP_URL"; \
    \
    if [ -n "$PHP_SHA256" ]; then \
        echo "$PHP_SHA256 *php.tar.xz" | sha256sum -c -; \
    fi; \
    if [ -n "$PHP_MD5" ]; then \
        echo "$PHP_MD5 *php.tar.xz" | md5sum -c -; \
    fi; \
    \
    if [ -n "$PHP_ASC_URL" ]; then \
        wget -O php.tar.xz.asc "$PHP_ASC_URL"; \
        export GNUPGHOME="$(mktemp -d)"; \
        for key in $GPG_KEYS; do \
            gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
        done; \
        gpg --batch --verify php.tar.xz.asc php.tar.xz; \
        rm -rf "$GNUPGHOME"; \
    fi; \
    \
    apk del .fetch-deps \

    && set -xe \
    && apk add --no-cache --virtual .build-deps \
        $PHPIZE_DEPS \
        coreutils \
        curl-dev \
        libedit-dev \
        libxml2-dev \
        libressl \
        libressl-dev \
        sqlite-dev \
    \
    && export CFLAGS="$PHP_CFLAGS" \
        CPPFLAGS="$PHP_CPPFLAGS" \
        LDFLAGS="$PHP_LDFLAGS" \
    && docker-php-source extract \
    && cd /usr/src/php \
    && gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
    && ./configure \
        --build="$gnuArch" \
        --with-config-file-path="$PHP_INI_DIR" \
        --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
        \
        --disable-cgi \
        \
# --enable-ftp is included here because ftp_ssl_connect() needs ftp to be compiled statically (see https://github.com/docker-library/php/issues/236)
        --enable-ftp \
# --enable-mbstring is included here because otherwise there's no way to get pecl to use it properly (see https://github.com/docker-library/php/issues/195)
        --enable-mbstring \
# --enable-mysqlnd is included here because it's harder to compile after the fact than extensions are (since it's a plugin for several extensions, not an extension in itself)
        --enable-mysqlnd \
        \
        --with-curl \
        --with-libedit \
        --with-openssl \
        --with-zlib \
        \
# bundled pcre is too old for s390x (which isn't exactly a good sign)
# /usr/src/php/ext/pcre/pcrelib/pcre_jit_compile.c:65:2: error: #error Unsupported architecture
        --with-pcre-regex=/usr \
        \
        $PHP_EXTRA_CONFIGURE_ARGS \
    && make -j "$(nproc)" \
    && make install \
    && { find /usr/local/bin /usr/local/sbin -type f -perm +0111 -exec strip --strip-all '{}' + || true; } \
    && make clean \
    && cd / \
    && git clone https://github.com/phpredis/phpredis.git \
    && cd phpredis \
    && git checkout php7 \
    && phpize \
    && ./configure \
    && make \
    && make install \
    && cd .. \
    && rm -rf phpredis \
    && docker-php-source delete \
    \
    && runDeps="$( \
        scanelf --needed --nobanner --recursive /usr/local \
            | awk '{ gsub(/,/, "\nso:", $2); print "so:" $2 }' \
            | sort -u \
            | xargs -r apk info --installed \
            | sort -u \
    )" \
    && apk add --no-cache --virtual .php-rundeps $runDeps \
    \
    && apk del .build-deps \
    \
# https://github.com/docker-library/php/issues/443
    && pecl update-channels \
    && rm -rf /tmp/pear ~/.pearrc \

    && mkdir -p /usr/local/etc/php-fpm.d \
    && mkdir -p /run/php/ \

    && set -ex \
    && { \
        echo '[global]'; \
        echo 'pid = /run/php/php7.1-fpm.pid'; \
        echo 'error_log = /proc/self/fd/2'; \
        echo 'daemonize = no'; \
        echo; \
        echo '[www]'; \
        echo 'catch_workers_output = yes'; \
        echo 'access.log = /proc/self/fd/2'; \
        echo 'user = www-data'; \
        echo 'group = www-data'; \
        echo 'listen = [::]:9000'; \
        echo 'pm = dynamic'; \
        echo 'pm.max_children = 5'; \
        echo 'pm.start_servers = 2'; \
        echo 'pm.min_spare_servers = 1'; \
        echo 'pm.max_spare_servers = 3'; \
    } | tee /usr/local/etc/php-fpm.d/vendi.conf

EXPOSE 9000

# CMD ["php-fpm"]
CMD docker-php-entrypoint --fpm-config /usr/local/etc/php-fpm.d/vendi.conf
