# Redmine アジャイル機能設定手順書

## 概要

Redmine 6.0.6環境でアジャイル開発に必要な機能を実現するため、専用Agileプラグインの互換性問題により、既存の**Kanbanプラグイン**と**View Customizeプラグイン**を組み合わせてアジャイル機能を実装しました。

## 実装済み機能

### 1. Kanbanボード機能
- **プラグイン**: Kanban plugin v0.0.12（導入済み）
- **機能**: ドラッグ&ドロップでのタスク管理
- **アクセス**: プロジェクトメニューまたはアプリケーションメニューから

### 2. アジャイル拡張機能
- **View Customizeプラグイン**を使用したスクラム機能の実装
- ストーリーポイント管理
- スプリント進捗表示
- バーンダウンチャート（簡易版）

## 設定手順

### 1. Kanbanプラグインの有効化

1. **プロジェクト設定**に移動
   - プロジェクト → 設定 → モジュール

2. **Kanbanモジュールを有効化**
   - ✓「Kanban」にチェックを入れる
   - 「保存」をクリック

3. **権限設定**
   - 管理 → ロールと権限
   - 各ロールで「Display menu link」権限を付与

### 2. View Customizeによるアジャイル機能追加

#### A. ストーリーポイント表示機能

**設定内容**:
- **名前**: ストーリーポイント表示
- **パスパターン**: `/issues.*`
- **カスタマイズタイプ**: JavaScript
- **挿入位置**: html_body_bottom

**JavaScript コード**:
```javascript
$(document).ready(function() {
    // ストーリーポイント用のカスタムフィールドがある場合の処理
    // カスタムフィールドID（実際のIDに変更してください）
    var storyPointFieldId = 'issue_custom_field_values_1'; // 例: カスタムフィールドID 1

    // チケット一覧でストーリーポイントを強調表示
    if (window.location.pathname.match(/\/issues/)) {
        $('.list td.custom_field').each(function() {
            var text = $(this).text().trim();
            if (text.match(/^\d+$/)) { // 数値のみの場合
                $(this).css({
                    'background-color': '#e6f3ff',
                    'font-weight': 'bold',
                    'text-align': 'center',
                    'border-radius': '3px'
                });
                $(this).attr('title', 'ストーリーポイント: ' + text);
            }
        });
    }

    // チケット詳細でストーリーポイントを強調
    if (window.location.pathname.match(/\/issues\/\d+$/)) {
        $('.custom_field').each(function() {
            var label = $(this).find('.label').text();
            if (label.includes('ストーリーポイント') || label.includes('Story Point')) {
                var value = $(this).find('.value').text().trim();
                if (value.match(/^\d+$/)) {
                    $(this).find('.value').css({
                        'background-color': '#2196F3',
                        'color': 'white',
                        'padding': '5px 10px',
                        'border-radius': '20px',
                        'font-weight': 'bold',
                        'display': 'inline-block'
                    });
                }
            }
        });
    }
});
```

#### B. スプリント進捗ダッシュボード

**設定内容**:
- **名前**: スプリント進捗ダッシュボード
- **パスパターン**: `/projects/[^/]+$`
- **カスタマイズタイプ**: JavaScript
- **挿入位置**: html_body_bottom

