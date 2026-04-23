# ============================================================
# ベースイメージ
# 公式 PHP + Apache イメージを使用
# ============================================================
FROM php:8.4-apache

# ============================================================
# ビルド引数
# ホスト側ユーザーの UID / GID を受け取る
# 例:
#   UID=1000
#   GID=1000
# ============================================================
ARG UID=1000
ARG GID=1000

# ============================================================
# www-data の UID / GID をホスト側に合わせる
#
# 目的:
# - bind mount 環境で、コンテナ経由で作成されたファイルの
#   所有者ズレを減らす
# - ホスト側で編集・削除しやすくする
#
# 注意:
# - これは「今後そのユーザーで作られるファイル」に効く
# - 既存ファイルの所有者は自動では変わらない
# ============================================================
RUN usermod -u ${UID} www-data \
    && groupmod -g ${GID} www-data

# ============================================================
# 必要パッケージ / PHP拡張インストール
# ============================================================
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
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# ============================================================
# Apache モジュール有効化
# rewrite : .htaccess の URL 書き換え用
# headers : セキュリティヘッダー等の制御用
# ============================================================
RUN a2enmod rewrite headers

# ============================================================
# OS 側のタイムゾーン設定
#
# 目的:
# - date コマンド
# - Apache ログ
# - OSレベルの時刻表示
# を Asia/Tokyo に合わせる
#
# PHP 側の timezone は php.ini で設定済み
# ============================================================
ENV TZ=Asia/Tokyo
RUN ln -snf /usr/share/zoneinfo/${TZ} /etc/localtime \
    && echo ${TZ} > /etc/timezone

# ============================================================
# PHP 設定ファイル配置
# php.ini 側に date.timezone = Asia/Tokyo を設定済み
# ============================================================
COPY php.ini /usr/local/etc/php/php.ini

# ============================================================
# ドキュメントルート
# ============================================================
WORKDIR /var/www/html

# ============================================================
# 画像アップロード先ディレクトリ作成
#
# アプリの書き込み先:
# - /var/www/html/images/icon
# - /var/www/html/images/thumbnail
#
# 注意:
# - Dockerfile 内の chown/chmod はイメージ内には効く
# - ただし bind mount 環境では、起動後はホスト側の
#   実ディレクトリが優先される
# - そのため、開発環境ではホスト側の所有者/権限も
#   整っている必要がある
# ============================================================
RUN mkdir -p \
    /var/www/html/images/icon \
    /var/www/html/images/thumbnail \
    && chown -R www-data:www-data /var/www/html/images \
    && chmod -R 775 /var/www/html/images

# ============================================================
# Apache 公開ポート
# ============================================================
EXPOSE 80