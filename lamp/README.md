# LAMP Docker 環境

PHP 8.4 + Apache + MySQL 8.4 + phpMyAdmin を Docker Compose で起動する開発環境です。
アプリ本体は別リポジトリの `review-app` を bind mount します。

通常環境では Docker 公式の MySQL イメージを使用し、古い CPU を搭載する LIFEBOOK（Ubuntu）ではローカル専用の `compose.override.yaml` で構成を切り替えます。

## 構成

| サービス | イメージ | バージョン・備考 |
|---|---|---|
| PHP | `php:8.4.27-apache`（Docker 公式） | PHP 8.4.27（パッチバージョン指定） |
| Apache | `php:8.4.27-apache` に内包 | Apache 2.4 系 |
| MySQL | `mysql:8.4`（Docker 公式） | 8.4 LTS（標準構成） |
| MySQL（LIFEBOOK） | `ubuntu/mysql:8.4-24.04_edge` | ローカル override で切り替え。digest 固定 |
| phpMyAdmin | `phpmyadmin:latest` | phpMyAdmin 5.2.3 / PHP 8.3.36（2026-10-22 検証済み）。ローカルからのみアクセス可能 |

> **検証済みの環境：** LIFEBOOK Ubuntu 24.04.4 LTS / Intel Pentium P6200。2026-10-22 に Web コンテナを PHP **8.4.26 → 8.4.27**、phpMyAdmin コンテナを PHP **8.3.33 → 8.3.36** に更新しました。Web のゲスト閲覧画面・PHP 構文チェック、および phpMyAdmin の管理画面・DB 接続を確認済みです。`Dockerfile` は `php:8.4.27-apache` を明示指定しているため、次回のパッチ更新には再変更が必要です。一方、`phpmyadmin:latest` は可変タグであり、今後のイメージ更新は別途取得と動作確認が必要です。

## ディレクトリ構成

```text
~/projects/
├── docker-lab/
│   └── lamp/                         # Docker 環境
│       ├── .env                      # 機密情報（Git に含めない）
│       ├── .env.example              # .env テンプレート（Git 管理）
│       ├── .gitignore
│       ├── compose.override.yaml     # ホスト固有の設定（Git に含めない）
│       ├── Dockerfile
│       ├── SECURITY_HISTORY.md       # LAMP の脆弱性対応履歴
│       ├── docker-compose.yml        # 共通構成（Git 管理）
│       ├── php.ini
│       ├── apache/
│       │   └── vhost.conf
│       └── mysql/
│           ├── conf.d/
│           │   └── my.cnf            # MySQL チューニング設定
│           └── init/                 # 初期 SQL・シーダー
├── review-app/                       # スクラッチ PHP 映画レビューアプリ（別 Git リポジトリ）
└── review-app-private/               # 非公開設定（Git リポジトリ外）
    └── portfolio/
        ├── env.php                   # 実際の値は公開・共有しない
        └── database.php              # 実際の値は公開・共有しない
```

`compose.override.yaml` と `review-app-private/` は LIFEBOOK で使う構成例です。ほかのホストでは必要に応じて構成してください。

`src/` を Docker 環境内に複製せず、アプリリポジトリを直接 bind mount します。共通の `docker-compose.yml` には以下が設定されています。

```yaml
volumes:
  - ../../review-app:/var/www/html
```

アプリの公開ディレクトリは `/var/www/html` です。実設定ファイルはその外側の `/var/www/private_config/portfolio` にマウントします。

## 初回セットアップ

以下のコマンドは、特記がない限り `~/projects/docker-lab/lamp` で実行します。

### 1. 環境変数の設定

```bash
cd ~/projects/docker-lab/lamp
cp .env.example .env
code .env

# ホスト側 UID / GID を確認する
id -u
id -g
```

`.env` のパスワードを開発環境用に設定し、`UID`・`GID` が実際のユーザーと一致していることを確認してください。テンプレートのパスワードをそのまま使わないでください。`.env` は Git に登録しません。

### 2. LIFEBOOK 固有の設定（該当する場合のみ）

LIFEBOOK（Pentium P6200）では、MySQL の CPU 互換性への対応として Ubuntu 版 MySQL 8.4 イメージを使用しました。また、スクラッチアプリの非公開設定ファイルを read-only でコンテナにマウントします。

**事前準備：** `review-app-private/portfolio/` に、権限のある安全なバックアップから `env.php` と `database.php` を配置してください。実設定値を README・Issue・コミットに記載せず、ファイルの内容もターミナルへ表示しないでください。