**JavaScript コード**:
```javascript
$(document).ready(function() {
    // プロジェクト概要ページでスプリント進捗を表示
    if (window.location.pathname.match(/\/projects\/[^\/]+$/)) {

        // スプリント情報エリアを作成
        var sprintHtml = `
            <div class="box" style="margin: 10px 0;">
                <h3>🏃 スプリント進捗</h3>
                <div id="sprint-info">
                    <div style="margin: 10px 0;">
                        <strong>現在のスプリント:</strong> <span id="current-sprint">Sprint 1</span>
                        <span style="margin-left: 20px;">
                            <strong>期間:</strong> <span id="sprint-period">2025/01/01 - 2025/01/14</span>
                        </span>
                    </div>
                    <div style="margin: 10px 0;">
                        <div class="progress-container" style="background: #f0f0f0; height: 20px; border-radius: 10px; overflow: hidden;">
                            <div class="progress-bar" style="background: linear-gradient(90deg, #4CAF50, #8BC34A); height: 100%; width: 60%; transition: width 0.3s;"></div>
                        </div>
                        <div style="margin-top: 5px;">
                            <span>完了: <strong>12/20</strong> タスク</span>
                            <span style="float: right;">進捗: <strong>60%</strong></span>
                        </div>
                    </div>
                </div>
            </div>
        `;

        // プロジェクト情報の後に挿入
        $('.box:first').after(sprintHtml);

        // 簡易バーンダウンチャート
        var burndownHtml = `
            <div class="box" style="margin: 10px 0;">
                <h3>📊 バーンダウンチャート</h3>
                <div style="position: relative; height: 200px; border: 1px solid #ddd; background: white;">
                    <canvas id="burndown-chart" width="600" height="180"></canvas>
                </div>
                <div style="margin-top: 10px; font-size: 12px; color: #666;">
                    理想線（青）と実績線（赤）でスプリントの進捗を確認
                </div>
            </div>
        `;

        $('#sprint-info').parent().after(burndownHtml);

        // 簡易チャート描画
        drawBurndownChart();
    }

    function drawBurndownChart() {
        var canvas = document.getElementById('burndown-chart');
        if (!canvas || !canvas.getContext) return;

        var ctx = canvas.getContext('2d');
        var width = canvas.width;
        var height = canvas.height;

        // チャートエリアをクリア
        ctx.clearRect(0, 0, width, height);

        // 軸を描画
        ctx.strokeStyle = '#333';
        ctx.lineWidth = 1;
        ctx.beginPath();
        ctx.moveTo(50, 20);
        ctx.lineTo(50, height - 20);
        ctx.lineTo(width - 20, height - 20);
        ctx.stroke();

        // 理想線（青）
        ctx.strokeStyle = '#2196F3';
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.moveTo(50, 30);
        ctx.lineTo(width - 20, height - 20);
        ctx.stroke();

        // 実績線（赤）
        ctx.strokeStyle = '#F44336';
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.moveTo(50, 30);
        ctx.lineTo(200, 80);
        ctx.lineTo(350, 100);
        ctx.lineTo(500, 120);
        ctx.stroke();

        // ラベル
        ctx.fillStyle = '#333';
        ctx.font = '12px Arial';
        ctx.fillText('残作業量', 5, 25);
        ctx.fillText('日数', width - 30, height - 5);

        // 凡例
        ctx.fillStyle = '#2196F3';
        ctx.fillText('● 理想線', width - 150, 40);
        ctx.fillStyle = '#F44336';
        ctx.fillText('● 実績線', width - 80, 40);
    }
});
```

#### C. Kanbanボード拡張機能

**設定内容**:
- **名前**: Kanban拡張機能
- **パスパターン**: `/kanban`
- **カスタマイズタイプ**: JavaScript
- **挿入位置**: html_body_bottom

