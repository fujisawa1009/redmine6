# View Customize Plugin 設定手順書

## 概要

View Customize PluginはRedmineのページをJavaScript、CSS、HTMLで自由にカスタマイズできるプラグインです。
Redmine 6.0.6環境への導入が完了しています。

## 導入済み作業

### 1. プラグインのダウンロードと配置
```bash
cd docker/redmine/plugins
git clone https://github.com/onozaty/redmine-view-customize.git view_customize
```

### 2. 依存関係のインストール
```bash
docker compose exec redmine bash -c "cd /usr/src/redmine && bundle install --without development test"
```

### 3. データベースマイグレーション実行
```bash
docker compose exec redmine bash -c "cd /usr/src/redmine && RAILS_ENV=production bundle exec rake redmine:plugins:migrate"
```

### 4. Redmineコンテナ再起動
```bash
docker compose restart redmine
```

## プラグイン設定手順

### 1. 管理者権限でのアクセス
- Redmine管理画面（http://localhost:3000）に管理者でログイン

### 2. プラグイン設定画面への移動
1. 「管理」メニューをクリック
2. 「プラグイン」を選択
3. 「View customize」プラグインの「設定」をクリック

### 3. View Customizeの作成・編集
1. 管理画面で「View Customizes」メニューを選択
2. 「新しいView Customizeを追加」をクリック
3. 以下の項目を設定：

#### 基本設定項目
- **名前**: カスタマイズの名前（例：「ヘッダー色変更」）
- **コメント**: カスタマイズの説明
- **有効**: チェックを入れて有効化
- **プライベート**: 個人用の場合はチェック
設定参考：https://qiita.com/wfigo7/items/f13a89e1d3522f4a73b6

#### 適用条件
- **パスパターン**: 適用するページのURLパターン
  - 例：`/issues` （チケット一覧ページ）
  - 例：`/issues/\d+` （個別チケットページ）
  - 例：`.*` （全ページ）
- **プロジェクトパターン**: 特定プロジェクトのみに適用する場合

#### カスタマイズコード
- **カスタマイズタイプ**: JavaScript / CSS / HTML から選択
- **挿入位置**:
  - `html_head`: HTMLのheadタグ内
  - `html_body_top`: bodyタグの直後
  - `html_body_bottom`: bodyタグの直前
- **コード**: 実際のカスタマイズコードを記述

## よく使用するカスタマイズ例

### 1. ヘッダーの背景色変更（CSS）
```css
/* カスタマイズタイプ: CSS */
/* 挿入位置: html_head */
#header {
    background-color: #2c5aa0 !important;
}
```

### 2. チケット一覧に警告メッセージ表示（JavaScript）
```javascript
/* カスタマイズタイプ: JavaScript */
/* 挿入位置: html_body_bottom */
/* パスパターン: /issues */
$(function() {
    if (ViewCustomize.context.path === '/issues') {
        $('#content').prepend('<div class="warning">重要: チケット作成時は適切なカテゴリを選択してください</div>');
    }
});
```

### 3. サイドバーにカスタム情報追加（HTML）
```html
<!-- カスタマイズタイプ: HTML -->
<!-- 挿入位置: html_body_bottom -->
<script>
$(function() {
    $('#sidebar').append('<div class="box"><h3>お知らせ</h3><p>システムメンテナンス予定：月末</p></div>');
});
</script>
```

## 注意事項

1. **バックアップ**: カスタマイズ前は必ずデータベースのバックアップを取得
2. **テスト環境**: 本番環境への適用前にテスト環境での動作確認を推奨
3. **JavaScriptエラー**: コードエラーがあると他の機能に影響する可能性があります
4. **パフォーマンス**: 重い処理は避け、必要な場合のみ実行されるよう条件を設定

## コンテキスト情報の活用

`ViewCustomize.context`オブジェクトで現在のページ情報が取得可能：

```javascript
console.log(ViewCustomize.context.path);        // 現在のパス
console.log(ViewCustomize.context.project);     // プロジェクト情報
console.log(ViewCustomize.context.user);        // ユーザー情報
```

## トラブルシューティング

### プラグインが表示されない場合
```bash
# プラグインの確認
docker compose exec redmine bash -c "ls -la /usr/src/redmine/plugins/"

# ログの確認
docker compose logs redmine
```

### カスタマイズが適用されない場合
1. パスパターンが正しいか確認
2. JavaScriptコンソールでエラーが出ていないか確認
3. 「有効」にチェックが入っているか確認

## 更新・メンテナンス

### プラグインの更新
```bash
cd docker/redmine/plugins/view_customize
git pull origin master
docker compose exec redmine bash -c "cd /usr/src/redmine && RAILS_ENV=production bundle exec rake redmine:plugins:migrate"
docker compose restart redmine
```

### アンインストール
```bash
# マイグレーションの巻き戻し
docker compose exec redmine bash -c "cd /usr/src/redmine && RAILS_ENV=production bundle exec rake redmine:plugins:migrate NAME=view_customize VERSION=0"

# プラグインディレクトリの削除
rm -rf docker/redmine/plugins/view_customize

# コンテナ再起動
docker compose restart redmine
```

---

**導入完了日**: 2025年9月26日
**対応バージョン**: Redmine 6.0.6 + View Customize Plugin latest