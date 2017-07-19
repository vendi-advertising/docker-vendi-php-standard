FROM ubuntu:16.04
LABEL maintainer="Chris Haas <chris@vendiadvertising.com>"

# Add PHP 7 repo
RUN apt-get update \
&& apt-get -y install software-properties-common \
&& LC_ALL=C.UTF-8 add-apt-repository ppa:ondrej/php

# Set this to the version of PHP you want
ENV VENDI_PHP_VERSION_BASE 7.1

# These are the common packages that we install
ENV VENDI_PHP_VERSION_PACKAGES \
        php${VENDI_PHP_VERSION_BASE} \
        php${VENDI_PHP_VERSION_BASE}-bcmath \
        php${VENDI_PHP_VERSION_BASE}-cli \
        php${VENDI_PHP_VERSION_BASE}-common \
        php${VENDI_PHP_VERSION_BASE}-curl \
        php${VENDI_PHP_VERSION_BASE}-dev \
        php${VENDI_PHP_VERSION_BASE}-fpm \
        php${VENDI_PHP_VERSION_BASE}-gd \
        php${VENDI_PHP_VERSION_BASE}-intl \
        php${VENDI_PHP_VERSION_BASE}-json \
        php${VENDI_PHP_VERSION_BASE}-mbstring \
        php${VENDI_PHP_VERSION_BASE}-mcrypt \
        php${VENDI_PHP_VERSION_BASE}-mysql \
        php${VENDI_PHP_VERSION_BASE}-opcache \
        php${VENDI_PHP_VERSION_BASE}-readline \
        php${VENDI_PHP_VERSION_BASE}-xml \
        php${VENDI_PHP_VERSION_BASE}-zip

# These packages are not installed:
# php7.1-bz2        php7.1-cgi      php7.1-dba
# php7.1-enchant    php7.1-gmp      php7.1-imap
# php7.1-interbase  php7.1-ldap     php7.1-odbc
# php7.1-pgsql      php7.1-phpdbg   php7.1-pspell
# php7.1-recode     php7.1-snmp     php7.1-soap
# php7.1-sqlite3    php7.1-sybase   php7.1-tidy
# php7.1-xmlrpc     php7.1-xsl

# One giant installer command because we need to purge after the run
# in order to reduce the layer size
RUN buildDeps='wget unzip git make' \
    && set -x \
    && apt-get update \
    && apt-get install -y --no-install-recommends $buildDeps \
        net-tools \
        php-common \
        $VENDI_PHP_VERSION_PACKAGES \
    && rm -rf /var/lib/apt/lists/* \
    && git clone https://github.com/phpredis/phpredis.git \
    && cd phpredis \
    && git checkout php7 \
    && phpize \
    && ./configure \
    && make \
    && make install \
    && cd .. \
    && rm -rf phpredis \
    && echo "extension=redis.so" > /etc/php/$VENDI_PHP_VERSION_BASE/mods-available/redis.ini \
    && ln -sf /etc/php/$VENDI_PHP_VERSION_BASE/mods-available/redis.ini /etc/php/$VENDI_PHP_VERSION_BASE/fpm/conf.d/20-redis.ini \
    && ln -sf /etc/php/$VENDI_PHP_VERSION_BASE/mods-available/redis.ini /etc/php/$VENDI_PHP_VERSION_BASE/cli/conf.d/20-redis.ini \
    && service php$VENDI_PHP_VERSION_BASE-fpm restart \
    && apt-get purge -y --auto-remove $buildDeps

RUN mkdir -p /usr/local/etc/php-fpm.d
RUN mkdir -p /run/php/

RUN set -ex \
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

CMD exec /usr/sbin/php-fpm7.1 --fpm-config /usr/local/etc/php-fpm.d/vendi.conf