```bash
# 非公開設定ディレクトリの作成（未作成の場合）
mkdir -p ~/projects/review-app-private/portfolio

# 非公開設定ファイルの配置後に実行
chmod 700 ~/projects/review-app-private ~/projects/review-app-private/portfolio
chmod 600 ~/projects/review-app-private/portfolio/{env.php,database.php}
```

`lamp/compose.override.yaml` を作成し、以下の構成を指定します。これは **ローカル専用ファイル**であり、Git にコミットしません。

```yaml
services:
  web:
    volumes:
      - ../../review-app-private/portfolio:/var/www/private_config/portfolio:ro

  db:
    image: ubuntu/mysql:8.4-24.04_edge@sha256:18aabb354db43e4c0200006686be1b53454dce9249ccc8240ecdcf311fc5d3e4
```

この override は共通 `docker-compose.yml` にマージされます。`web.volumes` の既存マウント（アプリ・Apache 設定）は維持し、非公開設定のマウントを追加する構成です。

**注意：** 上記の設定ファイルはホスト上で UID/GID `1000:1000` 所有の状態で動作確認しました。所有者・PHP 実行ユーザーが異なる場合、権限を確認してください。権限緩和のために `chmod 777` を使用しないでください。

### 3. ビルド・起動・確認

```bash
# 構成が妥当か確認（展開後の設定値を表示しない）
docker compose config --quiet

# ビルド＆起動
docker compose up -d --build

# 起動状態の確認
docker compose ps

# Web コンテナの PHP バージョンを確認
docker compose exec -T web php -v
```

LIFEBOOK では、`lamp_db` が healthy、`lamp_web`・`lamp_phpmyadmin` が起動することを確認済みです。更新済みイメージを確実に利用するには、必要な `pull` / `build --pull` とコンテナの再作成・バージョン確認を行ってください。今回の作業手順は [SECURITY_HISTORY.md](SECURITY_HISTORY.md) に記録しています。

## アクセス先

| サービス | URL | 備考 |
|---|---|---|
| Web (Apache) | http://localhost/ | `127.0.0.1:80` バインド |
| phpMyAdmin | http://localhost:8081/ | `127.0.0.1:8081` バインド |
| MySQL | コンテナ間通信のみ | ホスト側にポートを公開しない |

**ローカル動作確認は `http://localhost/` を使用してください。** アプリの `.htaccess` の条件によっては `http://127.0.0.1/` が HTTPS にリダイレクトされます。LAMP の問題と混同しないよう、HTTP ステータスと `Location` ヘッダーを確認してください。

## バックアップからのインポート

### データベース（XServer の SQL バックアップ）

LIFEBOOK では phpMyAdmin から SQL バックアップをインポートし、アプリのゲスト作品一覧が表示されることを確認しました。

1. `http://localhost:8081/` を開く。
2. 対象のローカルデータベースを選択する（標準例：`app_db`。実際の `.env` 設定を優先）。
3. ［インポート］から SQL バックアップを指定する。
4. インポート後にテーブル・レコードの存在とアプリ表示を確認する。

phpMyAdmin のアップロード上限は Compose 側で `256M` に設定されています。実際にインポートできるサイズは PHP 設定やタイムアウトなどにも依存します。

CLI からインポートする場合は、標準入力を使うため `docker compose exec -T` を指定します。以下は MySQL コンテナ側の環境変数を利用する例です（`MYSQL_PWD` はコンテナ内プロセスの環境変数として扱われるため、信頼できるローカル環境でのみ使用）。

```bash
docker compose exec -T db sh -c \
  'MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -u root "$MYSQL_DATABASE"' \
  < /path/to/backup.sql
```

### 画像ファイル（ポスター等）

DB の復元だけでは画像ファイルは戻りません。スクラッチアプリでは保存ファイル名がランダムなため、**DB の参照値と実ファイル名を一致させる**必要があります。

- 適切に権限を持つバックアップから、必要な画像をアプリ所定の保存先へ復元する。
- DB 上の画像パス・ファイル名と実ファイルが一致するか確認する。
- ゲスト作品一覧などでポスターの表示を確認する。
- 本番データ・画像や SQL ダンプを不用意に Git に追加しない。

## 動作確認（LIFEBOOK で検証済み）

### セキュリティ更新履歴（2026-10-22）