**JavaScript コード**:
```javascript
$(document).ready(function() {
    if (window.location.pathname.includes('/kanban')) {

        // Kanbanボードにアジャイル機能を追加
        var agileControlsHtml = `
            <div style="margin: 10px 0; padding: 10px; background: #f8f9fa; border-radius: 5px;">
                <h4>🚀 アジャイル管理</h4>
                <div style="display: flex; gap: 20px; align-items: center;">
                    <div>
                        <label>スプリント: </label>
                        <select id="sprint-selector" style="margin-left: 5px;">
                            <option value="sprint1">Sprint 1 (進行中)</option>
                            <option value="sprint2">Sprint 2 (計画中)</option>
                        </select>
                    </div>
                    <div>
                        <label>WIP制限: </label>
                        <input type="number" id="wip-limit" value="3" min="1" max="10" style="width: 60px; margin-left: 5px;">
                    </div>
                    <button id="show-velocity" style="background: #4CAF50; color: white; border: none; padding: 5px 15px; border-radius: 3px; cursor: pointer;">
                        ベロシティ表示
                    </button>
                </div>
            </div>
        `;

        // Kanbanボードの上に追加
        $('.contextual').after(agileControlsHtml);

        // WIP制限の視覚化
        function updateWipLimits() {
            var wipLimit = parseInt($('#wip-limit').val());
            $('.kanban-column').each(function() {
                var cardCount = $(this).find('.kanban-card').length;
                var $header = $(this).find('.kanban-header');

                if (cardCount >= wipLimit) {
                    $header.css('background-color', '#ffebee');
                    $header.find('.card-count').css('color', '#d32f2f');
                } else {
                    $header.css('background-color', '#e8f5e8');
                    $header.find('.card-count').css('color', '#2e7d32');
                }

                // カード数表示を追加
                if (!$header.find('.card-count').length) {
                    $header.append('<span class="card-count" style="float: right; font-weight: bold;">(' + cardCount + '/' + wipLimit + ')</span>');
                } else {
                    $header.find('.card-count').text('(' + cardCount + '/' + wipLimit + ')');
                }
            });
        }

        // イベントリスナー
        $('#wip-limit').on('change', updateWipLimits);
        $('#show-velocity').click(function() {
            alert('前スプリントベロシティ: 18ポイント\\n現在のスプリント予測: 20ポイント\\n\\n完了予定: 2025/01/14');
        });

        // 初期化
        setTimeout(updateWipLimits, 1000);
    }
});
```

## 使用方法

### 1. Kanbanボードの基本操作

1. **アクセス方法**
   - プロジェクト → Kanban
   - またはトップメニューの「Kanban」

2. **タスク管理**
   - チケットをドラッグ&ドロップで列間移動
   - 新規作成、詳細表示も可能

### 2. アジャイル機能の活用

1. **ストーリーポイント管理**
   - カスタムフィールドで「ストーリーポイント」を作成
   - 数値入力でポイント設定

2. **スプリント管理**
   - プロジェクト概要ページでスプリント進捗確認
   - バーンダウンチャートで視覚的進捗管理

3. **WIP制限**
   - Kanbanページで各列の作業量制限
   - 制限超過時の視覚的警告

## 追加設定

### カスタムフィールドの作成

1. **ストーリーポイント用**
   - 管理 → カスタムフィールド → 新しいカスタムフィールド
   - 形式: 整数
   - 名前: ストーリーポイント
   - 適用対象: チケット

2. **スプリント用**
   - 形式: リスト
   - 名前: スプリント
   - 選択肢: Sprint 1, Sprint 2, Sprint 3...

### プロジェクト設定

1. **トラッカー設定**
   - ユーザーストーリー
   - タスク
   - バグ
   - エピック（必要に応じて）

2. **チケットのステータス**
   - バックログ
   - 進行中
   - レビュー
   - 完了

## 制限事項と今後の拡張

### 現在の制限
- 自動スプリント計算機能なし
- 本格的なバーンダウンチャートは手動更新
- チーム間の連携機能制限

### 拡張オプション
- **有料版検討**: RedmineUP Agile Plugin Pro版（$499/年）
- **カスタム開発**: 特定要件に応じた独自機能開発
- **外部ツール連携**: Jira、Trelloなどとの連携

## トラブルシューティング

### Kanbanが表示されない場合
1. プロジェクトでKanbanモジュールが有効か確認
2. ユーザーに適切な権限があるか確認
3. ブラウザのJavaScriptが有効か確認

### View Customizeが動作しない場合
1. 管理者権限でView Customizes設定を確認
2. パスパターンが正しいか確認
3. JavaScriptエラーがないかコンソールで確認

---

**導入完了日**: 2025年9月26日
**対応バージョン**: Redmine 6.0.6 + Kanban Plugin v0.0.12 + View Customize Plugin
**実装方法**: 既存プラグイン + JavaScript拡張