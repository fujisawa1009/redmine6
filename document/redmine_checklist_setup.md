# Redmine チェックリスト機能設定手順書

## 概要

専用チェックリストプラグインの依存関係の問題により、既に導入済みの**View Customize Plugin**を使用してチェックリスト機能を実装しました。これにより安定した動作とRedmine 6.0.6での完全な互換性を確保できます。

## 実装方針

View Customizeプラグインを使用して以下の機能を実装：
1. チケット画面にチェックリスト入力フィールドを追加
2. チェックボックス付きのアイテム表示
3. ローカルストレージでの状態保存

## 設定手順

### 1. View Customizeの作成

1. **管理者でログイン**して以下にアクセス：
   ```
   http://localhost:3000/view_customizes
   ```

2. **「新しいView Customizeを追加」**をクリック

### 2. チェックリスト機能のView Customize設定

#### 基本設定
- **名前**: チケットチェックリスト機能
- **コメント**: チケット詳細画面にチェックリスト機能を追加
- **有効**: ✓ チェック
- **プライベート**: ✗ チェック解除（全ユーザーで利用）

#### 適用条件
- **パスパターン**: `/issues/\d+`
- **プロジェクトパターン**: （空欄 = 全プロジェクト）

#### カスタマイズ設定
- **カスタマイズタイプ**: JavaScript
- **挿入位置**: html_body_bottom

#### JavaScript コード

```javascript
$(document).ready(function() {
    // チケット詳細画面でのみ動作
    if (!window.location.pathname.match(/\/issues\/\d+$/)) {
        return;
    }

    var issueId = window.location.pathname.match(/\/issues\/(\d+)$/)[1];
    var storageKey = 'redmine_checklist_' + issueId;

    // チェックリストエリアを作成
    var checklistHtml = `
        <div id="checklist-area" class="box" style="margin: 10px 0;">
            <h3>チェックリスト</h3>
            <div id="checklist-items"></div>
            <div style="margin-top: 10px;">
                <input type="text" id="new-checklist-item" placeholder="新しいアイテムを入力..." style="width: 300px; margin-right: 10px;">
                <button id="add-checklist-item" class="button button-2">追加</button>
            </div>
        </div>
    `;

    // チケット詳細の下に追加
    $('.issue .details').after(checklistHtml);

    // ローカルストレージからチェックリストを読み込み
    function loadChecklist() {
        var checklist = JSON.parse(localStorage.getItem(storageKey) || '[]');
        var container = $('#checklist-items');
        container.empty();

        checklist.forEach(function(item, index) {
            var itemHtml = `
                <div class="checklist-item" style="margin: 5px 0; display: flex; align-items: center;">
                    <input type="checkbox" ${item.checked ? 'checked' : ''} data-index="${index}" style="margin-right: 10px;">
                    <span style="flex: 1; ${item.checked ? 'text-decoration: line-through; color: #999;' : ''}">${item.text}</span>
                    <button class="remove-item button button-2" data-index="${index}" style="margin-left: 10px; font-size: 12px;">削除</button>
                </div>
            `;
            container.append(itemHtml);
        });
    }

    // チェックリストを保存
    function saveChecklist() {
        var checklist = [];
        $('#checklist-items .checklist-item').each(function() {
            var $item = $(this);
            var checked = $item.find('input[type="checkbox"]').is(':checked');
            var text = $item.find('span').text();
            checklist.push({text: text, checked: checked});
        });
        localStorage.setItem(storageKey, JSON.stringify(checklist));
    }

    // 新しいアイテムを追加
    function addItem(text) {
        if (text.trim()) {
            var checklist = JSON.parse(localStorage.getItem(storageKey) || '[]');
            checklist.push({text: text.trim(), checked: false});
            localStorage.setItem(storageKey, JSON.stringify(checklist));
            loadChecklist();
            $('#new-checklist-item').val('');
        }
    }

    // イベントリスナー設定
    $('#add-checklist-item').click(function() {
        var text = $('#new-checklist-item').val();
        addItem(text);
    });

    $('#new-checklist-item').keypress(function(e) {
        if (e.which == 13) {
            var text = $(this).val();
            addItem(text);
        }
    });

    // チェックボックスの変更を監視
    $(document).on('change', '#checklist-items input[type="checkbox"]', function() {
        var index = $(this).data('index');
        var checked = $(this).is(':checked');
        var checklist = JSON.parse(localStorage.getItem(storageKey) || '[]');
        checklist[index].checked = checked;
        localStorage.setItem(storageKey, JSON.stringify(checklist));

        // スタイル更新
        var $span = $(this).siblings('span');
        if (checked) {
            $span.css({'text-decoration': 'line-through', 'color': '#999'});
        } else {
            $span.css({'text-decoration': 'none', 'color': ''});
        }
    });

    // アイテム削除
    $(document).on('click', '.remove-item', function() {
        var index = parseInt($(this).data('index'));
        var checklist = JSON.parse(localStorage.getItem(storageKey) || '[]');
        checklist.splice(index, 1);
        localStorage.setItem(storageKey, JSON.stringify(checklist));
        loadChecklist();
    });

    // 初期読み込み
    loadChecklist();
});
```

### 3. 設定完了後の確認

1. **「作成」**ボタンをクリックして保存
2. 任意のチケット詳細画面にアクセス
3. チケット詳細の下に「チェックリスト」エリアが表示されることを確認

## 使用方法

### チェックリストの操作

1. **アイテム追加**
   - テキストフィールドに項目を入力
   - 「追加」ボタンをクリック、またはEnterキー

2. **完了マーク**
   - チェックボックスをクリックして完了/未完了を切り替え
   - 完了したアイテムは取り消し線が表示される

3. **アイテム削除**
   - 各アイテムの「削除」ボタンをクリック

### データの保存について

- チェックリストのデータは**ブラウザのローカルストレージ**に保存
- チケットごとに独立したデータとして管理
- ブラウザを変更すると表示されません（サーバーサイドではなくクライアントサイド保存）

## 追加オプション設定

### 特定プロジェクトのみで有効にする場合

**プロジェクトパターン**に以下を設定：
```
プロジェクトA|プロジェクトB
```

### デザインのカスタマイズ

CSSでスタイルをカスタマイズする場合の追加View Customize：

#### CSS設定
- **カスタマイズタイプ**: CSS
- **挿入位置**: html_head

```css
#checklist-area {
    background: #f8f8f8;
    border: 1px solid #ddd;
    border-radius: 4px;
    padding: 15px;
}

#checklist-area h3 {
    margin-top: 0;
    color: #333;
    border-bottom: 1px solid #ddd;
    padding-bottom: 8px;
}

.checklist-item {
    padding: 8px;
    border-radius: 3px;
}

.checklist-item:hover {
    background-color: #f0f0f0;
}

#new-checklist-item {
    border: 1px solid #ccc;
    padding: 6px;
    border-radius: 3px;
}
```

## 制限事項

1. **データ共有**: ブラウザのローカルストレージを使用するため、他のユーザーや異なるブラウザでは共有されません
2. **データ永続性**: ブラウザデータクリア時に消去される可能性があります
3. **バックアップ**: 自動バックアップ機能はありません

## サーバーサイドでの共有機能が必要な場合

より高度な機能（チーム共有、データベース保存）が必要な場合は：
1. **RedmineUP Checklists Pro版**（有料: $99）の導入を検討
2. **カスタムフィールド**を使用したテキストベースの実装
3. **専用プラグイン開発**の検討

---

**導入完了日**: 2025年9月26日
**対応バージョン**: Redmine 6.0.6 + View Customize Plugin
**実装方法**: JavaScript/ローカルストレージベース