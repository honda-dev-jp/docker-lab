# LAMP セキュリティ対応履歴

この文書は `docker-lab/lamp/` の開発・検証環境で確認した脆弱性について、**公開済みの公式情報、影響範囲、対応内容、検証結果、残課題**を時系列で記録します。

- リポジトリ全体のセキュリティ方針：[../SECURITY.md](../SECURITY.md)
- LAMP 環境の構築・利用手順：[README.md](README.md)
- 調査・修正・レビューの追跡：GitHub Issue / Pull Request

この履歴は、個々の構成・検証時点についての記録であり、将来の安全性やインターネット公開の安全性を保証するものではありません。**未公開の脆弱性や秘密情報は、この公開ドキュメントに記載しません。**

## 対応状況一覧

| 対応日 | 識別子 | 対象 | 状況 | 追跡 |
|---|---|---|---|---|
| 2026-10-22 | CVE-2026-109618 / CVE-2026-109619 | `lamp_web`（PHP 8.4） | **修正版へ更新・主要機能の動作確認済み** | [Issue 4](https://github.com/honda-dev-jp/docker-lab/issues/4) |
| 2026-10-22 | 同上 | `lamp_phpmyadmin`（PHP 8.3） | **修正版へ更新・管理画面とDB接続を確認済み** | [Issue 4](https://github.com/honda-dev-jp/docker-lab/issues/4) |

## 2026-10-22：PHP Critical 脆弱性への対応（Issue 4）

### 1. 脆弱性の概要

PHP が2026年10月21日に公表した、**認証なしのリモートコード実行（RCE）につながる可能性のある Critical 脆弱性2件**への対応です。PHP公式は修正版への更新を推奨しており、設定変更だけでは完全な緩和にならないとしています。

| 識別子 | 公式名称 | 概要 | 深刻度 |
|---|---|---|---|
| [CVE-2026-109618](https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w) | RCE via OOM corruption | メモリ不足（OOM）時の異常終了処理により、リクエスト用メモリが不正な状態で後処理に使われる問題。条件次第でRCEにつながる | Critical（CVSS v4.0 9.2） |
| [CVE-2026-109619](https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp) | RCE via stale global state | リクエスト間で適切に初期化されなかったグローバル状態を参照することで、解放済みメモリへのアクセスが発生し得る問題。条件次第でRCEにつながる | Critical（CVSS v4.0 9.2） |

両件の修正バージョンは次のとおりです（**記載したバージョンより前の同系列が影響対象**）。

| PHP系列 | 修正バージョン |
|---|---|
| 8.2 | 8.2.35 |
| 8.3 | 8.3.36 |
| 8.4 | **8.4.27** |
| 8.5 | 8.5.12 |

**注意：** 脆弱性が存在することと、実際に当該環境が攻撃を受けたことは別です。今回の検証では、攻撃の発生や不正侵入を確認したという事実はありません。公開範囲がローカルに限定されていても、PHP自体の修正版適用は必要です。

### 2. 影響範囲と更新前の状態

2026年10月21日から22日にLIFEBOOK（Ubuntu）上のDocker環境を確認しました。

| 対象 | 更新前に確認した状態 | 対応判断 |
|---|---|---|
| `lamp_web` | PHP **8.4.26**。`lamp/Dockerfile` は `FROM php:8.4-apache` | PHP 8.4.27へ更新する |
| `lamp_phpmyadmin` | PHP **8.3.33**。`phpmyadmin:latest` を使用 | PHP 8.3.36以降を含む公式イメージを確認してから更新する |
| `lamp_db` | Ubuntu版 MySQL 8.4系、正常稼働 | 今回のPHP脆弱性の直接の更新対象ではない。構成と永続データを保持する |

ホスト公開はWebが `127.0.0.1:80`、phpMyAdminが `127.0.0.1:8081` で、MySQLのホスト向けポート公開はありませんでした。この構成があらゆる経路でのアクセスを不可能にすることまで証明するものではありません。

phpMyAdmin公式のDockerfileは `FROM php:8.3-apache` を使用します。当初確認したDocker Hubのamd64向け `phpmyadmin:latest` は **2026-10-21 01:25 UTC** 作成で、この時点では修正版PHPの取り込みを確認できませんでした。その後、新しいイメージが配信され、**PHP 8.3.36をイメージ内と稼働中コンテナ内の双方で実測**しています（後述）。**タグ名や作成日時だけでパッチ適用済みとは判定しません。**

### 3. Webコンテナで実施した修正

作業ブランチ：`security/php-critical-20261021`
関連Issue：[honda-dev-jp/docker-lab Issue 4](https://github.com/honda-dev-jp/docker-lab/issues/4)

`lamp/Dockerfile` のベースイメージを、PHP 8.4系列の浮動タグから修正済みパッチを明示するタグに変更しました。

```diff
-FROM php:8.4-apache
+FROM php:8.4.27-apache
```

**実施した操作（`lamp/` ディレクトリ）：**

```bash
# Compose設定を検証（設定値全体は表示しない）
docker compose config --quiet

# Webイメージだけを再ビルド
docker compose build --pull web

# 新しいイメージ内のPHPを確認（実行中コンテナは置き換えない）
docker run --rm --network none --entrypoint php lamp-web:latest -v

# Webコンテナだけを再作成
docker compose up -d --no-deps --force-recreate web

# 稼働中WebコンテナのPHPとComposeの状態を確認
docker exec lamp_web php -v
docker compose ps
```

**このWeb更新の操作では**、MySQL・phpMyAdminコンテナを再作成していません（phpMyAdminは後続の別操作で更新）。**`docker compose down -v` は実施していません。** ローカル専用の `.env`、`compose.override.yaml`、非公開設定ファイルの内容は変更・公開していません。

### 4. 検証結果

**Docker・PHP・ネットワーク**

| 項目 | 実測結果 |
|---|---|
| `docker compose config --quiet` | 正常終了 |
| `docker compose build --pull web` | 成功（約565.5秒） |
| 新イメージの `php -v` | **PHP 8.4.27** |
| Webコンテナ再作成 | 成功（`lamp_web` Started） |
| 稼働中の `lamp_web` の `php -v` | **PHP 8.4.27**、Zend Engine 4.4.27、OPcache 8.4.27 |
| `docker compose ps`（Web更新後） | `lamp_web` Up / `lamp_db` Up（healthy）/ `lamp_phpmyadmin` Up |
| Webポート | `127.0.0.1:80` のまま |
| phpMyAdminポート | `127.0.0.1:8081` のまま |
| MySQLホスト公開ポート | なし |

**スクラッチ版映画レビューアプリの動作確認（Chrome・`http://localhost/`）**

- ログイン画面の表示
- ゲスト作品一覧、ページネーション、作品画像の表示
- 作品詳細のタイトル・ポスター・あらすじの表示
- 平均評価、レビュー件数、評価グラフ、レビュー一覧、ユーザーアイコンの表示
- 上記の閲覧画面に必要なMySQLデータ取得と画像読み込み
- 会員ログインの成功（2026-10-22、ローカル環境）
- レビュー投稿・削除の成功（2026-10-22、ローカル環境）

**PHP構文チェック**

WebコンテナのPHP 8.4.27で、下記コマンドによる構文チェックを再実行し、**出力された全PHPファイルで `No syntax errors detected` を確認**しました。

```bash
docker compose exec -T web sh -c \
  'cd /var/www/html && find . -path "./vendor" -prune -o -type f -name "*.php" ! -path "./config/env.php" ! -path "./config/database.php" -exec php -l {} \;'
```

対象はアプリ内のPHPファイルで、`vendor/` と非公開の `env.php`・`database.php` は対象外です。**構文チェックは実行時の全機能を保証しません。**

**ローカルURLの確認**

`http://127.0.0.1/` はHTTP 301で `https://127.0.0.1/` へリダイレクトされましたが、443番ポートは公開されていないため接続拒否となりました。`http://localhost/` では画面表示に成功しています。アプリのリダイレクト設定は変更していません。

### 5. phpMyAdminコンテナで実施した修正（2026-10-22）

Webコンテナの更新後、Docker Hubの `phpmyadmin:latest` に新しいイメージが配信されたことを確認しました。

**公開イメージの確認結果（linux/amd64）：**

| 項目 | 確認結果 |
|---|---|
| 利用したタグ | `phpmyadmin:latest`（可変タグ） |
| phpMyAdminバージョン | `5.2.3-apache`（公式イメージの注記） |
| Docker Hubの新しいイメージインデックスDigest | `sha256:527a161fa6c0192cc66180e17e5541aaea028a23a9590e86ce08ecf7daa4fbc8` |
| amd64イメージの作成日時 | 2026-10-21 16:26:49 UTC（日本時間 2026-10-22 01:26:49） |
| ベースイメージ | `php:8.3-apache` |
| 修正済みPHPの実測 | **PHP 8.3.36** |

当初確認したインデックスDigest `sha256:cb525665c0669c74a117a63cb918a90662c786d9d4d04212717575546f6d7e29` から更新されていることを確認しました。**Digestの変化だけで修正を確定せず、下記のコマンドでPHPの実行バージョンを確認**しています。

**実施した操作（`lamp/` ディレクトリ）：**

```bash
# Docker Hubから新しい公式イメージを取得
docker pull phpmyadmin:latest

# 取得したイメージを一時コンテナで確認（既存コンテナを変更しない）
docker run --rm --network none --entrypoint php phpmyadmin:latest -v

# phpMyAdminコンテナのみ再作成（Web・MySQLには触れない）
docker compose up -d --no-deps --force-recreate phpmyadmin

# 稼働中のphpMyAdminコンテナのPHPを確認
docker exec lamp_phpmyadmin php -v
```

**実測・目視の結果：**

| 検証項目 | 結果 |
|---|---|
| `docker pull phpmyadmin:latest` | 成功。新しいイメージを取得 |
| 一時コンテナの `php -v` | **PHP 8.3.36**、Zend Engine 4.3.36、OPcache 8.3.36 |
| `docker compose up -d --no-deps --force-recreate phpmyadmin` | 成功（`lamp_phpmyadmin` Started） |
| 稼働中コンテナの `php -v` | **PHP 8.3.36**、Zend Engine 4.3.36、OPcache 8.3.36 |
| ブラウザ（`http://localhost:8081/`） | phpMyAdminの管理画面にログインして表示を確認 |
| MySQL接続・閲覧 | 既存データベースのテーブル一覧とレコード件数が表示されることを確認 |
| Web・MySQLコンテナ | phpMyAdmin更新操作では再作成せず維持 |

phpMyAdminの操作では**既存のMySQLデータの削除や変更は行っていません**。データベース名、認証情報、画面の実データは公開ドキュメントに記載しません。

**運用上の注意：** `phpmyadmin:latest` は可変タグのままで、`docker-compose.yml` のイメージ指定は変更していません。DockerfileのPHPパッチバージョン指定とは異なり、後日のイメージ更新には明示的な `pull` とコンテナの再作成、内部PHPバージョン・動作の再検証が必要です。上記Digestは**今回の検証対象を識別するための履歴**であり、Compose側でDigest固定したという意味ではありません。

### 6. 一次情報・関連資料

- [PHP公式：2026年10月21日 セキュリティ告知](https://www.php.net/archive/2026.php#2026-10-21-5)
- [PHP公式：8.4.27リリース](https://www.php.net/releases/8_4_27.php)
- [PHP公式：8.3.36リリース](https://www.php.net/releases/8_3_36.php)
- [PHP公式GHSA：CVE-2026-109618（OOM corruption）](https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w)
- [PHP公式GHSA：CVE-2026-109619（stale global state）](https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp)
- [PHP公式Dockerイメージ：更新PR 1694](https://github.com/docker-library/php/pull/1694)
- [Docker Official Images：登録PR 22541](https://github.com/docker-library/official-images/pull/22541)
- [phpMyAdmin公式Dockerfile](https://github.com/phpmyadmin/docker/blob/master/apache/Dockerfile)
- [対応Issue 4](https://github.com/honda-dev-jp/docker-lab/issues/4)

## 今後の記録方針

今後別の脆弱性に対応した場合は、**対応日、識別子（CVE / GHSA）、影響を受けるイメージ・バージョン、更新内容、確認コマンドと結果、残課題、関連Issue / PR**を追記します。未検証の項目を確認済みと書き換えず、検証日を明記します。
