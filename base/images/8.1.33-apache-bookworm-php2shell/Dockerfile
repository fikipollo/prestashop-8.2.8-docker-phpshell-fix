FROM php:8.1.33-apache-bookworm

ARG PHP2SHELL_COMMIT=1d7dec178a

LABEL org.opencontainers.image.description="PHP 8.1.33 Apache Bookworm patched for PHP2Shell"
LABEL org.opencontainers.image.php2shell.patch="${PHP2SHELL_COMMIT}"

ENV PHP2SHELL_PATCH="${PHP2SHELL_COMMIT}"

RUN set -eux; \
    \
    savedAptMark="$(apt-mark showmanual)"; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        apache2-dev \
        libargon2-dev \
        libcurl4-openssl-dev \
        libonig-dev \
        libreadline-dev \
        libsodium-dev \
        libsqlite3-dev \
        libssl-dev \
        libxml2-dev \
        zlib1g-dev \
        patch \
    ; \
    \
    docker-php-source extract; \
    cd /usr/src/php; \
    \
    curl -fsSL \
        "https://github.com/php/php-src/commit/${PHP2SHELL_COMMIT}.patch" \
        -o /tmp/php2shell.patch; \
    \
    patch -p1 < /tmp/php2shell.patch; \
    \
    export \
        CFLAGS="$PHP_CFLAGS" \
        CPPFLAGS="$PHP_CPPFLAGS" \
        LDFLAGS="$PHP_LDFLAGS" \
        PHP_BUILD_PROVIDER='https://github.com/docker-library/php + PHP2Shell backport' \
        PHP_UNAME='Linux - Docker'; \
    \
    gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
    debMultiarch="$(dpkg-architecture --query DEB_BUILD_MULTIARCH)"; \
    \
    if [ ! -d /usr/include/curl ] && [ ! -e /usr/local/include/curl ] && [ ! -L /usr/local/include/curl ]; then \
        ln -sT "/usr/include/$debMultiarch/curl" /usr/local/include/curl; \
    fi; \
    \
    ./configure \
        --build="$gnuArch" \
        --with-config-file-path="$PHP_INI_DIR" \
        --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
        --enable-option-checking=fatal \
        --with-mhash \
        --with-pic \
        --enable-ftp \
        --enable-mbstring \
        --enable-mysqlnd \
        --with-password-argon2 \
        --with-sodium=shared \
        --with-pdo-sqlite=/usr \
        --with-sqlite3=/usr \
        --with-curl \
        --with-iconv \
        --with-openssl \
        --with-readline \
        --with-zlib \
        --disable-phpdbg \
        --with-pear \
        $(test "$gnuArch" = 'riscv64-linux-gnu' && echo '--without-pcre-jit') \
        --with-libdir="lib/$debMultiarch" \
        --disable-cgi \
        --with-apxs2 \
    ; \
    \
    make -j "$(nproc)"; \
    find -type f -name '*.a' -delete; \
    make install; \
    \
    find /usr/local -type f -perm '/0111' \
        -exec sh -euxc 'strip --strip-all "$@" || :' -- '{}' +; \
    \
    make clean; \
    \
    cd /; \
    docker-php-source delete; \
    rm -f /tmp/php2shell.patch; \
    \
    apt-mark auto '.*' > /dev/null; \
    [ -z "$savedAptMark" ] || apt-mark manual $savedAptMark; \
    \
    find /usr/local -type f -executable -exec ldd '{}' ';' \
        | awk '/=>/ { \
            so = $(NF-1); \
            if (index(so, "/usr/local/") == 1) next; \
            gsub("^/(usr/)?", "", so); \
            printf "*%s\n", so \
        }' \
        | sort -u \
        | xargs -rt dpkg-query --search \
        | awk 'sub(":$", "", $1) { print $1 }' \
        | sort -u \
        | xargs -r apt-mark manual \
    ; \
    \
    apt-get purge -y --auto-remove \
        -o APT::AutoRemove::RecommendsImportant=false \
        patch \
    ; \
    rm -rf /var/lib/apt/lists/*; \
    \
    php --version; \
    apache2ctl -M | grep php
