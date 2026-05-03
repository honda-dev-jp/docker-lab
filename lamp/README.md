# LAMP Docker 環境

## 構成

| サービス    | イメージ              | バージョン |
|------------|----------------------|-----------|
| PHP        | php (Docker公式)      | 8.4       |
| Apache     | php:8.4-apache 内包  | 安定版     |
| MySQL      | mysql (Docker公式)    | 8.4 LTS   |
| phpMyAdmin | phpmyadmin (公式)     | latest    |

## ディレクトリ構成

```
~/projects/
├── docker-lab/
│   └── lamp/                    # Docker環境（使いまわし可能）
│       ├── .env                 # 機密情報（Gitに含めない）
│       ├── .env.example         # .env テンプレート（Gitで管理）
│       ├── .gitignore
│       ├── Dockerfile
│       ├── docker-compose.yml
│       ├── php.ini
│       ├── apache/
│       │   └── vhost.conf
│       └── mysql/
│           ├── conf.d/
│           │   └── my.cnf       # MySQLチューニング設定
│           └── init/            # 初期SQL・シーダー（.sqlファイル）
└── review-app/                  # PHPアプリ本体（Gitリポジトリ）
```

> **補足：** `src/` はDocker環境に含めず、アプリリポジトリを直接bind mountする構成です。
> `docker-compose.yml` の volumes でアプリのパスを指定してください。
>
> ```yaml
> volumes:
>   - ../../review-app:/var/www/html
> ```

## 初回セットアップ

```bash
# 1. .env を作成してパスワードを設定する
cp .env.example .env
nano .env

# 2. ビルド＆起動
docker compose up -d --build

# 3. 起動確認
docker compose ps
```

## アクセス先

| サービス       | URL                   | 備考               |
|--------------|-----------------------|--------------------|
| Web (Apache) | http://localhost      | localhost のみ公開 |
| phpMyAdmin   | http://localhost:8081 | localhost のみ公開 |
| MySQL        | コンテナ間通信のみ     | 外部ポート非公開   |

## バックアップからのインポート

```bash
# CLI から直接流し込む（推奨）
docker exec -i lamp_db mysql -u root -p "${MYSQL_ROOT_PASSWORD}" app_db < backup.sql

# phpMyAdmin からインポートする場合
# http://localhost:8081 → 対象DB → インポート（上限256MB）
```

## ログ確認

```bash
# 全サービス
docker compose logs -f

# MySQL スロークエリログ
docker exec lamp_db tail -f /var/log/mysql/slow.log

# PHPエラーログ
docker exec lamp_web tail -f /var/log/apache2/php_errors.log
```

## .htaccess について

`AllowOverride All` と `mod_rewrite` が有効化済みです。アプリの `.htaccess` を置くだけで機能します。

## 停止・再開・削除

```bash
# 停止（データ保持・再開可能）
docker compose stop

# 再開
docker compose start

# 停止＋コンテナ削除（データ保持）
docker compose down

# 停止＋コンテナ＋データ全削除（クリーンな再構築時）
docker compose down -v
```

## ⚠️ ハマりポイント

### `version:` は不要（obsolete警告）
Docker Compose V2以降、`version:` の記載は不要です。記載すると毎回WARNが表示されます。

### `skip-character-set-client-handshake` は MySQL 8.4 で廃止済み
`my.cnf` に記載するとMySQLが起動直後にクラッシュして無限再起動ループに陥ります。該当行を削除してください。

### `output_buffering` はデフォルトで無効
DockerのPHPは `output_buffering` がデフォルト無効です。`header()` の前に出力があるとエラーになります。
`php.ini` に以下を記述してからビルドしてください。

```ini
[Output]
output_buffering = On
```

### DB接続ホスト名は `localhost` ではなく `db`
Dockerではコンテナ間の接続にサービス名を使います。アプリの設定ファイルのホスト名を `db` に変更してください。

```php
// NG
'host' => 'localhost'
// OK
'host' => 'db'
```

### `.htaccess` のリファラ制限に `localhost` を追加
本番URLのみリファラを許可している場合、`localhost` からの画像アクセスが403になります。
`OR` 条件で `localhost` を追加してください。

```apache
RewriteCond %{HTTP_REFERER} ^https://example.com [NC,OR]
RewriteCond %{HTTP_REFERER} ^http://localhost [NC]
RewriteRule ^ - [L]
```

### `.env` のパスワード特殊文字はシングルクォート必須
`$` などの特殊文字が含まれる場合はシングルクォートで囲んでください。

```env
# NG
MYSQL_PASSWORD="$を含むパスワード"
# OK
MYSQL_PASSWORD='$を含むパスワード'
```

## セキュリティメモ

- `.env` は `.gitignore` に追加済みです。Git にコミットしないでください
- DB ポートは外部に公開していません（コンテナ間通信のみ）
- Web・phpMyAdmin は `127.0.0.1` バインドでローカル限定公開です
- 本番環境では `.env` のパスワードを必ず強力なものに変更してください
