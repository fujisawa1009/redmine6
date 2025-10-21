# CLAUDE.md

## 基本方針
このプロジェクトはClaudeCodeを使用した日本語環境での開発です。
必ず日本語で回答してください。

このファイルはこのリポジトリでClaudeCodeが作業する際のガイダンスを提供します。

## 概要

DockerベースのRedmine 6.0.6環境（MariaDB 11.5使用）です。プラグイン、テーマ、データベースの永続化にボリュームマウントを使用したカスタムDockerコンテナ構成です。

## アーキテクチャ

アプリケーションは2つのDockerサービスで構成されています：
- **redmine-app**: Redmine 6.0.6アプリケーションサーバー（プラグインコンパイル用にmake、gccを含む）
- **redmine-db**: MariaDB 11.5データベース（UTF8MB4文字セット）

主要ディレクトリ：
- `docker/redmine/plugins/`: アクティブなRedmineプラグイン（`/usr/src/redmine/plugins`にマウント）
- `docker/redmine/plugins_backup/`: Redmine 4.2から無効化されたプラグイン（kanban、redmine_issues_tree）
- `docker/redmine/themes/`: カスタムテーマ（`/usr/src/redmine/public/themes`にマウント）
- `docker/db/data/`: MariaDBデータ永続化

## 開発コマンド

### コンテナ管理
```bash
# 全サービスをビルド・起動
docker compose up -d

# キャッシュリセットしてビルド
docker compose build --no-cache

# 特定サービスの再起動
docker compose restart redmine
docker compose restart redmine-db

# コンテナ状態確認
docker compose ps

# ログ表示
docker compose logs redmine
docker compose logs redmine-db
```

### Redmine操作
```bash
# Redmineコンテナにアクセス
docker compose exec redmine bash

# データベースマイグレーション実行
docker compose exec redmine rake db:migrate RAILS_ENV=production

# Redmine初期データ投入（日本語）
docker compose exec redmine rake redmine:load_default_data RAILS_ENV=production REDMINE_LANG=ja

# シークレットキートークン生成
docker compose exec redmine rake generate_secret_token RAILS_ENV=production

# Railsコンソールアクセス
docker compose exec redmine bash -c "cd /usr/src/redmine && RAILS_ENV=production bundle exec rails console"
```

### データベース操作
```bash
# MariaDBアクセス
docker compose exec redmine-db mariadb -u root -p

# データベース手動作成
docker compose exec redmine-db mariadb -u root -predmine -e "CREATE DATABASE IF NOT EXISTS redmine CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

## プラグイン管理

Redmine 6.0互換性の問題により、初期セットアップ時はプラグインを無効化：
- 元のプラグインは`docker/redmine/plugins_backup/`に保存
- アクティブプラグインディレクトリ：`docker/redmine/plugins/`
- プラグインの追加・削除後はRedmineコンテナの再起動が必要

## 設定情報

- データベース認証情報：root/redmine
- MariaDBはUnicode完全サポートのためutf8mb4文字セットで構成
- Redmineはポート3000でアクセス可能
- Docker Compose V2互換性のためversionディレクティブを削除

## よくある問題

- プラグイン互換性：Redmine 4.2プラグインは6.0で動作しない可能性
- コンテナ再起動ループ：通常はプラグイン初期化エラーが原因
- データベース接続：RedmineよりもMariaDBコンテナが完全に起動していることを確認