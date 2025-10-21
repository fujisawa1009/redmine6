# Redmine リアルタイム通知システム (redmine_rt) セットアップガイド

## 概要

redmine_rtプラグインは、Redmineにリアルタイム通知機能を追加します。チケットのコメントや更新が他のユーザーにリアルタイムで通知されます。

## アーキテクチャ

### 技術スタック
- **ActionCable**: Rails 7.2のWebSocketフレームワーク
- **Redis**: メッセージブローカー（pub/sub）
- **Deface**: Redmineビューのオーバーライド
- **WebSocket**: ブラウザとサーバー間のリアルタイム通信

### データフロー
1. ユーザーAがチケットにコメントを投稿
2. `Journal#after_save`コールバックが発火
3. `RedmineRt::Broadcaster`がRedisにメッセージをpublish
4. ActionCableがRedisからメッセージを受信
5. WebSocket経由で接続中のユーザーBにメッセージを配信
6. JavaScriptが通知を表示

## セットアップ手順

### 1. 前提条件

- Redmine 6.0以上
- Rails 7.2以上
- Redis 7.x
- Docker & Docker Compose（推奨）

### 2. Redisサービスの追加

`docker-compose.yml`にRedisサービスを追加：

```yaml
services:
  redmine:
    # ... 既存の設定 ...
    volumes:
      # ... 既存のボリューム ...
      - ./docker/redmine/config/cable.yml:/usr/src/redmine/config/cable.yml  # 追加

  redmine-db:
    # ... 既存の設定 ...

  redis:  # 新規追加
    image: redis:7-alpine
    container_name: redmine-redis
    restart: always
    ports:
      - 6379:6379
```

### 3. ActionCable設定ファイルの作成

`docker/redmine/config/cable.yml`を作成：

```yaml
production:
  adapter: redis
  url: redis://redmine-redis:6379/1
  channel_prefix: redmine_rt_production

development:
  adapter: redis
  url: redis://redmine-redis:6379/1
  channel_prefix: redmine_rt_development

test:
  adapter: test
```

### 4. プラグインのインストール

```bash
# プラグインディレクトリに移動
cd docker/redmine/plugins

# redmine_rtプラグインをクローン
git clone https://github.com/MayamaTakeshi/redmine_rt.git

# 依存関係のインストール
docker compose exec redmine bash -c "cd /usr/src/redmine && bundle install"
```

### 5. Redmine側の設定

管理者権限でRedmineにログインし、以下を設定：

#### 5.1 自動ログイン機能の有効化
1. **管理 → 設定 → 認証**
2. 「**自動ログイン**」を有効化（例: 30日）
3. 「適用」をクリック

#### 5.2 REST APIの有効化
1. **管理 → 設定 → API**
2. 「**RESTによるWebサービスを有効にする**」にチェック
3. 「適用」をクリック

### 6. コンテナの再起動

```bash
# コンテナを再起動
docker compose down
docker compose up -d

# ログで起動確認
docker compose logs redmine | grep -i "redmine_rt\|cable"
```

### 7. ユーザー側の設定（重要）

**各ユーザーは以下の手順が必須：**

1. 現在ログインしている場合は**ログアウト**
2. ログイン画面で**「ログイン状態を保持する」にチェック**を入れる
3. ログイン
4. チケットページをリロード

> **重要**: 「ログイン状態を保持する」にチェックを入れないと、autologin cookieが設定されず、WebSocket認証が失敗します（`unauthorized`チャンネルに接続されてしまいます）。

### 8. 動作確認

#### 8.1 WebSocket接続の確認

1. チケットページを開く
2. ブラウザの開発者ツール（F12）→ Networkタブ → WSフィルター
3. `/cable` という接続が表示され、ステータスが`101 Switching Protocols`であることを確認

#### 8.2 通知のテスト

1. **ブラウザA**: ユーザー1でログイン、チケット#Xを開く
2. **ブラウザB**: ユーザー2でログイン、同じチケット#Xを開く
3. **ブラウザA**: コメントを投稿
4. **ブラウザB**: 画面右上に通知が表示される

#### 8.3 Redisログの確認（トラブルシューティング用）

```bash
# Redisのpub/subアクティビティをモニター
docker compose exec redis redis-cli MONITOR

# 別ターミナルでコメント投稿後、以下のようなログが表示されるべき：
# "subscribe" "redmine_rt_production:issue:XXX"
# "publish" "redmine_rt_production:issue:XXX" "{\"event\":\"journal ... saved\",..."
```