PHP の Critical 脆弱性（CVE-2026-109618 / CVE-2026-109619）への対応として、2026-10-22 に以下を実施しました。

- **Web コンテナ：** PHP **8.4.26 → 8.4.27**。再ビルド・稼働中バージョン・ゲスト閲覧画面・PHP 構文チェックを確認済み。
- **phpMyAdmin コンテナ：** PHP **8.3.33 → 8.3.36**。新しい公式イメージを取得してコンテナのみ再作成し、稼働中バージョン・管理画面ログイン・既存DBのテーブル一覧表示を確認済み。
- **MySQL：** 今回のPHP更新対象外。既存コンテナと永続データを維持。

**PHPを使用する2コンテナの更新と基本動作確認は完了**しました。会員向け書き込み機能などの未検証項目は残り、変更のコミット・PR・`main` への反映はこれからです。対応Issueは [Issue 4](https://github.com/honda-dev-jp/docker-lab/issues/4) を参照してください。

脆弱性の概要・影響範囲・更新コマンド・実測結果と残課題は **[SECURITY_HISTORY.md](SECURITY_HISTORY.md)** にまとめています。

### HTTP・アクセス制御

```bash
# ローカルの HTTP 応答を確認
curl -I http://localhost/

# アプリ付属の一括チェックスクリプト（ホスト側 PHP・PHP-CS-Fixer が必要）
# cd ~/projects/review-app && ./scripts/check.sh
```

アプリ付属の `scripts/check.sh` は、ホスト側に `php` と `php-cs-fixer` がない LIFEBOOK では一部の項目が FAIL になります。ただし、HTTP 疎通・想定した公開 URL とアクセス拒否 URL のチェックは PASS しました。スクリプト自体の Docker 対応は未実施です。

### PHP 構文チェック（Web コンテナ）

LIFEBOOK ではホストに PHP を追加せず、Web コンテナの PHP 8.4.26 で構文チェックを実行し、対象ファイルすべてで構文エラーがないことを確認しました。

```bash
cd ~/projects/docker-lab/lamp

docker compose exec -T web sh -c \
  'cd /var/www/html && find . -path "./vendor" -prune -o -type f -name "*.php" ! -path "./config/env.php" ! -path "./config/database.php" -exec php -l {} \;'
```

**再検証結果（2026-10-22）：** Web コンテナを PHP 8.4.27 に更新後、上記のコマンドで構文チェックを再実行しました。出力されたすべての対象 PHP ファイルで `No syntax errors detected` を確認しました。`vendor/` と非公開設定ファイル（`env.php`・`database.php`）は検証対象外です。

### PHP-CS-Fixer（独立した Docker イメージ）

Composer・PHP-CS-Fixer をホストや LAMP の Web コンテナへ追加せず、専用 Docker イメージで検証します。`review-app` を読み取り専用でマウントし、`--dry-run`・`--using-cache=no` で実行します。

```bash
cd ~/projects/docker-lab/lamp

docker run --rm \
  --user "$(id -u):$(id -g)" \
  -v "$HOME/projects/review-app:/app:ro" \
  -w /app \
  ghcr.io/php-cs-fixer/php-cs-fixer:3.95.25-php8.4 \
  fix \
  --dry-run \
  --diff \
  --using-cache=no \
  --config=.php-cs-fixer.dist.php
```

**検証結果（2026-10-10）：** PHP-CS-Fixer 3.95.25 / 実行ランタイム PHP 8.4.25 で **77/77 ファイル、修正対象 0 件**。

`composer.json` がないため、最低 PHP バージョンを判定できない旨の警告が出ましたが、検査は完了しました。Docker イメージ・規約設定・検査対象が変われば結果も変わるため、将来の検査結果を保証するものではありません。

## ログ確認

```bash
# 全サービス
docker compose logs -f

# Apache のエラーログ
docker compose exec web tail -f /var/log/apache2/error.log

# MySQL のスロークエリログ（my.cnf で有効化済み）
docker compose exec db tail -f /var/log/mysql/slow.log
```

## .htaccess について

`AllowOverride All` と `mod_rewrite` が有効化済みです。アプリの `.htaccess` を Apache が読み込みます。

HTTPS 強制・www 正規化・探索系アクセス拒否などはアプリ側 `.htaccess` のルールに依存します。現在進行中の Cloudflare 移行前棚卸し（`review-app` Issue 28）とは別に、LAMP 環境構築だけを理由として `.htaccess` を変更しないでください。

## 停止・再開・削除

```bash
# 停止（データ保持・再開可能）
docker compose stop

# 再開
docker compose start

# 停止＋コンテナ削除（データ保持）
docker compose down

# 停止＋コンテナ＋データ全削除（クリーンな再構築時のみ）
docker compose down -v
```

> **注意：** `docker compose down -v` は名前付きボリュームを削除します。インポート済みの MySQL データが失われるため、バックアップを確認せず実行しないでください。

## ⚠️ ハマりポイント

### `version:` は不要（obsolete 警告）

Docker Compose V2 以降、トップレベルの `version:` は不要です。記載すると WARN が表示されます。

### `skip-character-set-client-handshake` は MySQL 8.4 で使用できない

`my.cnf` に記載すると MySQL が起動できない場合があります。設定から削除してください。

### `output_buffering` は標準イメージで無効の場合がある

`header()` を使用する前に出力があるとヘッダー関連の警告が発生します。必要に応じて PHP 設定を確認してください。既存の `php.ini` では次の設定を使用します。

```ini
[Output]
output_buffering = On
```

### DB 接続ホスト名は `localhost` ではなく `db`

コンテナ間通信には Compose サービス名を使用します。アプリの接続設定では `db` を指定します。**実設定ファイルの中身を Issue・ログ・README に貼り付けないでください。**

```php
// 例：接続先ホスト名のみ
'host' => 'db'
```

### `.htaccess` のリファラ制限とローカル URL

本番 URL だけを許可するリファラ制限があると、ローカルの画像読み込みが 403 になる場合があります。既存の `.htaccess` と実際の要件を確認してください。**現在の Issue 28 の作業内容を勝手に変更しないこと。**

### `.env` のパスワードに特殊文字が含まれる場合

Docker Compose の `.env` の解釈規則に注意してください。`$` などを展開させたくない場合は、単一引用符によるクォートが使えます。

```dotenv
# 形式の例。実際のパスワードは記載しない。
MYSQL_PASSWORD='replace-with-a-strong-password'
```

### LIFEBOOK の CPU 互換性と MySQL イメージ

Pentium P6200 搭載の LIFEBOOK では、MySQL の互換性対策として `ubuntu/mysql:8.4-24.04_edge` の digest 固定イメージに切り替えました。標準構成の `mysql:8.4` は変更せず、ホスト固有の override にのみ記述します。

これは **LIFEBOOK で動作確認できた構成**の記録です。ほかの CPU で同じ切り替えが必要とは限りません。

### HTTP 500：非公開設定ファイルのマウント不足

LIFEBOOK の初期構築時は、`http://localhost/` が 500 を返しました。`/var/www/private_config/portfolio` へのマウントを追加した後、HTTP 200 を確認できました。

Compose のマウント設定を変更した場合、既存コンテナを単に再起動するだけでは反映されません。次のように再作成します。

```bash
docker compose up -d --force-recreate web
```

### `scripts/check.sh` がホスト側 PHP を検出できない

LIFEBOOK の Ubuntu ホストには PHP と PHP-CS-Fixer を入れていないため、従来の `scripts/check.sh` は一部失敗します。PHP 構文チェックは `web` コンテナ、PHP-CS-Fixer は専用 Docker イメージで個別に検証してください。

## セキュリティメモ

LAMP の脆弱性対応履歴は [SECURITY_HISTORY.md](SECURITY_HISTORY.md) に記録します。リポジトリ全体の報告・保守方針は [../SECURITY.md](../SECURITY.md) を参照してください。

- `.env` は Git に含めません。テンプレートのパスワードを実運用に使用しないでください。
- `compose.override.yml` / `compose.override.yaml` はローカル設定なので、両方を `.gitignore` で除外します。
- Git 管理対象外の `review-app-private/portfolio/` に実設定ファイルを置き、`/var/www/html` の外側へ `:ro` でマウントします。
- 非公開設定のパーミッションは、LIFEBOOK ではディレクトリ `700`・ファイル `600` で動作確認しました。実行ユーザーとの所有者整合性も必要です。
- DB ポートは外部に公開していません（コンテナ間通信のみ）。
- Web・phpMyAdmin は `127.0.0.1` バインドでローカル限定公開です。
- SQL ダンプ、画像バックアップ、秘密鍵、実設定ファイルを Git に追加しないでください。
- この README に記載した Docker 構成は開発環境用です。本番公開時の設定は別途セキュリティ評価してください。
