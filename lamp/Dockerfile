# ============================================================
# ベースイメージ: 公式 PHP 8.4 + Apache
# ============================================================
FROM php:8.4-apache

# ────────────────────────────────────────
# システムライブラリ + PHP拡張インストール
# ────────────────────────────────────────
RUN apt-get update && apt-get install -y \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libzip-dev \
    libonig-dev \
    libxml2-dev \
    libicu-dev \
    zip \
    unzip \
    curl \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install \
        gd \
        pdo \
        pdo_mysql \
        mysqli \
        mbstring \
        zip \
        xml \
        opcache \
        intl \
        bcmath \
        exif \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# ────────────────────────────────────────
# Apache モジュール有効化
# ────────────────────────────────────────
# mod_rewrite: .htaccess のURL書き換えを有効化
# mod_headers: レスポンスヘッダー制御を有効化
RUN a2enmod rewrite headers

# ────────────────────────────────────────
# php.ini 配置
# ────────────────────────────────────────
COPY php.ini /usr/local/etc/php/php.ini

# ────────────────────────────────────────
# ドキュメントルート
# ────────────────────────────────────────
WORKDIR /var/www/html

EXPOSE 80