## 本番環境への移行

### データベースを保持する場合

現在のリポジトリを本番環境に移行する際、**データベースはそのまま維持されます**：

```bash
# 本番サーバーで

# 1. 既存コンテナを停止（データベースは削除しない）
docker compose down

# 2. 最新のリポジトリを取得
git pull origin main

# 3. コンテナを再起動
docker compose up -d
```

**重要なポイント**：
- `docker/db/data/`ディレクトリはボリュームマウントされているため、コンテナを削除してもデータは保持されます
- `docker compose down -v`（-vオプション付き）を実行すると、ボリュームが削除されデータが消失します
- データベースをリセットしたい場合のみ、`docker/db/data/`を削除してください

### データベースをリセットする場合

```bash
# データベースを完全にリセットしたい場合のみ実行

# 1. コンテナとボリュームを削除
docker compose down -v

# 2. データディレクトリを削除（任意）
rm -rf docker/db/data/*

# 3. コンテナを再起動
docker compose up -d

# 4. データベース初期化
docker compose exec redmine rake db:migrate RAILS_ENV=production
docker compose exec redmine rake redmine:load_default_data RAILS_ENV=production REDMINE_LANG=ja
```

### 本番環境チェックリスト

- [ ] Redisサービスが起動している
- [ ] `cable.yml`が正しくマウントされている
- [ ] 管理設定で「自動ログイン」が有効
- [ ] 管理設定で「REST API」が有効
- [ ] 全ユーザーが「ログイン状態を保持する」でログイン
- [ ] WebSocket接続が確立している（/cable エンドポイント）
- [ ] 通知が正常に動作している

## トラブルシューティング

### 問題: 通知が表示されない

**確認項目**：

1. **ログイン状態を保持しているか？**
   ```bash
   # ブラウザの開発者ツール → Application → Cookies
   # "autologin" cookieが存在するか確認
   ```

2. **正しいチャンネルに接続しているか？**
   ```bash
   # Redisログを確認
   docker compose exec redis redis-cli MONITOR

   # "subscribe" "redmine_rt_production:issue:XXX" が表示されるべき
   # "subscribe" "redmine_rt_production:unauthorized" の場合は認証失敗
   ```

3. **WebSocket接続が確立しているか？**
   ```bash
   # Redmineログを確認
   docker compose logs redmine | grep -i websocket

   # "Successfully upgraded to WebSocket" が表示されるべき
   ```

4. **ブロードキャストが送信されているか？**
   ```bash
   # Redisログでpublishを確認
   docker compose exec redis redis-cli MONITOR | grep publish

   # コメント投稿時に "publish" "redmine_rt_production:issue:XXX" が表示されるべき
   ```

### 問題: "unauthorized" チャンネルに接続される

**原因**: autologin cookieが設定されていない

**解決方法**：
1. ログアウト
2. 「ログイン状態を保持する」にチェックを入れてログイン
3. ページをリロード

### 問題: WebSocket接続が確立しない

**確認項目**：

1. Redisが起動しているか
   ```bash
   docker compose ps redis
   # State: Up であるべき
   ```

2. cable.ymlがマウントされているか
   ```bash
   docker compose exec redmine ls -la /usr/src/redmine/config/cable.yml
   # ファイルが存在するべき
   ```

3. Redis接続テスト
   ```bash
   docker compose exec redmine bash -c "cd /usr/src/redmine && RAILS_ENV=production bundle exec rails runner \"
   redis = Redis.new(url: 'redis://redmine-redis:6379/1')
   puts redis.ping
   \""
   # "PONG" が返ってくるべき
   ```

## 仕組みの詳細

### コールバックフロー

#### Journal作成時（コメント投稿）

```ruby
# 1. ユーザーがコメントを投稿
QuickNotesController#add_quick_notes
  → issue.init_journal(user)
  → journal.notes = params[:issue][:notes]
  → issue.save!

# 2. Journal#after_save コールバック
RedmineRt::JournalPatch#handle_journal_after_save
  → Broadcaster.broadcast "issue:#{issue_id}", { type: 'journal_saved', ... }

# 3. Journal#after_create コールバック（通知対象ユーザーに配信）
RedmineRt::JournalPatch#handle_journal_after_create
  → [issue.author_id, issue.assigned_to_id].each do |user_id|
       Broadcaster.broadcast "user:#{user.login}", { command: 'show_notification', ... }
     end
```

#### Issue更新時

```ruby
# Issue#after_save コールバック
RedmineRt::IssuePatch#handle_issue_after_save
  → Broadcaster.broadcast "issue:#{issue.id}", { type: 'issue_saved', ... }
```

### Broadcasterの仕組み

```ruby
# lib/redmine_rt/broadcaster.rb
module RedmineRt
  class Broadcaster
    def self.broadcast(channel_name, data)
      ActionCable.server.broadcast channel_name, data
    end
  end
end
```

ActionCable.server.broadcast → Redis PUBLISH → ActionCable購読者に配信

### WebSocket認証フロー

```ruby
# app/channels/application_cable/connection.rb
def find_verified_user
  # 1. URLパラメータのaccess_keyをチェック
  token = Token.find_by(action: :api, value: request.params[:access_key])

  # 2. autologin cookieをチェック
  if not token
    token = Token.find_by(action: :autologin, value: cookies[:autologin])
  end

  # 3. トークンからユーザーを取得
  if token
    return User.find(token.user_id)
  else
    request.params[:unauthorized] = true
    return nil
  end
end
```

### チャンネル購読

```ruby
# app/channels/channel.rb
def subscribed
  if not current_user or current_user[:unauthorized]
    stream_from "unauthorized"  # 認証失敗
  else
    stream_from params['name']  # "issue:608" など
  end
end
```

### JavaScript（クライアント側）

```javascript
// assets/javascripts/cable.js
App.ws_setup = function(event_handler) {
  var channel_name = $('meta[name=page_specific_js]').attr('channel_name')
  // channel_name = "issue:608" など

  App.cable = ActionCable.createConsumer(base_url + "/cable");
  App.cable.subscriptions.create({
    channel: 'Channel',
    name: channel_name
  }, {
    received: event_handler  // メッセージ受信時の処理
  });
};
```

### Defaceによるビューオーバーライド

```ruby
# app/overrides/issues/show.rb
Deface::Override.new(
  :virtual_path => 'issues/show',
  :name => "add quick notes",
  :insert_before => "#history"
) do
  # metaタグでチャンネル名を設定
  <%= tag :meta, name: :page_specific_js, channel_name: 'issue:' + @issue.id.to_s %>

  # ActionCable JavaScriptを読み込み
  <%= javascript_include_tag(:cable, :plugin => 'redmine_rt') %>
end
```

## セキュリティ考慮事項

### CSRF保護の無効化

```ruby
# init.rb
Rails.application.config.action_cable.disable_request_forgery_protection = true
```

**理由**: WebSocket接続時はCSRFトークンの検証ができないため無効化しています。代わりにautologin tokenで認証を行います。

### プライベートコメントの扱い

- プライベートコメントもWebSocket経由で配信されます
- 表示制御はクライアント側（JavaScript）で行われます
- サーバー側で権限チェックを追加する場合は、`handle_journal_after_create`を修正してください

### 通知の表示制限

```javascript
// assets/javascripts/channels/issue_channel.js
const fromLoggedInUser = ($message) => {
  const author = $message.find('.user.active').attr('href')
  const logged_in = $('#loggedas').find('.user').attr('href')
  return author == logged_in
}

if(fromLoggedInUser($message)) {
  console.log(`Message from self. Ignoring.`)
  return  // 自分の投稿は通知しない
}
```

## パフォーマンス考慮事項

### Redis接続数

- 各Redmineプロセスが1つのRedis接続を持ちます
- 同時接続ユーザー数に比例してRedis接続が増加します
- 大規模環境ではRedisの`maxclients`設定を調整してください

### WebSocket接続数

- ユーザーがチケットページを開くたびに1つのWebSocket接続が確立されます
- 接続はページを離れるまで維持されます
- nginx等のリバースプロキシを使用する場合、WebSocket接続のタイムアウト設定に注意してください

## まとめ

redmine_rtプラグインは以下のコンポーネントで構成されています：

1. **サーバー側**
   - RedmineモデルPatch（Journal, Issue）
   - ActionCableチャンネル
   - Redis pub/sub

2. **クライアント側**
   - ActionCable JavaScript
   - WebSocket接続管理
   - 通知UI

3. **インフラ**
   - Redisコンテナ
   - cable.yml設定

**本番環境への移行は、docker-compose.ymlとcable.ymlが含まれていれば、そのままデプロイ可能です。データベースは`docker/db/data/`ボリュームに永続化されているため、コンテナ再起動時も保持されます。**
